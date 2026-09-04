# Nagare, the drifting market

`nagare.html` is the whole thing. One file, no build step, no asset folder. Double
click it and it runs on `file://`.

A floating Japanese night market rendered in real time with three.js, built as a
scroll driven landing page. You start standing in the middle of the market at
golden hour. Scrolling flies the camera through seven chapters (the gate, the
boardwalk, the lantern tower, the teahouse, the crossing, and a wide closing shot)
while the clock runs from dusk into deep night. Drag at any point to look around,
or take the dial bottom left and drive the sun yourself.

## Why it is built this way

**On the deprecation warning.** three.js logs that `build/three.min.js` is legacy.
That build is deliberate, not drift: `file://` refuses ES module scripts, and this
file is meant to be opened by double-clicking. Migrating to modules would break
that, so it is a trade to decide on rather than routine cleanup. r160 is the last
release shipping a UMD build.

three.js is loaded as a **classic script** from jsDelivr:

```
https://cdn.jsdelivr.net/npm/three@0.160.1/build/three.min.js
```

r160 is the last release that still ships a UMD build. That matters because a
`type="module"` script would refuse to run from `file://`, and the point of this
file is that you can double click it. If three.js ever fails to load, the page
shows a short message instead of a blank screen.

Everything in the scene is **procedural geometry**. There are no models, no
textures, no photographs. That is what keeps it genuinely single file. Every
building, boat, lantern and tree is generated from boxes, cylinders and lathes at
load time, then merged down to about twenty draw calls and 230k triangles.

## Materials without textures

Because there are no image files, surface detail is done in the shader. Every
merged vertex carries a **surface id** (`S_WOOD`, `S_PLANK`, `S_TILE`, `S_STONE`,
`S_ROCK`, `S_PLASTER`, `S_FABRIC`, `S_METAL`, `S_LEAF`, `S_BAMBOO`, `S_PAINT`,
`S_HULL`), written by `push()` from whatever `SURF` is current. `applySurfaceShader`
then injects a block into MeshStandardMaterial that gives each class its own
grain, roughness and normal relief.

Each merged vertex also carries **`aSeed`, `aAxis` and `aAlong`**, written in
`mergeBucket()` where positions are already in world space. `aSeed` gives every
piece its own stable tone, roughness and weathering. `aAxis` is that piece's own
long axis and `aAlong` is 0..1 along it, which is what lets wood grain run down a
plank and darken at the cut ends. Working purely in world space, as the first
version did, could not tell a standing post from a plank lying flat.

Three parts are worth knowing about:

**Tile courses follow the roof, not the world.** The shader builds a frame from
the surface normal (down-slope, and across it) and repeats the kawara ribs along
the across vector. That means courses run correctly on any roof at any rotation
without needing UVs.

**Everything fine fades with distance.** `det` and `det2` are distance ramps, and
every high frequency term is multiplied by one of them. Without that the grain
and the tile ribs alias into noise as soon as anything is more than a few metres
away. If detail looks too strong or too weak, those two ramps are the dial.

**Cavity darkening does more than fine noise.** Downward-facing surfaces lose up
to 40% at the end of the surface block. That is what puts shadow under eaves,
deck soffits and beam intersections, and it contributes more to solidity than any
amount of extra high-frequency detail.

Rock also gets a wet band at the waterline and lichen on dry upward faces, which
is what stops the islets reading as grey blobs.

## The lantern tower

Built as foundation -> structural frame -> lantern chamber -> beacon and finial,
not three copies of one lattice. The tiers do different jobs: the lower is open
and braced, the middle encloses a shoji lantern room holding the main light, the
upper is a lighter beacon. Corner posts thin as it rises (0.14 / 0.12 / 0.10) and
the roofs taper too (+2.30 / +1.80 / +1.35), so the top does not read as heavy.
It stands on its own platform lifted 0.28 above the market deck with a damp skirt,
stone footings and entry steps, which is what gives it a readable foundation.

It no longer carries a position of its own: it sits at `marketPoint(2.0, -1.6)`
on the square's own grid, turned to `MARKET_RY`, which is what squares it to the
dock and keeps a walkable ring around it (1.75m at the tightest side). Two things
to preserve if you move it: the square's lantern chains are routed around it
rather than through it, and the chapter 3 camera target is offset off the tower
so it does not sit dead centre. Moving the teahouse once put its roof straight
across the tower's base in that shot, so re-render chapter 3 after any change on
that side of the market.

## Where the sun and moon actually show up

