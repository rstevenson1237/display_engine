# Display Engine: design and build plan

- **Purpose:** Approve or correct the design of a reusable three.js game engine, then build it phase by phase from this plan.
- **For:** The repo owner, who approves the plan, and the Claude Opus and Sonnet sessions that will build each phase.
- **Version:** Draft 4, 2026-11-15. Status: proposal, waiting for approval of the new parts. No engine code exists yet.
- **Instead of this:** For how art is made, read the [artgen README in image_tools](https://github.com/rstevenson1237/image_tools/blob/main/README.md). This plan only covers how the engine uses that art.

## Contents

- [1. Summary](#1-summary)
- [2. The target](#2-the-target)
- [3. What was checked before writing this](#3-what-was-checked-before-writing-this)
- [4. Decisions](#4-decisions)
- [5. Architecture](#5-architecture)
- [6. Content lives in KDL files](#6-content-lives-in-kdl-files)
- [7. Assets are generated from code](#7-assets-are-generated-from-code)
- [8. What runs on the graphics card](#8-what-runs-on-the-graphics-card)
- [9. Beyond your brief: what was added and what to consider](#9-beyond-your-brief-what-was-added-and-what-to-consider)
- [10. Build phases](#10-build-phases)
- [11. How each phase is run](#11-how-each-phase-is-run)
- [12. Risks](#12-risks)
- [13. How the plan changed across drafts](#13-how-the-plan-changed-across-drafts)
- [Appendix A. What game code looks like](#appendix-a-what-game-code-looks-like)
- [Appendix B. What content files look like](#appendix-b-what-content-files-look-like)

## 1. Summary

**The design.** One TypeScript package, `display-engine`, built on three.js. A small core holds the game state and runs
it at a fixed tick rate with no browser needed. Optional modules add rendering, assets, input, physics, navigation,
audio and a 2D menu and overlay layer. Three genre kits sit on top: first-person, third-person and real-time strategy.
Each kit is proved by a small example game.

**Three rules shape a game built on it.**

- **Code holds the rules, content files hold everything else.** All content sits in a `content/` folder as KDL files
  (KDL is a small document language, described at [kdl.dev](https://kdl.dev)): entity templates, levels, input
  bindings, sounds, menus, text and tuning numbers.
- **Assets are generated from code, not stored as libraries.** Models, sprites, textures, levels and sounds come from
  short recipes and seeds. artgen makes the art. The repo holds recipes, and the build or the game makes the files.
- **Drawing runs on the graphics card.** The engine draws through WebGL 2, which is hardware accelerated in every
  current desktop browser.

**The build.** Eighteen phases, E0 to E17, each sized for Opus or Sonnet sessions and each ending on automated checks.
The engine is useful if you stop early: after E6 you can build a small game by hand, and after E11 a full first-person
game.

**What you need to do.**

1. Approve or change decisions 11 to 13 in [section 4](#4-decisions). They cover KDL, generated assets and the artgen
   work this needs.
2. Read [section 9](#9-beyond-your-brief-what-was-added-and-what-to-consider). Say whether to drop anything I added,
   and whether to pull in anything I only listed.

**Qualifications you should know.**

- The session estimates in section 10 are planning guesses. Nothing has been measured yet.
- Graphics speed on real hardware is not checked. The build environment draws in software, although players' browsers
  will use the graphics card. Phase E16 adds a benchmark page for you to run.
- Generating artgen assets while the game loads needs changes in `image_tools` (section 7.3). Until then the engine
  generates them at build time, which works today.
- Version 1 leaves out multiplayer, phones and tablets, virtual reality, and a visual level editor.
- artgen can rig only humanoid models today. Vehicles and animals in 3D need static meshes, or work in artgen first.

## 2. The target

### 2.1. What you asked for

A reusable game engine built on three.js that supports several genres, among them real-time strategy, first-person
shooter and third-person games. It should use the artgen component of `image_tools` to generate models where it can.
It should handle what a full engine is expected to handle: 2D menus and overlays, events, physics, sound, and the rest.

You then added three constraints:

- All content goes in a `content/` folder as KDL files.
- The engine should take advantage of WebGL and hardware acceleration.
- Assets should lean heavily on procedural generation in code, not on large content libraries, with artgen changed
  or extended where needed.

### 2.2. What version 1 will do

- Run a game from one fixed-rate simulation that is separate from drawing, so the same game logic runs in a browser or
  in a Node test with no screen.
- Read every piece of content from KDL files, check it against a schema, and reload it in a running game during
  development.
- Generate assets from recipes and seeds: at build time, while a level loads, or both.
- Draw 3D scenes on the graphics card: models, sprites, lights, shadows, fog, sky, and a pixel-art display mode that
  matches artgen's look.
- Load art made by artgen with no hand tuning of scale, facing or animation names.
- Read keyboard, mouse and gamepad through named actions that the player can rebind.
- Simulate rigid bodies, collisions, triggers and character movement.
- Find paths on grids and on navigation meshes, and steer groups of units.
- Show menus, a heads-up display (HUD), dialogs and settings as a 2D layer over the game.
- Play positional sound effects and music, with sound effects generated from parameters.
- Build levels from data, from artgen dungeon and terrain maps, or from a seed while the game runs.
- Save and load a game, and record and replay a play session.
- Give debugging tools: an inspector, timing figures, and collision and path drawings.

### 2.3. What version 1 will not do

| Left out | Why | What keeps the door open |
|---|---|---|
| Multiplayer | It doubles the work of every gameplay feature | Fixed ticks, player commands as data, seeded random numbers and state snapshots are in the core from E1 |
| Phones, tablets and touch | Each genre needs its own touch controls | Input is read through named actions, so touch is a new device module later |
| Virtual and augmented reality | The 2D layer and the cameras assume a flat screen | None planned |
| A visual level editor | Levels are KDL files that Claude writes, and artgen's map tools already edit maps | The KDL parser can rewrite a file and keep its comments, so an editor can come later |
| The WebGPU renderer | Section 8.3 | All renderer set-up lives in one module |
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
| Is there a working KDL parser? | Yes. `@bgotink/kdl` 0.4.2 parsed the files in Appendix B as KDL version 2. A broken file gave an error with its line and column |
| Can artgen generate a model from code inside a game, not only from its command line? | Partly. The released `artgen-scene` library built three textured prop meshes from code in 34 to 91 ms each in Node. Running a full artgen asset recipe needs `artgen-core`, which is written to run in a browser but is not released as a library yet |
| Can maps be generated inside a game? | The `artgen-mapgen` library is released and installable. Generating a dungeon from a seed inside a browser is not yet tested |

Current library versions on npm, checked today: three 0.187.0, `@dimforge/rapier3d-compat` 0.21.0, `recast-navigation`
0.44.0, `@bgotink/kdl` 0.4.2. `image_tools` builds with TypeScript 5.7, Vite 6, Vitest 4 and Node 22.

## 4. Decisions

### 4.1. Decisions you accepted on 2026-11-15

1. **Scope.** Version 1 is for single-player games in desktop browsers.
2. **Renderer.** Draw with the three.js `WebGLRenderer`. Section 8 says what this gives you.
3. **Game state.** Our own small entity component system (ECS): entities are ids, components are plain data, and
   systems are functions.
4. **Physics and navigation.** Rapier for physics. Our own grid pathfinding, plus Recast for navigation meshes.
5. **Menus and overlays.** HTML and CSS over the canvas, with no UI framework required.
6. **Sound.** Our own thin layer over the Web Audio API, with sound effects defined as parameters.
7. **Packaging.** One package with an entry point per module, installed from a GitHub release file.
8. **Conventions.** artgen's: metres, y up, models face +z, animation states named `idle`, `walk`, `attack`, `hurt`.
9. **Toolchain.** The same as `image_tools`: TypeScript 5, Vite 6, Vitest 4, Node 22, Playwright for browser checks.
10. **Proof.** Three example games, with three rough slices built early in E6.

### 4.2. New decisions that need your approval

#### Decision 11. All content is KDL version 2, checked and compiled at build time

- **Recommended:** content files are checked against schemas that the engine modules register, then compiled into one
  data file that the game loads. In development, a changed file reloads in the running game. Parser: `@bgotink/kdl`.
- **Alternative:** parse KDL in the browser every time the game starts.
- **Why:** a mistake is caught by a command, with the file, line and column, before anyone runs the game. The game
  ships no parser and starts faster. Section 6 has the detail.

#### Decision 12. The repo holds recipes, and each asset says when it is generated

- **Recommended:** every asset is a recipe plus a seed. Each asset names one of two times to generate it. *Build*
  makes a file when the game is built: fast to start, one fixed result. *Load* makes it in a background worker when a
  level loads, and caches it: slower the first time, but one recipe gives many variants. Generated files are build
  outputs. Committing them is optional.
- **Alternative:** build time only, as artgen works today.
- **Why:** build time works now and keeps start-up fast. Load time is what makes "every goblin different" and
  "a new dungeon each run" possible without storing each result. Section 7 has the detail.

#### Decision 13. artgen gains a generation library and wider 3D coverage

- **Recommended:** approve the work packages in section 7.3 as requests on `image_tools`. The engine does not wait
  for them. Each one unlocks a named engine feature when it lands.
- **Alternative:** the engine carries its own mesh and texture generator.
- **Why:** artgen already has the art direction, the palette, the meshing, the rigs and the review loop. A second
  generator would drift from it in look and scale.

## 5. Architecture

### 5.1. What each layer is responsible for

A layer may import only from the layers above it in this table. A script checks this on every commit.

| Layer | Modules | Responsibility | Needs a browser |
|---|---|---|---|
| Core | `core`, `content` | Entities, components, systems, the fixed-tick loop, events, player commands, seeded random numbers, save and restore, and content: prefabs, levels and tables read from compiled KDL | No |
| Simulation | `physics`, `nav`, `ai`, `procgen` (generators) | Bodies and collisions, pathfinding, steering, behaviour, and making levels and terrain from seeds | No |
| Client | `render`, `assets`, `input`, `audio`, `ui`, `artgen`, `procgen` (workers and cache), `debug` | Drawing, loading, devices, sound, the 2D layer, the artgen bridge, asset generation, developer tools | Yes |
| Kits | `kit-common`, `kit-fps`, `kit-tps`, `kit-rts` | Genre rules and controls. Each kit has a simulation half and a view half | View half only |
| Games | `examples/*`, your games | Systems in TypeScript and content in KDL for one game | Yes |

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
| Component | Plain data attached to an entity, declared with a schema. The schema drives KDL checking, save and load, and the inspector |
| System | A function that runs in one step of the frame over the entities that have certain components |
| Resource | One shared object per world, such as the clock or the physics world |
| Event | A fact raised during a tick, such as "entity 12 took 30 damage". Listeners hear it after the tick |
| Command | A player's intent as data, such as "move this way" or "order these units there" |
| Prefab | A named set of components used to create entities: a soldier, a crate, a door. Written in KDL |
| Table | Named rows of tuning data: weapons, unit types, loot, enemy waves. Written in KDL |
| Level | The environment, the things to place or generate, and the rules that react to events. Written in KDL |
| Plugin | A bundle of components, systems, resources, content kinds and view factories. Every module and every game is a plugin |

Systems always visit entities in ascending id order, so two runs with the same inputs give the same result.

### 5.4. What each module provides

| Module | Provides |
|---|---|
| `core` | The data model above, the frame loop, timers, a state machine helper, a spatial hash, logging |
| `content` | Section 6 |
| `procgen` | Section 7 |
| `render` | The renderer and scene. A view for each drawable entity. Greybox shapes (plain untextured boxes, capsules and planes) for prototyping. Lights, shadows, fog, sky. Two display profiles: `pixel` (low internal resolution, sharp upscale) and `smooth`. Camera rigs: first-person, third-person with a spring arm (a camera boom that shortens when a wall is in the way), strategy, top-down and isometric, and free. Camera shake. Debug lines. Recovery when the browser drops the graphics context |
| `assets` | A typed manifest, loading with progress, groups that load and unload with a level, and a visible placeholder for any missing asset |
| `artgen` | The bridge in section 7.2 |
| `input` | Keyboard, mouse with pointer lock, and gamepad. Named actions and axes. Contexts, so a menu blocks gameplay input. Rebinding that is saved |
| `physics` | Rigid bodies, collider shapes, sensors, collision layers, ray and shape casts, collision and trigger events, a character body that climbs steps and slopes |
| `nav` | Grid pathfinding (A*), flow fields (one direction map that a whole group follows), moving obstacles, and navigation meshes through Recast |
| `ai` | Steering and separation, a state machine, a small behaviour tree, sight and hearing checks |
| `audio` | Volume buses (master, music, effects, interface), sound cues with variation and voice limits, 3D positioning, music with crossfade, sounds generated from parameters, sounds triggered by game events, captions for cues |
| `ui` | Layers (world labels, HUD, menu, dialog, debug), a screen stack, reactive values, a widget set, keyboard and gamepad focus, a theme taken from the artgen palette, icons from artgen packs, text in more than one language, a settings store, the game-state flow from boot to pause |
| `debug` | Timing per system, an entity inspector, collision and path drawings, pause and single-step, a command console, a test hook for browser checks, and a command-line tool for sessions |
| `kit-common` | Health and damage, teams, weapons, projectiles, pickups, triggers, spawners |
| `kit-fps` | First-person movement and look, a weapon view drawn over the scene, instant-hit and projectile weapons, enemy behaviour, a HUD |
| `kit-tps` | Camera-relative movement, the spring-arm camera, melee attacks timed from clip data, target lock |
| `kit-rts` | The strategy camera, click and box selection, control groups, order queues, group movement with formations, buildings and placement, production, resources, fog of war, a minimap, a simple computer opponent |

## 6. Content lives in KDL files

### 6.1. The line between code and content

| Goes in TypeScript | Goes in `content/*.kdl` |
|---|---|
| Systems: the rules of the game | Prefabs: what a thing is made of |
| New component types | Values for those components |
| New action types for level rules | Level rules that use them |
| New generators | Which generator a level or asset uses, with its seed and settings |
| Custom widgets and complex screens | Menus, HUD layout, themes, all player-facing text |
| | Input actions and default bindings |
| | Sound cues and music lists |
| | Tables: weapons, units, items, waves, loot, technology trees |
| | Game settings: title, start level, tick rate, display profile, which kits to load |

Two kinds of file stay outside `content/`. artgen's own files stay in `art/`, because artgen's tools own them. Asset
recipes stay as code, because a recipe is a program.

### 6.2. How the folder is laid out

```text
content/
  game.kdl          title, start level, tick rate, kits, display profile
  input.kdl         actions and default bindings
  audio.kdl         sound cues and music
  prefabs/          one file per thing or family of things
  levels/           one file per level
  tables/           weapons, units, items, waves, loot
  ui/               screens, HUD layout, theme
  text/             one file per language
```

Appendix B shows what the files look like.

### 6.3. How a content file reaches the game

1. **Register.** Each engine module and each game registers the node kinds it understands. A component's schema is
   its KDL schema, so `health hp=60 max=60` is checked against the `Health` component with no extra work.
2. **Check.** `npm run content:check` parses every file and reports each mistake with its file, line and column, and
   what was expected. An unknown node, a misspelt property and a reference to a prefab or asset that does not exist
   are all errors.
3. **Compile.** The build writes one data file, and a TypeScript file of typed ids, so code that names a prefab or a
   level is checked by the compiler.
4. **Reload.** In development, saving a KDL file updates the running game. Changed prefabs and tables apply to live
   entities where that is safe. A changed level reloads the level.
5. **Document.** The same schemas generate `docs/content-reference.md`, the list of every node and property. A session
   writing content reads that file, so it does not guess.

### 6.4. Rules that keep content simple

- **KDL holds data, not logic.** Level rules are "when this event happens, do these actions" from a fixed list that
  plugins extend. There are no expressions, variables or loops. Anything that needs them is a system in TypeScript.
- **A prefab can extend another prefab.** Content is layered: engine defaults, then kit defaults, then the game. A
  later layer overrides an earlier one by name. This is also how content packs and mods would load later.
- **Saved games record the content version.** Phase E15 adds migration when a schema changes.

## 7. Assets are generated from code

### 7.1. What is generated, and when

| Asset | Recipe | Generator | Generated at |
|---|---|---|---|
| Sprites, tiles, textures, effects, icons | artgen asset module | artgen | Build today. Load after artgen work A1 |
| Static and rigged models | artgen asset module | artgen | Build today. Load after A1 |
| Variants of one model (size, colours, parts) | The same module with other parameters or seed | artgen | Load, after A1 and A3 |
| Colour swaps and team colours | A palette row | The graphics card (section 8.2) | Every frame, at no extra cost |
| Dungeons and settlements | Seed and settings in the level file | `artgen-mapgen` library | Build or load |
| Outdoor terrain | Seed and settings, or an artgen terrain map | Engine noise generator, or `artgen-mapgen` | Build or load |
| Scattered props and vegetation | Density rules in the level file | Engine | Load |
| Sky, fog, lighting | Settings in the level file | Engine shaders | Every frame |
| Sound effects | Parameters in `audio.kdl` | Engine sound generator | Load |
| Names, loot, enemy waves | Tables in KDL plus the game seed | Engine | During play |

How generation at load time works:

1. A level lists what it needs. Each need is a recipe id, parameters and a seed.
2. The engine hashes those three with the generator version. If the cache holds that hash, it uses the cached result.
3. Otherwise a pool of background workers runs the generators, so the page stays responsive, and a progress bar shows.
4. Results go to the graphics card and into the browser's cache for the next visit.

Every seed derives from the one game seed. The same game seed gives the same world on every run.

**Art is never a blocker.** Before a recipe exists, the engine draws a greybox shape at the size the content file
gives. Real art replaces it later under the same id.

### 7.2. How the engine uses artgen today

The `artgen` module wraps `artgen-runtime` and its three.js adapter. Content names an asset. The bridge does the rest.

| artgen output | Engine component | What the bridge handles |
|---|---|---|
| Sprite in an atlas pack | `ArtSprite` | Drawing it as a billboard (a flat picture that turns to face the camera), sized in metres, the facing picked from the camera angle, animation state, lit sprites with normal maps |
| Rigged model (`artgen rig`) | `ArtModel` | Clip playback from the entity's animation state, crossfades, level of detail by camera distance |
| Static mesh (`artgen mesh`) and voxel model | `ArtModel` | Loading, and a collider from the model's bounds |
| Tile and texture | `ArtSurface` | Repeating textures on floors, walls and terrain |
| Effect | `ArtEffect` | Spawning at a point or on an entity's named anchor |
| Interface icon | The `ui` icon widget | The frame cut from the atlas as a CSS sprite |
| Palette in `art/direction.json` | The `ui` theme | CSS variables for colours |
| Dungeon, terrain and settlement maps | Level builders (E12) | Cells become floors, walls, doors, terrain, colliders and a path grid. Rooms and symbols become entities |

One command, `npm run art:sync`, runs every build-time generator and writes one typed manifest:

1. Run `artgen export` for the atlas packs, and `artgen rig` or `artgen mesh` for each model the content names.
2. Read each model file for its clips, clip lengths, height and bounds.
3. Read the pixels-per-metre scale from the art direction.
4. Write `src/assets.gen.ts` with typed ids, and a theme stylesheet.

The simulation reads clip lengths and model bounds from that manifest. It never loads a model file, so tests still run
without a browser.

### 7.3. What artgen needs to gain

These are work packages for `image_tools`. The engine does not wait for any of them.

| Package | What artgen gains | What it unlocks in the engine | Until then |
|---|---|---|---|
| A1. Generation library | `artgen-core`'s asset renderer released as a browser library with one stable call: recipe, direction, parameters and seed in; mesh arrays, textures, skeleton and clips, or sprite frames, out | Generating artgen assets at load time | Build-time generation through the command line |
| A2. Recipe and model export | `artgen export --recipes` bundles approved recipes, the direction and their parameter lists. `--models` exports every model and lists them in `assets.ts` | A1 has something to run. `art:sync` gets simpler | `art:sync` drives the command line asset by asset |
| A3. Reviewed variant ranges | A brief declares which parameters may vary and how far. A sweep command runs the checks over sampled seeds and draws a sample sheet for review | Variants made at load time are covered by an approval | One approved variant per brief |
| A4. More 3D templates | Props, buildings, modular wall and floor pieces, vegetation, vehicles with turrets, and rigs for quadrupeds and flyers | Strategy units, outdoor scenes, and dungeon levels built from pieces | Static meshes, greybox shapes, and textured cell blocks |
| A5. Game data in the output | A collision shape, stride length per walk cycle, the contact time of an attack, and levels of detail for static meshes | Better colliders, walk speed and hit timing with no hand tuning | Bounds, hand-tuned speeds, a fraction of clip length |
| A6. Speed | Fewer triangles on flat faces, faster character meshing, results as transferable arrays | Shorter first loads | Caching, and build-time generation for heavy assets |

A1 and A2 matter most. `artgen-core` is already written with no Node or browser-page imports, so A1 is mainly
packaging and a stable interface.

**A condition on A3.** artgen's strength is that a person approves each asset. Variants made while the game loads are
never seen by a reviewer. A3 keeps that promise by approving a range and checking samples from it. Without A3, the
engine uses load-time variants only for parameters a recipe marks as safe.

## 8. What runs on the graphics card

### 8.1. The engine is hardware accelerated

Yes. three.js's `WebGLRenderer` draws through WebGL 2, which every current desktop browser runs on the graphics card.
The earlier note about software drawing describes only this build environment, which has no graphics card. It does not
describe what players get.

| Work | Where it runs |
|---|---|
| All drawing: geometry, textures, lighting, shadows, fog | Graphics card |
| Skinned animation of models | Graphics card (vertex shader) |
| The pixel-art upscale and other screen effects | Graphics card |
| Physics and navigation meshes | WebAssembly on the processor |
| Asset generation | Background workers on the processor |
| Game simulation | The main thread, in fixed ticks |

### 8.2. Where the plan uses the graphics card deliberately

- **Batching (E16).** Repeated props and tiles are drawn in one call each, not one call per object.
- **Colour by palette (E16).** artgen textures use exact palette colours. A shader can look colours up in a palette
  row, so team colours and colour variants cost no extra textures and no extra draw calls.
- **Particles (E16).** Particle motion runs in the vertex shader, so thousands cost little.
- **Fog of war and terrain blending (E14).** Both are textures blended in a shader.
- **Level of detail (E3).** Distant models draw fewer triangles. artgen already exports the levels.
- **Quality settings (E3, E8).** Resolution scale, shadow quality and draw distance, set by the player or picked from
  a short start-up measurement.
- **Recovery (E3).** If the browser drops the graphics context, the engine reloads its resources and carries on.
- **A benchmark page (E16).** A fixed scene that reports frame times as text you can copy back, because the build
  environment cannot measure a real graphics card.

### 8.3. Why not WebGPU yet

WebGPU would add compute shaders, which could later move particle and generation work onto the graphics card. It is
left out of version 1 for three reasons. artgen's adapter is tested on WebGL. The build environment cannot test
WebGPU. WebGL 2 already covers everything in this plan. One module owns the renderer, so the switch stays possible.

## 9. Beyond your brief: what was added and what to consider

### 9.1. Added to the plan

Each of these was outside what you asked for. Each is small next to the cost of adding it late. Say if any should go.

| Addition | Why | Phase |
|---|---|---|
| Content reloads in the running game | The edit-and-see loop is where most game-building time goes | E2 |
| A command-line tool for sessions: check content, run a level for a number of ticks, take a screenshot, replay a session, run the benchmark | A Claude session can test a game without a person, as artgen's command line lets it check art | E0, growing |
| Recovery from a dropped graphics context, window resize and full screen, pause when the tab is hidden | Browsers do all of these to real players | E3 |
| A top-down and isometric camera, and ground drawn from artgen tile maps | Most of artgen's output is top-down and isometric. A strategy game on an artgen terrain map needs both | E4, E12 |
| Accessibility: interface scale, captions for sounds, a contrast check on the theme, a reduced-motion setting, hold or toggle for held actions | Cheap now, costly to retrofit. Menus in HTML already work with the keyboard | E8, E9 |
| Text in more than one language | All player-facing text is already in KDL, so this is one more folder | E2, E8 |
| Level rules in KDL: when an event happens, do these actions | Without it, every door and ambush needs code, which breaks "content in KDL" | E12 |
| A reproduction bundle: one file holding the seed, the content version, the recorded commands and a save | Any bug can be replayed exactly. It falls out of the fixed-tick design | E15 |
| Versions and migration for saves and content | A schema change must not break old saves | E15 |
| Licences: MIT for the engine, as artgen, and a notices file for three.js, Rapier, Recast and fonts | Needed before anyone else uses a game built on it | E0 |
| A feedback route from game repos to the engine, and from the engine to artgen | Your artgen tooling plan already works this way | E17 |
| A deploy workflow for GitHub Pages in the game scaffold | `image_tools` already deploys this way | E17 |

### 9.2. Worth considering, not in the plan

| Idea | What it would give | Cost and my view |
|---|---|---|
| Content packs and mods loaded at run time | Players or you add content with no rebuild | Low, since content is already layered KDL. It would ship the parser. Worth doing after version 1 |
| Props defined as shapes in KDL | Simple models with no recipe code at all: boxes, capsules and slabs in the art direction's materials | Medium. It is tested as possible in section 3, but it bypasses artgen's review. Better as an artgen feature |
| Music generated from patterns in KDL | Music with no audio files | Medium. Without it, music is the one asset that still needs files |
| A sound generator as a sibling of artgen | Reviewed, direction-led sound, as artgen does for art | High. A separate project. The engine's own generator covers effects until then |
| Dialogue, quests, inventory and cutscenes | Role-playing and story games | Medium each. Best added as a fourth kit when a game needs them |
| A session driving the game through a tool server | Claude plays the game step by step and sees screenshots | Low once the command-line tool exists. Worth doing in E17 if playtesting by session proves useful |
| Editing in the running game, written back to KDL | Tune a value in the inspector and save it to the file | Medium. The chosen parser keeps comments and layout on rewrite, so it is possible |
| A desktop app wrapper | Distribution outside the browser | Low to medium. No engine change needed. Decide per game |
| Multiplayer | Section 2.3 | High. The core keeps it possible |
| Usage statistics and crash reports sent to a server | Knowing how games are played | Not recommended by default. It needs a server and a privacy notice. The reproduction bundle covers bug reports |

## 10. Build phases

Sizes follow the artgen plans: S is about one session, M is two or three, L is four or more. "Windows" are five-hour
usage windows, as in the artgen tooling plan, and are guesses. Every phase ends with all checks green (section 11.2).
The tick budget is 4 ms of simulation work per tick, measured in Node, unless a phase says otherwise.

| Phase | What it builds | Size | Model | Windows |
|---|---|---|---|---|
| E0 | Repo scaffold and guard rails | S | Sonnet | 1–2 |
| E1 | Core | M | Opus designs, Sonnet builds | 4–6 |
| E2 | Content in KDL | M | Opus designs, Sonnet builds | 3–5 |
| E3 | Render, assets, artgen bridge | M–L | Opus designs, Sonnet builds | 5–8 |
| E4 | Input and cameras | M | Sonnet | 3–4 |
| E5 | Physics and character movement | M–L | Opus for the character body | 4–7 |
| E6 | Three greybox slices | M | Sonnet builds, Opus reviews | 3–5 |
| E7 | Generation at load time | M–L | Opus designs, Sonnet builds | 5–8 |
| E8 | 2D layer and game-state flow | M | Sonnet | 4–7 |
| E9 | Audio | S–M | Sonnet | 2–4 |
| E10 | Navigation and AI | M–L | Opus for flow fields and steering | 5–8 |
| E11 | First-person kit and example game | M–L | Opus designs, Sonnet builds | 5–8 |
| E12 | Levels from data, maps and seeds | M–L | Opus designs, Sonnet builds | 5–8 |
| E13 | Third-person kit and example game | M | Sonnet | 3–5 |
| E14 | Strategy kit and example game | L | Opus designs and builds movement | 8–14 |
| E15 | Save, load and replay | S–M | Sonnet | 2–4 |
| E16 | Effects and graphics performance | M | Sonnet | 4–7 |
| E17 | Packaging for game repos | M | Sonnet | 3–5 |

The total is roughly 70 to 115 windows. Order: E0 → E1 → E2 → E3 → E4 and E5 → E6 → E7 to E10 in any order → E11.
After E11, phases E12 to E17 can be reordered to suit the first real game. For example, pull E17 forward to start a
game repo early.

**Work outside this table.** The artgen packages A1 to A6 are built in `image_tools` and sized there. Art for the
example games is made with artgen: one art direction for this repo, then a batch of recipes per example. It can start
any time after E3. Until then the examples use greybox shapes and the billboard-crawler's assets as fixtures.

### E0. Scaffold the repo and its guard rails (S)

- **Builds:** npm workspace (`packages/engine`, `examples/*`). TypeScript, Vite and Vitest set up. The boundary lint
  for section 5.1. A Playwright harness that serves an example, drives it and takes a screenshot. The first commands
  of the session tool. Continuous integration (CI). The licence and notices files. `CLAUDE.md` and an
  engine-development skill that state the rules in section 11.
- **Accept:** `npm run check`, `npm test`, `npm run lint:boundaries` and `npm run accept:smoke` pass locally and in CI.
  The smoke test draws the section 3 scene and compares it to a stored picture.
- **Makes later work faster:** every later phase has its checks from the first commit.

### E1. Build the core with no browser (M)

- **Builds:** entities, components with schemas, queries, add and remove hooks, the system scheduler, the fixed-tick
  clock, events, commands, the seeded random number generator with derived seeds, resources, plugins, snapshot and
  restore, and a rule check that bans clocks, `Math.random` and browser objects in simulation code.
- **Accept:** unit tests for each part. Two runs with the same seed and commands end with the same state hash. A run
  that is saved, restored and continued matches an unbroken run. 10,000 entities and 20 queries cost under 1 ms of
  overhead a tick in Node.
- **Makes later work faster:** every module is written against one fixed contract.

### E2. Read all content from KDL (M)

- **Builds:** the `content` module in section 6: the schema registry, the checker, the compiler, typed ids, prefabs
  with inheritance, tables, level files, text tables, layering, reload in development as a Vite plugin, the generated
  content reference, and `npm run content:check`.
- **Accept:** the files in Appendix B check and compile. Twenty deliberately broken files each give an error that
  names the file, line, column and the fix. A prefab spawned from KDL equals the same prefab built in code. Changing a
  value in a file changes a live entity in the browser without a page reload.
- **Makes later work faster:** every later module adds content by registering a schema, not by writing a loader.

### E3. Draw the world and load art (M–L)

- **Builds:** `render` (backend, views with tick blending, greybox shapes, lights, environment, both display profiles,
  level of detail, quality settings, context recovery, resize, debug lines), `assets`, the `artgen` bridge, the
  `art:sync` script, an animation-state component, and the sandbox example.
- **Accept:** the sandbox shows greybox shapes, the warden walking a path and the ghoul sprite turning with the camera,
  matched against stored pictures. 1,000 moving entities hold the tick budget. Loading then unloading a level returns
  the renderer's memory counts to where they started. A forced context loss recovers to the same picture.
- **Makes later work faster:** anything built after this can be seen.

### E4. Read input and move cameras (M)

- **Builds:** `input` (devices, actions and bindings from `input.kdl`, contexts, rebinding, pointer lock) and the
  camera rigs with shake and blending between rigs.
- **Accept:** Playwright sends keys and mouse moves and the right actions fire. Rig maths is unit tested. The sandbox
  switches rigs on a key.

### E5. Add physics and character movement (M–L)

- **Builds:** `physics` (Rapier world, body and collider components, state sync both ways, casts, events, layers,
  debug drawing, state save) and the character body. The third-person rig gains camera collision.
- **Accept:** in Node: a stack of boxes settles, the state hash repeats, and a character climbs steps and slopes
  within its limits and never passes through a wall at speed. In the browser: walking round a greybox room.

### E6. Build three rough slices to test the design (M)

- **Builds:** three tiny greybox games using only E1 to E5, with all their content in KDL. First-person: walk and
  shoot a ray at targets. Third-person: run round obstacles with the orbit camera. Strategy: pan the camera,
  box-select 20 units and order them to move on flat ground. Then a findings note, and the core changes it calls for.
- **Accept:** a Playwright script plays each slice to its goal. No slice has a tuning number or asset name in
  TypeScript. `docs/findings/E6.md` lists what was awkward and what changed.
- **Makes later work faster:** the kits are built on an interface that three genres have already used.

### E7. Generate assets and worlds at load time (M–L)

- **Builds:** the `procgen` module in section 7.1: the generator registry, the build-or-load choice per asset, the
  worker pool, the hash-keyed cache, progress reporting, and the engine's own generators for terrain, scatter and sky.
  Dungeon generation from a seed through `artgen-mapgen`. When artgen package A1 has landed: artgen recipes at load
  time, with variants.
- **Accept:** the same seed gives byte-identical results on two runs. A second load of the same level generates
  nothing and reads the cache. Generation never blocks a frame for more than 50 ms. A dungeon generated from a seed in
  the browser matches the same seed generated in Node. The artgen part has its own accept when A1 lands: 20 variants
  of one recipe load, all different, within a time budget set in the design note.
- **Makes later work faster:** example games get new content by adding recipes and seeds, not files.

### E8. Add the 2D layer and the game-state flow (M)

- **Builds:** `ui` as listed in section 5.4, with screens, HUD layout and theme read from KDL. A main menu, pause
  menu, settings (key bindings, volume, display, accessibility) and a HUD in the sandbox. The flow from boot to
  loading, menu, play and pause.
- **Accept:** Playwright works the menus with the keyboard alone. Pausing stops ticks. A rebound key survives a
  reload. A click on the 2D layer does not reach the game. Switching language changes every visible string. The theme
  passes the contrast check.

### E9. Add audio (S–M)

- **Builds:** `audio` as listed in section 5.4, with cues read from `audio.kdl` and a starter set of generated sounds.
- **Accept:** a generated sound renders to the same non-silent samples each time. In the browser a game event produces
  output on its bus, the volume setting changes it, and its caption shows. With no audio device the module does
  nothing and nothing fails.

### E10. Add navigation and AI (M–L)

- **Builds:** `nav` and `ai` as listed in section 5.4, with debug drawings.
- **Accept:** in Node: correct shortest paths on test maps, 200 agents pass a narrow gap and arrive within the tick
  budget, and the run repeats exactly. A navigation mesh path crosses a greybox level with stairs.

### E11. Build the first-person kit and its example game (M–L)

- **Builds:** `kit-common`, `kit-fps`, and a small crypt game: enemies, a weapon, a pickup, a door and an exit.
- **Accept:** a scripted player clears a room of three enemies, takes the pickup, opens the door and reaches the exit.
  A recorded session replays to the same end state.

### E12. Build levels from data, maps and seeds (M–L)

- **Builds:** the full level format, level rules (events to actions), a prefab library, import of a glTF scene (the
  3D file format artgen exports) whose nodes name prefabs, the builders that turn artgen dungeon and terrain maps into
  levels, ground drawn from artgen tile maps, and moving between levels.
- **Accept:** an artgen dungeon file, and a dungeon generated from a seed, each load as a first-person level that the
  E11 scripted player can cross. An artgen terrain map loads as terrain that units can path across. A level rule opens
  a door when a trigger fires, with no code written for that level.

### E13. Build the third-person kit and its example game (M)

- **Builds:** `kit-tps` and a small arena game with waves of enemies.
- **Accept:** a scripted player survives a wave using movement, attack and target lock. The camera never clips through
  a wall. The session replays to the same end state.

### E14. Build the strategy kit and its example game (L)

- **Builds:** `kit-rts` as listed in section 5.4, a batch renderer for health bars, and a skirmish game on generated
  terrain. The simulation runs at 30 ticks a second.
- **Accept:** in Node, a scripted match of two sides with 50 units each runs to the same result every time within the
  tick budget. In the browser, a scripted player selects units, gives orders, builds and wins a small scenario.

### E15. Save, load and replay (S–M)

- **Builds:** save slots in browser storage, versions and migration for saves and content, replay recording and
  playback in the debug tools, the reproduction bundle, and a replay check for each example in CI.
- **Accept:** each example saves mid-game, reloads and continues to the same end state as an unbroken run. A
  reproduction bundle made in the browser replays to the same state hash from the command line.

### E16. Add effects and tune graphics performance (M)

- **Builds:** the items in section 8.2: batching, colour by palette, shader particles, artgen effects on anchors,
  sprite stand-ins for distant models, shadow tuning, budgets in the debug overlay, and the benchmark page.
- **Accept:** each example holds its stated budgets for draw calls and tick time. A strategy scene with 200 units in
  two team colours uses one texture set per unit type. You run the benchmark page on your hardware and the numbers go
  into `docs/metrics/`.

### E17. Package the engine for game repos (M)

- **Builds:** a `create-game` scaffold (Vite, the engine, the artgen install, a starter `content/` folder, `CLAUDE.md`,
  a game-building skill and a deploy workflow), a release workflow that attaches the package file, a changelog,
  generated reference pages, and the feedback route in section 9.1.
- **Accept:** in a clean container, the scaffold makes a new repo where `npm run dev` shows a walking placeholder
  character, `npm run content:check` passes and `npm run art:sync` runs.

## 11. How each phase is run

### 11.1. One phase per session, from a written design

1. **Design.** An Opus session writes `docs/design/E<n>-<name>.md`: the exact interfaces, the files, the tests and
   anything this plan left open. Small phases marked Sonnet skip straight to step 2 with a short note.
2. **Build.** Sonnet sessions build against that note, one sub-step per session, committing as checks pass.
3. **Record.** The same note gains three closing parts, as the artgen design notes have: what was built, where it
   differs from the design, and what was not done.

Start each session from a short brief that names the phase. Use no subagents or parallel runs. Let the checks find
mistakes instead of re-reading files.

### 11.2. The checks every phase must pass

| Check | Command | What it proves |
|---|---|---|
| Types | `npm run check` | The code compiles under strict TypeScript |
| Unit and simulation tests | `npm test` | Logic is right, with no browser |
| Content | `npm run content:check` | Every KDL file matches its schema and every reference resolves |
| Boundaries | `npm run lint:boundaries` | No layer imports from a lower one, and simulation code uses no clocks or browser objects |
| Browser acceptance | `npm run accept:<name>` | The real page runs, drawn in headless Chromium and driven by script |
| Repeatability | part of `npm test` | Same seed and commands give the same state hash |

### 11.3. Rules for all engine and game code

- Game state changes only in fixed ticks (section 5.2).
- Game logic is written as systems. There is no second way.
- Tuning numbers, asset names and player-facing text live in KDL, never in TypeScript.
- No binary asset is committed without a recipe that regenerates it. Music files are the one exception.
- A feature moves from a kit into a shared module only when a second kit needs it.
- Every drawable and every loaded asset is released when its entity or level goes. Tests check for leaks.
- A missing asset shows a placeholder and a warning. It never stops the game.

## 12. Risks

| Risk | Effect | What reduces it |
|---|---|---|
| The scope is large | The work stalls before a game ships | Phases are useful alone. E6 gives a usable base, E11 a whole genre |
| "Any genre" abstractions turn out wrong | Rework in the kits | E6 tests three genres early. The second-kit rule in 11.3 |
| KDL grows into a programming language | Content becomes hard to check and to read | The rule in 6.4: data only, with actions from a fixed list |
| Generation makes first loads slow | Players wait | The cache, background workers, build-time generation for heavy assets, and artgen package A6 |
| The artgen packages land late | No artgen variants at load time | Build-time generation works today. The engine's own generators and `artgen-mapgen` need no artgen change |
| Variants at load time skip review | Poor assets reach players | artgen package A3, and safe-parameter marking until then |
| Many skinned units are slow | The strategy game misses its frame rate | Levels of detail, sprite stand-ins, batching and palette colour in E16, a 1,000-entity gate in E3. Real hardware is measured with the benchmark page |
| Character movement feels wrong | First- and third-person games feel poor | Opus builds it in E5 with step, slope and speed tests. Feel is tuned by you in the sandbox |
| artgen rigs only humanoids | Strategy units and animals lack animation | Static meshes with moving parts, or artgen package A4 |
| Music has no generator | Games have no music without files | Audio files for music, or the pattern idea in 9.2 |
| Trigonometry differs between browsers | Replays and generated assets differ across browsers | Only matters for multiplayer and shared caches, which are out of scope. Checks run on one browser |
| Libraries move on | Builds break | E0 pins exact versions. Updates are their own small tasks |

## 13. How the plan changed across drafts

**Draft 1** set out the layers, the modules and a core-first build order, with the three kits at the end.

**Draft 2** was a review of draft 1 for scope and for the "any genre" claim. It made five changes:

- Added the three greybox slices, so three genres use the core before any kit is designed.
- Removed behaviour scripts. Systems are now the only way to write game logic.
- Split each kit into a simulation half and a view half, so kit rules can be tested without a browser.
- Made generated sounds part of the audio phase, because nothing else can produce sound effects without files.
- Moved prefabs and level data into the core, so example games do not wait for the level phase.

**Draft 3** was a review against what a full engine needs and against the checks in section 3. It made four changes:

- Chose `WebGLRenderer` after finding no WebGPU adapter in the test environment.
- Moved the map-to-level phase to straight after the first-person kit, where it pays off first.
- Added the memory-leak check, the missing-asset placeholder rule and the 1,000-entity gate.
- Added the `art:sync` manifest so the simulation can read clip lengths without loading models.

**Draft 4** followed your reply of 2026-11-15, which accepted decisions 1 to 10 and added three constraints. It made
these changes:

- Added the `content` module and phase E2, so all content is KDL (section 6, decision 11).
- Added the `procgen` module and phase E7, the build-or-load choice per asset, and the artgen work packages A1 to A6
  (section 7, decisions 12 and 13).
- Added section 8 to answer the hardware acceleration question, and moved graphics-card features into named phases.
- Reviewed the plan for gaps outside the brief. Twelve items went into the plan and ten are listed to consider
  (section 9).
- Renumbered the phases. Old E2 to E4 are now E3 to E5, old E5 is E6, old E6 to E15 are now E8 to E17.

## Appendix A. What game code looks like

This sketch shows the intended feel. Exact names are fixed in the E1 to E3 design notes. Code holds only the rule.
The numbers and the things it applies to are in Appendix B.

```ts
import { createEngine, defineComponent, defineSystem, f32, type Plugin } from 'display-engine/core';
import { render } from 'display-engine/render';
import { artgen } from 'display-engine/artgen';
import { procgen } from 'display-engine/procgen';
import { input } from 'display-engine/input';
import { physics } from 'display-engine/physics';
import { audio } from 'display-engine/audio';
import { ui } from 'display-engine/ui';
import { Health } from 'display-engine/kit-common';
import { fps } from 'display-engine/kit-fps';
import { content } from './content.gen';

// A component is plain data with a schema. Its KDL node is `regen rate=2`.
export const Regen = defineComponent('Regen', { rate: f32(1) });

// A system is a function that runs in one step of the frame.
const regen = defineSystem({
  name: 'regen',
  phase: 'fixed',
  run(world, dt) {
    for (const e of world.query(Health, Regen)) {
      const h = world.get(e, Health);
      h.hp = Math.min(h.max, h.hp + world.get(e, Regen).rate * dt);
    }
  },
});

// A game is a plugin: its own components and systems. Everything else comes from content/.
const game: Plugin = { name: 'crypt', components: [Regen], systems: [regen] };

const engine = await createEngine({
  canvas: document.querySelector('canvas')!,
  content,
  plugins: [render(), artgen(), procgen(), input(), physics(), audio(), ui(), fps(), game],
});
engine.start();
```

## Appendix B. What content files look like

These five files were parsed as KDL version 2 in the section 3 checks. Node and property names are fixed in the E2
design note.

```kdl
// content/prefabs/warden.kdl
prefab "warden" {
    art-model "warden" bake="load" {
        vary "coat" "short" "long"
        vary "height" min=1.6 max=1.9
    }
    animator state="idle"
    health hp=60 max=60
    regen rate=2
    character-body radius=0.3 height=1.7
    team "wardens"
    sound-on "damaged" cue="grunt"
}

// content/levels/crypt-1.kdl
level "crypt-1" {
    environment {
        fog "#06070c" near=2.5 far=9
        ambient "#8090b0" intensity=0.35
    }
    generate "dungeon" seed=42 rooms=8 theme="crypt"
    spawn "warden" room=3
    on "trigger-entered" zone="vault" {
        spawn "warden" count=2 room=5
        play-sound "alarm"
        show-text "vault-alarm"
    }
}

// content/audio.kdl
cue "grunt" bus="effects" spatial=#true {
    synth wave="noise" attack=0.01 decay=0.18 pitch=180 slide=-60
    vary pitch=0.1 volume=0.15
}

// content/input.kdl
action "fire" {
    key "Mouse0"
    pad "RightTrigger"
}
axis2 "move" {
    keys "KeyW" "KeyS" "KeyA" "KeyD"
    pad "LeftStick"
}

// content/text/en.kdl
text "vault-alarm" "The vault is open. Something heard it."
```
