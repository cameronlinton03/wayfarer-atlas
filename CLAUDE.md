# Wayfarer Atlas

A free, self-owned fantasy map maker in the style of 16th–17th century engraved maps (ink on parchment, light hand-tinted washes). One self-contained HTML file, no build step, no dependencies beyond Google Fonts. Owner: Cameron (D&D DM). Desktop is the main target; phone is a rough-outlining mode only. A map is either a world/region/city map or a dungeon sheet (`S.kind`), in the manner of Dungeon Scrawl.

## Running it

Open `wayfarer-atlas.html` in a browser. That is the whole app. Maps are stored in the browser (IndexedDB, falling back to localStorage); "Save map file" writes JSON.

## Hard rules from the owner

- Engraved style only. No colour themes, no cartoonish art. Nothing with a "face" or comic-book shape. Serious, gritty linework.
- Light comes from the left: shading goes on the right of raised things, and on the LEFT inside hollows (craters, pits).
- Mirrored stamps must be redrawn mirrored with shading still on the right, never flipped as a picture.
- Colour washes stay faint and desaturated; ink marks stay sparse. Do not let it get busy.
- He values honest pushback and wants weak results flagged, not hidden.

## Architecture (all in the one file)

- State `S = {w,h,name,lands,cliffs,paints,lines,stamps,labels,grid,frame,rhumb,scale,layers,active}`. Lists are immutable (replace on edit) so undo snapshots share arrays. `snap()`, `pushUndo()`, `loadState()`, `migrate()`, `serialize()`.
- Render scale `R` (1 desktop, .5 phone, higher for export). Map-sized buffers are created in `initBuffers()`; draw in map units.
- `rebuildLand()` builds the land mask (rivers are carved out of it). `compose(full)` builds the base picture: paper, sea, coast lining, land, realms, cliffs. `drawVectors(c,k)` draws lines, stamps, labels per layer, then multiplies `tintC` (all colour washes) over everything.
- Terrain: ink marks are tiles from `INK_TILES` painted into `terrainC`; washes go to small blurred layers `washLand` / `washSea`; `buildTint()` combines them. `cleanInk()` removes marks cut by coasts, rivers and cliff edges.
- Stamps: `STAMP_SETS` → `STAMP_GROUPS` → `SKETCH_DRAW[kind](c,n)` drawn in a 40-unit box, ground at y≈14. Helpers: `engInk`, `engScratch`, `engShade`, `engPoly`, `engForm`, `engLimb`, `engHump`, `engTree`, `engHouse`, `engTower`, plan-view `cRoof`, `cSpire`, `cRound`. Use `EX(x)` for x positions so mirroring works (`ENG_M`). `paintStamp` scales pen weight (`penOf`) and hatch density (`ENG_K`) with stamp size. Sprites are cached in `sprites`.
- Hand-placed hatching in building stamps (`engHouse`, `engTower`, `cRoof`, cottage) divides its spacing by `ENG_K` so bigger stamps get more strokes; keep doing this for any new hatch loop. `engFurrow` lays ploughed rows across a field polygon.
- Fields: `fieldplot` (8 variants), `orchard` and `vineyard` (3 each) are listed in `VARIANTS`; the city generator mixes them in the outskirts.
- Roads: `drawRoads` / `roadNet` cut roads at crossings and at ends that land within 12 units of another road, fit the dashes per piece so every arm starts with a dash at the junction, and add a small blot. Drawing only; the saved lines are untouched.
- Dungeons (`S.kind==='dungeon'`, `S.dun={hatch,shadow}`): floors are ordinary `S.lands` entries flagged `floor:1` (`sharp:true` = straight-walled room that keeps its corners, see `sharpPoly`; `sharp+brush` = flat-ended corridor ribbon, see `sharpCorridor`; `mode:'sea'` digs a hole; `tex` = flag/plank/dirt/none). Inner walls are lines of kind `dwall`. `compose()` hands over to `composeDungeon(full)`: floor mask in `dgF`, a distance field over the rock (`distField`) drives the hatching (`dgHatch`, thick at the wall, thinning out), the wall is the floor silhouette nudged round in ink, textures go in `dgT` (`dgTexture`), then the grid (clipped to floors) and a band of vertical shading down the LEFT inside of each wall. `full` is false while dragging, which skips the wide hatching and textures. Tools `room`, `corridor`, `dwall` are tagged `m:'d'` in `TOOLS` (world-only tools `m:'w'`); `buildRail()` filters by map kind. Polygon rooms, corridors and walls are click-by-click drafts (`draft`, `draftClick`, `draftEnd`); `gsnap` snaps to the grid (a one-square corridor runs down the middle of its squares, see `corrOff`).
- Dungeon stamps: set `dungeon`, groups Doors / Stairs & pits / Furnishings / Hazards & features, art at the end of `SKETCH_DRAW` (`dBox`, `dRing`, `dPost`). `DUN_FOOT` gives each kind's footprint in grid squares (default size = footprint x cell). `dungSnap` puts doors on the middle of a square's edge and turns them to cross the passage (it looks at the floor either side), and everything else on half-squares. `generateDungeon()` makes example dungeons (rooms, spanning-tree corridors, doors, themed rooms, numbered labels).
- Cliffs: `rebuildCliffs()` (plateaus h>0, pits h<0, taper, facets, cast shadows).
- Lines: `LINES` kinds river, road, border, realm, street, wall, fence. Streets are drawn in two passes so junctions merge (`drawStreets`).
- Generators: `generateMap(W,H,kind,G)` (island/continent/archipelago) and `generateCity(W,H,G)`; `housesAlong()` lines buildings along streets.
- Library: `store`, `lib`, `saveNow()`, `openMap()`, `addMap()`.
- Tools are listed in `TOOLS`; the side panel is built by `renderPanel()`.

## Working conventions

- Keep it a single file. Make targeted edits; many lines are long, so match exact strings carefully.
- After any change, load the page in a headless browser and check for page errors, then exercise the affected tool. A good smoke test: new blank map → draw land, sea, cliff, paint, river, road, stamp, label → move a landmass → undo/redo → `serialize()` round trip → export at 2×.
- Look at rendered output before calling art finished; judge it at whole-map zoom and close up.

## Known gaps

- Battle maps are undecided. Dungeons have no lighting or per-room colour, no stairs linking levels, and no way to rotate a room; round and cave rooms get no doors in the generator.
- A full dungeon compose is ~0.5 s on a software-rendered browser (hatching and textures dominate); dragging uses a cheaper compose.
- The `bones` and `pool` dungeon stamps are the roughest art.
- City district names are not placed to avoid streets.
- Brush hardness no longer affects washes (they are always soft).
