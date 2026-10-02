---
name: cannon-es
description: cannon-es rigid-body physics for three.js — worlds, bodies, shapes, materials, constraints, collision events, raycasting, fixed-timestep stepping, sleeping, and mesh sync. Use when adding physics to a 3D scene, simulating rigid bodies, handling collisions or triggers, building character controllers, or debugging jitter, sinking, exploding stacks, or objects falling through floors — even if the user just says "physics" without naming an engine.
---

# cannon-es — Physics for three.js

cannon-es is the maintained ESM fork of cannon.js (pmndrs). Always use `cannon-es`, never the abandoned `cannon` package — several APIs changed (see Common Pitfalls). Ships with TypeScript types.

```bash
npm install cannon-es
```

```javascript
import * as THREE from "three";
import * as CANNON from "cannon-es";
```

## Quick Start

```javascript
const world = new CANNON.World({ gravity: new CANNON.Vec3(0, -9.82, 0) });
world.broadphase = new CANNON.SAPBroadphase(world);
world.allowSleep = true;

// Static ground — Plane is infinite and faces +Z by default, tilt it to Y-up
const groundBody = new CANNON.Body({ mass: 0, shape: new CANNON.Plane() });
groundBody.quaternion.setFromEuler(-Math.PI / 2, 0, 0);
world.addBody(groundBody);

// Falling box — Box takes HALF extents
const boxBody = new CANNON.Body({
  mass: 1,
  shape: new CANNON.Box(new CANNON.Vec3(0.5, 0.5, 0.5)),
  position: new CANNON.Vec3(0, 5, 0),
});
world.addBody(boxBody);

// Render mesh
const boxMesh = new THREE.Mesh(
  new THREE.BoxGeometry(1, 1, 1),
  new THREE.MeshStandardMaterial({ color: 0x00ff00 }),
);

const clock = new THREE.Clock();
function animate() {
  requestAnimationFrame(animate);
  world.fixedStep(); // fixed 1/60 internally, framerate-independent
  boxMesh.position.copy(boxBody.interpolatedPosition);
  boxMesh.quaternion.copy(boxBody.interpolatedQuaternion);
  renderer.render(scene, camera);
}
```

## Core Concepts

| Physics concept | cannon-es | Pairs with (three.js) |
| --- | --- | --- |
| Simulation container | `World` | `Scene` |
| Rigid object | `Body` (+ `Shape`s) | `Mesh` (+ `Geometry`) |
| Surface behavior | `Material` / `ContactMaterial` | `Material` (unrelated) |
| Joints | `Constraint` subclasses | — |
| Query by ray | `world.raycastClosest(...)` | `Raycaster` |

Physics and rendering are fully decoupled: the world knows nothing about meshes. You step the world, then copy transforms into meshes. Keep arrays of `{ mesh, body }` pairs and sync in one loop.

## Units and Scale

Use **1 unit = 1 meter**, mass in **kg**, gravity **-9.82** on Y. The solver is tuned for objects roughly 0.1–10 m; sub-centimeter objects jitter and skyscraper-sized ones feel floaty. Size your scene for the physics engine — don't rescale gravity to compensate for a giant scene.

## World Setup

```javascript
const world = new CANNON.World({
  gravity: new CANNON.Vec3(0, -9.82, 0),
  allowSleep: true, // bodies sleep when still — big perf win
  broadphase: new CANNON.SAPBroadphase(world), // pass world BEFORE adding to it
  // solver: defaults to GSSolver
});

// Solver quality vs speed (defaults: 10 iterations)
world.solver.iterations = 10;
world.solver.tolerance = 0.001; // early-out for stable-enough stacks

// Global fallback for pairs without a specific ContactMaterial
world.defaultContactMaterial.friction = 0.4;
world.defaultContactMaterial.restitution = 0.1;
```

- **Broadphase**: default `NaiveBroadphase` checks all pairs (O(n²)). `SAPBroadphase` (sweep-and-prune) is the right default for anything beyond a few dozen bodies. `GridBroadphase` suits evenly spaced worlds.
- **`world.hasActiveBodies`** becomes `false` when every body is asleep — skip `step()` entirely until something wakes (forces and impulses wake bodies automatically).

## Bodies

```javascript
const body = new CANNON.Body({
  mass: 1, // mass: 0 → STATIC, the usual way to make static geometry
  shape: new CANNON.Sphere(0.5), // or add shapes below
  position: new CANNON.Vec3(0, 2, 0),
  // velocity, angularVelocity, quaternion also accepted here
  linearDamping: 0.01, // velocity retained per second; use as cheap air drag
  angularDamping: 0.01,
  allowSleep: true,
  fixedRotation: true, // never tips over — bullets, characters
  collisionFilterGroup: 1,
  collisionFilterMask: -1, // collide with everything
  collisionResponse: true, // false = overlap events only, no push-back
});
world.addBody(body); // world.removeBody(body) to remove
```