The camera always looks down at the diorama, so the sky above the horizon is never
in frame. A disc drawn in the sky is therefore never seen directly. What you see is
its **reflection**, and that lands at `cameraHeight / tan(elevation)` away along the
light's azimuth.

Two consequences worth knowing before touching `elev` or `azim` in `KEYS`:

- The azimuth has to point roughly where the cameras look (about 130 to 160 degrees
  for most chapters) or the reflection falls off the side of frame entirely.
- The elevation sets how far out it lands. Too high and it lands on the market and
  is hidden behind it; too low and it lands hundreds of metres out and fogs away.
  The night moon sits at 21 degrees, which puts it on open water just past the
  market.

So it is a genuine trade: a low moon gives a disc in the sky but no light path, a
higher one gives the path but puts the disc above the frame. The current setting
takes the path.

## Making buildings differ

`withPalette(PAL, fn)` swaps the timber and roof colours for one building while it
is built from the shared kit. `PAL_TEAHOUSE` is cedar and warm timber, `PAL_SHRINE`
is vermilion posts under weathered copper, `PAL_KURA` is a near-black roof over
white plaster, and `STALL_PALS` cycles three roof tones through the stall rows so
they are not all the same. `BOAT_PALS` does the same for hulls.

## The scroll journey

Two lists, deliberately separate. `CAMERA_KEYS` holds the camera beats -- there
are more of them than there are chapters, because a chapter can contain an
approach, a reveal and a hold. `CHAPTERS` holds the copy: a panel, a scrim side
and a rail tick each. `CHAP_P` and `CHAP_S` tie the two together, and the long
version of how is under **The camera journey** below.

Scroll position is damped once, on the way in. Nothing eases per segment: that
was the old stutter.

Dragging adds an offset on top of the chapter camera rather than replacing it,
and that offset eases back toward zero while the page is moving, so the
composition reasserts itself without fighting you mid gesture.

## Typography

**Display and Japanese: Hina Mincho. Interface: Zen Kaku Gothic New.** Both come
from Google Fonts, both carry Latin and Japanese, and both fall back to a real
stack when there is no network.

Hina Mincho is a single 400 weight and it is deliberately hairline, so every rule
that uses `var(--serif)` asks for 400 explicitly. Hand a browser `font-weight:500`
for a family that has no bold and it synthesises one, which on a face this
delicate smears the thin strokes into the thick ones. If you add a serif rule,
set the weight to 400.

The one thing to watch is contrast. Hairlines over lantern light on black water
are the failure mode this face has, and it is why the display text keeps its
`text-shadow`. It reads well down to about 20px; below that the interface sans
does the work.

The page uses **22 non-Latin glyphs** in total — とのりれ・塔居屋市月橋水流渡灯社籠茶道門霧鳥.
Google's `&text=` parameter returns a font containing only the glyphs you name,
which brings those 22 back as **7.4 KB** for Hina Mincho and **4.1 KB** for Zen
Kaku Gothic New: small enough to inline as a data URI in the standalone build, so
the kanji survive on `file://` with no connection. Check coverage before swapping
a face — a missing kanji falls back silently and you will not notice it in a
Latin-only test.

A small harness that renders the page with three.js dropped on the floor is
worth keeping. Typography is a CSS question, and answering it should not cost
four minutes of SwiftShader.

## The interface

**The next stop.** One button at the bottom centre, not one per panel: it always
offers the next stop on the journey and takes its label from wherever the camera
is settled -- *Walk the market*, *Cross the old water*, *Reach the tower*, *Find
the teahouse*, *Plan your visit*. It fades out while you are between chapters, so
it never sits there pointing at a stop you have already left, and it reuses the
same `data-scroll-chapter` smooth-scroll the hero's "Cross the gate" uses, so
buttons and rail land on identical anchors. Labels live in `NEXT_STOP`, indexed
by chapter; `null` means no button, which is why the hero (its button is in the
copy) and the last chapter have none.



The controls used to sit open in the bottom-left corner, which is exactly where
the hero copy and the market both want to be, with a hint line across the bottom
centre through everything else. Both are now one pill on the same header line as
the wordmark and the nav: it carries the clock, the dial and tools drop below it
on hover, on focus or on tap, and the hint sits under it and fades as soon as
you start scrolling.

The pill's panel is positioned out of flow on purpose. In flow it left a tall
invisible box over the middle of the scene. `:hover` still reaches an ancestor
from a descendant wherever that descendant is painted, so the pointer can travel
from pill to panel without the menu closing under it -- and the bridge between
them is a `::after` with no size at all until something is already hovering the
dock, because a permanent invisible box at the top centre of the screen would
eat camera drags.

