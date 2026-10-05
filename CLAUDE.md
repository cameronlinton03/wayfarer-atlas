# Wayfarer Atlas

A free fantasy map maker in the style of 16th–17th century engraved maps (ink on parchment, faint hand-tinted washes). One self-contained HTML file (`wayfarer-atlas.html`), no build step, no dependencies beyond Google Fonts. Owner: Cameron (D&D DM). Desktop first; phone (`#phone`, R=.5) is rough outlining only. A map is a world/region/city map or a dungeon sheet (`S.kind`).

Open `wayfarer-atlas.html` in a browser. Maps live in IndexedDB (localStorage fallback); "Save map file" writes JSON. `index.html` forwards to it for GitHub Pages.

## Owner's rules

- Engraved style only: serious, gritty linework. No colour themes, nothing cartoonish, nothing with a face.
- Light from the left: shading on the right of raised things, on the LEFT inside hollows.
- Mirrored stamps are redrawn mirrored (shading still on the right), never flipped as a picture: art uses `EX(x)` / `ENG_M`.
- Washes faint and desaturated; ink marks sparse. Don't let it get busy.
- He wants honest pushback and weak results flagged, not hidden.

## Working conventions

- Keep it one file; make targeted edits (lines are long, match strings exactly).
- After a change, load the page headless (Chromium at `/opt/pw-browsers/chromium`), check for page errors, exercise the tool. Smoke test: blank map → land, sea, cliff, paint, river, road, stamp, label → move a landmass → undo/redo → `serialize()` round trip → export at 2×.
- Look at rendered output, whole map and close up, before calling art done. For performance work, pixel-compare against the previous build: nothing may change the picture.
- Never end a line with `// comment` followed by more code: the comment swallows it (this has broken things several times). Check: flag any `//` followed later on the line by `; const|let|if(|name(`.
- Stamp kind names are global across the `SKETCH_DRAW` blocks; a later duplicate silently replaces earlier art.

## Map of the code

State and history
- `S` holds lists (`lands, cliffs, paints, lines, stamps, labels, elev, fpaint`) plus settings (`grid, frame, rhumb, scale, layers, tiers, relief, gen, height, dun`). Lists are immutable (replace on edit), so undo snapshots (`snap`) share arrays. `pushUndo(label)`, `undo/redo/goTo`, cap 60. `loadState` + `migrate` read old files; `serialize` writes.
- `S.hm` (height view) is not undone. `restore` rebuilds the land only if what shapes it changed (`sameLand`), else `restoreNeeds` → full compose, territories only, or nothing.

