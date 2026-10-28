# TouchDesigner submission pack

Comment text ready to paste, plus the things that will break if you submit the
`.toe` on its own.

---

## 0. Review of the Kinect network

From the screenshots. Ordered by what a marker will notice.

### 0.1 It fires a find on load, with nothing in front of the sensor

In the `sculptures` table, **Maori Wrasse is sitting at progress 1.0 and
contact 1** while `kinect2` is showing a warning triangle. That isn't a
coincidence.

With no sensor, every joint channel reads 0. `hand_screen()` then returns a
hand at (0, 0) — dead centre of frame — with a radius from
`max(z, 0.5)`, so `hr = 0.5 × 0.10 / (0.709 × 0.5) = 0.141`. Maori Wrasse is
at (0.1062, −0.007) with `r = 0.06`. Distance from centre is 0.106,
`tr + hr` is 0.201, so the circles overlap. The accumulator fills, the find
fires, and it stays fired.

Anyone opening this file without a Kinect plugged in sees a sculpture
permanently triggered before they've done anything. Reads as broken, is
actually a missing guard.

Fix. Add to `hit_logic.py`:

```python
def joints_live():
    """False when the sensor is dead and every channel sits at exactly zero."""
    m = op('merge_joints')
    try:
        vals = [m[c][0] for c in ('hand_l_x', 'hand_l_y', 'hand_r_x', 'hand_r_y')]
    except (TypeError, AttributeError):
        return False
    return any(abs(v) > 1e-6 for v in vals)
```

and at the top of `check()`, straight after `table = op('sculptures')`:

```python
    if not joints_live():
        for row in range(1, table.numRows):
            table[row, 'contact'] = 0
            table[row, 'progress'] = 0.0
            held[row] = 0.0
            fired[row] = False
        return
```

Worth noting in the journal too. "A dropout parks the hand at frame centre and
whatever is nearest the middle charges by itself" is a real failure mode of the
coordinate design, not just a bug — it's what happens when absence and
origin are the same number.

Related: **Maori Wrasse at x = 0.1062 is almost dead centre.** Even with the
guard in, any brief dropout parks the hand right on it. Worth moving it out
toward its real position on the map.

### 0.2 Bywa is on a map it isn't on

`Bywa` is in the sculptures table and has a Circle TOP. Your own 08.10 entry
says Bywa is at Horseshoe Bay in **Bowen**, outside the Hook Island crop.

So the prototype is inviting people to find a sculpture at a location where it
doesn't exist. Either drop the row, or keep it and label it as off-map. A
marker who reads the journal and then opens the file will spot it.

### 0.3 The "removing z" comment is wrong on this path

The sticky note by the select chain reads
`//Select an rename -> map range 1 - 0 (removing z)`, but `select_z` is wired
into `merge_joints` and `hand_l_z` / `hand_r_z` are in the channel list.

That's correct — on the Kinect path z is essential, it's the perspective
divide. The comment is left over from the MediaPipe version, where there is no
usable z. Fix it, because it contradicts the most important idea in the build.

Replace with:

```
Select and rename the joints we use.
x / y / z are split so each axis can be flipped independently —
a Math CHOP's range applies to every channel in the stream, so there's
no way to negate x and leave y alone.

z is NOT discarded here. It drives the perspective divide in hit_logic:
at distance z, half the frame width in metres is tan(35.3) * z = 0.709z.
That's what makes the mapping independent of standing distance.
(The MediaPipe version has no usable z — it substitutes shoulder span.)
```

### 0.4 The aspect problem is in `resolution`, and widening it isn't free

`resolution` is **512 × 424**, which is the Kinect depth sensor's own
resolution and gives the 1.21:1 comp. The map is 2.08:1. That's the mismatch
you've been deferring, and it now has an address.

What makes it awkward: hand x is a fraction of the *sensor* frame, target x is
a fraction of the *comp*, and both are ±0.5, so they map onto each other 1:1
regardless of aspect. Widen the comp to 2.08:1 and the interaction still
*works* — targets placed with `place()` stay consistent — but horizontal hand
movement now covers more screen than the same vertical movement. It stops
feeling like a cursor.

Three options, cheapest first:

1. **Crop the map to 1.21:1.** You already know tight crops are better, since
   the 255 levels get spent on what you keep. Costs you some of the island
   chain.
2. **Letterbox.** Keep the interaction region at sensor aspect, sit the wide
   map inside it with dead space above and below. Honest, slightly ugly.