The dial and the scroll share the clock. By default time follows the scroll; the
link button shows that. Touching the dial or pressing play hands the clock to
you, and the link button gives it back.

## What is in there

- Hand written orbit controls, layered on top of the scroll camera
- A planar water reflection (the scene renders twice, mirrored, with a clip plane)
- Animated wave normals, fresnel mixing, sun sparkle and expanding ripple rings
- A hand written post chain: bright pass, dual filter bloom, tilt shift band,
  chromatic aberration, ACES tonemap, vignette, film grain
- Nine keyframed sky states interpolated continuously, driving sun angle, fog,
  water colour, lantern brightness, bloom and exposure together
- A shadow casting sun, paper lantern point lights, drifting lanterns, embers,
  low mist, lily pads and flowers
- Two boat types: a round hulled sampan and a flat bottomed cargo punt
- Stroke-built Japanese lettering on the banners and the shrine's name-board

## Changing things

**The look of a time of day.** Everything lives in the `KEYS` array. Each entry is
one hour of the clock with a full palette. Add or move entries and the whole scene
follows. There is a guard that throws if a keyframe is missing a field, because a
missing colour silently reads as white and it is horrible to track down.

**The layout of the settlement.** `buildWorld()` reads as a route: torii,
approach, Gate Landing, bazaar, end landing, bridge, tower square, teahouse, with
the shrine and the kura leaving as square side branches. It opens with a block of
market-local constants and derives everything from them. Everything notable goes
through `place({label, kind, x, z, ry, w, d}, fn)`, which records a footprint into
`PLAN` as well as positioning the thing. The kit it draws from (`roof`, `shoji`,
`shojiWall`, `slidingDoor`, `deck`, `chamferedDeck`, `chochin`, `torii`, `boat`,
`punt`, `bridge`, `flatBridge`, `stall`, `teahouse`, `bazaar`, `pottedPlant`, and
so on) is all above it.

### Two layout frames

The settlement used to be a set of independently rotated pieces, and from above
it read as a scatter rather than a plan. There are now exactly two grids.

**The market frame** is the settlement. Everything past the gate takes
`MARKET_RY`, `MARKET_RY + PI/2`, or one of those reversed by `PI`, and takes its
position from `marketPoint()`:

```js
var MARKET_X = 7.8, MARKET_Z = -6.4, MARKET_RY = 0.10;
var STREET_RY = MARKET_RY + PI/2;      // the bazaar runs across the grid
marketPoint(localX, localZ)            // a point in the square's own coordinates
marketVector(localX, localZ)           // a direction in it
edgePoint(platform, edge, t, offset)   // a point on one edge of a placed rectangle
bridgeBetween(a, b, width)             // a bridge whose ends land exactly on a and b
```

Nothing in the settlement carries a hand-tuned angle. `buildWorld()` opens with a
block of market-local constants and derives every platform, bridge and landing
from them, so moving `MARKET_X/Z/RY` moves the whole thing as one piece. The
entire settlement block contains no world coordinates at all.

**The approach frame** exists only so the torii can keep its ceremonial
diagonal. `APRY` is derived as `STREET_RY - APPROACH_SKEW + PI`, so even the
diagonal is expressed against the market grid rather than typed in.

`APPROACH_SKEW` is **exactly `PI/4`, and it has to be.** The chamfer's own
normal is at 45 degrees, so any other angle lands the approach off-centre on
that face. At 0.55 rad its centreline met the chamfer 1.5m from the midpoint and
1.2m of the deck's width hung past the end of the face into open water. From
above it looked plausible; from the deck it read as two decks bumped together.
`validate.py` now checks the approach's full width against the face's extent,
not just that its axis passes through the octagon's centre.

The two grids meet at exactly one place, and that is the point of the Gate
Landing. The previous version resolved the change of grid by overlapping several
differently rotated rectangles, which is what produced the fan-shaped junctions
and the hidden double floors.

### The Gate Landing

An octagon, built by `chamferedDeck(w, d, cornerCut, opt)` as **one** extruded
top surface with plank-seam strips laid over it, perimeter beams on all eight
edges and piles at the vertices. Not a stack of rotated `deck()` calls: a single
slab cannot produce an accidental triangular sliver or a second floor at a
slightly different height.

Three routes cross it, and the furniture is placed against them rather than at
the corners:

- the approach arrives on the `-x/-z` chamfer at 31.5 degrees
- the bazaar leaves through `+x`
- the kura bridge leaves through `+z`

