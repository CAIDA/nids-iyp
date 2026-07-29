[README](README.md) | [Introduction](Introduction.md) | [Datasets](Datasets.md) | [Cypher](Cypher.md) | [Tasks](Task.md) | [Task 1](Task-1.md) | Task 2 ⮕ | [Task 3](Task-3.md) | [Notebook](nids-iyp.ipynb)

# Task 2: Bridging BGP and DNS in One Traversal — How-To Guide

**Builds on:** [nids-asn-introduction](https://github.com/CAIDA/nids-asn-introduction), [nids-dns-ecosystem](https://github.com/caida/nids-dns-ecosystem)

## AS Prefixes and Hosted Domains

To find every hostname resolving into an AS's announced address space, chain three relationship types in one pattern: BGP origin (`ORIGINATE`), prefix containment (`PART_OF`), and DNS resolution (`RESOLVES_TO`). Use `OPTIONAL MATCH` for the resolution half so a prefix with no resolving hostnames still appears in the result (see [Cypher §3.4](Cypher.md#34-optional-match-for-possibly-missing-relationships)):

```cypher
// step 1: every BGPPrefix this AS originates
MATCH (a:AS {asn: $asn})-[:ORIGINATE]-(pfx:BGPPrefix)

// step 2 (OPTIONAL): every hostname resolving into an IP inside that prefix.
// OPTIONAL so prefixes with zero resolving hostnames still come back, instead
// of silently disappearing from the result
OPTIONAL MATCH (pfx)-[:PART_OF]-(:IP)-[:RESOLVES_TO]-(h:HostName)

RETURN a.asn AS asn, count(DISTINCT h) AS num_hostnames, count(DISTINCT pfx) AS num_prefixes
```

Run this once with `$asn = 2497` (IIJ) and once with `$asn = 3356` (Level3/Lumen) and compare. If you also want the per-prefix breakdown (which prefix hosts which hostnames, rather than just a total count), drop the aggregation and return `pfx.prefix, collect(DISTINCT h.name)` instead — that's the shape used in [Cypher §3.3](Cypher.md#33-multi-hop-traversal-chaining-relationship-types).

> Why IIJ (AS2497) and not one of Task 1's three running-example ASes? Level3/Lumen is a pure transit backbone — it carries other networks' traffic, but doesn't host much of its own content, which makes it a weak example for "domains hosted on this AS's address space." IIJ is IYP's own tutorial example for exactly this kind of query, and hosts enough of its own infrastructure to make the traversal interesting. Level3/Lumen is still useful here as the *contrast* case in Q2.a.

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

- **Q2.a** — How many distinct hostnames resolve into AS2497's vs. AS3356's announced prefixes? Does the gap match what you'd expect from a content/hosting network versus a pure transit backbone (see [`nids-asn-introduction`](https://github.com/CAIDA/nids-asn-introduction)'s discussion of AS business models)?
- **Q2.b** — Which nameservers manage the most domains inside AS2501's address space? Is domain hosting concentrated in a handful of nameservers, or spread evenly?
- **Q2.c** — This is the same kind of question [`nids-dns-ecosystem`](https://github.com/caida/nids-dns-ecosystem) answers by loading OpenINTEL DNS measurements and a separately-obtained BGP prefix-to-AS mapping, then joining them in Spark across two independently-fetched datasets. What did chaining `ORIGINATE` → `PART_OF` → `RESOLVES_TO` in one Cypher query do in a single step that took multiple stages there? Is there anything the manual approach gives you that the graph traversal doesn't (e.g. control over which snapshot date, or access to raw DNS response fields not modeled in the graph)?

[README](README.md) | [Introduction](Introduction.md) | [Datasets](Datasets.md) | [Cypher](Cypher.md) | [Tasks](Task.md) | [Task 1](Task-1.md) | Task 2 ⮕ | [Task 3](Task-3.md) | [Notebook](nids-iyp.ipynb)
