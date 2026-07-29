[README](README.md) | [Introduction](Introduction.md) | [Datasets](Datasets.md) | [Cypher](Cypher.md) | [Tasks](Task.md) | Task 1 ⮕ | [Task 2](Task-2.md) | [Task 3](Task-3.md) | [Notebook](nids-iyp.ipynb)

# Task 1: The AS Ecosystem — How-To Guide

**Builds on:** [nids-asn-introduction](https://github.com/CAIDA/nids-asn-introduction)

## Canonical Names and CAIDA ASRank

An AS's name isn't a single property in IYP — it's a separate `Name` node, reached through a `NAME` relationship, and different sources can (and do) report different names for the same AS (see [Introduction](Introduction.md#nodes-relationships-and-provenance)). To get one merged name back, `OPTIONAL MATCH` each source individually and `coalesce()` them:

```cypher
// step 1: the three running-example ASes
MATCH (a:AS)
WHERE a.asn IN [3356, 2906, 4837]

// step 2: CAIDA ASRank -- OPTIONAL because not every AS is ranked
OPTIONAL MATCH (a)-[r:RANK {reference_name: 'caida.asrank'}]->(:Ranking)

// step 3: one NAME source per OPTIONAL MATCH, so a missing source doesn't
// drop the whole row -- each is filtered to a single reference_org/reference_name
// so we don't accidentally average together sources that disagree
OPTIONAL MATCH (a)-[:NAME {reference_org: 'PeeringDB'}]->(n1:Name)
OPTIONAL MATCH (a)-[:NAME {reference_org: 'BGP.Tools'}]->(n2:Name)
OPTIONAL MATCH (a)-[:NAME {reference_name: 'ripe.as_names'}]->(n3:Name)

// step 4: coalesce picks the first non-null -- the "canonical" name -- but we
// also return each source individually so a disagreement is visible, not hidden
RETURN a.asn AS asn,
       r.rank AS as_rank,
       coalesce(n1.name, n2.name, n3.name) AS canonical_name,
       n1.name AS peeringdb_name,
       n2.name AS bgptools_name,
       n3.name AS ripe_name
ORDER BY as_rank
```

Run it with `run_query()` (see [Cypher](Cypher.md#1-running-cypher-from-python)) and the rows come back as a DataFrame with one row per AS.

## IXP Membership

Two separate queries: a global top-10 (no filter — this is a graph-wide question, not one about the three running-example ASes), and a per-AS membership count for the three ASes.

```cypher
// Global: which 10 IXPs have the most AS members?
MATCH (m:AS)-[:MEMBER_OF]->(ix:IXP)
RETURN ix.name AS ixp_name, count(DISTINCT m) AS num_members
ORDER BY num_members DESC
LIMIT 10
```

```cypher
// Per-AS: how many IXPs does each running-example AS belong to?
MATCH (a:AS)-[:MEMBER_OF]->(ix:IXP)
WHERE a.asn IN [3356, 2906, 4837]
RETURN a.asn AS asn, count(DISTINCT ix) AS num_ixps
ORDER BY num_ixps DESC
```

## Peering Degree

```cypher
// How many distinct ASes does each running-example AS peer with directly?
MATCH (a:AS)-[:PEERS_WITH]-(b:AS)
WHERE a.asn IN [3356, 2906, 4837]
RETURN a.asn AS asn, count(DISTINCT b) AS num_peer_ases
ORDER BY num_peer_ases DESC
```

> **Gotcha:** `PEERS_WITH` can also connect an `AS` to a `BGPCollector` — a route-collector monitoring session, not a real peering relationship. The query above already excludes those by requiring the other end to match `(b:AS)`, but if you write a variant without an explicit label on both ends, double-check what's actually on the far side of the relationship before you count it.

## What Your Write-Up Should Address

Ground each answer in what your own queries actually return, not in the example structure above:

- **Q1.a** — Do PeeringDB, BGP.Tools, and RIPE NCC agree on each AS's name? What does agreement or disagreement tell you about why IYP models `NAME` as multiple relationships instead of a single property?
- **Q1.b** — How does each of the three ASes' IXP membership count compare to the global top 10? Is IXP membership something all large networks do, or does it vary by the kind of network (transit ISP vs. content provider vs. national gateway) discussed in [`nids-asn-introduction`](https://github.com/CAIDA/nids-asn-introduction)?
- **Q1.c** — Do ASRank, peering degree, and IXP membership move together for these three ASes, or does one stand out from the others? What does that tell you about what CAIDA ASRank is actually measuring?

[README](README.md) | [Introduction](Introduction.md) | [Datasets](Datasets.md) | [Cypher](Cypher.md) | [Tasks](Task.md) | Task 1 ⮕ | [Task 2](Task-2.md) | [Task 3](Task-3.md) | [Notebook](nids-iyp.ipynb)
