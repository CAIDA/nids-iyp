# IYP Schema Reference

See the CSVs in this directory for the underlying data. This rollup covers only the labels and
relationship types the three tasks in this module actually use; the full IYP schema (80+ datasets)
is documented at
[github.com/InternetHealthReport/internet-yellow-pages/documentation](https://github.com/InternetHealthReport/internet-yellow-pages/tree/main/documentation).

**Snapshot:** labels, relationship types, property names, and the `node_count`/`relationship_count`
columns were all read from the public IYP instance (`neo4j://iyp-bolt.ihr.live:7687`) on
**2026-08-03**. Relationship counts are directed -- one per stored relationship, not one per
traversal direction.

Treat the counts as a point-in-time reading, not a fixed property of IYP. The public instance is
reloaded as its sources update, so counts drift and query results will not match a write-up made on
a different day. That is the same provenance problem the module teaches: a number is only meaningful
alongside where and when it came from.

## Node Labels

See [`iyp_labels.csv`](iyp_labels.csv) for the full list. The ones used in this module:

| Label | Represents |
| --- | --- |
| `AS` | An Autonomous System |
| `Prefix` | An IP prefix — a supertype label also carried by every `BGPPrefix` and `RPKIPrefix` |
| `BGPPrefix` | An IP prefix observed announced in BGP |
| `RPKIPrefix` | An IP prefix covered by an RPKI ROA |
| `IP` | A single IP address |
| `HostName` | A fully-qualified domain name |
| `DomainName` | A non-FQDN registrable domain |
| `AuthoritativeNameServer` | A DNS authoritative nameserver |
| `IXP` | An Internet exchange point |
| `Name` | A name string an entity is known by |
| `Ranking` | A specific ranking system |
| `Tag` | A classification label |

## Relationship Types

See [`iyp_relationship_types.csv`](iyp_relationship_types.csv) for the full list, and
[`iyp_relationship_properties.csv`](iyp_relationship_properties.csv) for their properties.
Every relationship type also carries the reference/provenance properties described in
[Introduction.md](../Introduction.md#nodes-relationships-and-provenance) -- these aren't
re-listed per type in the CSV since they're common to all of them.
