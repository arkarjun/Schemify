# Topology Inference & Route Bundling — Design Doc

Status: Phase 1 and Phase 2 built and tested (jsdom harnesses `test_harness11.js`,
`test_harness12.js`); Phase 3 (bundled parallel-line rendering) not started. Written in
response to feedback from a working transit cartographer (July 2026) — see the "Known
limitation" note in README.md.

**What shipped, concretely:**
- The "Cross-route merge tolerance" Shape slider (0% default = off, identical behavior to
  before this existed). At tolerance &gt; 0:
  - `mergeNearbyNodes()` (Phase 1) collapses nearby vertices from independently-digitized
    routes into one, via grid-bucketed union-find on top of the existing tight `snapEps`
    jitter pass.
  - `projectStopsOntoLines()` (Phase 2) takes any point still standalone after that —
    stations recorded near a route rather than snapped onto it, the literal case a user hit
    in testing — and projects it onto the nearest line within the same tolerance, splicing
    it in as a real vertex (splitting the edge) rather than leaving it floating disconnected.
  - Both share one tolerance control by design, rather than two separate sliders, per the
    "opt-in, explicit control" resolution below.
- The edge-dedup bug fix (`edgeSet` now a `Map`, tracks `edge.routeFeatures`) — Phase 3's
  prerequisite, done independent of clustering.

**Not done:** Phase 1's diagnostics panel (flagging structurally-suspicious merges instead
of merging silently), Phase 3's actual bundled rendering, and validation against the
cartographer's real fixture data — every tolerance default here is still a reasonable guess,
not something checked against a real case.

## Problem

Schemify's schematization pipeline (`runPipeline()` in `index.html`) assumes the input
GeoJSON is already close to a resolved topology graph: points that represent the same
real-world junction or station are expected to already sit at (near-)identical
coordinates. That assumption holds for the OSM-derived city feeds Schemify fetches, where
shared nodes are literally the same OSM node.

It does not hold for raw route-level data. Two routes that run the same physical corridor
are commonly digitized as two separate, non-coincident lines, and stops are recorded near
a line rather than snapped onto it. Feeding that kind of data through Schemify today would
silently produce a broken or misleading graph rather than a useful schematic.

## What Schemify actually does today (not "nothing")

It's worth being precise here, because Schemify isn't starting from zero — it already does
a form of coordinate snapping, just tuned for a different problem (floating-point/projection
jitter, not cross-route entity resolution):

- `coordKey(x, y, eps)` (line 566) buckets a point onto a grid of cell size `eps` and
  returns a string key. Two points within the same cell collapse to one map key.
- `snapEps = Math.max(diag * 0.0015, 0.01)` (line 837) sets that cell size to 0.15% of the
  loaded data's bounding-box diagonal. This is small by design — enough to absorb
  reprojection noise, not enough to merge two independently-digitized lines running the
  same street a few meters apart.
- `getNode(x, y)` (line 847) is the single shared lookup all features funnel through
  (`nodeByKey` is one map across every loaded feature), so cross-feature merging already
  happens *when points are already almost coincident*. The gap is purely that the tolerance
  is too tight for real-world route divergence, and there's no mechanism for stops that
  aren't already vertices on a line.
- A second, separate gap: edge dedup. `edgeSet` (line 843) keys edges by their two node
  ids and **silently drops** a second feature's edge if the same A→B pair already exists
  (line 874: `if (edgeSet.has(key1) || edgeSet.has(b.id+"_"+a.id)) continue;`). So even
  when two routes *do* snap onto the same edge today, Schemify has no record that both
  routes use it — the second route's edge just vanishes with no multiplicity tracked. This
  needs fixing regardless of clustering tolerance, since it's the data route-bundling
  (Phase 3) would need to draw parallel offset lines at all.

## Prior art

Rather than invent this from scratch, two existing open-source tools solve adjacent
problems and are worth borrowing the *approach* from (both are Python/GIS-backend tools —
not directly embeddable in a single-file client-side JS app, but their algorithms port
cleanly to plain JS):

