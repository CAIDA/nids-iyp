[README](README.md) | [Introduction](Introduction.md) | [Datasets](Datasets.md) | Cypher ⮕ | [Tasks](Task.md) | [Task 1](Task-1.md) | [Task 2](Task-2.md) | [Task 3](Task-3.md) | [Notebook](nids-iyp.ipynb)

# Cypher Guide: Querying IYP from Python

This guide builds up the Cypher you need for the three tasks, one piece at a time, and shows how to run each query from Python so the result comes back as a pandas DataFrame. Every query targets the labels and relationship types described in [Datasets](Datasets.md); read that page first for the full schema and the [Writing Efficient Queries](Datasets.md#writing-efficient-queries) rules the examples below follow.

Each [Task guide](Task.md) contains the complete, commented query for that task. This page explains the *building blocks* those queries are assembled from.

## 1. Running Cypher from Python

The notebook opens one Neo4j driver (see [Datasets](Datasets.md#connecting)):

```python
from neo4j import GraphDatabase
db = GraphDatabase.driver(IYP_URI, auth=(IYP_USER, IYP_PASSWORD))
```

You never format query results yourself. Hand a Cypher string (and any parameters) to `db.execute_query`, which runs the query and returns `(records, summary, keys)`:

```python
records, _, keys = db.execute_query("""
    MATCH (a:AS {asn: 2497})
    RETURN a.asn AS asn
""")

import pandas as pd
df = pd.DataFrame(records, columns=keys)
df.head()   # a normal pandas DataFrame from here on
```

The notebook wraps this pattern in a `run_query()` helper (see [Task.md](Task.md#task-0-setup-and-data-access)) that returns the DataFrame directly and refuses to execute anything containing a write keyword — use `run_query(cypher, **params)` everywhere below instead of calling `db.execute_query` yourself.

## 2. Node and Relationship Types

The full label/relationship-type reference is in [Datasets](Datasets.md#node-labels). Everything in this module traverses a small subset of it: `AS`, `BGPPrefix`, `RPKIPrefix`, `IP`, `HostName`, `IXP`, `Ranking`, `Name`, `Tag`, connected by `ORIGINATE`, `ROUTE_ORIGIN_AUTHORIZATION`, `PEERS_WITH`, `MEMBER_OF`, `PART_OF`, `RESOLVES_TO`, `MANAGED_BY`, `NAME`, `RANK`, and `CATEGORIZED`.

## 3. Building Blocks

### 3.1 Filter by a property

`(a:AS {asn: 2497})` matches `AS` nodes with `asn` equal to `2497` — Neo4j uses the index on `AS.asn` for this, so it's a cheap lookup rather than a scan of every `AS` node:

```cypher
MATCH (a:AS {asn: 2497})
RETURN a.asn, a
```

The equivalent with a `WHERE` clause is logically the same query, and sometimes clearer when you're filtering on a value computed elsewhere:

```cypher
MATCH (a:AS)
WHERE a.asn = 2497
RETURN a.asn, a
```

### 3.2 Traverse a relationship (one hop)

A relationship pattern names the edge type in square brackets. This finds every `IXP` a given AS belongs to:

```cypher
MATCH (a:AS {asn: 2497})-[:MEMBER_OF]-(ix:IXP)
RETURN ix.name
```

Relationships in Cypher patterns are undirected by default (`-[:TYPE]-`); add an arrowhead (`-[:TYPE]->`) only when the direction actually matters to the question you're asking.

### 3.3 Multi-hop traversal (chaining relationship types)

This is the technique that makes IYP different from the siloed datasets in other modules: chain several relationship types in one `MATCH` to walk from one kind of node to a completely different kind, in a single query. This finds every hostname that resolves into an AS's announced address space — BGP origin, prefix containment, and DNS resolution, joined in one pattern:

```cypher
MATCH (a:AS {asn: 2501})-[:ORIGINATE]-(pfx:BGPPrefix)-[:PART_OF]-(:IP)-[:RESOLVES_TO]-(h:HostName)
RETURN pfx.prefix, collect(DISTINCT h.name)
```

Read it left to right: start at one `AS`, walk to the `BGPPrefix` nodes it originates, walk to the `IP` addresses contained in those prefixes, walk to the `HostName` nodes that resolve to those IPs. Compare this to `nids-dns-ecosystem`, where the same "which domains does this AS's address space host" question requires separately loading a BGP prefix-to-AS table and a DNS resolution table, then joining them yourself.

### 3.4 `OPTIONAL MATCH` for possibly-missing relationships

A plain `MATCH` silently drops a row when the pattern doesn't match at all — which looks identical to "this AS really has zero results" even when it just means "this particular relationship happens to be missing." Use `OPTIONAL MATCH` for the part of the pattern that might not exist, so you keep the row with a null value instead of losing it:

```cypher
MATCH (a:AS {asn: 2501})-[:ORIGINATE]-(pfx:BGPPrefix)
OPTIONAL MATCH (pfx)-[:PART_OF]-(:IP)-[:RESOLVES_TO]-(h:HostName)
RETURN pfx.prefix, collect(DISTINCT h.name)
```

Now every one of the AS's prefixes appears in the result, even the ones with no resolving hostnames (`collect()` just returns an empty list for those).

### 3.5 Filtering by `reference_org` / `reference_name`

Because multiple sources can populate the same relationship type, add an inline property filter on the *relationship* (not just the nodes) when you want one source's claim specifically:

```cypher
MATCH (a:AS {asn: 2497})-[mem:MEMBER_OF]-(ix:IXP)
WHERE mem.reference_org <> 'PeeringDB'
RETURN ix.name, mem.reference_org
```

Skipping this filter when a task calls for it means silently mixing multiple sources' claims together — see [Datasets](Datasets.md#writing-efficient-queries).

### 3.6 Aggregation: `collect()`, `count()`, `DISTINCT`

`collect()` gathers values across matched rows into a list; `count()` counts them. Both respect `DISTINCT` the same way SQL's aggregates do:

```cypher
MATCH (m:AS)-[:MEMBER_OF]->(ix:IXP)
RETURN ix.name AS ixp_name, count(DISTINCT m) AS num_members
ORDER BY num_members DESC
LIMIT 10
```

### 3.7 Staging a query with `WITH`

`WITH` passes the result of one part of a query into the next, letting you compute an intermediate aggregate (or filter) before continuing the pattern — the Cypher equivalent of a SQL CTE:

```cypher
MATCH (a:AS)-[:PEERS_WITH]-(b:AS)
WITH a, count(DISTINCT b) AS num_peers
WHERE num_peers > 100
RETURN a.asn, num_peers
ORDER BY num_peers DESC
```

### 3.8 `coalesce()` across multiple sources

When several `NAME` relationships (one per source) might exist for the same node, `OPTIONAL MATCH` each source separately and pick the first non-null with `coalesce()` — this is how you get one canonical name back instead of one row per source:

```cypher
MATCH (a:AS {asn: 2497})
OPTIONAL MATCH (a)-[:NAME {reference_org: 'PeeringDB'}]->(n1:Name)
OPTIONAL MATCH (a)-[:NAME {reference_org: 'BGP.Tools'}]->(n2:Name)
RETURN a.asn, coalesce(n1.name, n2.name) AS name
```

### 3.9 Parameterized queries from Python

Pass values as query parameters instead of formatting them into the query string — `execute_query` (and the notebook's `run_query()` wrapper) accept them as keyword arguments:

```python
df = run_query(
    "MATCH (a:AS {asn: $asn})-[:ORIGINATE]-(pfx:BGPPrefix) RETURN count(DISTINCT pfx) AS num_prefixes",
    asn=2497,
)
```

Prefer this over building the query with an f-string whenever the value comes from a variable — it's one less thing to get wrong, and it lets Neo4j reuse the query plan across calls with different parameter values.

## 4. Using the Results in Pandas

Once a query returns a DataFrame, the analysis is plain pandas — the same idioms used elsewhere in this curriculum:

- **Sort and limit** — `df.sort_values("num_members", ascending=False).head(10)`.
- **Count distinct within groups** — `df.groupby("asn")["name"].nunique()`.
- **Label codes with names** — `df["asn"].map(names_by_asn)` to swap ASNs for the canonical names you resolved with `coalesce()` in §3.8.

## 5. Writing Efficient Queries

The full rules are in [Datasets](Datasets.md#writing-efficient-queries); the short version:

- **Filter on an indexed property first** (`{asn: ...}`, `{prefix: ...}`) — don't scan a whole label and filter after.
- **Filter by `reference_org`/`reference_name` before aggregating**, whenever more than one source can populate a relationship type.
- **Avoid unbounded variable-length paths** (`-[*]-`) — chain explicit relationship types instead.
- **Prototype with `LIMIT`**, and return only the properties you need rather than whole nodes.

[README](README.md) | [Introduction](Introduction.md) | [Datasets](Datasets.md) | Cypher ⮕ | [Tasks](Task.md) | [Task 1](Task-1.md) | [Task 2](Task-2.md) | [Task 3](Task-3.md) | [Notebook](nids-iyp.ipynb)