Nothing sits within 1.3m of any of those centre lines. The approach's overlap
onto the landing is computed from the octagon's actual boundary along the
diagonal, so it stays at 0.15m whatever `APPROACH_SKEW` is set to.

**Its size is not arbitrary.** Every route has to meet a face wide enough to
take it, or the incoming deck's corners hang over open water in a thin wedge,
which looks exactly like the fan junction this replaced. So:

```
flat faces    = GL_W - 2*GL_CUT = 6.0m   for the 5.8m bazaar and the kura bridge
chamfer faces = GL_CUT * sqrt(2) = 3.96m for the 3.8m approach
```

That is what fixes `GL_W` at 11.6 and `GL_CUT` at 2.8. The first version was
8.0 by 2.0, whose 4.0m flat face was narrower than the street leaving through
it. If you change the street width or the approach width, re-derive these two
numbers before anything else.

### The bazaar

`bazaar(spec)` lays the street: one long deck on `STREET_RY`, stalls set back on
both sides, and the lantern runs. Two things about it are deliberate.

**Positions are metres from the gate end, not fractions.** A fraction silently
changes what it means the moment the street's length is edited, and a stall
creeping into a bridge mouth is exactly the failure this layout exists to make
impossible.

**The clear spine is a constraint, not an outcome.** Stalls sit back far enough
to leave it, the passing bay and the shrine junction are chosen gaps rather than
leftovers, and the lantern poles stand only in their own side's gaps at the very
edge of the deck. Only the two end chains cross the street, at the one place
where both sides happen to be free. A pole pitched in the middle of a walking
corridor is the classic way to make a market impassable.

### The arrival sequence

The street deliberately does not touch the tower square. It ends, and the walk
crosses open water:

```
bazaar  ->  widened end landing  ->  flat bridge  ->  tower-square landing
```

The bridge is `flatBridge`, not the decorative arch: this is the primary
pedestrian route and an arch here would read as a garden feature. It is aligned
to the grid, perpendicular to the square's edge, centred on its landing, and
lands on the tower's centre line so the tower is straight ahead as you cross.

The shrine leaves the bazaar the same way, as a square T-junction: a widened side
landing, then a bridge perpendicular to the street, centred on the pavilion
entrance. The kura leaves the Gate Landing on its own side. No branch shares a
corner with another.

### The camera journey

`CAMERA_KEYS` is nineteen keys built from the same named nav nodes that build
the world, so the route moves when the settlement does. How it is sampled
matters more than the keys themselves.

**It is a spline, not a polyline.** Position and target run through Catmull-Rom,
so the path curves through its keys instead of turning corners at them.

**There is no per-segment easing.** An earlier version ran a smootherstep across
every segment, which eased the camera to a dead stop at every key and launched
it again: sixteen little stop-starts, felt as a stutter rather than a journey.
The segment parameter is linear, and the only easing left is the scroll damping,
which is what should own the feel of starting and stopping.

**Scroll maps to how much the shot changes, not to key index or to distance.**
Keys are spaced by editorial intent, so a constant scroll rate ran the camera
6.4x faster through some segments than others. But distance alone is not the
answer either: a pan covers no ground at all, so distance-based pacing gave the
two places where the camera turns to look at something -- the storehouse, the
shrine -- almost none of the page, and they snapped. `buildArcTable()` charges
rotation at `ARC_TURN_M` metres per radian, currently 15, so a quarter turn
costs about as much page as walking twenty metres.

**And chapter budgets sit on top of that.** One arc-length map across the whole
journey handed a third of the page to the tower orbit purely because an orbit is
long, and left the bazaar a fifth of a screen to be walked down. `CHAP_P` says
where each chapter's shot happens on the curve, `CHAP_S` says how much of the
page that chapter gets, and `arcParam()` maps between them with a Fritsch-Carlson
monotone cubic. Monotone matters: a plain spline can overshoot, and an
overshooting scroll map runs the journey backwards for a moment. Straight lines
between the same control points also work, but they put a kink in the pace at
every chapter boundary, which is the stutter this whole pass exists to remove.

The route, in order: outside the torii at golden hour, through the gate, a stop
held back over the approach deck with the storehouse three-quarter across the
water, then a low walk down the middle of the bazaar looking straight at the
tower, a left turn onto the shrine, and the tower.