- **[gtfs-route-segments](https://pypi.org/project/gtfs-route-segments/)** (MIT, active as
  of June 2026) solves stop-to-route snapping. Naive nearest-point projection breaks on
  loops/lollipops/out-and-back routes because a stop near the base of a loop has two valid
  projection points and naive projection may pick the wrong one. Their fix: project stops
  in visit order (from the route's stop sequence) and pick the *earliest* reasonable
  candidate along the route, biasing toward forward progress. They also emit diagnostics —
  `max_offset_m` per stop, and a count of stops offset more than ~30m — which flags likely
  bad data (park-and-ride stops, wrong-shape assignment) instead of silently misplacing
  points.
- **[GTFS-to-Roads-converter](https://github.com/TRIP-Lab/GTFS-to-Roads-converter)**
  chops all route shapes into elementary segments, dedupes near-duplicates, then runs
  DBSCAN clustering on segment endpoints within a tunable radius (their default: 0.4m,
  explicitly called out as needing manual tuning per city/dataset — too small under-merges,
  too large wrongly merges two real, distinct stations).
- For the visual "draw N routes sharing a corridor as offset parallel lines cleanly, without
  crossings" problem specifically, that's an open research area in the metro-map layout
  literature (e.g. ["Ordering Metro Lines by Block
  Crossings"](https://www.researchgate.net/publication/236589377_Ordering_Metro_Lines_by_Block_Crossings)) —
  no off-the-shelf library solves it. Treated as a stretch goal below, not a blocker.

## Goals

- Accept route-level input where lines from different routes on the same corridor are
  digitized as separate, non-coincident geometry, and where stops may sit near (not on) a
  line.
- Merge points that represent the same real junction/station within a tunable distance
  tolerance, without wrongly merging two distinct nearby stations.
- Track, per graph edge, which routes/features traverse it (fixing the silent-drop bug
  above), as the prerequisite for any bundled rendering.
- Surface diagnostics (offset distances, merge counts) so bad input is visible rather than
  silently mis-rendered — matching the diagnostics precedent in gtfs-route-segments.

## Non-goals (this round)

- Crossing-free line ordering for bundled corridors. First pass uses a fixed deterministic
  order (e.g. route name); real minimization is a later, separate effort.
- Full GTFS ingestion (stop_times.txt, trips.txt parsing). Schemify takes GeoJSON; this
  only concerns how tolerant the topology-building step is once GeoJSON is loaded, not
  adding a GTFS importer.
- Automatic tolerance selection with no user input. See open question below.

## Proposed approach

### Phase 1 — Configurable, two-tier vertex snapping

Extend the existing `getNode()`/`coordKey()` mechanism rather than replacing it:

- Keep the current tight `snapEps` pass for jitter (it's cheap and already correct for
  that job).
- Add a second, coarser, user-configurable tolerance (a new Shape control, analogous to the
  existing simplify/straighten/min-segment-length sliders) for cross-route merging. Grid-
  bucketed nearest-neighbor at this scale is enough — full DBSCAN isn't needed at typical
  transit-network sizes (hundreds to low thousands of points), and avoids a new dependency.
- Where the coarse tolerance would merge two points that are farther apart than the tight
  pass but structurally suspicious (e.g. very different original feature names, or a merge
  distance close to the tolerance ceiling), log it to a diagnostics panel instead of
  merging silently — directly modeled on gtfs-route-segments' offset diagnostics.

### Phase 2 — Stop-to-route projection (only if/when needed)

If input ever arrives as ordered stop coordinates without an accompanying line geometry
(vs. requiring a shape per route, which is what GeoJSON LineStrings already give us), port
the sequential-projection idea: process stops in given order, project each onto the nearest
line within tolerance, pick the earliest valid candidate rather than the globally-nearest
point. Given Schemify's input is GeoJSON (which naturally carries line geometry already),
this phase may turn out to be unnecessary — worth confirming against the actual fixture
data before building it.

### Phase 3 — Corridor bundling (rendering)

- Fix the edge-dedup bug: track a list of contributing `featureIdx` values per edge instead
  of dropping duplicates.
- Where an edge's route count > 1, render it as N parallel offset lines instead of one.
  First pass: fixed deterministic order (insertion order or alphabetical by layer/feature
  name). No crossing minimization.
- Reuse the existing `resolveOverlaps()` declutter pass conceptually (it already nudges
  nearby unconnected nodes apart) as a starting point for spacing logic, though it operates
  on nodes today and offsetting parallel edges is a different geometric operation.

## Validation plan

Tolerance values (Phase 1) and offset ordering (Phase 3) cannot be responsibly tuned by
eyeballing — they need a real input/expected-output pair to check against. This is exactly
what the cartographer offered: raw route GeoJSON plus a hand-built "ideal schematic" to use
as a fixture. Recommend getting that before writing the Phase 1 merge logic, so the default
tolerance and diagnostics thresholds are chosen against a real case rather than guessed and
re-guessed. Once available, it slots into the same jsdom-harness testing pattern already
used elsewhere in this project (`test_harness*.js`).

## Open questions

- **Tolerance: one global number, or density-aware?** Still open — shipped as a single
  fixed distance (0–5% of bbox diagonal). Dense downtown stops sitting closer together than
  suburban ones could still wrongly merge or under-merge one or the other; needs real
  fixture data to know if this is actually a problem in practice or a theoretical one.
  Density-aware tolerance would be a follow-up if so.
- **Where in the pipeline does clustering run — before or after `rdpSimplify`?** Resolved:
  after. `runPipeline()` still simplifies and tight-snaps each feature independently first
  (as before), *then* runs Phase 1 merge and Phase 2 projection once across the combined
  graph. Untested against real messy data, so worth revisiting if fixture data shows simplify
  is dropping a point that clustering should have kept, or vice versa.
- **Opt-in or always-on?** Resolved: opt-in, one explicit "Cross-route merge tolerance"
  slider defaulting to 0% (off) — existing clean inputs (the OSM city feeds) render exactly
  as before. Phase 1 and Phase 2 both gate on the same control rather than getting separate
  sliders, since from a user's perspective they're one "how sloppy is my input topology"
  knob, not two different settings to reason about.
- **Interaction with per-segment styling.** Schemify's Style & Layers panel supports
  selecting and styling a single segment between two points. Once one rendered line can
  represent multiple routes' shared corridor, the current one-path-per-feature/segment
  selection model needs to account for "this drawn line is actually N routes" — not yet
  designed.

## Rollout order

1. ~~Phase 1 (the direct blocker; reuses the existing graph step once points are pre-snapped)~~ — done
2. ~~Phase 3's dedup-bug fix (small, correctness fix, useful independent of clustering)~~ — done
3. ~~Phase 2~~ — done; needed after all (confirmed by direct testing, not just fixture data)
4. Phase 3's bundled rendering (depends on 1 and 2, both now in place) — not started
5. Phase 1's diagnostics panel — not started
6. Validation against real fixture data — still outstanding; still recommended before trusting
   the current default tolerance range (0–5% of bbox diagonal) on real messy data
