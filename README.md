# cannon-es-skills

AI-agent skills for [cannon-es](https://github.com/pmndrs/cannon-es) — the maintained ESM rigid-body physics engine — paired with [three.js](https://github.com/mrdoob/three.js).

The `cannon-es` skill covers world setup, bodies, shapes, materials and contact materials, constraints, collision events and triggers, raycasting, fixed-timestep stepping with interpolation, sleeping, the three.js mesh-sync pattern, performance tips, and 11 common pitfalls (half-extents, scale invisibility, NaN poisoning, tunneling, delta spikes, and more).

Every API claim was verified against the cannon-es source and docs (October 2026), including the details that differ from old cannon.js snippets: Y-aligned `Cylinder`, center-of-mass-relative `applyForce`, and the fact that `fixedStep()` updates `interpolatedPosition`/`interpolatedQuaternion`.

## Install

With the [skills CLI](https://skills.sh):

```bash
npx skills add erikoey/cannon-es-skills
```

or a single skill:

```bash
npx skills add erikoey/cannon-es-skills@cannon-es
```

## Skills

| Skill | Description |
| --- | --- |
| [`cannon-es`](skills/cannon-es/SKILL.md) | Rigid-body physics for three.js — worlds, bodies, shapes, materials, constraints, collision events, raycasting, stepping, sleeping, mesh sync, pitfalls |

## Related

- [cloudai-x/threejs-skills](https://github.com/cloudai-x/threejs-skills) — companion three.js skill suite (fundamentals, geometry, materials, shaders, …)

## License

[MIT](LICENSE)
