README ⮕ | [Introduction](Introduction.md) | [Datasets](Datasets.md) | [Cypher](Cypher.md) | [Tasks](Task.md) | [Task 1](Task-1.md) | [Task 2](Task-2.md) | [Task 3](Task-3.md) | [Notebook](nids-iyp.ipynb)

# Exploring the Internet Yellow Pages (IYP): A Graph Model of Internet Infrastructure

**GitHub:** https://github.com/CAIDA/nids-iyp

## Authors

Bradley Huffaker, Romain Fontugne, Malte Tashiro

## Learning Objectives

Other NIDS modules hand you one dataset at a time: an AS-ranking API, a table of BGP routes, a pile of DNS measurements, a set of RPKI ROAs. Answering a question that spans more than one of those means writing your own join logic to stitch them together. The [Internet Yellow Pages (IYP)](https://iyp.iijlab.net/) takes the opposite approach: it loads over 80 of these same kinds of Internet-measurement datasets into a single Neo4j graph database, where an Autonomous System, a BGP prefix, a hostname, and an IXP are all typed nodes connected by typed relationships. In this module you will learn Cypher, Neo4j's graph query language, and use it to explore an AS's ecosystem (ranking, IXP membership, peering), trace a single multi-hop path from an AS's announced address space to the domain names hosted inside it, and check whether an AS's RPKI authorizations match what is actually observed in BGP. By the end you should be able to write a multi-hop Cypher query that does in one step what would otherwise require joining several separate datasets by hand — and reason about what it means when independent data sources disagree about the same fact.

## Slides

- [ETP week 08 IYP](slides/ETP-Week-08-iyp.pptx)

## Overview

- step 1 [read the introduction](Introduction.md)
- step 2 [read the dataset overview](Datasets.md)
- step 3 [read the Cypher guide](Cypher.md)
- step 4 [review the tasks](Task.md)
- step 5 set up your local Python environment (see "Running Locally" below)
- step 6 launch Jupyter and complete nids-iyp.ipynb
  - complete each task by replacing the `# YOUR CODE HERE` sections
  - answer all nine questions
- step 7 save the notebook, commit, and push to github

### Running Locally

This notebook runs on your own machine — no hosted JupyterHub required. Pick one:

```bash
# Option A: uv
uv sync
uv run jupyter lab nids-iyp.ipynb
```

```bash
# Option B: pip + venv
python3 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
jupyter lab nids-iyp.ipynb
```

No database setup is required. The notebook queries the public IYP instance at `neo4j://iyp-bolt.ihr.live:7687`, which is read-only and needs no credentials — see [Datasets](Datasets.md#access-model). Because it is a live shared service rather than a frozen snapshot, record the date you ran your queries: the data is reloaded as its sources update.

### Directory Structure

```
nids-iyp
├- Introduction.md                          # Reading list and IYP data-model concepts
├- Datasets.md                              # Node/relationship schema and access model
├- Cypher.md                                # Cypher query-language guide
├- Task.md                                  # Task checklist and instructions
├- Task-1.md                                # Task 1 how-to guide
├- Task-2.md                                # Task 2 how-to guide
├- Task-3.md                                # Task 3 how-to guide
├- nids-iyp.ipynb                       ⬅  # Complete / Commit / Push
├- iyp_csv/                                 # Schema reference: labels, relationship types, properties
├- requirements.txt                         # Dependencies (pip + venv)
├- pyproject.toml / uv.lock                 # Dependencies (uv)
```

### Glossary

- **IYP (Internet Yellow Pages)**: A Neo4j graph database, maintained by the Internet Health Report (IHR) project, that unifies over 80 Internet-measurement datasets into one queryable graph of typed nodes and relationships.
- **Neo4j**: The graph database management system IYP is built on.
- **Cypher**: Neo4j's query language, used to describe and match patterns of nodes and relationships in the graph.
- **Node**: A single entity in the graph — an `AS`, a `BGPPrefix`, a `HostName`, an `IXP`, etc. Every node has one or more **labels** naming its type.
- **Relationship**: A typed, directed edge connecting two nodes, e.g. `ORIGINATE`, `RESOLVES_TO`, `MEMBER_OF`. Relationships carry their own properties, separately from the nodes they connect.
- **Reference / provenance properties**: Every relationship in IYP records where the fact came from — `reference_org`, `reference_name`, `reference_time_fetch`, `reference_url_data`, etc. Because different datasets can disagree about the same fact, IYP stores each source's claim as its own relationship rather than overwriting a single property.
- **AS (Autonomous System)**: A network under one administrative control, identified by an ASN; see `nids-asn-introduction` for a deeper introduction.
- **BGP / Prefix / Origin**: The routing protocol ASes use to announce which IP prefixes they carry traffic for; the AS that announces a prefix is its _origin_. See `nids-bgp-control-plane`.
- **RPKI / ROA**: Resource Public Key Infrastructure and its core object, the Route Origin Authorization — a cryptographically signed statement of which AS is authorized to originate a given prefix. See `nids-irr-rpki-whois`.
- **IXP (Internet Exchange Point)**: A physical facility where multiple ASes interconnect and exchange traffic directly, rather than through transit.
- **Peering / peering degree**: In this module, _peering_ means a BGP peering session — a direct routing adjacency between two ASes, recorded as a `PEERS_WITH` relationship — regardless of the commercial arrangement behind it. An AS's _peering degree_ is the number of distinct ASes it has such a session with. The word is overloaded: in a business context, "peering" (p2p, settlement-free) is contrasted with "transit" (p2c, one AS paying another to carry its traffic). IYP records that distinction separately, in `PEERS_WITH`'s `rel` property.
- **HostName / DomainName**: A fully-qualified domain name (`www.example.com`) is a `HostName`; a non-FQDN registrable name (`example.com`) is a `DomainName`. See `nids-dns-ecosystem`.
- **Authoritative Name Server**: The DNS server responsible for answering queries for a given domain's zone.
- **Ranking**: A node representing a specific popularity or importance ranking (e.g. CAIDA ASRank, Tranco, Chrome UX Report), linked to the ASes or domains it ranks via `RANK` relationships.
- **Tag**: A node representing a classification applied to a resource (e.g. `"RPKI Invalid"`, `"Academic"`), linked via `CATEGORIZED` relationships.

README ⮕ | [Introduction](Introduction.md) | [Datasets](Datasets.md) | [Cypher](Cypher.md) | [Tasks](Task.md) | [Task 1](Task-1.md) | [Task 2](Task-2.md) | [Task 3](Task-3.md) | [Notebook](nids-iyp.ipynb)
