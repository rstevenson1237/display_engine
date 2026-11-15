# Display Engine: design and build plan

- **Purpose:** Approve or correct the design of a reusable three.js game engine, then build it phase by phase from this plan.
- **For:** The repo owner, who approves the plan, and the Claude Opus and Sonnet sessions that will build each phase.
- **Version:** Draft 3, 2026-11-15. Status: proposal, waiting for approval. No engine code exists yet.
- **Instead of this:** For how art is made, read the [artgen README in image_tools](https://github.com/rstevenson1237/image_tools/blob/main/README.md). This plan only covers how the engine uses that art.

## Contents

- [1. Summary](#1-summary)
- [2. The target](#2-the-target)
- [3. What was checked before writing this](#3-what-was-checked-before-writing-this)
- [4. Decisions that need your approval](#4-decisions-that-need-your-approval)
- [5. Architecture](#5-architecture)
- [6. How the engine uses artgen](#6-how-the-engine-uses-artgen)
- [7. Build phases](#7-build-phases)
- [8. How each phase is run](#8-how-each-phase-is-run)
- [9. Risks](#9-risks)
- [10. How the plan changed across drafts](#10-how-the-plan-changed-across-drafts)
- [Appendix A. What game code looks like](#appendix-a-what-game-code-looks-like)

## 1. Summary

**The design.** One TypeScript package, `display-engine`, built on three.js. A small core holds the game state and runs
it at a fixed tick rate with no browser needed. Optional modules add rendering, assets, input, physics, navigation,
audio and a 2D menu and overlay layer. Three genre kits sit on top: first-person, third-person and real-time strategy.
Each kit is proved by a small example game. Art comes from artgen: sprites, textures and rigged 3D models load through
artgen's own runtime, and artgen's dungeon and terrain maps become playable levels.

**The build.** Sixteen phases, E0 to E15, each sized for Opus or Sonnet sessions and each ending on automated checks.
The engine is useful if you stop early: after E5 you can build a small game by hand, and after E9 a full first-person
game.

**What you need to do.** Read the ten decisions in [section 4](#4-decisions-that-need-your-approval) and say which you
accept or want changed. Everything else follows from them.

**Qualifications you should know.**

- The session estimates in section 7 are planning guesses. Nothing has been measured yet.
- Graphics speed on real hardware is not checked. This build environment renders in software.
- Version 1 leaves out multiplayer, phones and tablets, virtual reality, and a visual level editor.
- artgen can rig only humanoid models today. Vehicles and animals in 3D need static meshes, or work in artgen first.

## 2. The target

### 2.1. What you asked for

A reusable game engine built on three.js that supports several genres, among them real-time strategy, first-person
shooter and third-person games. It should use the artgen component of `image_tools` to generate models where it can.
It should handle what a full engine is expected to handle: 2D menus and overlays, events, physics, sound, and the rest.
This phase produces a finished plan that Opus and Sonnet sessions can build from.

### 2.2. What version 1 will do

- Run a game from one fixed-rate simulation that is separate from drawing, so the same game logic runs in a browser or
  in a Node test with no screen.
- Draw 3D scenes with three.js: models, sprites, lights, shadows, fog, sky, and a pixel-art display mode that matches
  artgen's look.
- Load art exported by artgen with no hand tuning of scale, facing or animation names.
- Read keyboard, mouse and gamepad through named actions that the player can rebind.
- Simulate rigid bodies, collisions, triggers and character movement.
- Find paths on grids and on navigation meshes, and steer groups of units.
- Show menus, a heads-up display (HUD), dialogs and settings as a 2D layer over the game.
- Play positional sound effects and music, with sounds defined in code so a game has audio before it has audio files.
- Load levels from data files, and build levels from artgen dungeon and terrain maps.
- Save and load a game, and record and replay a play session.
- Give debugging tools: an inspector, timing figures, and collision and path drawings.

### 2.3. What version 1 will not do

| Left out | Why | What keeps the door open |
|---|---|---|
| Multiplayer | It doubles the work of every gameplay feature | Fixed ticks, player commands as data, a seeded random number generator and state snapshots are in the core from E1 |
| Phones, tablets and touch | Each genre needs its own touch controls | Input is read through named actions, so touch is a new device module later |
| Virtual and augmented reality | The 2D layer and the cameras assume a flat screen | None planned |
| A visual level editor | Levels are data files that Claude writes, and artgen's Dungeon Map and Terrain Map tools already edit maps | Levels are plain JSON |
| The WebGPU renderer | See decision 2 in section 4 | All renderer set-up lives in one module |
| Animation blending by body part, and inverse kinematics | artgen exports whole-body clips | None planned |

## 3. What was checked before writing this

A throwaway test project was built in this session to check the riskiest assumptions. None of it is committed.

| Question | Result |
|---|---|
| Does artgen's runtime install as a normal dependency? | Yes. `artgen-runtime-1.5.0.tgz` installs from the `artgen-v0.14.0` release with npm |
| Do artgen sprites and a rigged artgen model draw in three.js 0.187? | Yes. The ghoul sprite and the warden model from the billboard-crawler example drew together in headless Chromium. The warden loaded with 4 clips, 12 bones and 3 levels of detail |
| Can the build environment test drawing? | Yes, with WebGL 2 in software. It has no WebGPU adapter, so WebGPU could not be tested here |
| Does the Rapier physics library run without a browser, and repeatably? | Yes. 200 bodies stepped 600 times in Node gave the same state hash on two runs, in about 0.6 s each. Rapier can also save its whole state |
| Does the Recast navigation library run without a browser? | Yes. It built a navigation mesh and found a path in Node |
| Does three.js maths run without a browser? | Yes. Vectors, quaternions and object transforms work in Node |

Current library versions on npm, checked today: three 0.187.0, `@dimforge/rapier3d-compat` 0.21.0, `recast-navigation`
0.44.0. `image_tools` builds with TypeScript 5.7, Vite 6, Vitest 4 and Node 22.

## 4. Decisions that need your approval

Each decision gives the recommended choice, the main alternative and the reason. Say which numbers you want changed.

### Decision 1. Version 1 is for single-player games in desktop browsers

- **Alternative:** add multiplayer or touch controls now.
- **Why:** section 2.3 gives the cost of each and what keeps it possible later.

### Decision 2. Draw with the three.js `WebGLRenderer`

- **Alternative:** the newer `WebGPURenderer`.
- **Why:** artgen's adapter is tested on WebGL, and WebGL is what this environment can test. One module owns the
  renderer, so a later switch stays possible.

### Decision 3. Hold game state in our own small entity component system

In an entity component system (ECS), entities are ids, components are plain data, and systems are functions.

- **Alternative:** a library such as bitecs or miniplex, or game objects with behaviour scripts.
- **Why:** we control save and load, iteration order and repeatability. miniplex has had no release since 2023. One
  pattern for all game logic is easier for model-written code to follow.

### Decision 4. Use Rapier for physics, and our own grid pathfinding plus Recast

- **Alternative:** Jolt for physics, or no physics library.
- **Why:** Rapier is the usual three.js choice, has a character controller, and passed the checks in section 3. Recast
  builds navigation meshes for levels that are not grids. All three are optional modules.

### Decision 5. Build menus and overlays in HTML and CSS over the canvas

- **Alternative:** menus drawn inside the 3D canvas.
- **Why:** text, layout, focus and form controls come free and look sharp. The engine requires no UI framework. A game
  may mount Svelte or any other inside a screen.

### Decision 6. Write our own thin sound layer, with sound effects defined in code

- **Alternative:** the Howler library, or three.js audio objects.
- **Why:** it is small and not tied to the scene. Claude can write a sound as parameters, the way artgen writes art as
  code.

### Decision 7. Ship one package with an entry point per module

- **Alternative:** many packages.
- **Why:** a game installs one version from a GitHub release file, as it does for artgen's libraries. A lint keeps the
  module boundaries honest instead.

### Decision 8. Adopt artgen's conventions

The conventions are: metres, y up, models face +z, and animation states named `idle`, `walk`, `attack` and `hurt`.

- **Alternative:** our own conventions.
- **Why:** art then loads with no per-asset fixes.

### Decision 9. Use the same toolchain as `image_tools`

That is TypeScript 5, Vite 6, Vitest 4, Node 22, and Playwright for browser checks.

- **Alternative:** the newest version of each tool.
- **Why:** one toolchain across your repos.

### Decision 10. Prove the engine with three example games, starting with three rough slices

- **Alternative:** build the core fully, then the games.
- **Why:** the early slices in E5 test the "any genre" claim before the kits are built on it.

## 5. Architecture

### 5.1. What each layer is responsible for

A layer may import only from the layers above it in this table. A script checks this on every commit.

| Layer | Modules | Responsibility | Needs a browser |
|---|---|---|---|
| Core | `core` | Entities, components, systems, the fixed-tick loop, events, player commands, seeded random numbers, save and restore, prefabs, level data | No |
| Simulation | `physics`, `nav`, `ai` | Bodies and collisions, pathfinding, steering, state machines and behaviour trees | No |
| Client | `render`, `assets`, `input`, `audio`, `ui`, `artgen`, `debug` | Drawing, loading, devices, sound, the 2D layer, the artgen bridge, developer tools | Yes |
| Kits | `kit-common`, `kit-fps`, `kit-tps`, `kit-rts` | Genre rules and controls. Each kit has a simulation half and a view half | View half only |
| Games | `examples/*`, your games | Content and rules for one game | Yes |

The core imports only the maths classes from three.js (`Vector3`, `Quaternion`, `Matrix4` and similar). They run in
Node, and using them everywhere avoids converting between vector types.

### 5.2. How one frame runs

1. **Input.** Read devices and turn them into named actions.
2. **Commands.** Turn actions into player commands, which are plain data stamped with a tick number.
3. **Fixed ticks.** Run zero or more ticks to catch up with real time. Each tick applies commands, runs gameplay
   systems, steps physics, then delivers the events that the tick raised.
4. **Update.** Run per-frame systems that do not change game state, such as animation playback.
5. **Cameras.** Move the camera rigs.
6. **Sync.** Copy each entity's position to its three.js object, blended between the last two ticks so motion is smooth.
7. **Render.** Draw the scene, then update the 2D layer.

The split in step 3 is the main design rule. **Game state changes only inside fixed ticks, and only from commands,
other state, and the seeded random number generator.** Drawing, sound and the 2D layer read state and react to events.
They never change it. This gives repeatable tests, replays, save and load, and a path to multiplayer later.

### 5.3. The data model

| Thing | What it is |
|---|---|
| Entity | A number. It has no data of its own |
| Component | Plain data attached to an entity, declared with a schema. The schema drives save and load, prefabs and the inspector |
| System | A function that runs in one step of the frame over the entities that have certain components |
| Resource | One shared object per world, such as the clock or the physics world |
| Event | A fact raised during a tick, such as "entity 12 took 30 damage". Listeners hear it after the tick |
| Command | A player's intent as data, such as "move this way" or "order these units there" |
| Prefab | A named set of components used to create entities: a soldier, a crate, a door |
| Level | A data file listing the environment, the prefabs to place, and the static geometry |
| Plugin | A bundle of components, systems, resources and view factories. Every module and every game is a plugin |

Systems always visit entities in ascending id order, so two runs with the same inputs give the same result.

### 5.4. What each module provides

| Module | Provides |
|---|---|
| `core` | The data model above, the frame loop, timers, a state machine helper, a spatial hash, logging |
| `render` | The renderer and scene. A view for each drawable entity. Greybox shapes (plain untextured boxes, capsules and planes) for prototyping. Lights, shadows, fog, sky. Two display profiles: `pixel` (low internal resolution, sharp upscale) and `smooth`. Camera rigs: first-person, third-person with a spring arm (a camera boom that shortens when a wall is in the way), strategy, and free. Camera shake. Debug lines |
| `assets` | A typed manifest, loading with progress, groups that load and unload with a level, and a visible placeholder for any missing asset |
| `artgen` | Section 6 |
| `input` | Keyboard, mouse with pointer lock, and gamepad. Named actions and axes. Contexts, so a menu blocks gameplay input. Rebinding that is saved |
| `physics` | Rigid bodies, collider shapes, sensors, collision layers, ray and shape casts, collision and trigger events, a character body that climbs steps and slopes |
| `nav` | Grid pathfinding (A*), flow fields (one direction map that a whole group follows), moving obstacles, and navigation meshes through Recast |
| `ai` | Steering and separation, a state machine, a small behaviour tree, sight and hearing checks |
| `audio` | Volume buses (master, music, effects, interface), sound cues with variation and voice limits, 3D positioning, music with crossfade, sounds defined as parameters, sounds triggered by game events |
| `ui` | Layers (world labels, HUD, menu, dialog, debug), a screen stack, reactive values, a widget set, keyboard and gamepad focus, a theme taken from the artgen palette, icons from artgen packs, a settings store, the game-state flow from boot to pause |
| `debug` | Timing per system, an entity inspector, collision and path drawings, pause and single-step, a command console, and a test hook for browser checks |
| `kit-common` | Health and damage, teams, weapons, projectiles, pickups, triggers, spawners |
| `kit-fps` | First-person movement and look, a weapon view drawn over the scene, instant-hit and projectile weapons, enemy behaviour, a HUD |
| `kit-tps` | Camera-relative movement, the spring-arm camera, melee attacks timed from clip data, target lock |
| `kit-rts` | The strategy camera, click and box selection, control groups, order queues, group movement with formations, buildings and placement, production, resources, fog of war, a minimap, a simple computer opponent |

## 6. How the engine uses artgen

### 6.1. What the bridge does

The `artgen` module wraps `artgen-runtime` and its three.js adapter. Game code names an asset. The bridge does the rest.

| artgen output | Engine component | What the bridge handles |
|---|---|---|
| Sprite in an atlas pack | `ArtSprite` | Drawing it as a billboard (a flat picture that turns to face the camera), sized in metres, the facing picked from the camera angle, animation state, lit sprites with normal maps |
| Rigged model (`artgen rig`) | `ArtModel` | Clip playback from the entity's animation state, crossfades, level of detail by camera distance |
| Static mesh (`artgen mesh`) and voxel model | `ArtModel` | Loading, and a collider from the model's bounds |
| Tile and texture | `ArtSurface` | Repeating textures on floors, walls and terrain |
| Effect | `ArtEffect` | Spawning at a point or on an entity's named anchor |
| Interface icon | The `ui` icon widget | The frame cut from the atlas as a CSS sprite |
| Palette in `art/direction.json` | The `ui` theme | CSS variables for colours |
| Dungeon, terrain and settlement maps | Level builders (E10) | Section 6.3 |

### 6.2. One command brings art into the game

A script, `npm run art:sync`, runs the artgen exports and writes one typed manifest:

1. Run `artgen export` for the atlas packs.
2. Run `artgen rig` or `artgen mesh` for each asset the game lists as a 3D model.
3. Read each model file for its clips, clip lengths, height and bounds.
4. Read the pixels-per-metre scale from the art direction.
5. Write `src/assets.gen.ts` with typed ids for every pack asset and model, and a theme stylesheet.

The simulation reads clip lengths and model bounds from that manifest. It never has to load a model file, so tests
still run without a browser.

**Art is never a blocker.** Before an asset exists, the engine draws a greybox shape at the size the brief gives. With
`artgen export --placeholders`, sprites show artgen's own placeholders. Real art replaces both later under the same ids.

### 6.3. Maps become levels

artgen's map engine already turns a dungeon, terrain map or settlement into a grid of classed cells with named objects
(the `TileGrid` described in [TILEMAPS.md in image_tools](https://github.com/rstevenson1237/image_tools/blob/main/docs/TILEMAPS.md)).
The level builders in E10 read that grid:

- **Dungeon to interior level.** Floor, wall and door cells become merged meshes textured with artgen tiles, with
  colliders and a path grid. Rooms, doors and symbols become entities, such as spawn points and triggers.
- **Terrain map to outdoor level.** The height field becomes a terrain mesh with a height-field collider. Terrain
  classes set the ground texture and the movement cost.
- **Settlement to placed buildings.** Building footprints place building prefabs. This part is optional in E10.

So `/artgen-dungeon a flooded crypt with eight rooms` can end as a level you walk through.

### 6.4. What would help from artgen later

None of these block the engine. Each one removes a workaround. You own both repos, so they are listed as requests.

| Request | Workaround until then |
|---|---|
| Export models in bulk and list them in the generated `assets.ts` | The `art:sync` script does it |
| Rigged templates for quadrupeds, flyers and vehicles with turrets | Static meshes, with parts as separate entities (a tank hull and a turret) |
| Levels of detail for static meshes | One mesh; distant props fall back to sprites in E14 |
| Stride length per walk cycle in the clip data | Walk clip speed is tuned by hand per unit type |
| A named contact time in attack clips | The kit takes a fraction of the clip length |
| A collision shape in the model file | A box or capsule from the bounds |

## 7. Build phases

Sizes follow the artgen plans: S is about one session, M is two or three, L is four or more. "Windows" are five-hour
usage windows, as in the artgen tooling plan, and are guesses. Every phase ends with all checks green (section 8.2).
The tick budget is 4 ms of simulation work per tick, measured in Node, unless a phase says otherwise.

| Phase | What it builds | Size | Model | Windows |
|---|---|---|---|---|
| E0 | Repo scaffold and guard rails | S | Sonnet | 1–2 |
| E1 | Core | M | Opus designs, Sonnet builds | 4–6 |
| E2 | Render, assets, artgen bridge | M–L | Opus designs, Sonnet builds | 5–8 |
| E3 | Input and cameras | M | Sonnet | 3–4 |
| E4 | Physics and character movement | M–L | Opus for the character body | 4–7 |
| E5 | Three greybox slices | M | Sonnet builds, Opus reviews | 3–5 |
| E6 | 2D layer and game-state flow | M | Sonnet | 4–6 |
| E7 | Audio | S–M | Sonnet | 2–4 |
| E8 | Navigation and AI | M–L | Opus for flow fields and steering | 5–8 |
| E9 | First-person kit and example game | M–L | Opus designs, Sonnet builds | 5–8 |
| E10 | Levels from data and from artgen maps | M–L | Opus designs, Sonnet builds | 4–7 |
| E11 | Third-person kit and example game | M | Sonnet | 3–5 |
| E12 | Strategy kit and example game | L | Opus designs and builds movement | 8–14 |
| E13 | Save, load and replay | S–M | Sonnet | 2–4 |
| E14 | Effects and performance | M | Sonnet | 4–6 |
| E15 | Packaging for game repos | M | Sonnet | 3–5 |

The total is roughly 60 to 100 windows. Order: E0 → E1 → E2 → E3 and E4 → E5 → E6, E7 and E8 in any order → E9. After
E9, phases E10 to E15 can be reordered to suit the first real game. For example, pull E15 forward to start a game repo
early.

**Art for the examples** is separate work done with artgen: one art direction for the repo, then a batch of assets per
example game. It can start any time after E2. Until then the examples use greybox shapes and the billboard-crawler's
existing assets as fixtures.

### E0. Scaffold the repo and its guard rails (S)

- **Builds:** npm workspace (`packages/engine`, `examples/*`). TypeScript, Vite and Vitest set up. The boundary lint
  for section 5.1. A Playwright harness that serves an example, drives it and takes a screenshot. Continuous
  integration (CI). `CLAUDE.md` and an engine-development skill that state the rules in section 8.
- **Accept:** `npm run check`, `npm test`, `npm run lint:boundaries` and `npm run accept:smoke` pass locally and in CI.
  The smoke test draws the section 3 scene and compares it to a stored picture.
- **Makes later work faster:** every later phase has its checks from the first commit.

### E1. Build the core with no browser (M)

- **Builds:** entities, components with schemas, queries, add and remove hooks, the system scheduler, the fixed-tick
  clock, events, commands, the seeded random number generator, resources, plugins, snapshot and restore, prefabs, level
  data, and a rule check that bans clocks, `Math.random` and browser objects in simulation code.
- **Accept:** unit tests for each part. Two runs with the same seed and commands end with the same state hash. A run
  that is saved, restored and continued matches an unbroken run. 10,000 entities and 20 queries cost under 1 ms of
  overhead a tick in Node.
- **Makes later work faster:** every module is written against one fixed contract.

### E2. Draw the world and load art (M–L)

- **Builds:** `render` (backend, views with tick blending, greybox shapes, lights, environment, both display profiles,
  debug lines), `assets`, the `artgen` bridge, the `art:sync` script, an animation-state component, and the sandbox
  example.
- **Accept:** the sandbox shows greybox shapes, the warden walking a path and the ghoul sprite turning with the camera,
  matched against stored pictures. 1,000 moving entities hold the tick budget. Loading then unloading a level returns
  the renderer's memory counts to where they started.
- **Makes later work faster:** anything built after this can be seen.

### E3. Read input and move cameras (M)

- **Builds:** `input` (devices, actions, contexts, rebinding, pointer lock) and the four camera rigs with shake and
  blending between rigs.
- **Accept:** Playwright sends keys and mouse moves and the right actions fire. Rig maths is unit tested. The sandbox
  switches rigs on a key.

### E4. Add physics and character movement (M–L)

- **Builds:** `physics` (Rapier world, body and collider components, state sync both ways, casts, events, layers,
  debug drawing, state save) and the character body. The third-person rig gains camera collision.
- **Accept:** in Node: a stack of boxes settles, the state hash repeats, and a character climbs steps and slopes
  within its limits and never passes through a wall at speed. In the browser: walking round a greybox room.

### E5. Build three rough slices to test the design (M)

- **Builds:** three tiny greybox games using only E1 to E4. First-person: walk and shoot a ray at targets.
  Third-person: run round obstacles with the orbit camera. Strategy: pan the camera, box-select 20 units and order them
  to move on flat ground. Then a findings note, and the core changes it calls for.
- **Accept:** a Playwright script plays each slice to its goal. `docs/findings/E5.md` lists what was awkward and what
  changed.
- **Makes later work faster:** the kits are built on an interface that three genres have already used.

### E6. Add the 2D layer and the game-state flow (M)

- **Builds:** `ui` as listed in section 5.4, with a main menu, pause menu, settings (key bindings, volume, display) and
  a HUD in the sandbox. The flow from boot to loading, menu, play and pause.
- **Accept:** Playwright works the menus with the keyboard alone. Pausing stops ticks. A rebound key survives a
  reload. A click on the 2D layer does not reach the game.

### E7. Add audio (S–M)

- **Builds:** `audio` as listed in section 5.4, with a starter set of code-defined sounds.
- **Accept:** a code-defined sound renders to the same non-silent samples each time. In the browser a game event
  produces output on its bus, and the volume setting changes it. With no audio device the module does nothing and
  nothing fails.

### E8. Add navigation and AI (M–L)

- **Builds:** `nav` and `ai` as listed in section 5.4, with debug drawings.
- **Accept:** in Node: correct shortest paths on test maps, 200 agents pass a narrow gap and arrive within the tick
  budget, and the run repeats exactly. A navigation mesh path crosses a greybox level with stairs.

### E9. Build the first-person kit and its example game (M–L)

- **Builds:** `kit-common`, `kit-fps`, and a small crypt game: enemies, a weapon, a pickup, a door and an exit.
- **Accept:** a scripted player clears a room of three enemies, takes the pickup, opens the door and reaches the exit.
  A recorded session replays to the same end state.

### E10. Load levels from data and from artgen maps (M–L)

- **Builds:** the full level format, a prefab library, import of a glTF scene (the 3D file format artgen exports) whose nodes name prefabs, and the level
  builders in section 6.3. Moving between levels.
- **Accept:** an artgen dungeon file loads as a first-person level that the E9 scripted player can cross. An artgen
  terrain map loads as terrain that units can path across. Both are tested with fixture maps.

### E11. Build the third-person kit and its example game (M)

- **Builds:** `kit-tps` and a small arena game with waves of enemies.
- **Accept:** a scripted player survives a wave using movement, attack and target lock. The camera never clips through
  a wall. The session replays to the same end state.

### E12. Build the strategy kit and its example game (L)

- **Builds:** `kit-rts` as listed in section 5.4, a batch renderer for health bars, and a skirmish game on an artgen
  terrain map. The simulation runs at 30 ticks a second.
- **Accept:** in Node, a scripted match of two sides with 50 units each runs to the same result every time within the
  tick budget. In the browser, a scripted player selects units, gives orders, builds and wins a small scenario.

### E13. Save, load and replay (S–M)

- **Builds:** save slots in browser storage, versioned save data with migration, replay recording and playback in the
  debug tools, and a replay check for each example in CI.
- **Accept:** each example saves mid-game, reloads and continues to the same end state as an unbroken run.

### E14. Add effects and tune performance (M)

- **Builds:** particle emitters, artgen flipbook effects on anchors, batching of repeated static props, sprite
  stand-ins for distant models, shadow tuning, and budgets shown in the debug overlay.
- **Accept:** each example holds its stated budgets for draw calls and tick time. Frame rate on real hardware is
  measured by you, because this environment cannot measure it.

### E15. Package the engine for game repos (M)

- **Builds:** a `create-game` scaffold (Vite, the engine, the artgen install, `CLAUDE.md` and a game-building skill),
  a release workflow that attaches the package file, a changelog, and generated reference pages.
- **Accept:** in a clean container, the scaffold makes a new repo where `npm run dev` shows a walking placeholder
  character and `npm run art:sync` runs.

## 8. How each phase is run

### 8.1. One phase per session, from a written design

1. **Design.** An Opus session writes `docs/design/E<n>-<name>.md`: the exact interfaces, the files, the tests and
   anything this plan left open. Small phases marked Sonnet skip straight to step 2 with a short note.
2. **Build.** Sonnet sessions build against that note, one sub-step per session, committing as checks pass.
3. **Record.** The same note gains three closing parts, as the artgen design notes have: what was built, where it
   differs from the design, and what was not done.

Start each session from a short brief that names the phase. Use no subagents or parallel runs. Let the checks find
mistakes instead of re-reading files.

### 8.2. The checks every phase must pass

| Check | Command | What it proves |
|---|---|---|
| Types | `npm run check` | The code compiles under strict TypeScript |
| Unit and simulation tests | `npm test` | Logic is right, with no browser |
| Boundaries | `npm run lint:boundaries` | No layer imports from a lower one, and simulation code uses no clocks or browser objects |
| Browser acceptance | `npm run accept:<name>` | The real page runs, drawn in headless Chromium and driven by script |
| Repeatability | part of `npm test` | Same seed and commands give the same state hash |

### 8.3. Rules for all engine code

- Game state changes only in fixed ticks (section 5.2).
- Game logic is written as systems. There is no second way.
- A feature moves from a kit into a shared module only when a second kit needs it.
- Every drawable and every loaded asset is released when its entity or level goes. Tests check for leaks.
- A missing asset shows a placeholder and a warning. It never stops the game.

## 9. Risks

| Risk | Effect | What reduces it |
|---|---|---|
| The scope is large | The work stalls before a game ships | Phases are useful alone. E5 gives a usable base, E9 a whole genre |
| "Any genre" abstractions turn out wrong | Rework in the kits | E5 tests three genres early. The second-kit rule in 8.3 |
| Many skinned units are slow | The strategy game misses its frame rate | Levels of detail from artgen, sprite stand-ins in E14, a 1,000-entity gate in E2. Real hardware must still be measured |
| Character movement feels wrong | First- and third-person games feel poor | Opus builds it in E4 with step, slope and speed tests. Feel is tuned by you in the sandbox |
| artgen rigs only humanoids | Strategy units and animals lack animation | Static meshes with moving parts, or the artgen requests in 6.4 |
| No audio files exist | Games are silent | Code-defined sounds in E7. Music still needs files from elsewhere |
| Trigonometry differs between browsers | Replays differ across browsers | Only matters for multiplayer, which is out of scope. Replays are checked on one browser |
| Libraries move on | Builds break | E0 pins exact versions. Updates are their own small tasks |

## 10. How the plan changed across drafts

**Draft 1** set out the layers, the modules and a core-first build order, with the three kits at the end.

**Draft 2** was a review of draft 1 for scope and for the "any genre" claim. It made five changes:

- Added E5, the three greybox slices, so three genres use the core before any kit is designed.
- Removed behaviour scripts. Systems are now the only way to write game logic.
- Split each kit into a simulation half and a view half, so kit rules can be tested without a browser.
- Made code-defined sounds part of the audio phase, because nothing else can produce sound effects without files.
- Moved prefabs and level data into the core, so example games do not wait for the level phase.

**Draft 3** was a review against what a full engine needs and against the checks in section 3. It made four changes:

- Chose `WebGLRenderer` after finding no WebGPU adapter in the test environment.
- Moved the map-to-level phase to straight after the first-person kit, where it pays off first.
- Added the memory-leak check, the missing-asset placeholder rule and the 1,000-entity gate.
- Added the `art:sync` manifest so the simulation can read clip lengths without loading models.

## Appendix A. What game code looks like

This sketch shows the intended feel. Exact names are fixed in the E1 and E2 design notes.

```ts
import { createEngine, defineComponent, defineSystem, f32, type Plugin } from 'display-engine/core';
import { render } from 'display-engine/render';
import { assets } from 'display-engine/assets';
import { artgen } from 'display-engine/artgen';
import { input } from 'display-engine/input';
import { physics } from 'display-engine/physics';
import { audio } from 'display-engine/audio';
import { ui } from 'display-engine/ui';
import { fps } from 'display-engine/kit-fps';
import { manifest } from './assets.gen';

// A component is plain data with a schema.
export const Health = defineComponent('Health', { hp: f32(100), max: f32(100) });

// A system is a function that runs in one step of the frame.
const regen = defineSystem({
  name: 'regen',
  phase: 'fixed',
  run(world, dt) {
    for (const e of world.query(Health)) {
      const h = world.get(e, Health);
      h.hp = Math.min(h.max, h.hp + 2 * dt);
    }
  },
});

// A game is a plugin.
const game: Plugin = {
  name: 'crypt',
  components: [Health],
  systems: [regen],
  prefabs: {
    warden: { Transform: {}, ArtModel: { asset: 'warden' }, Animator: { state: 'walk' }, Health: { hp: 60, max: 60 } },
  },
};

const engine = await createEngine({
  canvas: document.querySelector('canvas')!,
  tickRate: 60,
  seed: 1,
  plugins: [render({ profile: 'pixel', height: 270 }), assets({ manifest }), artgen(), input(), physics(), audio(), ui(), fps(), game],
});
await engine.loadLevel('crypt-1');
engine.start();
```