**Types** (`body.type`): `Body.DYNAMIC` (simulated, finite mass), `Body.STATIC` (never moves), `Body.KINEMATIC` (moves only by its set velocity — elevators, moving platforms; ignores forces). Static bodies don't collide with static/kinematic bodies — a moving platform must be KINEMATIC, not STATIC.

**Filters**: two bodies interact only if `(A.collisionFilterGroup & B.collisionFilterMask) && (B.collisionFilterGroup & A.collisionFilterMask)`. Assign each group a power-of-two bit (`GROUND = 1, PLAYER = 2, ENEMY = 4, ...`).

**Compound bodies** — multiple shapes on one body, each with an optional offset and orientation:

```javascript
// Capsule ≈ sphere + cylinder, matching a three.js group of the same parts
const body = new CANNON.Body({ mass: 5 });
body.addShape(new CANNON.Sphere(0.5), new CANNON.Vec3(0, 0.9, 0));
body.addShape(new CANNON.Cylinder(0.5, 0.5, 1, 12));
body.addShape(new CANNON.Sphere(0.5), new CANNON.Vec3(0, -0.9, 0));
```

Changing `mass`, shapes, or `fixedRotation` after creation requires `body.updateMassProperties()`.

## Shapes

| Shape | Notes |
| --- | --- |
| `Box(halfExtents)` | **Half** extents: `new CANNON.Box(new CANNON.Vec3(w/2, h/2, d/2))` |
| `Sphere(radius)` | Cheapest, most robust — prefer for characters and balls |
| `Plane` | Infinite half-space, faces +Z; tilt for floors; static only |
| `Cylinder(rTop, rBottom, height, segments)` | **Y-aligned** in cannon-es (matches `THREE.CylinderGeometry` — no rotation needed) |
| `ConvexPolyhedron({ vertices, faces })` | `faces` = arrays of vertex indices, wound CCW seen from outside |
| `Trimesh(vertices, indices)` | Arbitrary mesh **static** collision only; no Trimesh↔Trimesh contacts |
| `Heightfield(data, { elementSize })` | Terrain grid in local X–Y, height on +Z; rotate −90° on X for Y-up |
| `Particle` | Zero-size point — triggers, raycast targets, idealized joints |

Rules of thumb:

- Approximate visual meshes with primitives or compounds of primitives. Save `Trimesh` for static level geometry, and remember dynamic Trimesh bodies can't collide with each other.
- `Heightfield` example:

```javascript
const matrix = []; // matrix[x][y] = height value
for (let i = 0; i < 20; i++) {
  matrix.push([]);
  for (let j = 0; j < 20; j++) matrix[i].push(Math.sin(i * 0.3) * Math.cos(j * 0.3));
}
const hfShape = new CANNON.Heightfield(matrix, { elementSize: 1 });
const hfBody = new CANNON.Body({ mass: 0, shape: hfShape });
hfBody.quaternion.setFromAxisAngle(new CANNON.Vec3(1, 0, 0), -Math.PI / 2);
hfBody.position.set(-10, 0, 10); // center the grid on origin
world.addBody(hfBody);
```

## Stepping the World

Physics must advance in fixed timesteps — variable steps break solver stability. Prefer `world.fixedStep()`: it tracks wall-clock time internally, runs 0–n substeps of 1/60 per frame, and updates `interpolatedPosition`/`interpolatedQuaternion` on every body (first call excepted).

```javascript
// Option A (recommended): clock is handled for you
world.fixedStep(); // fixedStep(dt = 1/60, maxSubSteps = 10)

// Option B: manual, with THREE.Clock — clamp dt or a tab-switch stalls the frame
const dt = Math.min(clock.getDelta(), 0.1);
world.step(1 / 60, dt, 3); // step(dt, timeSinceLastCalled, maxSubSteps)
```

Render from `body.interpolatedPosition` / `body.interpolatedQuaternion` — they're blended between the last two fixed steps, so motion stays smooth at any refresh rate. Raw `body.position` is only correct if you render exactly on step boundaries.

`maxSubSteps` caps catch-up work per frame: at 10 substeps of 1/60 the sim falls behind on hitches instead of freezing the main thread (the "spiral of death").

## Syncing Bodies to Meshes

