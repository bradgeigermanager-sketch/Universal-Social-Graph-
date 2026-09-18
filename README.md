# Universal-Social-Graph-
A central hub format social media identity file for people, groups, places and organizations, useful to map all their social media pages with.



META-PROMPT (hand this to any LLM to bootstrap the project)

Build a Unified Social Graph (USG) hub that ingests social/platform identities for people, places, groups, and organizations into one canonical graph, using bidirectional transformers per platform. Treat Social Analyzer (qeeqbox, AGPL-3.0) as the discovery sidecar for ~1000 platforms and OSGE (Open Social Graph Explorer) as the graph-store + analysis + viz layer. The hub stores ONLY canonical USG nodes/edges; every platform-specific shape lives behind an adapter that implements to_usg(raw) -> USG and from_usg(usg) -> raw.
Canonical core (v1):
- Entity {id, label:"Entity", kind:person|place|group|org|brand, canonical_name, aliases[], trust, first_seen}
- Account {id:"acct::", label:"Account", platform, handle, display_name, url, verified, discovery_conf, native:{}, first_seen}
- Edge {from, to, type:OWNED_BY|ASSERTS|MEMBER_OF|CROSS_POSTS|IMPERSOSTATES|REBRAND_OF|SIBLING_OF, relation_conf, evidence, source, observed_at}
Adapter ABC: to_usg_account(raw,ctx), to_usg_edges(ent,acct,raw,ctx), from_usg_account(acct), optional discover(seed,opts).
Hub: ingest_raw(adapter,raw,ctx), ingest_discovery(adapter,seed,opts,entity_id), export_to(adapter,acct_id).
SA integration: run SA as Docker sidecar on :9005, call POST /api/check, translate its detected[] (link/rate_percent/status/title/desc) into the Account+DISCOVERY_MATCH edge shape; never import SA source into the hub (AGPL boundary = HTTP only).
OSGE integration: use its Entity-container vs Account-node split, edge-confidence+evidence model, PageRank/community detection; back it with MemStore for v0, Neo4j for persistence.
Imposter scoring: platform-agnostic module over USG Account nodes (handle Levenshtein to verified official, unverified + official-like keywords, no CROSS_POSTS to verified cluster, SA conf≥0.6, scam lingo) → IMPERSOSTATES edge ≥0.6.
Seed example: Thetan World (studio) → Thetan Arena (rebrand sub-brand) → Thetan Rivals (sibling); official accounts on x/discord/reddit/web/telegram; SA discovers thetanarena permutations; imposter scorer flags thetanarena_giftcode IG clone.
Deliverables: usg/core.py, usg/hub.py, adapters/base.py, adapters/x.py, adapters/discord.py, adapters/reddit.py, adapters/web.py, adapters/sa_discovery.py (wraps SA sidecar), scoring/imposter.py, storage/seed_theta.py, tests/test_x_adapter.py, tests/test_discovery_theta.py. Sidecar compose for SA, README with deploy order.

That block is the whole spec. Everything below is the human-readable expansion of it, so you can see how SA and OSGE map in.

How Social Analyzer maps into the hub

SA concept	USG mapping
POST /api/check username query	adapter.sa_discovery.discover(seed, opts)
detected[] entry	one candidate Account node (unverified, discovery_conf=rate_percent/100)
status: good/maybe/bad	kept as sa_status on Account; bad skipped on ingest
link field	url
rate_percent	discovery_conf
title/desc/screenshot in SA extras	folded into native + used by imposter scorer
SA sites.json 1000+ platforms	each becomes a platform key; no per-site code needed, transformer is generic
Ixora force-graph in SA	not reused — OSGE/Neo4j Browser/Sigma.js is the viz layer instead
SA stays a black-box HTTP service. The hub's sa_discovery.py is the only file that knows SA's JSON shape.

How OSGE maps into the hub

OSGE concept	USG mapping
Identity container	Entity node
Account node	Account node (identical)
Edge with weight+confidence+evidence	Edge (identical schema)
Platform adapter in OSGE	promoted to the ABC above
PageRank / betweenness / community detect	run on USG store as-is
SQLite/Neo4j store	store interface behind USGHub
Local-first + public-only scope	kept as an adapter boundary rule (no private Discord channels, no logged-in scrape)
OSGE is not "called" — its ideas* are the hub's graph engine. You can literally drop osge's store + analysis modules under usg/store/ and usg/analysis/ and rename them.

Transformer flow (both directions)

Ingest (platform → USG)
SA sidecar --(detected[])--> sa_discovery.to_usg_account --> Account node
X API JSON --> XAdapter.to_usg_account --> Account node
                                   |
                                   v
                            USGHub.ingest_raw --> store.upsert_node
                            adapter.to_usg_edges --> OWNED_BY / CROSS_POSTS / etc.

Export (USG → platform)
USG Account --> XAdapter.from_usg_account --> {username, ...}  (read-only returns minimal)
USG Account --> MastodonAdapter.from_usg_account --> status-draft payload (writable)

Round-trip invariant: to_usg(from_usg(x)) reproduces the 6 canonical fields exactly; native may differ. That's the per-adapter test.

Why this is LLM-ready
- The meta-prompt at the top is self-contained: an LLM with no context can emit the file tree, the schemas, the SA-translation shim, and the imposter heuristic from it.
- SA and OSGE are named with their real roles (sidecar vs graph engine), so the LLM won't try to "import OSGE into SA" or violate AGPL by inlining SA.
- Thetan is referenced only as a seed fixture, not a dependency — swap seed_theta.py for seed_acme_corp.py and nothing else changes.

Minimal file tree the LLM should produce
usg/
  core.py            # Entity/Account/Edge dataclasses, json serialize
  hub.py             # USGHub: ingest_raw/ingest_discovery/export_to
  scoring/imposter.py
  adapters/
    base.py          # PlatformAdapter ABC
    sa_discovery.py  # wraps SA :9005, detected[] -> USG
    x.py  discord.py  reddit.py  web.py
  store/
    mem_store.py     # v0
    neo_store.py     # optional
  storage/
    seed_theta.py    # USG-only seed (no adapter imports)
tests/
  test_x_adapter.py
  test_discovery_theta.py
docker-compose.yml   # social-analyzer sidecar :9005
README.md            # deploy order: SA sidecar -> pip install -e . -> pytest -> run_theta

That's the fusion: A generalized-hub design + SA/OSGE mapping + LLM-bootstrap prompt in one. Hand the quoted meta-prompt to a fresh LLM and it regenerates the v0; the prose below it is just the rationale so you can debug the output.
