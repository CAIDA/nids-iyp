[README](README.md) | [Introduction](Introduction.md) | Datasets ⮕ | [Cypher](Cypher.md) | [Tasks](Task.md) | [Task 1](Task-1.md) | [Task 2](Task-2.md) | [Task 3](Task-3.md) | [Notebook](nids-iyp.ipynb)

## Database

This module queries the public **Neo4j** instance of IYP operated by the IHR project (see [Access Model](#access-model) below for the endpoint and what that implies). Every node has one or more **labels** naming its type; every relationship has exactly one **type** and carries reference/provenance properties describing which dataset it came from (see [Introduction](Introduction.md#nodes-relationships-and-provenance)). The full schema (80+ datasets) is much larger than what's below — this page lists only the labels and relationship types the three tasks actually use. The complete reference lives at [github.com/InternetHealthReport/internet-yellow-pages/documentation](https://github.com/InternetHealthReport/internet-yellow-pages/tree/main/documentation); the schema snapshot this course pins to is regenerated into [`iyp_csv/`](iyp_csv/iyp_data_dictionary.md).

#### Node Labels

| Label | Represents | Key properties |
| --- | --- | --- |
| `AS` | An Autonomous System | `asn` |
| `Prefix` | An IP prefix — a supertype label carried by *every* prefix node, alongside the more specific label below | `prefix`, `af` (4 or 6) |
| `BGPPrefix` | An IP prefix observed announced in BGP | `prefix`, `af` |
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
| `Organization` | An organization that operates one or more ASes | `name` |
| `BGPCollector` | A route collector (e.g. a RIPE RIS or RouteViews session) | `project`, `name` |

#### Relationship Types

| Type | Connects | Key properties | Notes |
| --- | --- | --- | --- |
| `ORIGINATE` | `AS` → `BGPPrefix` | `reference_name` (e.g. `bgpkit.pfx2asn`) | Observed BGP origin |
| `ROUTE_ORIGIN_AUTHORIZATION` | `AS` → `RPKIPrefix` | `reference_name` | An RPKI ROA |
| `PEERS_WITH` | `AS` — `AS`, or `AS` — `BGPCollector` | `rel` (`AS`—`AS`); `num_v4_pfxs`, `num_v6_pfxs` (`AS`—`BGPCollector`) | A BGP peering session — any direct adjacency, not only settlement-free peering — or a route-collector monitoring session. `rel` records the business relationship: `0` peer-to-peer, `-1` (CAIDA) or `1` (BGPKIT) provider-to-customer, stored as `(provider)->(customer)` |
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

The full graph is tens of millions of nodes across 80+ datasets, so query shape matters just as much as it would against a large SQL table:

- **Start from an indexed property lookup, not a label scan.** For example, `MATCH (a:AS {asn: 2497})` or `MATCH (a:AS) WHERE a.asn = 2497` uses Neo4j's index on `AS.asn`.
- **Filter by `reference_name`/`reference_org` before aggregating**, whenever more than one source can populate the same relationship type (e.g. `NAME`, `RANK`, `COUNTRY`). Aggregating across sources that disagree silently mixes incompatible claims into one number.
- **Avoid unbounded variable-length paths** (`-[*]-` with no upper bound) — on a graph this size they can match combinatorially many paths. If you need multi-hop traversal, chain explicit relationship types (as the tasks do) rather than a wildcard.
- **Use `OPTIONAL MATCH`** when a relationship might not exist for every row in your result — a plain `MATCH` silently drops rows with no match at all, which can look like "zero results" when the real answer is "some rows have no name from this source."
- **Prototype with `LIMIT`**, confirm the pattern returns what you expect, then remove it before computing a real count or aggregate.
- **Return only the properties you need** (`RETURN a.asn`, not `RETURN a`) rather than whole nodes, especially once you start `collect()`-ing many rows together.

### Access Model

This module queries the **public IYP instance** the IHR project operates at
`neo4j://iyp-bolt.ihr.live:7687`. It is read-only and takes **no credentials**, so there is nothing
to request and nothing to configure — the notebook connects out of the box.

Because it is a shared public service, be a considerate guest: prototype with `LIMIT`, don't loop a
heavy traversal over hundreds of ASes, and expect a multi-hop query over a large AS to take a few
seconds. Never send `CREATE`, `MERGE`, `SET`, `DELETE`, `DETACH DELETE`, `DROP`, `REMOVE`, or
`LOAD CSV`. The notebook's `run_query()` helper refuses any query containing those keywords, which
catches accidental mistakes — a courtesy check, not a security boundary.

One consequence of using a live public instance rather than a frozen dump: **it is reloaded as its
sources update**, so the counts you report will drift over time and will not exactly match a
classmate's run on a different day. Record the date you ran your queries alongside your answers.
That is the provenance discipline this module is about, applied to your own write-up.

### Connecting

```python
from neo4j import GraphDatabase

IYP_URI = "neo4j://iyp-bolt.ihr.live:7687"
db = GraphDatabase.driver(IYP_URI, auth=None)
db.verify_connectivity()
```

No credentials, no config file, nothing to copy — this module only ever targets the public instance.

[README](README.md) | [Introduction](Introduction.md) | Datasets ⮕ | [Cypher](Cypher.md) | [Tasks](Task.md) | [Task 1](Task-1.md) | [Task 2](Task-2.md) | [Task 3](Task-3.md) | [Notebook](nids-iyp.ipynb)