The gate stop used to sit out on the landing, square on to the storehouse at
twelve metres. It filled the frame, and the walk in was over almost before it
started. From eighteen metres and off-axis the deck and the kura bridge lead
into it, and the same stretch of scroll covers less ground -- so the arrival
reads as slower without anything about the pacing changing. `pathcheck.py`
checks the aim still sits on the storehouse's own axis, so "three-quarter"
cannot quietly drift into "pointed at nothing".

The market is walked at head height on purpose. It used to run at seven to eight
metres, which put the reader on a crane rather than on the boards: the stalls are
2.6m deep, and their roofs only read as something you walk between if the camera
is actually between them.

The tower is one continuous move rather than three shots: arrive low at 12.6m,
climb to 14.5m front-on, then a 133 degree orbit. The angle only ever increases
and the radius only ever grows -- 12.6m at the foot to 54m at the last frame --
so the camera never doubles back on itself.

The tail used to fly across to the far side of the settlement for a jetty
close-up. That put the camera five metres from the thing it was pointing at, and
crossing your own subject swings the view through 150 degrees in a couple of per
cent of the page: the one real whip left in the journey. The orbit leaves the
camera east of the tower and still travelling east, so the last three shots keep
going that way and look back.

The gaze reverses direction exactly three times, and every one is a deliberate
beat: settling onto the storehouse, turning to the shrine, and turning back to
the tower. There is no way to look at a thing beside the route and then carry on
without turning back, so the goal is not zero reversals but no *accidental*
ones.

The gate stop is pinned to exactly 20:00. The clock runs **18:30 to 09:00 the
next morning**: the whole night passes over
the market. Values run past 24 rather than wrapping to 1, because the pacing
spline needs a monotonic clock; the wrap happens once, in `updateStory`, where
`TIME.hour` is taken modulo 24 for the sky and the dial. `sampleSky` never sees
an hour above 24.

The distribution is deliberately uneven. The arrival keeps its golden hour and
the shrine its blue hour minute for minute, and the dead middle of the night --
which looks the same at 01:00 as at 03:00 -- is crossed quickly under the tower
orbit. The scroll spends its time where the sky is doing something.

**A key's `y` is a request, not a promise.** `updateCamera()` rebuilds the camera
position from `(target, azimuth, elevation, radius)`, so anything that touches
the elevation silently rewrites the height. `EL_MIN` used to clamp the *total*
elevation to +0.06 rad, and a key standing at 2.9m on the bazaar deck looking up
at the tower has a *negative* elevation -- so the clamp put it straight back up
at 6.4m. Every low shot in the journey was being flown at twice its written
height, which is exactly the "too high" the low walk was meant to fix. The floor
now sits below whatever the shot itself asks for, and `pathcheck.py` compares
the rendered height against the authored one so it cannot drift back.

`pathcheck.py` samples the shipped curve 800 times and checks direction
reversals, shot-rate evenness, acceleration spikes, pan rate, how many times the
gaze turns back, clock monotonicity, that the market keys stay under 3.4m, and
that the gate stop actually frames the storehouse. It mirrors the page's pacing
in Python rather than rendering, because a headless render of this scene takes
minutes and spline maths does not need one. If you change `ARC_TURN_M`,
`ARC_BLEND`, `CHAP_P` or `CHAP_S` in the page, change them in `pathcheck.py` too
or it is measuring a different journey.

### Shadows

The shadow camera **follows the camera target**. It used to be a fixed 68m box
on the world origin, which was fine until the settlement grew to about 70m long
and the torii ended up outside it entirely. Samples outside a shadow frustum
clamp to the border rather than missing, and the result was hard diagonal bands
down the gate's legs that looked like a texture problem and were not. If you
move the settlement further again, check `sc.left/right/top/bottom` still covers
what a single view can contain.

### The navigation spine

No character walks it yet, but the layout is built around it. `NAV_PATHS` records
world-space nodes for four routes and `NAV_CLEAR` records their widths: 2.6m for
the primary corridor, 1.8m for the branches. `__nagare.debugNav(true)` draws the
routes and their clearance circles.

This is not decoration. The corridor widths are the reason several stalls sit
where they do, and they are much easier to argue about drawn than in a table.

### Moorings

Boats are positioned against the platform they are tied to, not by world
coordinate, so none of them is left stranded inland when a platform moves:

```js
mooringPoint(platform, edge, along, normalOffset)
placeMooredBoat(platform, edge, spec)   // spec.gap is hull-to-edge
placeOpenWaterBoat(x, z, heading, spec)
```

`moorFleet()` places eleven hulls in five working clusters plus two distant
silhouettes, rather than fourteen evenly scattered objects. `auditLayout()` then
checks hull footprints against every platform, bridge and islet and against each
other, and `__nagare.audit()` returns what it found. Set `DEBUG_LAYOUT = true` to
have it warn on the console at build time.

