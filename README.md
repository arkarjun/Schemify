# Schemify

**An Open Source Schematization Tool by GeoMinds**

Schemify turns real-world GeoJSON data into clean, schematic maps — the kind of simplified,
straight-line, angle-snapped look you see on subway and transit maps — directly in your browser.
No server, no build step, no install: it's a single HTML file.

🔗 **Live demo:** https://arkarjun.github.io/Schemify/ *(once GitHub Pages is enabled — see below)*

## What it does

1. **Generate** — Upload or paste a GeoJSON file (points, lines, and/or polygons). Schemify
   simplifies the geometry, builds a graph of shared points, and iteratively straightens and
   snaps edges to a small set of angles (45° octilinear, 90° orthogonal, or off) — the same idea
   behind classic transit-map schematization.
2. **Style** — Switch to Style mode to customize each layer: line color, width, dash pattern,
   and opacity; junction/station dot visibility, radius, and shape; label visibility, size, and
   color; layer draw order; and canvas background. Export your styling as a JSON preset and
   re-import it later or share it with someone else.

Two sample datasets (a small transit network, and a mixed polygons/line/points map) are built in
so you can try it without a file of your own.

## Usage

Just open `index.html` in a browser — locally or via GitHub Pages. There's nothing to build or
install.

1. Upload a `.geojson`/`.json` file, paste GeoJSON text, or click one of the sample buttons.
2. Tune simplification, straightening strength, and angle-snap mode in the **Generate** tab.
3. Switch to the **Style** tab to color and label each layer, reorder them, and set a background.
4. Click **Download schematic SVG** to export the result, or **Export style JSON** to save your
   styling as a reusable preset.

## How the schematization works

- **Simplify:** each line/polygon ring is reduced with Douglas–Peucker simplification.
- **Graph:** shared coordinates (e.g. line intersections) are snapped together into common nodes
  so topology is preserved.
- **Straighten:** edge angles are snapped to the nearest allowed direction (45° or 90°
  increments), then node positions are relaxed iteratively so edges align to their snapped
  angles while staying connected.
- **Declutter:** a light overlap-resolution pass nudges nearby unconnected nodes apart.

This is a fast heuristic, not a constraint solver — degree-2 paths (typical roads/lines) snap
cleanly, but busy junctions with 3+ connections can land a few degrees off-grid since not every
angle constraint can be satisfied at once. A future version may explore a proper optimization-based
layout for those cases.

## Status / roadmap

- [x] GeoJSON → schematic map generation (points, lines, polygons)
- [x] Per-layer styling (color, width, dash, opacity, nodes, labels, order, background)
- [x] Style export/import as JSON presets
- [ ] Optimization-based (non-heuristic) angle solving for complex junctions
- [ ] PNG export
- [ ] Saving/loading full map projects (data + style together)

Issues and PRs welcome.

## License

MIT — see [LICENSE](./LICENSE). © GeoMinds, with contributions from Ark Arjun.
