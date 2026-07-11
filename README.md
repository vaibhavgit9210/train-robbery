# How to Build a Single-File 3D Scene Without Any Library

This is a guide for an LLM (or a human) to reproduce `train-robbery.html` — a
self-contained WebGL animation of robbers on horseback robbing a steam train —
**in one HTML file with zero dependencies**. No Three.js, no CDN, no build step.

Read this before writing any code. The order of sections is the order you
should build in. Each step ends with something you can verify.

---

## 0. Why no library?

The file must work when opened from disk and inside strict Content-Security-Policy
sandboxes (which block ALL external requests, including CDN scripts). So every
line of engine code is written by hand. The good news: you only need a tiny
subset of a 3D engine, listed below. Total is ~700 lines of JavaScript.

## 1. The one big architectural decision (read this twice)

**Do NOT move the train through the world. Keep the train at the origin and
scroll the world past it.**

- The train, horses, and camera all stay near `(0, 0, 0)` forever.
- A single number `scroll = time * SPEED` represents how far the train has "traveled".
- Everything that should appear to rush past (railroad sleepers, cacti, rocks,
  telegraph poles) is drawn at `x = wrap(originalX - scroll)` so it repeats forever.
- Things that are uniform along the direction of travel (the two rails, the
  gravel ballast strip, the flat ground) don't need to move at all — you cannot
  see a uniform strip sliding along itself.
- Distant mountains can simply stay static; at that distance the eye forgives it.

This kills three problems at once: no floating-point drift over time, the camera
math stays trivial, and the world is effectively infinite.

The wrap function that makes scenery loop over a window of `RANGE` units
centered on the train:

```js
function wrapX(x) { var v = (x % RANGE + RANGE * 1.5) % RANGE; return v - RANGE / 2; }
```

**Pop-in trick:** items would visibly appear at the wrap edge. Fix: compute
`fade = clamp((RANGE/2 - |x|) / 36, 0, 1)` and multiply the item's *height* by
it — objects grow out of the ground at the horizon instead of popping, and the
distance fog hides the rest.

## 2. The minimal math kit (~40 lines)

You need exactly these, using flat 16-element arrays, **column-major** (OpenGL
convention):

| Function | Purpose |
|---|---|
| `T(x,y,z)`, `RX(a)`, `RY(a)`, `RZ(a)` | translation / axis rotations |
| `mul(a,b)`, `chain(...)` | matrix multiply, left-to-right composition |
| `tp(m, x,y,z)` | transform a **point** (uses translation column) |
| `td(m, x,y,z)` | transform a **direction** (ignores translation — for normals) |
| `persp(fov, aspect, near, far)` | projection matrix |
| `lookAt(eye, target, up)` | view matrix |
| `norm3`, `cross` | vector helpers |

Column-major cheat sheet: element `[12],[13],[14]` is the translation;
`tp` is `m[0]*x + m[4]*y + m[8]*z + m[12]` for the x component, etc.

`chain(T(...), RZ(...), T(...))` reads like a scene-graph path: "go to the
shoulder, rotate the arm, go down the arm". This is your entire hierarchy
system — there is no scene graph object, just nested matrix products.

## 3. Everything is boxes (the "geometry bucket")

Model **every** object out of transformed cuboids, plus a cylinder for the
boiler and wheels. A horse is ~25 boxes. The whole locomotive is ~30 boxes and
3 cylinders. This low-poly "voxel toy" style is a feature: it reads as charming
instead of as a failed attempt at realism.

Implement a `Bucket` class wrapping a pre-allocated `Float32Array`:

- Vertex layout: **position(3) + normal(3) + rgba color(4) = 10 floats**,
  interleaved, one shared layout for every draw call.
- `bucket.box(matrix, sizeX, sizeY, sizeZ, color, alpha)` — writes 36 vertices
  (6 faces × 2 triangles). Transform corner **points** with `tp`, face
  **normals** with `td` + normalize.
- `bucket.cyl(matrix, radius, length, segments, color)` — a prism along local X.
  Need it along Y or Z? Just pass a rotated matrix.
- **Disable back-face culling** (`gl.disable(gl.CULL_FACE)`). You lose a little
  GPU efficiency and gain total immunity to triangle-winding bugs. Worth it.