### Weathering

Everything is procedural and lives in one shader, branched on the surface id.
The rule the whole thing follows: **weathering follows exposure and use**, not
noise for its own sake.

- **Painted timber** (the torii) is old lacquer over cedar: grain along the
  piece, paint worn through on the arrises and in patches, fine scratches, and
  chalking where the sun has hit it. The previous version had a single noise
  term and read as brand new plastic.
- **Planking** gets knots with stain bleeding out of them, scuffing along the
  grain, and polished centres where people actually walk.
- **Roofs** vary tile to tile and course to course, and their moss is gated by
  height: a stall roof at 2.8m sits in the splash and the shade, the tower's
  upper roofs are swept clean. That is also what keeps the tower reading as the
  cleanest, darkest silhouette in the settlement.
- **Metals** are one surface class with three behaviours, taken from the base
  colour so the palette stays the single source of truth: iron stays dark and
  rough, brass takes a tighter highlight, and only the green metals oxidise.
- **Contact darkening** puts a shadow band at the foot of anything standing on a
  deck, and again where anything breaks the water. It is keyed off the deck
  plane rather than real ambient occlusion, which merged geometry cannot do.

### Lettering and signs

There are no textures anywhere in this scene, so a sign is built the way a real
sign is built: strokes, laid on a board. `KANJI` holds each glyph as a list of
brush strokes in a unit square:

```js
[ centre x, centre y, length, angle, thickness ]
```

The angle is the stroke's own direction, so a diagonal is one rotated box rather
than a staircase, and a curved stroke is read as its chord -- which is what a
brush stroke resolves to at the size these are actually seen at. Two glyphs so
far: `ichi` (市, market) on the banners, and `mizu` (水, water) on the shrine's
name-board.

`glyph(name, size, z, col, surf, bucket)` lays one in the current group's xy
plane. `hengaku()` builds the framed name-board that hangs on a shrine or a shop
front -- slab, mitred surround, sunk field, character standing proud of it, two
iron fixings -- and `tatefuda()` builds the small standing notice at the foot of
the steps. The notice carries ruled lines rather than invented characters,
because at that size a real one reads as ruling anyway.

The banner word lives on the cloth, not beside it. The cloth is a hero object
and sways; a character left behind in the merged mesh would slide off its own
banner the moment the wind picked up.

### The noren

Adapted from the Washi Noren source in ThreeUI's Woven Cloth family. That file
is a standalone full-screen scene: its own renderer, camera, shoji backdrop and
lights, on a `MeshPhysicalMaterial` with `transmission: 0.82`. None of that
survives the move. Here it is one prop standing on a deck inside a scene that
already has a sun, fog, and a planar water reflection that renders everything
twice -- transmission would add a **third full scene pass for a single object**.
So the paper is lit rather than physically transmissive: an emissive term keyed
to `SKYSTATE.glow`, standing in for the doorway behind it, which is what lantern
light actually does to a sheet of kozo. It goes out by itself at dawn.

What is kept is the good part: the Verlet sheet with the top row pinned, the
horizontal links cut below the sleeve so the three panels sway as their own
pieces of cloth, and the papermaking treatment -- laid and chain lines, long
kozo fibres, and a torn deckle edge sampled by *both* the alpha mask and the
shading, so the two agree about where the sheet ends. Grid is 27x20 rather than
60x40, for the size it actually occupies on screen.

It hangs in front of the storehouse door, which is where a noren belongs and
which happens to be a lit shoji. It stops short of the threshold the way a real
one does -- you brush through it -- so the lower half of the lit door still
shows beneath. It replaced a nobori, which is a roadside banner on a single
upright and was the wrong object for a doorway.

The panels read 水 / 流 / 市 -- water, flow, market -- one glyph per panel, the
way a noren carries a name, with the wordmark in the middle. They are drawn with
the page's own stroke table rather than `fillText`, so the noren, the shrine
plaque and the seal are the same hand and none of them need a webfont.

**A warning from building it.** The stroke table's 水 was wrong for a long time:
both wings hung off the same point near the top of the stem, which draws 木.
Rendering the stroke-built glyph beside the real typeface at the same size is
the only way to catch that, and it is worth re-checking whenever a glyph is
added.

### Lanterns and the light they spill

`LANTERNS` entries are `[x, y, z, radius, intensity, soft]`, and every one of
them draws an additive billboard as well as feeding the point lights. That sixth
flag matters more than it looks.

