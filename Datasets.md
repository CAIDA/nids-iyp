[README](README.md) | [Introduction](Introduction.md) | Datasets ⮕ | [Cypher](Cypher.md) | [Tasks](Task.md) | [Task 1](Task-1.md) | [Task 2](Task-2.md) | [Task 3](Task-3.md) | [Notebook](nids-iyp.ipynb)

## Database

This module uses a self-hosted **Neo4j** instance loaded from a pinned IYP dump, running on CAIDA's National Research Platform (NRP) namespace. Every node has one or more **labels** naming its type; every relationship has exactly one **type** and carries reference/provenance properties describing which dataset it came from (see [Introduction](Introduction.md#nodes-relationships-and-provenance)). The full schema (60+ datasets) is much larger than what's below — this page lists only the labels and relationship types the three tasks actually use. The complete reference lives at [github.com/InternetHealthReport/internet-yellow-pages/documentation](https://github.com/InternetHealthReport/internet-yellow-pages/tree/main/documentation); the schema snapshot this course pins to is regenerated into [`iyp_csv/`](iyp_csv/iyp_data_dictionary.md).

#### Node Labels

| Label | Represents | Key properties |
| --- | --- | --- |
| `AS` | An Autonomous System | `asn` |
| `BGPPrefix` | An IP prefix observed announced in BGP | `prefix`, `af` (4 or 6) |
| `RPKIPrefix` | An IP prefix covered by an RPKI ROA | `prefix`, `af` |
| `IP` | A single IP address | `ip` |
| `HostName` | A fully-qualified domain name | `name` |
| `DomainName` | A non-FQDN registrable domain | `name` |
| `AuthoritativeNameServer` | A DNS authoritative nameserver | `name` |
| `IXP` | An Internet exchange point | `name` |
| `Name` | A name string an entity is known by | `name` |
| `Ranking` | A specific ranking system (e.g. CAIDA ASRank, Tranco) | `name` |
| `Tag` | A classification label applied to a resource | `label` |
| `Country` | A country | `country_code`, `alpha3` |
| `Organization` | An organization that operates one or more ASes | — (reached via `Name`) |
| `BGPCollector` | A route collector (e.g. a RIPE RIS or RouteViews session) | `project`, `name` |

#### Relationship Types

| Type | Connects | Key properties | Notes |
| --- | --- | --- | --- |
| `ORIGINATE` | `BGPPrefix` — `AS` | `reference_name` (e.g. `bgpkit.pfx2asn`) | Observed BGP origin |
| `ROUTE_ORIGIN_AUTHORIZATION` | `RPKIPrefix` — `AS` | `reference_name` | An RPKI ROA |
| `PEERS_WITH` | `AS` — `AS`, or `AS` — `BGPCollector` | `num_v4_pfxs`, `num_v6_pfxs` | BGP peering, or a monitoring session |
| `MEMBER_OF` | `AS` → `IXP` | `reference_org` | IXP membership |
| `PART_OF` | `IP` → `Prefix`; `HostName` → `DomainName` | — | Containment |
| `RESOLVES_TO` | `HostName` → `IP` | `reference_name` | DNS A/AAAA resolution |
| `MANAGED_BY` | `DomainName` → `AuthoritativeNameServer`; `AS` → `Organization` | — | Zone management / org ownership |
| `NAME` | any node → `Name` | `reference_org` | Canonical naming; **multi-source** |
| `RANK` | any node → `Ranking` | `rank` | Ranking assignment |
| `CATEGORIZED` | any node → `Tag` | `reference_name` | Classification / tagging |
| `COUNTRY` | any node → `Country` | `reference_name` | Geolocation *or* registration country — meaning depends on the source |

Every relationship above also carries `reference_org`, `reference_time_fetch`, `reference_time_modification`, `reference_url_data`, and `reference_url_info`, even where not called out — see [Introduction](Introduction.md#nodes-relationships-and-provenance).

### Writing Efficient Queries

The full graph is tens of millions of nodes across 60+ datasets, so query shape matters just as much as it would against a large SQL table:

- **Start from an indexed property lookup, not a label scan.** `MATCH (a:AS {asn: 2497})` uses Neo4j's index on `AS.asn`; `MATCH (a:AS) WHERE a.asn = 2497` is logically identical but push the filter into the pattern whenever you can — it reads the same to Neo4j's planner, but the `{prop: value}` form is easiest to get right by habit.
- **Filter by `reference_name`/`reference_org` before aggregating**, whenever more than one source can populate the same relationship type (e.g. `NAME`, `RANK`, `COUNTRY`). Aggregating across sources that disagree silently mixes incompatible claims into one number.
- **Avoid unbounded variable-length paths** (`-[*]-` with no upper bound) — on a graph this size they can match combinatorially many paths. If you need multi-hop traversal, chain explicit relationship types (as the tasks do) rather than a wildcard.
- **Use `OPTIONAL MATCH`** when a relationship might not exist for every row in your result — a plain `MATCH` silently drops rows with no match at all, which can look like "zero results" when the real answer is "some rows have no name from this source."
- **Prototype with `LIMIT`**, confirm the pattern returns what you expect, then remove it before computing a real count or aggregate.
- **Return only the properties you need** (`RETURN a.asn`, not `RETURN a`) rather than whole nodes, especially once you start `collect()`-ing many rows together.

### Access Model

Ask your instructor for the read-only username and password. Copy `neo4j_credentials.env.example` to `neo4j_credentials.env`, fill in `IYP_READ_URI`/`IYP_READ_USER`/`IYP_READ_PASSWORD`, and place it in the same directory as the notebook — it is git-ignored, so never commit real credentials.

**Note:** Neo4j Community Edition (what this instance runs) has no role-based access control — there is no database-enforced way to give you a credential that can only read. Treat `neo4j_credentials.env` as a shared-trust credential, not a sandboxed one: never run `CREATE`, `MERGE`, `SET`, `DELETE`, `DETACH DELETE`, `DROP`, `REMOVE`, or `LOAD CSV` against this instance. The notebook's `run_query()` helper refuses to send a query containing any of those keywords, which will catch accidental mistakes — but it is a courtesy check, not a security boundary, so don't rely on it to protect against anything adversarial.

### Connecting

```python
import os
from pathlib import Path
from neo4j import GraphDatabase
from dotenv import dotenv_values

CREDS_FILE = Path(os.environ.get("IYP_CREDS_FILE", "./neo4j_credentials.env"))
_creds = dotenv_values(CREDS_FILE)
IYP_URI = os.environ.get("IYP_URI") or _creds.get("IYP_READ_URI") or "bolt://localhost:7687"
IYP_USER = os.environ.get("IYP_USER") or _creds.get("IYP_READ_USER") or "neo4j"
IYP_PASSWORD = os.environ.get("IYP_PASSWORD") or _creds.get("IYP_READ_PASSWORD") or "CHANGE_ME"

db = GraphDatabase.driver(IYP_URI, auth=(IYP_USER, IYP_PASSWORD))
db.verify_connectivity()
```

If your JupyterHub session runs inside the same NRP namespace as the IYP instance, `IYP_URI` will be an in-cluster service address your instructor gives you; otherwise, open a `kubectl port-forward` tunnel yourself (your instructor will tell you which command to run) and leave `IYP_URI` at `bolt://localhost:7687`.

[README](README.md) | [Introduction](Introduction.md) | Datasets ⮕ | [Cypher](Cypher.md) | [Tasks](Task.md) | [Task 1](Task-1.md) | [Task 2](Task-2.md) | [Task 3](Task-3.md) | [Notebook](nids-iyp.ipynb)