```javascript
// THREE.Vector3.copy() accepts any {x,y,z} — CANNON.Vec3 qualifies (and same for quaternions)
mesh.position.copy(body.interpolatedPosition);
mesh.quaternion.copy(body.interpolatedQuaternion);
```

- Sync **after** `step()`, once per frame, for every pair.
- **Never scale a mesh to change collision size** — `mesh.scale` is invisible to the physics engine and non-uniform scaling of shapes is unsupported. Bake dimensions into the shape and rebuild the geometry instead.
- For hundreds of simple objects, render with `THREE.InstancedMesh` and write `body.interpolatedPosition` into `instanceMatrix` each frame.
- Moving a dynamic body by teleport (`body.position.set(...)`) is for resets only; zero its velocity too or contacts will explode.

## Materials and Contact Materials

`Material` is just a tag you attach to bodies; `ContactMaterial` defines what happens when two tags meet.

```javascript
const ice = new CANNON.Material("ice");
const tire = new CANNON.Material("tire");

world.addContactMaterial(
  new CANNON.ContactMaterial(ice, tire, {
    friction: 0.05, // default 0.3
    restitution: 0.1, // default 0.3 — bounciness
    // Solver tuning, rarely needed (defaults: stiffness 1e7, relaxation 3):
    // contactEquationStiffness, contactEquationRelaxation,
    // frictionEquationStiffness, frictionEquationRelaxation
  }),
);

bouncyBall.material = ice;
```

Materials pair up combinatorially (n materials → n(n−1)/2 contact materials). Keep it to a handful of behavior classes (default / slippery / bouncy / grippy) and lean on `world.defaultContactMaterial` for everything else.

## Forces, Impulses, Torque

```javascript
body.applyForce(new CANNON.Vec3(0, 500, 0)); // N — accumulates, cleared each step
body.applyImpulse(new CANNON.Vec3(0, 10, 0)); // N·s — instant velocity change
body.applyTorque(new CANNON.Vec3(0, 50, 0));
// Local-space variants and off-center application:
body.applyLocalForce(localForce, localPoint);
body.applyForce(force, relativePoint); // point relative to CENTER OF MASS
```