Three buckets / three draw calls per frame:

1. **Static** — terrain, rails, ballast, mesas. Built once, `STATIC_DRAW`.
2. **Dynamic opaque** — train, horses, riders, scenery items. **Rebuilt from
   scratch on the CPU every frame** and re-uploaded with `DYNAMIC_DRAW`.
   ~25k vertices/frame is nothing; do not build a scene graph or dirty-flag
   system, just re-push everything. Simplicity wins.
3. **Translucent** — smoke, dust, muzzle flashes, lamp glow. Drawn last with
   blending on and `gl.depthMask(false)` (test depth, don't write it).

## 4. Two tiny shaders

**Main shader.** Vertices are already in world space (the CPU did all
transforms), so the vertex shader is just `gl_Position = uVP * vec4(aP, 1.0)`
plus passing varyings. Fragment lighting is one hardcoded formula — ambient +
warm sun diffuse + faint blue skylight, then fog:

```glsl
vec3 col = albedo * (vec3(0.40,0.35,0.33)            // ambient
         + vec3(1.05,0.80,0.58) * max(dot(n,sun),0.) // low golden sun
         + vec3(0.16,0.19,0.25) * max(n.y,0.));      // sky fill from above
float fog = 1.0 - exp(-distance(worldPos, eye) * 0.0036);
col = mix(col, fogColor, fog);
```

> **THE BUG THAT WILL BITE YOU:** compute fog in the **fragment** shader, not
> the vertex shader. A 960-unit-long rail box has vertices only at its far
> ends, where fog ≈ 1. Per-vertex fog then interpolates "fully fogged" across
> the entire strip — the rails render sky-colored even 5 meters from the
> camera. This actually happened in this build; screenshot verification caught
> it. Also use `precision highp float` in the fragment shader, because
> `distance()` over ~900 units overflows mediump on mobile GPUs.

**Sky shader.** One fullscreen triangle drawn first with depth test off.
Reconstruct the per-pixel view ray from camera basis vectors (`fwd + x*aspect*
tanFov*right + y*tanFov*up` — no inverse matrix needed), then mix three colors
by `ray.y` (horizon peach → dusty orange → slate blue) and add the sun as
`smoothstep` disc + `pow(dot(ray,sunDir), 9.0)` glow. **The horizon color, the
fog color, and `gl.clearColor` must be the same value** — that's what makes
ground and sky melt together seamlessly.

Emissive trick with no extra shader: vertex colors are floats, so a muzzle
flash can use color `(2.6, 2.1, 1.1)` — values above 1.0 stay bright after the
lighting multiply.

## 5. Animating the actors

All animation is `sin()` of time with phase offsets. No keyframes, no easing
library.

**Locomotive.** Drive wheels rotate `angle = -scroll / wheelRadius` (tie
rotation to distance, not time, so it never slips). Spokes are 2 crossed thin
boxes per wheel — rotation is invisible on a plain cylinder. The side rod is a
horizontal box whose center rides a circle of crank radius around the wheel
centers; because both cranks share a phase, the rod stays horizontal and just
orbits — one box, very convincing. Cars get tiny independent
`sin(t*5 + i*1.9) * 0.02` bobs and rolls so the train feels alive.

**Horse gallop.** Body: `y = 1.18 + sin(cyc)*0.1`, pitch `sin(cyc+1.2)*0.075`,
where `cyc = (t*2.55 + phase) * 2π`. Each of 4 legs is a 2-segment pendulum:
upper `sin(cyc + legPhase)*0.78`, knee `-max(0, sin(cyc + legPhase + 1.05))*1.1`
(clamped so the knee only folds one way). Leg phase offsets `{hindL:0,
hindR:.25, frontL:.55, frontR:.8}` produce a rotary gallop. Each horse also
drifts alongside the train with a slow sine on its x-position so the pack
surges and falls back. **Give every horse a different phase** — synchronized
horses instantly look robotic.

**Riders and the roof robber** share one `drawBandit(matrix, crouch, ...)`
function — a rider is just a bandit with low crouch sitting on a saddle
matrix; the roof man is the same function with high crouch on the car roof.
Gun arm: `chain(shoulderT, RZ(raisedAngle), T(down the arm), RZ(level))`, and
the muzzle tip for the flash is `tp(gunMatrix, 0.32, 0.03, 0)`.

**Muzzle flash timing:** `f = fract(t*0.55 + perBanditOffset)`; if `f < 0.05`
draw 3 crossed bright boxes at the tip, scaled by `(0.9 - f*12)` so each flash
decays over ~3 frames. Add a recoil term to the arm angle during the window.

**Particles** (one array of plain objects): spawn smoke at the stack every
55 ms with upward velocity + `-SPEED*0.82` backward drift (remember: the world
moves, so exhaust must too); spawn dust at any hoof whose leg-phase sine is
below −0.75 (i.e., planted). Each particle: position += velocity, gravity or
buoyancy on `vy`, size lerps up, alpha lerps to 0. Render as camera-facing
quads: corners = `pos ± cameraRight*s ± cameraUp*s`. Cap the array (~600) by
shifting the oldest out.

## 6. Camera

Spherical orbit around a target just above the train: yaw, pitch, distance.
Auto-drift `yaw += dt*0.045` for a cinematic feel; any pointer drag takes over
and auto resumes after 8 idle seconds. Wheel zoom clamps distance to [7, 48].
Clamp pitch so the camera never dives underground. Honor
`prefers-reduced-motion` by disabling the auto-orbit and bob.

## 7. Order of operations per frame

```
1. resize canvas to clientSize × devicePixelRatio (cap DPR at 2)
2. update camera → view, proj, eye, fwd/right/up basis
3. reset dynamic buckets (just set count = 0)
4. push train, scenery, horses into opaque bucket; particles into translucent
5. clear color+depth to fog color
6. draw sky triangle          (depth test OFF)
7. draw static + dynamic VBOs (depth ON, blend OFF)
8. draw translucent VBO       (blend ON, depthMask OFF)
9. requestAnimationFrame
```

## 8. Verify with screenshots — do not trust your mental render

You cannot eyeball WebGL correctness from source. After every visual change:

```bash
# syntax check the extracted <script> first
node --check main.js

# then actually render it headless (SwiftShader = no GPU needed)
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless=new --disable-gpu --use-angle=swiftshader \
  --window-size=1280,800 --virtual-time-budget=6000 \
  --screenshot=shot.png "file:///path/to/train-robbery.html"
```

`--virtual-time-budget=6000` fast-forwards 6 simulated seconds so you screenshot
mid-animation. Look at the image and ask: is anything floating, inside-out,
fog-colored up close, or the wrong scale? Every bug found in this build
(washed-out rails, a glow quad rendering as a hard white square, salmon-pink
cowcatcher, oversized mesas) was found from a screenshot, not from the code.

## 9. Traps, ranked by how likely you are to fall in

1. **Per-vertex fog on long geometry** (§4). Fragment fog, always.
2. Trying to move the train instead of scrolling the world (§1).
3. Enabling face culling and losing an hour to invisible inside-out boxes —
   just disable it.
4. Forgetting `depthMask(false)` for particles → smoke punches rectangular
   holes in later smoke.
5. Rotating wheels by time instead of `scroll/radius` → wheels visibly slip
   when you change speed.
6. Synchronized animation phases → everything marches in lockstep like toys.
7. mediump `distance()` overflow on large worlds → black/NaN speckle on phones.
8. Hard-edged billboard quads look fine for dust/smoke at low alpha (~0.3–0.6);
   they read as "stylized". At high alpha they read as bugs (that white lamp
   square). When a glow looks wrong, halve its size AND its alpha.
9. Colors: pick a 5–6 value palette up front (sand `#C08552`, oxide red
   `#7A3B2A`, iron `#26221f`, fog/horizon `#EFBA85`, cactus `#5C7345`) and
   derive everything from it. Random per-object colors are what make scenes
   look AI-generated.

## 10. Files

| File | What it is |
|---|---|
| `train-robbery.html` | the complete scene — open it in any browser |
| `README.md` | this guide |

Ideas if you want to extend it: a second track with a passing train, a canyon
bridge section spliced into the scroll cycle, lasso physics (a swinging chain
of 5–6 segments), day/night cycle by lerping the sun direction and palette.