A lantern is an object: tight body, hot core, so it reads as a paper shell with
a flame in it. A room's spill is not an object, and given the same profile it
becomes one -- a glowing disc hanging in the air with nothing holding it up.
That is exactly what had happened to the storehouse doorway, the stall
interiors, the teahouse veranda and the shrine: eighteen halos with no lamp
under any of them. Soft halos drop the hot core entirely and fall off slowly,
which is what light on air actually looks like.

The other half of the fix is geometry. `hungChochin(x, y, z, r, h, hue, topY)`
draws the hook and the cord as well as the paper, and the storehouse door now
has a bracket lantern on each side. Nothing in this scene should hang in mid
air. `pathcheck.py` enforces the rule from the other end: no hard halo may be
larger than 0.72m, because at that size it is not a paper lantern any more.

### Secondary motion

The scene should read as inhabited by weather and water, not mechanised. At any
moment most of it is almost still.

The architecture stays in the merged static mesh. A short list of hero objects is
lifted out instead, captured by `dynamic()` and rebuilt as its own small mesh on
the shared materials in `build()`. Geometry is captured **already rotated into
world orientation**, so the surface shader's grain axis stays correct and the
object's own rotation only ever carries the motion deltas.

What moves: five hero boats (bob 2 to 3cm, pitch and roll under a degree, periods
between 8 and 13 seconds), three banners, and three threshold lanterns. Cloth and
lanterns pivot on their support, never about their own centre, so nothing
detaches from what is holding it. `motion(seedIdx, opts)` walks phases and
periods by the golden ratio, because shared periods are what make a scene read as
mechanised.

The rest of the lanterns get brightness variation instead of a moving body: each
takes its rate from its own position, so nothing pulses in step, and the
amplitude is about 2%. The tower's chamber and beacon are held steadier still, by
scaling the amplitude down with halo size. A beacon that reads as unstable undoes
its whole job.

`prefers-reduced-motion` drops all of it to a tenth rather than to nothing.
Freezing the boats dead still next to moving water looks like a bug.

`apPt(t, side)` returns a point on the approach deck's own axis, and the torii,
its lamps and the boat dock are all derived from it. The torii had previously been
positioned by hand and drifted 2.2m off the centreline and 13 degrees out of
alignment, so the boardwalk ran through one side of the arch. Deriving it means
that cannot happen again.

The islets are generated as **clusters** with uneven angular steps and the odd
skipped sector. Evenly spaced placement, which is what they had first, reads as a
circle drawn around the market rather than as scattered land.

**The plan.** `__nagare.plan()` returns `PLAN`, so a labelled top-down map can be
drawn from exactly the numbers the scene is built from. That is much easier to
argue about than a perspective view when you want to move things.

Rules worth keeping when you move things around. `auditLayout()` now enforces
the first of them automatically, but the rest are still on you:

- **No boat should overlap a deck footprint**, or it disappears under the
  planking. `auditLayout()` tests exactly this, and `__nagare.audit()` returns
  what it found. Remember that the kit lays some decks larger than the recorded
  footprint (a stall adds 0.7, the teahouse 2.6 by 2.2, the kura 1.4 by 1.2);
  `DECK_GROW` holds those margins, so add to it if you add another such kit part.
- **A moored boat's edge is not its hull.** An oar reaches about 0.3m past the
  beam, which is more than the gap to the deck it is tied to, so every moored
  hull passes `oar:false`. Only the two distant open-water boats keep theirs out.
- **A pier's long edges run back into whatever it grows out of.** Both approach
  dock boats were originally moored on the same edge at a low `along`, which put
  the near one flat on top of the approach deck. The audit caught it; from above
  it was invisible.
- **A market lamp shorter than about `h = 2.7` buries its lantern in a roof** if
  it happens to stand near a stall. The lantern hangs at `baseY + h - 0.26` and
  stall roofs ridge out around 2.8.
- **Nothing should stand across the middle of a bridge, and nothing should narrow
  its mouth.** These are different failures. A stall can miss a bridge deck
  entirely and still leave only a metre to walk onto it, so measure the clear
  corridor across the landing, not whether the footprints touch. On the bazaar
  the stall positions are chosen against the junction gaps for exactly this.

**The journey.** `CHAPTERS` is the copy and `CAMERA_KEYS` is the camera, and
there are more keys than chapters on purpose. Change the chapter list and three
things have to move with it: the number of scroll sections in `#story`, the
number of `.panel` articles, and both `CHAP_P` (where each chapter's shot sits on
the camera curve) and `CHAP_S` (how much of the page it gets). `side` picks which
scrim lights up: 0 left, 1 right, 2 bottom, 3 the centred vignette the hero uses.

