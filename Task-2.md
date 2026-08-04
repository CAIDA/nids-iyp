[README](README.md) | [Introduction](Introduction.md) | [Datasets](Datasets.md) | [Cypher](Cypher.md) | [Tasks](Task.md) | [Task 1](Task-1.md) | Task 2 ⮕ | [Task 3](Task-3.md) | [Notebook](nids-iyp.ipynb)

# Task 2: Bridging BGP and DNS in One Traversal — How-To Guide

**Builds on:** [nids-asn-introduction](https://github.com/CAIDA/nids-asn-introduction), [nids-dns-ecosystem](https://github.com/caida/nids-dns-ecosystem)

Running examples: **AS2497** (IIJ) vs. **AS6461** (Zayo) for Q2.a, **AS2501** for Q2.b. As with Task 1, the fragments below are the pieces — you assemble the graded query yourself.

## AS Prefixes and Popular Domains

Finding the popular hostnames resolving into an AS's announced address space means chaining three relationship types: BGP origin (`ORIGINATE`), prefix containment (`PART_OF`), and DNS resolution (`RESOLVES_TO`) — the exact shape [Cypher §3.3](Cypher.md#33-multi-hop-traversal-chaining-relationship-types) walks through:

```cypher
MATCH (a:AS {asn: 2497})-[:ORIGINATE]-(pfx:BGPPrefix)-[:PART_OF]-(:IP)-[:RESOLVES_TO]-(h:HostName)
RETURN pfx.prefix, h.name
LIMIT 10
```

Run that and look at what comes back before going further: is every one of the AS's prefixes represented, or only some? A plain `MATCH` silently drops a prefix the moment any hop in the chain has nothing to match — see [Cypher §3.4](Cypher.md#34-optional-match-for-possibly-missing-relationships) for why the resolution half of this chain (`PART_OF`→`RESOLVES_TO`) needs to be `OPTIONAL MATCH`, not `MATCH`, if you want every prefix to show up even when it hosts nothing.

"Popular" needs its own filter. [Cypher §3.6](Cypher.md#36-filtering-a-node-by-property-mid-chain) covers pinning a node in the middle of a chain by property — apply that here to keep only hostnames that appear on one specific popularity list:

```cypher
MATCH (h:HostName)-[:RANK]-(:Ranking {name: 'Cisco Umbrella Top 1 million'})
RETURN h.name
LIMIT 10
```

### Why the ranking is pinned to one list

`RANK` is a multi-source relationship, and IYP holds hundreds of separate `Ranking` nodes — Cisco Umbrella, per-country Chrome UX Report lists, per-country eyeball estimates, CAIDA ASRank, and more. An unpinned `-[:RANK]-(:Ranking)` therefore means "appears in *any* ranking IYP carries", which is a much weaker claim than "popular" and runs straight into the warning in [Datasets](Datasets.md#writing-efficient-queries) about aggregating across sources that measure different things. It is not a harmless looseness: try the same comparison unpinned and it reports Zayo hosting roughly ten times more popular hostnames *per prefix* than IIJ — the opposite of the pinned answer.

Pinning to `Cisco Umbrella Top 1 million` gives "popular" a single, checkable definition. Note that you cannot swap in `Tranco top 1M` here — Tranco ranks `DomainName` nodes, not `HostName` nodes, so the same pattern silently returns zero. Checking what a source actually attaches to is part of using a multi-source graph.

> **Worth trying:** re-run with a `CrUX top 1M (JP)` or `CrUX top 1M (US)` ranking instead. Popularity is regional, and the two lists do not agree — which is the point of keeping each source as its own relationship rather than merging them into one "popularity" property.

Put together: the AS→prefix→IP→hostname chain from above, with the resolution half `OPTIONAL MATCH`ed, and a fourth hop pinning `HostName` to the Umbrella `Ranking`. Run it once for AS2497, once for AS6461, and aggregate with `count(DISTINCT ...)` ([Cypher §3.9](Cypher.md#39-aggregation-collect-count-distinct)) — both a raw count and a count divided by number of prefixes, since the two ASes announce very different amounts of address space.

> Why IIJ (AS2497) here? IIJ is IYP's own tutorial example for exactly this kind of query, and it both operates a network and hosts a great deal of its own infrastructure, so the traversal has something to find. Zayo (AS6461) is the *contrast* case: it is a large fiber and IP-transit backbone — ASRank 8, better than IIJ's — that carries other networks' traffic almost exclusively and hosts essentially no popular content of its own. Two networks of comparable stature, opposite answers to "what lives in your address space."

## Authoritative Nameservers

Extend the same kind of traversal one relationship further, from `HostName` back to `DomainName` and on to the `AuthoritativeNameServer` that manages it, via `MANAGED_BY`:

```cypher
MATCH (a:AS {asn: 2501})-[:ORIGINATE]-(pfx:BGPPrefix)-[:PART_OF]-(:IP)-[:RESOLVES_TO]-(ns:AuthoritativeNameServer)
RETURN ns.name
LIMIT 10
```

```cypher
MATCH (ns:AuthoritativeNameServer)
OPTIONAL MATCH (dn:DomainName)-[:MANAGED_BY]-(ns)
RETURN ns.name, dn.name
LIMIT 10
```

Chain those two together (the nameservers reachable inside AS2501's address space, then the domains each one manages, `OPTIONAL MATCH`ed since a nameserver could show up with zero attributed domains) and aggregate with `count(DISTINCT ...)` per nameserver.

## What Your Write-Up Should Address

- **Q2.a** — How many distinct popular hostnames resolve into AS2497's vs. AS6461's announced prefixes? Report both the raw counts and the count per announced prefix, since the two ASes announce different amounts of address space. Does the gap match what you'd expect from a network that hosts content versus a pure transit backbone (see [`nids-asn-introduction`](https://github.com/CAIDA/nids-asn-introduction)'s discussion of AS business models)?
- **Q2.b** — Which nameservers manage the most domains inside AS2501's address space? Is domain hosting concentrated in a handful of nameservers, or spread evenly?
- **Q2.c** — This is the same kind of question [`nids-dns-ecosystem`](https://github.com/caida/nids-dns-ecosystem) answers by loading OpenINTEL DNS measurements and a separately-obtained BGP prefix-to-AS mapping, then joining them in Spark across two independently-fetched datasets. What did chaining `ORIGINATE` → `PART_OF` → `RESOLVES_TO` in one Cypher query do in a single step that took multiple stages there? Is there anything the manual approach gives you that the graph traversal doesn't (e.g. control over which snapshot date, or access to raw DNS response fields not modeled in the graph)?

[README](README.md) | [Introduction](Introduction.md) | [Datasets](Datasets.md) | [Cypher](Cypher.md) | [Tasks](Task.md) | [Task 1](Task-1.md) | Task 2 ⮕ | [Task 3](Task-3.md) | [Notebook](nids-iyp.ipynb)