- **Impulse** for discrete events (explosions, jumps, hits); **force** for continuous push (thrusters) applied every frame.
- In cannon-es the point argument is relative to the center of mass (old cannon.js took a world point — don't copy old snippets).
- Forces/impulses wake sleeping bodies automatically.

## Collision Events and Triggers

```javascript
boxBody.addEventListener("collide", (event) => {
  event.body; // the other body
  event.contact.getImpactVelocityAlongNormal(); // e.g. for fall damage
});

world.addEventListener("beginContact", ({ bodyA, bodyB }) => {});
world.addEventListener("endContact", ({ bodyA, bodyB }) => {});
world.addEventListener("postStep", () => {}); // after all substeps
```

- `collide` fires on **each** of the two bodies during the step. **Never add/remove bodies or constraints inside it** — the world is mid-iteration. Queue changes and apply them in a `postStep` handler (or next frame).
- `beginContact`/`endContact` are emitted after the step from the pair set — safe for game logic like score zones.
- **Trigger volumes**: a body with `isTrigger: true` (or `collisionResponse: false`) never pushes others but still raises contact events — pickups, checkpoints, pressure plates.

## Sleeping

```javascript
world.allowSleep = true; // master switch
body.allowSleep = true; // default
body.sleepSpeedLimit = 0.1; // below this speed the body gets sleepy
body.sleepTimeLimit = 1; // ...and sleeps after this many seconds

body.sleep(); // force
body.wakeUp();
if (body.sleepState === CANNON.Body.SLEEPING) {} // AWAKE / SLEEPY / SLEEPING
body.addEventListener("wakeup", () => {}); // also 'sleepy' and 'sleep'
```

Sleeping bodies are skipped by the solver but still report collisions — when stacks settle, simulation cost drops to nearly zero. Tune `sleepSpeedLimit` up (e.g. 0.3) if objects never fall asleep because micro-jitter keeps them awake.

## Raycasting

```javascript
const result = new CANNON.RaycastResult();
world.raycastClosest(
  new CANNON.Vec3(0, 10, 0), // from
  new CANNON.Vec3(0, -10, 0), // to
  { skipBackfaces: true, collisionFilterMask: GROUND }, // options
  result,
);
if (result.hasHit) {
  result.body;
  result.hitPointWorld;
  result.hitNormalWorld;
  result.distance;
}
```

Also available: `raycastAny` (first hit, unordered, fastest), `raycastAll(from, to, options, callback)` (every hit). Use for mouse picking, ground checks, and line-of-sight.

## Constraints

| Constraint | Use |
| --- | --- |
| `PointToPointConstraint(bodyA, pivotA, bodyB, pivotB, maxForce?)` | Ball joint |
| `HingeConstraint(bodyA, bodyB, { pivotA, axisA, pivotB, axisB })` | Doors, wheels |
| `ConeTwistConstraint(bodyA, bodyB, { pivotA, pivotB, axisA, axisB, angle, twistAngle })` | Ragdoll limbs |
| `DistanceConstraint(bodyA, bodyB, distance?, maxForce?)` | Rope segment |
| `LockConstraint(bodyA, bodyB, options?)` | Rigid weld |

```javascript
const hinge = new CANNON.HingeConstraint(doorBody, frameBody, {
  pivotA: new CANNON.Vec3(0.5, 0, 0), // on the door edge
  axisA: CANNON.Vec3.UNIT_Y,
  pivotB: new CANNON.Vec3(1, 0, 0), // matching point on the frame
  axisB: CANNON.Vec3.UNIT_Y,
});
world.addConstraint(hinge); // world.removeConstraint(hinge)
```

## Character Controller Basics

```javascript
const player = new CANNON.Body({
  mass: 5,
  shape: new CANNON.Sphere(0.5), // spheres slide over edges; add shapes for a capsule
  fixedRotation: true,
  linearDamping: 0.9, // high damping = quick stop when input releases
});

// Move by setting velocity — never position
player.velocity.x = input.x * speed;
player.velocity.z = input.z * speed;

// Ground check: short raycast straight down from the body
const from = player.position;
const to = new CANNON.Vec3(from.x, from.y - 0.55, from.z); // slightly past the surface
world.raycastClosest(from, to, { skipBackfaces: true }, groundResult);
const grounded = groundResult.hasHit;
if (grounded && wantJump) player.velocity.y = 7;
```

## Performance Tips

1. Use `SAPBroadphase` and `allowSleep: true` — the two cheapest wins.
2. Primitive shapes over Trimesh; compound approximations over detailed geometry; static geometry as `mass: 0` bodies.
3. Keep `solver.iterations` at 7–15; raise only for tall stacks that won't settle.
4. Reuse `Vec3`/`Quaternion` temporaries — don't allocate inside the frame loop.
5. Merge static scenery into fewer compound bodies to cut broadphase pairs.
6. Dozens to low hundreds of dynamic bodies is cannon-es' comfort zone. Need thousands? That's an engine problem — reach for a WASM physics engine (e.g. Rapier) rather than tuning cannon-es past its limits.

## Common Pitfalls

1. **Box half-extents** — `new CANNON.Box(new CANNON.Vec3(w / 2, h / 2, d / 2))`. Passing full dimensions makes every collider double-sized. The #1 "physics feels wrong" bug.
2. **Scale is invisible to physics** — never resize colliders via `mesh.scale`; bake dimensions into shapes.
3. **Sync after step, from interpolated transforms** — syncing before `step()` shows one frame of latency; raw `position` jitters between fixed steps.
4. **Don't drive dynamic bodies by setting `position` each frame** — it fights the solver and injects contact energy. Use velocity/forces; teleport only for resets (and zero velocity).
5. **Changed mass/shapes/fixedRotation? → `updateMassProperties()`** — inertia tensors don't recompute themselves.
6. **cannon-es ≠ old cannon.js** — `applyForce`/`applyImpulse` points are center-of-mass-relative (was world-relative), `Cylinder` is Y-aligned (was Z), ESM import. Old snippets silently misbehave.
7. **No world mutation during step** — adding/removing bodies inside `collide`/`preStep` corrupts iteration. Queue, apply in `postStep`.
8. **Clamp frame delta** — after a tab switch `clock.getDelta()` returns seconds; with manual `step()` clamp to ~0.1 or maxSubSteps eats the frame. (`fixedStep()` bounds this for you.)
9. **No CCD (continuous collision detection)** — fast, small bodies tunnel through thin walls. Use thicker geometry, cap velocities, lower the step to 1/120, or add substeps.
10. **NaN poisoning** — one NaN position (huge force, bad division) spreads through the whole world. Cap applied forces; in debug builds check `Number.isNaN(body.position.x)` per frame.
11. **Static↔static and kinematic↔kinematic pairs don't collide** — moving platforms must be `KINEMATIC`.

## See Also

- `threejs-fundamentals` — scene, cameras, render loop, clock
- `threejs-geometry` — building the visual meshes these bodies drive
- `threejs-animation` — mixing physics-driven and keyframed motion