3. **Widen and accept the non-uniform mapping.** Only if you're happy to say
   so in the report.

Whichever you pick, say which and why. A deferred decision described as a
decision is worth more than a deferred decision left silent.

### 0.5 Rename the rest

The named nodes are good — `select_x`, `merge_joints`, `circ_hand_l`,
`comp_hands`, `blur_hands`, `comp_accum`, `level_decay`, `fb_loop`,
`watercut`, `premultiply_RGB_by_alpha`, `land_mask`, `water_final` all read.
These don't:

| Now | Suggested |
|---|---|
| `lookup` | `lookup_sand` |
| `lookup1` | `lookup_hand_falloff` |
| `lookup2` | `lookup_water` |
| `ramp1` | `ramp_hand` |
| `ramp2` | `ramp_water_keys_src` (or whatever it feeds) |
| `ramp3` | `ramp_sand` |
| `ramp5` | `ramp_sand_dark` |
| `reorder1` | `land_to_alpha` |
| `reorder2` | `sand_to_alpha` |
| `constant1` | `sculpt_light_colour` |
| `constant2` | `black_bg` |
| `constant3` | `land_colour` |
| `math1` | `sculpt_alpha` |
| `math2` | `carve_invert` |
| `math3` | `land_mask_level` |
| `limit1` | `carve_limit` |
| `noise1` | `water_noise` |
| `transform1` | `sculpt_transform` |
| `level1` | `sculpt_level` |
| `comp1` | `comp_markers` |
| `null1` | `out_markers` |
| `null2` | `out_water` |
| `datto1` | `progress_to_chop` |
| `trail1` | `progress_graph` |

Also: `seabed_8` and `moviefilein1` look like the same image loaded twice,
both feeding `lookup`. If one is redundant, delete it. If they're genuinely
different crops, name them so.

### 0.6 Keep the progress graph and label it

`datto1` → `trail1` plotting the accumulator over time is the best piece of
evidence in the network that the dwell mechanic does what you say it does.
Rename it, put an Annotate box around it saying what it shows, and leave it
visible. It's the kind of thing a marker notices.

### 0.7 Clear the warning triangles

Three visible: `kinect2`, `kinect_ref`, `level_decay`. `kinect2` is almost
certainly "no device", which is unavoidable on a machine without one — so the
Kinect file should **default to recorded footage through a Movie File In**,
with the live Kinect as the alternative, and the README should say so.
Otherwise the file looks broken on every machine but yours.

### 0.8 When you port the MediaPipe visuals across

Keep the two files structurally identical apart from the sensing block, so the
`SOURCE` constant remains a true claim. The moment the Kinect version has a
different comp chain, "one constant switches the sensor" stops being accurate
and becomes a thing you said in a report. Check `SOURCE` matches the file
before you zip each one.

---

## 1. Do this first — the file won't open properly for anyone else

A `.toe` stores **absolute paths** to external media. Submit the file alone and
the marker opens it to a wall of missing-file errors, which reads as a broken
prototype rather than a path problem.

Two fixes, do both:

**Make the paths relative.** Every Movie File In, Text DAT with a file
reference, and the FBX import currently points at
`C:/Users/mjbis/OneDrive - University of the Sunshine Coast/...`. Change each to
a path relative to the `.toe`, e.g. `media/seabed_8.png`. TouchDesigner resolves
relative paths from the project file's location.

**Ship a folder, not a file.** Structure:

```
BCI304_Bissell_TouchDesigner/
├─ README.txt
├─ whitsunday_mediapipe.toe
├─ whitsunday_kinect.toe
├─ hit_logic.py
└─ media/
   ├─ seabed_8.png
   ├─ land_mask.png
   ├─ mantaray.fbx
   └─ demo_footage.mp4      (if the Kinect version needs the recorded clip)
```

Zip the folder. Check it by extracting the zip somewhere else on your machine
and opening it from there — if it works from a different directory the paths
are relative.

**Include `hit_logic.py` as a loose file.** It's the actual technical artefact
and a marker who doesn't have TouchDesigner installed can still read it. It is
already commented; leave it as is.

---

## 2. README.txt — paste into the submission folder

