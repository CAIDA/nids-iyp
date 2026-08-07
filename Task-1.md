[README](README.md) | [Introduction](Introduction.md) | [Datasets](Datasets.md) | [Cypher](Cypher.md) | [Tasks](Task.md) | Task 1 ⮕ | [Task 2](Task-2.md) | [Task 3](Task-3.md) | [Notebook](nids-iyp.ipynb)

# Task 1: The AS Ecosystem — How-To Guide

**Builds on:** [nids-asn-introduction](https://github.com/CAIDA/nids-asn-introduction)

Running examples throughout this task: **AS6461** (Zayo, a transit backbone), **AS2906** (Netflix, a content provider), **AS4837** (China Unicom, a national gateway). None of the fragments below are assembled into a final query — you build each graded query yourself from the pieces here plus [Cypher](Cypher.md).

## Canonical Names and CAIDA ASRank

An AS's name isn't a single property in IYP — it's a separate `Name` node, reached through a `NAME` relationship, and different sources can (and do) report different names for the same AS (see [Introduction](Introduction.md#nodes-relationships-and-provenance)).

Start from all three running examples at once — [Cypher §3.1](Cypher.md#31-filter-by-a-property) covers filtering by one value, and `IN` extends the same indexed lookup to a list of them:

```cypher
MATCH (a:AS)
WHERE a.asn IN [6461, 2906, 4837]
RETURN a.asn
```

CAIDA's ASRank is one `RANK` relationship, pinned to its own `reference_name`, `caida.asrank` (see [Cypher §3.5](Cypher.md#35-filtering-by-reference_org--reference_name) for the general shape of a pinned relationship filter — here it's `reference_name` rather than `reference_org`). Use `OPTIONAL MATCH` ([Cypher §3.4](Cypher.md#34-optional-match-for-possibly-missing-relationships)) since not every AS is ranked:

```cypher
MATCH (a:AS {asn: 6461})
OPTIONAL MATCH (a)-[r:RANK {reference_name: 'caida.asrank'}]->(:Ranking)
RETURN a.asn, r.rank
```

Names work the same way, but with three sources worth checking, and they don't all key off the same property — PeeringDB and bgp.tools set `reference_org`, but the RIPE NCC source doesn't; it's identified by `reference_name: 'ripe.as_names'` instead. (If you're ever unsure which property a source uses, run an unfiltered `MATCH (a:AS {asn: ...})-[r:NAME]-(n:Name) RETURN r.reference_org, r.reference_name, n.name` for one AS and look at what comes back.) One source, `OPTIONAL MATCH`ed:

```cypher
MATCH (a:AS {asn: 6461})
OPTIONAL MATCH (a)-[:NAME {reference_org: 'PeeringDB'}]->(n:Name)
RETURN a.asn, n.name AS peeringdb_name
```

You'll want one `OPTIONAL MATCH` like this per source (PeeringDB, bgp.tools, RIPE), each returning its own named column — not merged yet, so a disagreement between sources stays visible instead of being hidden. [Cypher §3.11](Cypher.md#311-coalesce-across-multiple-sources) shows `coalesce()` combining two sources into one canonical value the same way you'll combine these three.

Put together: the multi-AS filter, the ASRank `OPTIONAL MATCH`, one `OPTIONAL MATCH` per `NAME` source, and a `coalesce()` over all three name columns, in a single query returning one row per AS.

## IXP Membership

This is two separate questions, at two different scopes: a graph-wide question (which IXPs are largest, full stop) and a running-examples question (how do these three ASes compare). `MEMBER_OF` is the relationship either way ([Cypher §3.2](Cypher.md#32-traverse-a-relationship-one-hop)):

```cypher
MATCH (a:AS {asn: 6461})-[:MEMBER_OF]->(ix:IXP)
RETURN ix.name
LIMIT 10
```

The global top-10 question has no `WHERE` at all — it isn't about the running examples, it's about every `AS` in the graph, aggregated by IXP (see [Cypher §3.9](Cypher.md#39-aggregation-collect-count-distinct) for `count(DISTINCT ...)` + `ORDER BY` + `LIMIT`). The per-AS question is the multi-AS filter from the names section above, aggregated the same way but grouped by AS instead of by IXP. Same building blocks, two different things being counted.

## Peering Degree

**"Peering" here means a BGP peering session, not a settlement-free peering agreement.** The term is badly overloaded. In a business context, "peering" (p2p) is contrasted with "transit" (p2c), where one AS pays another to carry its traffic. `PEERS_WITH` does not make that distinction — it records that two ASes exchange routes directly, whatever the commercial arrangement behind it. So an AS's *peering degree* in this task is simply how many other ASes it has a direct BGP adjacency with, transit providers and customers included.

`PEERS_WITH` connects two `AS` nodes directly — one hop, aggregated the same way as IXP membership above:

```cypher
MATCH (a:AS {asn: 6461})-[:PEERS_WITH]-(b:AS)
RETURN b.asn
LIMIT 10
```

> **Gotcha:** `PEERS_WITH` can also connect an `AS` to a `BGPCollector` — a route-collector monitoring session, not a real peering relationship. Label both ends explicitly (`(a:AS)-[:PEERS_WITH]-(b:AS)`, as above) so a collector session can't silently fold into your count. If you write a variant without an explicit label on the far end, check what's actually there before you count it.

> **Worth trying:** the p2p/p2c distinction *is* in the graph, just not as a separate relationship type — an `AS`—`AS` `PEERS_WITH` carries a `rel` property recording the business relationship. The two sources that populate it disagree on the encoding: CAIDA (`caida.as_relationships_v4`/`_v6`) uses `0` for peer-to-peer and `-1` for provider-to-customer, while BGPKIT (`bgpkit.as2rel_v4`/`_v6`) uses `0` and `1`. Reading `rel` without first pinning `reference_org` or `reference_name` therefore mixes two incompatible conventions — the same provenance trap [Datasets](Datasets.md#writing-efficient-queries) warns about. `PEERS_WITH` is also one of the few relationship types where direction is meaningful: a provider-to-customer edge is stored as `(provider:AS)-[:PEERS_WITH]->(customer:AS)`, which the undirected `-[:PEERS_WITH]-` above deliberately ignores. None of this is needed for Q1.c — that count is over all sessions — but returning `r.rel` alongside your peers shows how much of a network's degree is transit rather than settlement-free.

## What Your Write-Up Should Address

Ground each answer in what your own queries actually return, not in the example structure above:

- **Q1.a** — Do PeeringDB, BGP.Tools, and RIPE NCC agree on each AS's name? What does agreement or disagreement tell you about why IYP models `NAME` as multiple relationships instead of a single property?
- **Q1.b** — How does each of the three ASes' IXP membership count compare to the global top 10? Is IXP membership something all large networks do, or does it vary by the kind of network (transit ISP vs. content provider vs. national gateway) discussed in [`nids-asn-introduction`](https://github.com/CAIDA/nids-asn-introduction)?
- **Q1.c** — Do ASRank, peering degree, and IXP membership move together for these three ASes, or does one stand out from the others? What does that tell you about what CAIDA ASRank is actually measuring?

[README](README.md) | [Introduction](Introduction.md) | [Datasets](Datasets.md) | [Cypher](Cypher.md) | [Tasks](Task.md) | Task 1 ⮕ | [Task 2](Task-2.md) | [Task 3](Task-3.md) | [Notebook](nids-iyp.ipynb)
