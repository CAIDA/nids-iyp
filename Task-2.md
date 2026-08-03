[README](README.md) | [Introduction](Introduction.md) | [Datasets](Datasets.md) | [Cypher](Cypher.md) | [Tasks](Task.md) | [Task 1](Task-1.md) | Task 2 ⮕ | [Task 3](Task-3.md) | [Notebook](nids-iyp.ipynb)

# Task 2: Bridging BGP and DNS in One Traversal — How-To Guide

**Builds on:** [nids-asn-introduction](https://github.com/CAIDA/nids-asn-introduction), [nids-dns-ecosystem](https://github.com/caida/nids-dns-ecosystem)

## AS Prefixes and Popular Domains

To find the popular hostnames resolving into an AS's announced address space, chain three relationship types in one pattern: BGP origin (`ORIGINATE`), prefix containment (`PART_OF`), and DNS resolution (`RESOLVES_TO`). Use `OPTIONAL MATCH` for the resolution half so a prefix with no resolving hostnames still appears in the result (see [Cypher §3.4](Cypher.md#34-optional-match-for-possibly-missing-relationships)), and chain a fourth hop to a `Ranking` via `RANK` to keep only hostnames that appear on a popularity list:

```cypher
// step 1: every BGPPrefix this AS originates
MATCH (a:AS {asn: $asn})-[:ORIGINATE]-(pfx:BGPPrefix)

// step 2 (OPTIONAL): popular hostnames resolving into an IP inside that prefix.
// OPTIONAL so prefixes with zero popular hostnames still come back, instead
// of silently disappearing from the result
OPTIONAL MATCH (pfx)-[:PART_OF]-(:IP)-[:RESOLVES_TO]-(h:HostName)
               -[:RANK]-(:Ranking {name: 'Cisco Umbrella Top 1 million'})

RETURN a.asn AS asn, count(DISTINCT h) AS num_popular_hostnames, count(DISTINCT pfx) AS num_prefixes
```

Run this once with `$asn = 2497` (IIJ) and once with `$asn = 6461` (Zayo) and compare. If you also want the per-prefix breakdown (which prefix hosts which hostnames, rather than just a total count), drop the aggregation and return `pfx.prefix, collect(DISTINCT h.name)` instead — that's the shape used in [Cypher §3.3](Cypher.md#33-multi-hop-traversal-chaining-relationship-types).

> Why IIJ (AS2497) here? IIJ is IYP's own tutorial example for exactly this kind of query, and it both operates a network and hosts a great deal of its own infrastructure, so the traversal has something to find. Zayo (AS6461) is the *contrast* case in Q2.a: it is a large fiber and IP-transit backbone — ASRank 8, higher than IIJ's 100 — that carries other networks' traffic almost exclusively and hosts essentially no popular content of its own. Two networks of comparable stature, opposite answers to "what lives in your address space."

### Why the ranking is pinned to one list

`RANK` is a multi-source relationship, and IYP holds hundreds of separate `Ranking` nodes — Cisco Umbrella, per-country Chrome UX Report lists, per-country eyeball estimates, CAIDA ASRank, and more. An unpinned `-[:RANK]-(:Ranking)` therefore means "appears in *any* ranking IYP carries", which is a much weaker claim than "popular" and runs straight into the warning in [Datasets](Datasets.md#writing-efficient-queries) about aggregating across sources that measure different things. It is not a harmless looseness: unpinned, this query reports Zayo as hosting roughly ten times more popular hostnames *per prefix* than IIJ, reversing the answer.

Pinning to `Cisco Umbrella Top 1 million` gives "popular" a single, checkable definition. Note that you cannot swap in `Tranco top 1M` here — Tranco ranks `DomainName` nodes, not `HostName` nodes, so the same pattern silently returns zero. Checking what a source actually attaches to is part of using a multi-source graph.

> **Worth trying:** re-run with a `CrUX top 1M (JP)` or `CrUX top 1M (US)` ranking instead. Popularity is regional, and the two lists do not agree — which is the point of keeping each source as its own relationship rather than merging them into one "popularity" property.

## Authoritative Nameservers

Extend the same traversal one relationship further, from `HostName` back to `DomainName` and on to the `AuthoritativeNameServer` that manages it, via `MANAGED_BY`:

```cypher
// step 1: every BGPPrefix AS2501 originates
MATCH (a:AS {asn: 2501})-[:ORIGINATE]-(pfx:BGPPrefix)

// step 2: nameservers reachable inside that address space
MATCH (pfx)-[:PART_OF]-(:IP)-[:RESOLVES_TO]-(ns:AuthoritativeNameServer)

// step 3 (OPTIONAL): domains that nameserver manages -- OPTIONAL because a
// nameserver could in principle show up with zero attributed domains
OPTIONAL MATCH (dn:DomainName)-[:MANAGED_BY]-(ns)

RETURN ns.name AS nameserver, count(DISTINCT dn) AS nb_domains
ORDER BY nb_domains DESC
```

## What Your Write-Up Should Address

- **Q2.a** — How many distinct popular hostnames resolve into AS2497's vs. AS6461's announced prefixes? Report both the raw counts and the count per announced prefix, since the two ASes announce different amounts of address space. Does the gap match what you'd expect from a network that hosts content versus a pure transit backbone (see [`nids-asn-introduction`](https://github.com/CAIDA/nids-asn-introduction)'s discussion of AS business models)?
- **Q2.b** — Which nameservers manage the most domains inside AS2501's address space? Is domain hosting concentrated in a handful of nameservers, or spread evenly?
- **Q2.c** — This is the same kind of question [`nids-dns-ecosystem`](https://github.com/caida/nids-dns-ecosystem) answers by loading OpenINTEL DNS measurements and a separately-obtained BGP prefix-to-AS mapping, then joining them in Spark across two independently-fetched datasets. What did chaining `ORIGINATE` → `PART_OF` → `RESOLVES_TO` in one Cypher query do in a single step that took multiple stages there? Is there anything the manual approach gives you that the graph traversal doesn't (e.g. control over which snapshot date, or access to raw DNS response fields not modeled in the graph)?

[README](README.md) | [Introduction](Introduction.md) | [Datasets](Datasets.md) | [Cypher](Cypher.md) | [Tasks](Task.md) | [Task 1](Task-1.md) | Task 2 ⮕ | [Task 3](Task-3.md) | [Notebook](nids-iyp.ipynb)
