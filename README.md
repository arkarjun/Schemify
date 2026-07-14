# Schemify

**An Open Source Schematization Tool by Geominds**

Schemify turns real-world GeoJSON data into clean, schematic maps — the kind of simplified,
straight-line, angle-snapped look you see on subway and transit maps — directly in your browser.
No server, no build step, no install: it's a single HTML file.

🔗 **Live demo:** https://arkarjun.github.io/Schemify/
📦 **Source:** https://github.com/arkarjun/Schemify

## What it does

1. **Data & Shape** — Upload or paste a GeoJSON file (points, lines, and/or polygons), load a
   built-in sample, or fetch a real metro/subway network by city (via the
   [Organic Maps Subway Validator](https://cdn.organicmaps.app/subway/), built on OpenStreetMap
   data). Schemify simplifies the geometry, builds a graph of shared points, and iteratively
   straightens and snaps edges to a small set of angles (45° octilinear, 90° orthogonal, or off)
   — the same idea behind classic transit-map schematization. Shape controls (simplify tolerance,
   straightening strength, minimum segment length, angle snapping) live both on that first Data
   screen and in a standalone Shape section on the left rail, so you can keep tuning them after
   data is loaded without reopening the modal. A **Next** button confirms you're done and moves
   to the map, rather than the modal disappearing on its own the moment data loads.
2. **Style & layers** — Its icon lives on the left rail alongside Data/Shape/Help, but the panel
   itself opens on the right, out of the way of the map. Customize each layer: line color, width,
   dash pattern, and opacity; junction/station dot visibility, radius, and shape; layer draw
   order; and canvas background. Import a QGIS `.qml` layer style to bring in color/width/dash
   from an existing QGIS project, or import/export Schemify's own style as a JSON preset (an info
   icon next to those buttons explains the file format).
3. **Label / text styling** — Per layer: bold/italic/underline, a QGIS-style buffer (halo) with
   configurable width and color, text color, and a font picker across five categories
   (Sans-serif/Serif/Script/Display/Monospace). Each is led by an open-source font with native
   Malayalam support, backed by Noto per-script fallbacks for other Indian scripts. Sans-serif,
   Serif, Script, and Display use Manjari, Rachana, Chilanka, and Keraleeyam from
   [Swathanthra Malayalam Computing (SMC)](https://smc.org.in/); Monospace uses
   [JetBrains Mono](https://www.jetbrains.com/lp/mono/). See the in-app About panel for full credits.
4. **Pan / Select / Text tools** — A floating toolbar on the schematic panel switches between
   panning the map (default), selecting a layer or a single segment to style it, and clicking an
   existing label with the Text tool to jump straight to that layer's text styling.

Two sample datasets (a small transit network, and a mixed polygons/line/points map) are built in
so you can try it without a file of your own.

## Usage

Just open `index.html` in a browser — locally or via GitHub Pages. There's nothing to build or
install.

1. On first load (or anytime via the **Data & Shape** icon on the left rail) you land on a single
   start screen: upload a file, paste GeoJSON, load a sample, or fetch a city's metro network —
   then tune simplification, straightening strength, and angle-snap mode below it. The same Shape
   controls reappear in the left rail's **Shape** section once the start screen is closed.
2. Click **Style & layers** on the left rail to open its panel on the right: color each layer,
   reorder them, set a background, import a QGIS `.qml` style, or import/export a JSON style
   preset.
3. On the schematic panel, use the floating toolbar over the map: zoom in/out, then **Pan /
   Select / Text**. Pan moves around, Select clicks a line, station, or a single stretch between
   two points to style it, and Text clicks an existing label to edit its bold/italic/underline,
   buffer, color, and font.
4. Use the SVG/PNG buttons in the schematic panel's header to export the result, or
   **Export style** to save your styling as a reusable JSON preset.
5. The schematic panel's header also holds **Sync zoom** (pan/zoom both panels together) and
   **Reset view** (snap both back to full extent).
6. Switch how the raw/original map is shown with the icon group in that same header — **Inset**
   (default, a small mini-map over the schematic — click its eye icon to hide it, or its expand
   icon to view it full screen), **Split** (stacked top/bottom), or **Full** (schematic only).
7. Use **Report a bug** / **Request a feature** in the footer to open a prefilled GitHub issue
   (basic diagnostics included automatically; your map data is never sent).

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
- [x] Per-segment style overrides (style a single stretch between two points)
- [x] Style export/import as JSON presets
- [x] Import QGIS `.qml` layer styles (best-effort, single-symbol renderers)
- [x] SVG and PNG export
- [x] Synced pan/zoom between the two panels
- [x] Fetch real metro/subway networks by city (Organic Maps / OpenStreetMap)
- [x] Inset / split / full map layout modes
- [x] Shape controls available both in the Data modal and a standalone left-rail section
- [x] Style & layers panel: icon stays on the left rail, content opens in a right-hand drawer
- [x] Explicit Pan / Select / Text floating toolbar on the schematic panel, with zoom controls
- [x] Label text styling: bold/italic/underline, buffer/halo, color, multi-script font picker with
      live preview
- [x] Report a bug / Request a feature footer links (prefilled GitHub issue, with diagnostics)
- [x] Non-abrupt Data modal flow with an explicit "Next" button
- [x] Sync zoom / Reset view / layout switch consolidated into the schematic panel's own header
- [x] Accessibility pass: keyboard-operable dropzone, ARIA labels and pressed-states on icon-only
      controls, accessible SVG names, visible focus rings
- [ ] Optimization-based (non-heuristic) angle solving for complex junctions
- [ ] Saving/loading full map projects (data + style together)
- [ ] Broader city coverage for the network-fetch picker (currently ~45 curated cities; paste
      any direct GeoJSON URL to fetch others)

Issues and PRs welcome.

## About

Built by [Ark Arjun](https://arkives.in) for [Geominds](https://geominds.in), released as open
source under the MIT license. The Schemify logo is also designed by
[Ark Arjun](https://arkives.in). See the in-app **About** panel (footer link) for the same
information plus data attribution.

Label fonts — Sans-serif, Serif, Script, and Display — are Manjari, Rachana, Chilanka, and
Keraleeyam by [Swathanthra Malayalam Computing (SMC)](https://smc.org.in/); Monospace is
[JetBrains Mono](https://www.jetbrains.com/lp/mono/) by JetBrains. All are SIL Open Font License.

## License

MIT — see [LICENSE](./LICENSE). © Geominds, with contributions from Ark Arjun.
