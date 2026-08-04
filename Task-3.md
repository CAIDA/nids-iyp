[README](README.md) | [Introduction](Introduction.md) | [Datasets](Datasets.md) | [Cypher](Cypher.md) | [Tasks](Task.md) | [Task 1](Task-1.md) | [Task 2](Task-2.md) | Task 3 ⮕ | [Notebook](nids-iyp.ipynb)

# Task 3: RPKI-Authorized vs. BGP-Observed Origins — How-To Guide

**Builds on:** [nids-irr-rpki-whois](https://github.com/CAIDA/nids-irr-rpki-whois)

Running example: **AS6461** (Zayo) for Q3.a, a graph-wide scan cross-referenced against **AS6461, AS2906, AS4837, AS2497** for Q3.b. As with the other tasks, build the graded query yourself from these pieces.

## ROA Coverage for a Single AS

An RPKI ROA is a `ROUTE_ORIGIN_AUTHORIZATION` relationship between an `AS` and an `RPKIPrefix`; an observed BGP announcement is the same `ORIGINATE` relationship from Task 2, but pointing at a `BGPPrefix` instead:

```cypher
MATCH (a:AS {asn: 6461})-[:ROUTE_ORIGIN_AUTHORIZATION]-(rpfx:RPKIPrefix)
RETURN rpfx.prefix
LIMIT 10
```

A ROA with *no* exact matching observed announcement is a prefix that exists in the pattern above but not in an equivalent `BGPPrefix` pattern — "exists in one but not the other" is exactly what [Cypher §3.7](Cypher.md#37-where-not--excluding-a-pattern) is for. That section's example checks whether a *string* matches somewhere else; here you need the same idea applied to a prefix's own `prefix` property — i.e., the `WHERE NOT` pattern has to reference the property of a node you already matched, the way `{name: n1.name}` does in that example, but with `{prefix: rpfx.prefix}` instead.

Report both the count and a sample of the actual prefix strings — you'll want the list, not just the number, to reason about Q3.a.

## Global RPKI-Invalid Scan

IYP separately tags prefixes it classifies as RPKI-invalid, via a `CATEGORIZED` relationship to a `Tag` node. [Cypher §3.8](Cypher.md#38-starts-with) covers why this needs `STARTS WITH` rather than `=` — the graph carries more than one exact label in this family:

```cypher
MATCH (t:Tag)
WHERE t.label STARTS WITH "RPKI Invalid"
RETURN DISTINCT t.label
```

Chain that onto `ORIGINATE` ([Cypher §3.3](Cypher.md#33-multi-hop-traversal-chaining-relationship-types)) to find which ASes are actually announcing the tagged prefixes, aggregate with `count(DISTINCT ...)`, and cross-reference the result against AS6461, AS2906, AS4837 (Task 1) and AS2497 (Task 2) — do any of them show up?

## What Your Write-Up Should Address

- **Q3.a** — How many of AS6461's ROAs have no exact matching observed `BGPPrefix`? Looking at the actual prefixes, what's a plausible explanation for each (a prefix that's authorized but simply not announced right now vs. one where the announcement might use a different, unlisted prefix length)?
- **Q3.b** — Do any of the four running-example ASes appear in the top-20 RPKI-invalid list? If not, what does that suggest about them? If so, does it look like a genuine anomaly or something explainable?
- **Q3.c** — An `"RPKI Invalid"` tag means the ROA and the observed announcement disagree — it does not by itself mean malicious hijacking. What are at least two other explanations for the same tag (e.g. non-adoption of RPKI by a legitimate deaggregating announcer, a stale or missing ROA, a routing leak)? How does this graph-pattern comparison relate to the manual IRR/RPKI/BGP cross-comparison in [`nids-irr-rpki-whois`](https://github.com/CAIDA/nids-irr-rpki-whois) — what does the graph give you "for free" that you had to build there, and what nuance (if any) does the graph pattern flatten away?

[README](README.md) | [Introduction](Introduction.md) | [Datasets](Datasets.md) | [Cypher](Cypher.md) | [Tasks](Task.md) | [Task 1](Task-1.md) | [Task 2](Task-2.md) | Task 3 ⮕ | [Notebook](nids-iyp.ipynb)