**The camera.** `CAM` holds the live values. `EL_MIN` and `EL_MAX` clamp how far
you can tilt; `Z_MIN` and `Z_MAX` clamp pinch zoom.

**The page.** The overlay is plain DOM sitting over a fixed canvas. Three scrims
(`#scrimL`, `#scrimR`, `#scrimB`) plus a centred one (`#scrimC`) fade in and out
to follow whichever side the current chapter's copy is on, so the type stays
readable whatever the market is doing behind it. Colours are CSS variables at the
top. `#story` is seven empty sections giving the page something to scroll.

## Scripting it

`window.__nagare` is exposed for poking at the scene from the console:

```js
__nagare.setChapter(3)             // jump straight to a chapter
__nagare.chapters()                // their names, in order
__nagare.unlockScroll()            // hand control back to the scrollbar
__nagare.setTime(6.4)              // hour of the day, 0 to 24
__nagare.setCam(-0.62, 0.33, 60)   // azimuth, elevation, distance
__nagare.hideUI(true)              // photo mode
__nagare.freeze(x,y,z,az,el,dist)  // pin the camera for a look at one thing
__nagare.unfreeze()                // hand it back to the scroll
__nagare.setPost(false)            // skip the post chain
__nagare.stats()                   // draw calls, triangles, camera
__nagare.nav()                     // the navigation spine
__nagare.debugNav(true)            // draw the routes and their clearance circles
__nagare.boats()                   // every hull, derived positions and headings
__nagare.audit()                   // run the layout audit now
__nagare.issues()                  // what it found at build time
__nagare.dynamics()                // the hero objects lifted out of the merged mesh
__nagare.setMotion(0)              // freeze all secondary motion (1 = normal)
```

Set `DEBUG_LAYOUT = true` near the top of the script to have the audit warn on
the console during `build()` instead of only recording into `LAYOUT_ISSUES`.

## Keys

- `H` hides the interface
- `Space` plays and pauses time (and releases the clock from the scroll)
- The rail on the right jumps to a chapter

## Performance

Roughly 37 draw calls and 240k triangles, rendered twice per frame for the water
reflection, plus the post chain. Comfortable at 60fps. If a machine cannot keep
up, the render scale drops once or twice on its own and stops there.

## Two traps worth keeping

**A second `function foo()` silently replaces the first.** The noren frame was
first written as `noren()`, which the kit already had -- `noren(w, h, y, col)`
hangs a short curtain across a stall front. Hoisting made the later declaration
win, so every stall called the new one with four numbers: `cfg.span` was
undefined, `NaN` posts went into the merged mesh, and the last stall to build
overwrote the object the real noren had just set. Nothing threw. The cloth simply
never appeared, and the stall curtains quietly stopped existing. `check.sh` now
fails the build on any duplicate top-level declaration.

**`renderer.setSize` clears the drawing buffer.** The adaptive-resolution check
ran at the end of the frame, so the resize landed after the draw and the
compositor could present one empty frame -- a black flash a couple of seconds
into the page. It also acted on its first two-second window, which contains
shader compilation and the opening animation and is not evidence of anything, so
it was downgrading quality on machines that did not need it. The check now runs
before the draw and skips its warm-up window.

## Verifying it

Screenshots of this page are not trustworthy on their own -- the in-app browser
cannot open `file://` at all, and a headless capture of a scene this heavy is
its own small research project. So most of the checking is numeric:

- `validate.py` reads `__nagare.plan()`, `nav()`, `boats()` and `audit()` and
  runs about forty geometric assertions: overlaps, clearances, deck junctions,
  bridge centring, mooring gaps.
- `pathcheck.py` mirrors the page's own pacing maths and samples the shipped
  camera curve 800 times.
- `auditLayout()` runs in the page itself and returns what it found.

When a headless capture is needed, do not let the page drive its own frame loop.
Chrome's `--virtual-time-budget` advances the clock faster than SwiftShader can
render this scene, so rAF callbacks get coalesced and dropped, the budget
expires, and the DOM is dumped with the capture never having run -- silently,
with nothing in the log to say why. The working shape is to replace
`requestAnimationFrame` with a queue and run the queued callbacks from a timer
with a synthetic clock, so every frame is actually executed, and to kill CSS
animations at parse time so an infinite keyframe cannot burn virtual time in the
compositor where nothing can observe it.