```
WHITSUNDAY IN MOTION — Sculpture Finder
Interactive prototype. Matt Bissell, BCI304, 2026.

WHAT THIS IS
A motion-activated display for Whitsunday Coast Airport. A bathymetric map of
the water around Hook Island sits under a layer of water. A person in front of
the display parts the water with their hands. Holding over one of the Ngaro
underwater sculptures or a reef restoration site reveals it and names it.

TWO VERSIONS
whitsunday_mediapipe.toe   Runs from a webcam. Open this one first.
whitsunday_kinect.toe      Runs from a Kinect v2, or recorded Kinect footage.

Both drive the same logic. The only difference is which sensor feeds the
joint positions. The switch is one constant at the top of hit_logic.py:

    SOURCE = 'mediapipe'    # or 'kinect'

TO RUN
1. Open whitsunday_mediapipe.toe
2. Check /project1/videodevin1 is pointing at a working camera
3. Stand back far enough that your shoulders are in frame
4. Hold a hand over one of the marked sites for about a second

The interaction runs at 60 fps. Hit detection is in /project1/hit_logic,
called every frame by /project1/execute1.

DATA
Bathymetry from AusBathyTopo (Great Barrier Reef) 30m 2017, Geoscience
Australia, record 20170025C. Clipped to the Hook Island area in QGIS and
exported to 8-bit, clamped to the -78m to 0m range. Source raster is
single-band Float32; media/seabed_8.png is a derived image, not the data.

Sculpture locations from the Ngaro Underwater Marine Sculpture Trail.
Reef restoration sites from Reef Ecologic's WRAP project.

KNOWN LIMITATIONS
- Map is 2.08:1, comp is 1.21:1. Sculpture coordinates and composition do
  not fully agree. Unresolved.
- No airport hardware specification was available, so resolution and aspect
  decisions are against an assumption.
- MediaPipe single-pose only. Two people in frame will confuse it.
- Not tested with seated users. The accessibility reasoning in the journal
  is reasoning, not evidence.

Full process journal: [your URL]
```

---

## 3. Root `readme` Text DAT

Make a Text DAT at `/project1/readme`, paste this in, and park it top-left of
the network so it's the first thing anyone sees.

```
SCULPTURE FINDER — how this network works

Read left to right. Five stages.

1  SENSING          videodevin1 / kinect1  ->  select  ->  rename
                    Pull wrist and shoulder joints out of the sensor's
                    channel stream and give them workable names.

2  COORDINATES      select_x / select_y  ->  math_x / math_y  ->  merge_joints
                    Four conversions happen here. Units (metres to fraction
                    of frame, via the perspective divide), origin (sensor /
                    corner / centre), direction (mirrored x, flipped y) and
                    scale per axis (x is a fraction of width, y of height).
                    Getting any one of these wrong looks like "the tracking
                    is off" and they are four separate bugs.

3  LOGIC            sculptures (Table DAT)  +  hit_logic  +  execute1
                    execute1 calls hit_logic.check() every frame.
                    hit_logic reads merge_joints and the sculptures table,
                    works out which target each hand is nearest, and writes
                    progress (0-1) and contact (0/1) back into the table.
                    Nothing else in the network knows about sensors.

4  VISUAL           moviefilein1  ->  lookup (sand)  ->  lookup2 (water)
                    ->  carve  ->  composite
                    Seabed and water are both driven off the same bathymetry
                    through two different ramps. The water layer is carved by
                    a feedback mask that follows the hands.

5  OUTPUT           comp1  ->  out
                    Resolution is set from kinect_ref. Change it there, not
                    on individual operators.

Everything reads the sculptures table. To add a sculpture, append a row:
  name, x, y, r, progress, contact
and place it by standing where you want it and running, from the Textport:

  h = op('/project1/hit_logic').module
  h.add('Turtle Dream', 'hand_r')
```

---

## 4. Annotations per cluster

Add an Annotate COMP around each group (Palette → Tools → annotate), or
right-click → Comment on the key node. Keep them short — these are signposts,
not documentation.

