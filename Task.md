[README](README.md) | [Introduction](Introduction.md) | [Datasets](Datasets.md) | [Cypher](Cypher.md) | Tasks ⮕ | [Task 1](Task-1.md) | [Task 2](Task-2.md) | [Task 3](Task-3.md) | [Notebook](nids-iyp.ipynb)

# Tasks

Complete the tasks below in order. All three tasks are completed inside [nids-iyp.ipynb](nids-iyp.ipynb) — replace the `# YOUR CODE HERE` sections with your code and answer the questions in the markdown cells that follow.

## Task 0: Setup and Data Access

- step 1. Install dependencies and launch Jupyter — pick one:

  ```bash
  # Option A: uv
  uv sync
  uv run jupyter lab nids-iyp.ipynb
  ```

  ```bash
  # Option B: pip + venv
  python3 -m venv .venv
  source .venv/bin/activate        # Windows: .venv\Scripts\activate
  pip install -r requirements.txt
  jupyter lab nids-iyp.ipynb
  ```

- step 2. Run the notebook's setup cell. It connects to the public IYP instance at
  `neo4j://iyp-bolt.ihr.live:7687`, which needs no credentials — there is nothing to configure (see
  [Datasets](Datasets.md#connecting)).
- step 3. Confirm the connection check prints the node and relationship counts it queries. If it
  fails, you have a network problem rather than a credentials problem.
- step 4. Complete each task by replacing the `# YOUR CODE HERE` sections and answer all questions.
- step 5. Note the date you ran your queries. The public instance is reloaded as its sources update,
  so your numbers are only interpretable alongside when you collected them.
- step 6. Save your completed notebook, commit, and push.

## Task 1: The AS Ecosystem — Ranking, IXPs, and Peering

**Builds on:** [nids-asn-introduction](https://github.com/CAIDA/nids-asn-introduction)

Using three running-example autonomous systems — **Zayo (AS6461)**, **Netflix (AS2906)**, and **China Unicom (AS4837)** — explore how IYP represents an AS's position in the Internet: what it's called (and whether sources agree), how it ranks, which IXPs it belongs to, and how many other ASes it peers with directly.

- [ ] **Q1.a**: For each of the three ASes, resolve a canonical name across multiple `NAME` sources and look up its CAIDA ASRank. Do the sources agree on the name? Report both the merged name and each individual source's claim.
- [ ] **Q1.b**: Which 10 IXPs have the most AS members globally? Separately, how many IXPs does each of the three ASes belong to?
- [ ] **Q1.c**: How many distinct ASes does each of the three peer with directly (`PEERS_WITH` — every BGP peering session, not only settlement-free peering agreements; see [Task 1](Task-1.md#peering-degree))? Write a short paragraph relating peering degree, IXP membership, and ASRank for these three networks — do they move together the way you'd expect?

## Task 2: Bridging BGP and DNS in One Traversal

**Builds on:** [nids-asn-introduction](https://github.com/CAIDA/nids-asn-introduction), [nids-dns-ecosystem](https://github.com/caida/nids-dns-ecosystem)

Using **IIJ (AS2497)** and **AS2501** as the primary examples — the same two ASes IYP's own tutorial uses for this kind of query — walk from an AS's announced BGP address space to the domain names hosted inside it, in a single Cypher pattern. Compare against **Zayo (AS6461)**, a pure transit backbone of comparable stature, to see what a network with little hosted content looks like by the same measure.

- [ ] **Q2.a**: For AS2497 and AS6461, how many distinct popular hostnames resolve into each AS's announced prefixes, both in total and per announced prefix? What does the difference tell you about what kind of network each one is?
- [ ] **Q2.b**: For AS2501, which authoritative nameservers manage the most domains hosted in its address space? Produce a table of nameserver → domain count.
- [ ] **Q2.c**: This traversal joins BGP origin, prefix containment, and DNS resolution in one query. In [`nids-dns-ecosystem`](https://github.com/caida/nids-dns-ecosystem), the same kind of join required loading OpenINTEL DNS data and a separate BGP prefix-to-AS mapping, then joining them yourself in Spark. What did the graph traversal do in one step that took multiple stages there? What did you lose, if anything, by not doing it yourself?

## Task 3: RPKI-Authorized vs. BGP-Observed Origins

**Builds on:** [nids-irr-rpki-whois](https://github.com/CAIDA/nids-irr-rpki-whois)

Compare what an AS is cryptographically authorized to originate (an RPKI ROA) against what is actually observed in BGP, for **Zayo (AS6461)**, and then broaden the lens to a global scan of prefixes IYP has tagged `"RPKI Invalid"`.

- [ ] **Q3.a**: For AS6461, find every RPKI ROA (`ROUTE_ORIGIN_AUTHORIZATION`) that has no exact matching observed `BGPPrefix`. How many are there, and what might explain a ROA existing with no corresponding BGP announcement?
- [ ] **Q3.b**: Across the whole graph, which 20 ASes originate the most prefixes tagged `"RPKI Invalid"`? Do any of the four running-example ASes (AS6461, AS2906, AS4837, AS2497) appear in that list?
- [ ] **Q3.c**: Does an `"RPKI Invalid"` tag prove a prefix hijack? What else could produce that tag (non-adoption, misconfiguration, legitimate deaggregation)? How does this graph-pattern comparison relate to the manual IRR/RPKI/BGP comparison you did (or would do) in [`nids-irr-rpki-whois`](https://github.com/CAIDA/nids-irr-rpki-whois)?

[README](README.md) | [Introduction](Introduction.md) | [Datasets](Datasets.md) | [Cypher](Cypher.md) | Tasks ⮕ | [Task 1](Task-1.md) | [Task 2](Task-2.md) | [Task 3](Task-3.md) | [Notebook](nids-iyp.ipynb)
