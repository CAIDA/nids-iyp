[README](README.md) | Introduction ⮕ | [Datasets](Datasets.md) | [Cypher](Cypher.md) | [Tasks](Task.md) | [Task 1](Task-1.md) | [Task 2](Task-2.md) | [Task 3](Task-3.md) | [Notebook](nids-iyp.ipynb)

### Prerequisite NIDS Assignments

- [How the Internet assigns and uses Autonomous Systems (ASes)](https://github.com/CAIDA/nids-asn-introduction)
- [Understanding the BGP Control Plane](https://github.com/CAIDA/nids-bgp-control-plane)
- [Exploring Internet Topology with CAIDA's ITDK](https://github.com/CAIDA/nids-itdk)
- [Regional Internet Registry, IRR, and RPKI](https://github.com/CAIDA/nids-irr-rpki-whois)
- [Understanding the DNS Ecosystem](https://github.com/caida/nids-dns-ecosystem)

## What Is the Internet Yellow Pages?

The [Internet Yellow Pages (IYP)](https://iyp.iijlab.net/) is a graph database, built and maintained by the [Internet Health Report (IHR)](https://ihr.live) project, that aggregates over 80 Internet-measurement datasets — BGP routing tables, RPKI ROAs, IXP membership records, DNS measurements, AS rankings, geolocation, and more — into one queryable graph. It is built on [Neo4j](https://neo4j.com), a graph database management system, and queried with **Cypher**, Neo4j's pattern-matching query language.

### Why a Graph, and Not a Table?

Every other NIDS module you may have worked through hands you one dataset at a time, in whatever shape that dataset's provider chose — a REST API for AS rank, a relational schema for router topology, Parquet files for DNS measurements. Answering a question that touches more than one of these means writing your own join logic: looking up an AS's announced prefixes in one dataset, then manually cross-referencing those prefixes against DNS records from a completely different dataset, matching on the IP address in between.

IYP instead models each of these datasets as nodes and typed relationships in a single graph, so that a prefix, the AS that originates it, and a hostname that resolves into it are all directly connected — walking from one to the other is a single Cypher `MATCH`, not a manual join across two separately-fetched datasets. The tasks in this module deliberately mirror analyses from `nids-asn-introduction`, `nids-dns-ecosystem`, and `nids-irr-rpki-whois` so you can see the same kind of question answered as a graph traversal instead of a bespoke pipeline.

### Nodes, Relationships, and Provenance

Under the hood, every node and relationship in IYP is **typed**: a node carries one or more **labels** (`AS`, `BGPPrefix`, `HostName`, `IXP`, ...) and a relationship carries exactly one **type** (`ORIGINATE`, `RESOLVES_TO`, `MEMBER_OF`, ...). The full label and relationship-type reference is in [Datasets](Datasets.md).

The one modeling choice worth understanding before you write a single query: **IYP does not store one "true" value per fact — it stores every source's claim as its own relationship.** An AS's name, for example, is not a property on the `AS` node. It is a separate `Name` node, reached via a `NAME` relationship, and if PeeringDB, BGP.tools, and RIPE NCC all report a name for the same AS, that AS has _three_ `NAME` relationships, each carrying a `reference_org` property identifying which source it came from. This is because these sources genuinely disagree sometimes, and flattening them into a single property would silently hide that disagreement. Every relationship in IYP carries this same reference/provenance metadata: `reference_org`, `reference_name` (a unique per-dataset identifier, e.g. `caida.asrank`), `reference_time_fetch`, `reference_time_modification`, `reference_url_data`, and `reference_url_info`. You will use `reference_name`/`reference_org` filters directly in Task 1 to pick out one source's claim among several.

### Task 1 Background: The AS Ecosystem

An Autonomous System doesn't exist in isolation — its position in the Internet is defined by who ranks it, who it exchanges traffic with, and where it shows up physically. IYP surfaces all three of these as graph structure rather than separate lookups: a `RANK` relationship to a `Ranking` node (CAIDA's ASRank, Tranco, or Chrome UX Report, depending on `reference_name`), a `MEMBER_OF` relationship to an `IXP` node, and a `PEERS_WITH` relationship to another `AS`. Note that IYP has no dedicated "customer cone" relationship the way `nids-asn-introduction`'s AS Rank REST API does — ranking is instead just another `RANK` edge, differentiated by which `Ranking` node it points to.

### Task 2 Background: Bridging BGP and DNS

`nids-dns-ecosystem` answers "who hosts this domain" by loading OpenINTEL DNS measurements and joining them, in Spark, against a separately-obtained BGP prefix-to-AS mapping. IYP represents both halves of that join natively: an `AS` node connects to the `BGPPrefix` nodes it originates via `ORIGINATE`, a `BGPPrefix` contains the `IP` addresses within it via `PART_OF`, and an `IP` connects to the `HostName` nodes that resolve to it via `RESOLVES_TO`. Chaining these three relationship types in one `MATCH` walks from an AS's announced address space all the way to the domain names hosted inside it — the same question, answered as one graph pattern instead of a multi-stage join across two independently-fetched datasets.

### Task 3 Background: RPKI-Authorized vs. BGP-Observed Origins

`nids-irr-rpki-whois` has you compare IRR route objects, RPKI ROAs, and observed BGP announcements by hand to see where they agree, disagree, or go missing. IYP models the RPKI side of that comparison as a `ROUTE_ORIGIN_AUTHORIZATION` relationship between an `AS` and an `RPKIPrefix` (the ROA), and the BGP side as the same `ORIGINATE` relationship used in Task 2, but pointing at a `BGPPrefix` instead. A prefix that has a ROA but no matching observed `BGPPrefix`, or a `BGPPrefix` tagged `"RPKI Invalid"` via a `CATEGORIZED` relationship to a `Tag` node, are both direct graph patterns — no separate ROA-validator tool required.

### Reading

- [IYP Tutorial: What Is IYP?](https://tutorial.iyp.ihr.live/content/start/what-is-iyp.html)
- [IYP Tutorial: Overview of IYP Data](https://tutorial.iyp.ihr.live/content/start/iyp-data.html)
- [IYP Tutorial: Cypher — Querying IYP](https://tutorial.iyp.ihr.live/content/start/querying-iyp.html)
- Fontugne et al., ["The Wisdom of the Measurement Crowd: Building the Internet Yellow Pages, a Knowledge Graph for the Internet"](https://www.iijlab.net/en/members/romain/pdf/romain_imc2024.pdf), ACM IMC 2024 — the design paper behind IYP.
- [github.com/InternetHealthReport/internet-yellow-pages](https://github.com/InternetHealthReport/internet-yellow-pages) — the full node-type and relationship-type schema reference, source of the summary in [Datasets](Datasets.md).

[README](README.md) | Introduction ⮕ | [Datasets](Datasets.md) | [Cypher](Cypher.md) | [Tasks](Task.md) | [Task 1](Task-1.md) | [Task 2](Task-2.md) | [Task 3](Task-3.md) | [Notebook](nids-iyp.ipynb)