| Around | Title | Body |
|---|---|---|
| videodevin1 / kinect1, select, rename | **1. Sensing** | Wrists and shoulders out of the sensor stream. MediaPipe gives 33 landmarks normalised 0–1; Kinect gives 25 joints in metres. |
| select_x, select_y, math_x, math_y, merge | **2. Coordinate conversion** | Split per axis because a Math CHOP's range applies to every channel — no way to negate x and leave y alone. Merged back after. |
| sculptures, hit_logic, execute1 | **3. Hit logic** | execute1 → hit_logic.check() every frame. Writes progress and contact into the sculptures table. Winner-takes-all: each hand charges only its nearest target. |
| circ_hand_l, circ_hand_r | **Hand markers** | Centre from merge_joints, radius from the perspective divide. Same radius the hit test uses, so what you see is what's tested. |
| moviefilein1, ramp, lookup | **4a. Seabed** | seabed_8.png is 8-bit greyscale clamped to −78m–0m. Pixel Format must be set explicitly to 8-bit RGBA — "Use Input" propagates Mono, which has no colour and no alpha. |
| ramp_water, lookup2 | **4b. Water colour** | Same bathymetry, second ramp. Depth-tinted rather than flat turquoise, free because the map is already loaded. |
| feedback, level_decay, blur | **4c. Carve mask** | Hand circles into a Feedback TOP with decay. Feedback needs an explicit Custom resolution — Use Input makes a circular dependency and a checkerboard. |
| reorder, premultiply, comp | **4d. Compositing** | Reorder takes Input 2 **Luminance** into alpha (not Red — survives a coloured mask). Premultiply is required before Over or the blend goes additive. |
| blur_carve, subtract | **4e. Foam ring** | Blur the mask, subtract the original from the blur. What's left is a ring just outside the hole. Tracks the hole automatically, including while it closes. |
| noise1, noise2, displace | **4f. Refraction** | Two noises scrolling at 0.01 and 0.013 — speeds that don't divide evenly, so it doesn't read as one sliding texture. 16-bit float through here; repeated subtracts band badly in 8-bit. |
| mantaray, cam_sculpt, light_sculpt, render | **Sculpture model** | FBX, Y up / −Z forward. Background alpha 0 so it composites. Material override is on the Advanced tab. |

---

## 5. Node comments for the non-obvious ones

Right-click → Comment. These are the operators where someone will ask "why is
that there".

| Node | Comment |
|---|---|
| `math_x` | Negates x. Kinect's x is mirrored relative to the screen. |
| `math_y` | Flips y. MediaPipe runs top-down, TouchDesigner runs bottom-up. |
| `kinect_ref` | Resolution reference only. Everything else pulls width/height from here so there's one place to change it. |
| `moviefilein1` | Pixel Format set explicitly to 8-bit RGBA. Do not set to Use Input — the PNG is Mono and the Lookup will produce no colour. |
| `feedback1` | Custom resolution, not Use Input. Circular dependency otherwise. |
| `level_decay` | Drain rate on the carve. Lower = water closes faster. |
| `reorder1` | Input 2 Luminance → Output Alpha. Luminance rather than Red so a coloured mask still works. |
| `premultiply1` | Required before the Over. Without it the composite goes additive and the sculptures disappear. |
| `level_translucent` | Brightness 0.85 on the inverted mask. Water sits at 85% opacity so sculptures are faintly visible before you reach them. |
| `displace1` | Weight is deliberately low. Climb until obviously too much, then halve. |

---

## 6. Naming and colour

If the nodes are still on default names (`select1`, `math2`, `null3`), rename
them before you submit. It's ten minutes and it's the difference between a
network someone can read and one they can't.

Colour the five stages differently using the node colour palette (right-click →
Colour). Sensing one colour, coordinates another, logic another, visual another,
output another. A marker scanning the network sees structure before they read
anything.

---

## 7. Before you zip

- [ ] Both `.toe` files open from a fresh extract with no missing-file errors
- [ ] `SOURCE` in each file matches the sensor that file uses
- [ ] `hit_logic.py` included as a loose file
- [ ] `README.txt` in the folder root
- [ ] `readme` Text DAT visible top-left in both networks
- [ ] Nodes renamed off defaults
- [ ] Annotations in place
- [ ] Textport clear of errors on load
- [ ] Zip tested by extracting to a different folder and opening from there

---

## 8. What to submit, all up

| Item | What it is for the rubric |
|---|---|
| Field report, 2000 words | The assessed written piece. Still outstanding. |
| Process journal (the blog) | Field journal — named in the task description |
| TouchDesigner folder (zipped) | Creative artefact |
| `hit_logic.py` | Creative artefact, readable without TouchDesigner |
| Screen capture of the demo | Field documentation |
| Presentation deck | Field documentation (already submitted for Task 2, can be referenced) |
| Images, maps, QGIS exports | Field documentation — mostly embedded in the journal |
| AI acknowledgement | Required, in the report |

The journal covers artefacts and documentation. The report is the thing the
four criteria are actually written against, and it's the one still missing.
