[README](README.md) | [Introduction](Introduction.md) | [Datasets](Datasets.md) | [Cypher](Cypher.md) | [Tasks](Task.md) | [Task 1](Task-1.md) | [Task 2](Task-2.md) | Task 3 ⮕ | [Notebook](nids-iyp.ipynb)

# Task 3: RPKI-Authorized vs. BGP-Observed Origins — How-To Guide

## ROA Coverage for a Single AS

An RPKI ROA is a `ROUTE_ORIGIN_AUTHORIZATION` relationship between an `AS` and an `RPKIPrefix`; an observed BGP announcement is the same `ORIGINATE` relationship you used in Task 2, but pointing at a `BGPPrefix` instead. A ROA with no matching observed announcement is a prefix that exists in one graph pattern but not the other — expressed directly as a negative pattern with `WHERE NOT`:

```cypher
// step 1: every ROA naming AS3356 as the authorized origin
MATCH (roa_as:AS {asn: 3356})-[:ROUTE_ORIGIN_AUTHORIZATION]-(rpfx:RPKIPrefix)

// step 2: keep only the ones with NO matching observed BGPPrefix -- i.e., no
// BGPPrefix node exists with the same prefix string, contained in this RPKIPrefix
WHERE NOT (rpfx)-[:PART_OF]-(:BGPPrefix {prefix: rpfx.prefix})

RETURN rpfx.prefix
```

Report both the count and a sample of the actual prefix strings — you'll want the list, not just the number, to reason about Q3.a.

## Global RPKI-Invalid Scan

IYP separately tags prefixes it classifies as RPKI-invalid, via a `CATEGORIZED` relationship to a `Tag` node whose `label` starts with `"RPKI Invalid"`. Chain that onto `ORIGINATE` to find which ASes are actually announcing those tagged prefixes:

```cypher
// step 1: every BGPPrefix tagged RPKI Invalid
MATCH (pfx:BGPPrefix)-[:CATEGORIZED]-(t:Tag)
WHERE t.label STARTS WITH "RPKI Invalid"

// step 2: who originates it
MATCH (pfx)-[:ORIGINATE]-(a:AS)

RETURN a.asn AS asn, count(DISTINCT pfx) AS num_invalid_prefixes
ORDER BY num_invalid_prefixes DESC
LIMIT 20
```

Cross-reference the resulting list against AS3356, AS2906, AS4837 (Task 1), and AS2497 (Task 2) — do any of them show up?

## What Your Write-Up Should Address

- **Q3.a** — How many of AS3356's ROAs have no matching observed `BGPPrefix`? Looking at the actual prefixes, what's a plausible explanation for each (a prefix that's authorized but simply not announced right now vs. one where the announcement might use a different, unlisted prefix length)?
- **Q3.b** — Do any of the four running-example ASes appear in the top-20 RPKI-invalid list? If not, what does that suggest about them? If so, does it look like a genuine anomaly or something explainable?
- **Q3.c** — An `"RPKI Invalid"` tag means the ROA and the observed announcement disagree — it does not by itself mean malicious hijacking. What are at least two other explanations for the same tag (e.g. non-adoption of RPKI by a legitimate deaggregating announcer, a stale or missing ROA, a routing leak)? How does this graph-pattern comparison relate to the manual IRR/RPKI/BGP cross-comparison in `nids-irr-rpki-whois` — what does the graph give you "for free" that you had to build there, and what nuance (if any) does the graph pattern flatten away?

[README](README.md) | [Introduction](Introduction.md) | [Datasets](Datasets.md) | [Cypher](Cypher.md) | [Tasks](Task.md) | [Task 1](Task-1.md) | [Task 2](Task-2.md) | Task 3 ⮕ | [Notebook](nids-iyp.ipynb)