Rendering
- Render scale `R`; map-sized buffers in `initBuffers()`, drawn in map units. `invalidate()` runs one rAF: `need.land` → `rebuildLand()` (land mask with rivers carved out, except under mountain bodies: `riverMask`, a mask of the rivers less each mountain's own drawing (its sprite, filled down to its foot), so a river passes behind a mountain and overlapping mountains count once; river scraps under ~350 square units left between mountains are dropped; rivers are tinted in the base, only where they are open, not in the overall tint), `need.compose` ('fast'|'full') → `compose()`, `need.realms` → `composeRealmsOnly()`; then `draw()`.
- `compose` → `composeMap`: paper, sea, water paint, coast lining, land with ground paint (`cleanC`), woodland edge (`ecoC`), relief (`reliefC`), territories, cliffs. Dungeons: `composeDungeon` (a room clears corridor flagstones under it; corridor textures are cut to outside rooms); height view: `composeHeight` / `composePlates`.
- `drawVectors(c,k,vp)` draws lines, stamps, labels per layer every frame (only what is in view), then multiplies `tintC` (all washes, `buildTint`).
- Kept quick by: caches keyed on identity/versions (`landVer`, `washVer`, `oid`, `mtnSig`, `readAlpha`); a dirty box while painting ground/water/elevation (`liveBox`, `liveClip`, `rebuildRelief(rect)`, needs `baseIsMap`); stroke-sized scratch canvases (`scr`, `SB`) because drawing onto a canvas that was drawn from copies all of it; territory pictures cached per territory and relaid only where changed (`realmCache`, `realmLayers`, `realmDirty`, `prebaseC`). Sprites are capped (512 px, 1024 export), LRU-limited (`SPRITE_PIXELS`) and built under a per-frame budget (`spriteBudgetMs`). Export budget: `maxScale`, `bufCount()`, `releaseBuffers`.

Terrain and land
- Ground types `TERRAIN` (Grass, Grasses, Flowers, Stone, Sand, Cold) are `INK_TILES` recipes: faint wash + small sparse marks (sand: dune arcs, never the sea's wave mark). Water depths read from their lining: shallows stipple, open sea wave marks, deep broken level lines, abyss close lines and a darker wash; icy water angular floes; tropical, lagoon, cold and murky water each have their own wash and mark (`w_tropic`, `w_lagoon`, `w_cold`, `murk`), told apart at whole-map size. `paintInto`/`strokeAlpha` lay strokes; `syncTerrain`/`syncWater` replay them; `cleanInk` removes marks cut by coasts, banks and cliff edges (partial when only strokes were added/removed). Woodland edge (`rebuildEco`): shrubs/saplings where open ground (`isOpenGround`) meets forest, on a hashed lattice.
- Relief (`rebuildRelief`): one-direction diagonal hatching (top left to bottom right), on slopes facing away from the light past a middling steepness (slope > .0045, full at .011) and fairly steep sea cliffs, from the relief ground `er` in `heightGrid()` (generator heights where still valid, Elevation brush strokes `S.elev`), plus sea cliff bands and tapered-cliff ramps. `S.relief` sets strength. `heightGrid` also feeds the height view (estimate from the coast where nothing is stored, mountains via `featureBumps`, cliffs via `cliffRelief`).
- Cliffs (`rebuildCliffs`): plateaus (h>0; cut to the land they stand on, they never extend it; the outline is the foot, the top is lifted by h in overlapping slices on a taper; shadow: level lines thinning outward, three sweep masks, kept to land; ramp tops get only faint broken dashes), pits (h<0, rim on its own layer, drawn crisp; tone per pixel from depth: shallow floors stay clean, a floor fades in from its edge, a deep pit's walls darken toward the foot of the wall as seen, reaching black from about depth 180, into a black floor with no edge drawn), taper/dir ramps (`fadeLow`, `footLine`; face slivers under ~4 units across and specks under ~2.5 tall are dropped). By water: a river does not cut a plateau, it runs under it (`Lm` = land plus rivers) and comes out at the foot; the top, where it reaches out over the sea or a river, is drawn as plain land over the water, lining and coastline (`F`), takes the land wash and loses the sea tint (`cliffOver`, `cliffVer`, used by `buildTint`); a thread of ground between face and water joins the face (`K2`); scraps between the top and the water, pockets, nicks and seams are made top by a morphological closing near the shore (`closeGaps`, distance fields; real face `thick` and real water `sea` are left alone, small pieces of either count as scraps). A hollow whose floor reaches the water is flooded: the floor is drawn as ruled water and takes the sea tint (`cliffFlood`). Masks are soft-edged: joining two with a seam between them leaves a ghost line, so seams are stroked over or made solid. `faceHidden` hides stamps on faces; `ridersOn` carries stamps, paint, lines, elevation on moved land/plateaus.
- Shore: `rebuildCoast` (lining band, inner shading; none inside rivers; water too narrow to hold the band, a strait or a small lake, gets a thin ring instead). Rhumb lines are cut out of lakes and carved inlets (`mode:'sea'` shapes).

Lines
- `LINES`: river, road, border (no tool any more), realm, street, wall, fence, plus dungeon `dwall`. Roads are cased lines that merge (`drawRoads`), trails dotted (`drawTrails`), streets two-pass (`drawStreets`).
- Territories (`realm`, tool key K): loops or brush areas, `fill` wash/edge/hatch/dots/line, `col` from `REALM_COLS`, `op`. A territory = lines sharing `gid`; tiers `S.tiers` (`DEFAULT_TIERS`): lower tiers stack inside higher ones, same tier carves. Pieces (`pieceKey`) move independently; strokes over a piece merge (`mergeStroke`/`retrace` → ring lines); the eraser retraces pieces and is not stored. Rendering: `renderTerritory` (distance fields `edt2`, `edgeDist`), `TIER_BORDER` dashes.

Stamps
- `STAMP_SETS` → `STAMP_GROUPS` → `SKETCH_DRAW[kind](c,n)` in a 40-unit box, ground at y≈14. Helpers `engInk, engScratch, engShade` (`fade:true` with `clipX`: a shaded side that thins away, every stroke at the edge, every second further in, every fourth furthest; used by `dBox`/`dRing`, never a hard block of hatching), `engPoly, engForm, engLimb, engHump, engTree, engHouse, engTower, cRoof, cSpire, cRound`; hatch spacing divides by `ENG_K`. `paintStamp`/`sprite()` cache; `TINTS`/`tintSprite` for ground tints; `s.op` opacity.
- Mountain-like kinds need entries in `MTN_BODY` and `BASE_FADE` (foot fade); trees inside a mountain's drawing are hidden (`coverHidden`). Oversized art (`highpeak`) needs `SPR_SIDE`.
- Placement: next stamp decided ahead (`pend`, `pending()`), ghost = what lands; Placement bar (size, rotation, mirror, shape, opacity). Scatter (`scatterDab`), search and favourites (`chooseStamp`, `favs`).
- Dungeon set: Doors, Stairs & pits, Furnishings, Hazards, monsters (top-down, no faces: `mMan`, `mHead`, `engForm`), characters `ch_*`. `DUN_FOOT`, `dungSnap`.

Generators
- `generateMap(W,H,kind,G)`: tectonic plates (`mulberry(seed+1001)`, never `rnd`, so seeds repeat), coastline detail (`G.coast`), lakes and rivers (priority flood; continents keep only their 5 biggest, deepest basins of 12+ cells), mountains on collisions (`highpeak` at the thickest part of a range, only on a landmass of 4+ peak-areas with its whole width on land, shrinking from 150-200 to 125 or 105 to fit, no river or road across its foot; snow only on its caps, open hatching `sp:1.6`), range name placed near the range's middle, wholly on land and clear of other names, ground drifts (`groundAt`), cliffs and hollows (`G.cliffs`, `mulberry(seed+1501)`; ravines are long narrow clefts with ragged walls and kinks, opened along rift/fault boundaries `bnd`/`bang` where there are any), rivers ending in water are carried on until they reach the drawn shore, tributaries snap onto the river they join, any river still ending on dry land is dropped, routes by Dijkstra (`route`, Float64 costs, `mulberry(seed+1601)`). Stores `S.height`, plates, `gen` for Reroll (`reroll`, `generateFrom`, `genRef`).
- Generated world maps also get a ruled frame, a scale and a compass in the emptiest corner; icy-water floes are small flat slabs; peaks need dry feet and keep off roads; lake basins lose their pointed tips (erosion of the basin mask); rivers fade in from a thread at the source. Cities: roads, streets and the river are cut at the neatline and short of the title plate (`ok`/`edge`), district names are placed by least collision, no Fishgate without walls.
- `generateCity` (cathedral reserves room along its whole length, clear of the river and shore, `coastY` gentle under the walls with bays and headlands further off, `housesAlong`, fields/orchards/vineyards in `VARIANTS`), `generateDungeon` (rooms, spanning-tree corridors, doors, themed props, monsters, room numbers clear of props).

UI
- `TOOLS` (`m:'w'|'d'|'x'`), `etool()` says which job a multi-job tool is doing (Terrain: ground/water/cliff/elev; Wall: fences; Street: road/trail). `renderPanel()`, `slider()`/`bind()` (typeable values, arrows, wheel; one undo per run), `syncHint()`, `openKeys()` (`?`).
- Selection: `hitTest`, `boxSelect`, `makeSel` (Shift adds), `locked`, `alignSel`, transform handles (`xfGrab`).
- Labels: `drawLabel` with paper halo; `l.effects` overrides `LABEL_FX`; `LABEL_STYLES`.
- Library: `store`, `lib`, `saveNow`, `openMap`, `addMap`; untouched blank/example maps are not kept (`untouched`, `throwaway`, `leaveMap`). Maps dialog: search, sort, rename.

## Known gaps

- Monsters are darker with ragged fur (`mBody` defaults, `engForm` `fur`); the humanoid build (`mMan`) now has a square-shouldered back with the blades marked, arms with an elbow and wrist strap, and small fists (`mHand`), but pack, cloak and heads are still simple shapes: better, not yet a real redraw.
- Older reports: roads straight over long runs, city river mouth has a pale fan and no bridges, grid doubles flagstone joints, example map lacks border/compass/cartouche, paddock animals and hedges crude, river sources blunt and tributary banks cross the main river, streets across walls make no gate, hand-drawn walls wobble.
- Dungeons: no lighting, per-room colour, level links or room rotation; generator gives round/cave rooms no doors. A dungeon drag is ~350 ms on software rendering (a dirty box would not match: hatch jitter runs along whole lines).
- Battle maps undecided. Dragon, sea serpent and griffin still have eyes (the giant lost its face): Cameron to say whether animals may keep them. City bridges are just the street covering the river (its outline serves as parapets).
