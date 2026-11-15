# Generating artgen assets inside a game: design for artgen packages A1 to A3

- **Purpose:** Build, in `image_tools`, the library and exports that let a game generate artgen assets while it loads, and build the engine side that calls them, from one shared specification.
- **For:** The sessions that build A1, A2 and A3 in `image_tools`, and the session that builds phase E7 in this repo.
- **Version:** 1, 2026-11-15. Status: design, not built. Written against `image_tools` at commit `c659e8f` (release `artgen-v0.14.0`).
- **Instead of this:** For how a game uses artgen's build-time output today, read section 7.2 of [the engine plan](../PLAN.md). For the engine's data rules, read [the data contract](E1-E2-data-contract.md).

## Contents

- [1. Summary](#1-summary)
- [2. What was tested](#2-what-was-tested)
- [3. What artgen has today, and what has to move](#3-what-artgen-has-today-and-what-has-to-move)
- [4. A1: the generation library](#4-a1-the-generation-library)
- [5. A2: exporting recipes and models](#5-a2-exporting-recipes-and-models)
- [6. A3: reviewed variants](#6-a3-reviewed-variants)
- [7. The engine side, in phase E7](#7-the-engine-side-in-phase-e7)
- [8. Order of work and open points](#8-order-of-work-and-open-points)

## 1. Summary

A game should be able to ship an artgen *recipe* (the small JavaScript module that draws an asset) and turn it into a
model or sprites while a level loads. Today only artgen's command line can do that, and only to files.

The test in section 2 shows it already works in a browser worker with no change to artgen's code, only to how it is
packaged. So the work is mostly packaging and a stable interface. This note fixes five things:

1. **One library, `artgen-generate`.** It is a single browser-safe file with one call for models and one for sprites.
   It does no file or network access.
2. **The output is the file format artgen already writes.** A model comes back as the bytes of a `.glb` file (the
   binary form of glTF, the 3D format artgen exports). Sprites come back as a pack. The game loads them with the code
   it already uses for build-time assets.
3. **A recipe bundle.** `artgen export --recipes` writes the recipes, the art direction and their data in a form the
   library reads.
4. **A request has a key.** The same key always means the same bytes, so the game can cache results.
5. **Variants are numbered and approved by count.** An asset approved for 64 variants means variant numbers 1 to 64
   have each passed the checks. The game picks a number. It never invents parameter values.

**What you do with it.** In `image_tools`, build A1, then A2, then A3, each to the accept list in its section. In this
repo, build the artgen part of E7 from section 7 once A1 and A2 are released.

**Qualifications.**

- Models were tested end to end. Sprites were not: the pieces exist in `artgen-core`, but nobody has run them at load
  time, and their cost is unknown.
- All timings are from one machine with software drawing. Treat them as rough sizes, not budgets.
- Node and Chromium did not always produce identical bytes for the same request (section 2). The promise in section
  4.5 is therefore per JavaScript engine, and section 7.2 keeps the game's rules independent of generated assets.

## 2. What was tested

A throwaway project bundled `artgen-core` from the `image_tools` source with esbuild, and called it as a library.

| Test | Result |
|---|---|
| Six real recipes from the example games, generated in Node with no files written | All six worked. The warden came out rigged with 3 levels of detail and 4 clips. Five props came out as static meshes. Every mesh and skin check passed |
| The same request twice, in Node | Byte-identical `.glb` output for the warden and for two barrel variants |
| The same request in Node 22 and in Chromium | The two barrel variants were byte-identical. The warden was not. The likely cause is trigonometry in the rig, which may differ in the last digit between engine versions. It was not investigated further |
| The library's bytes against the warden file committed in the example game | Different. This shows nothing either way: the test re-implemented the command line's steps, and the file is from an earlier release. Accept step 3 in section 4.7 is the real test |
| Variant numbers 0, 1 and 2 of recipes that declare parameters | Different meshes for the barrel and the rubble. The chest and pillar gave two distinct results across the three numbers |
| The library in a browser worker, in headless Chromium | The warden generated in 550 ms, was written as `.glb` bytes in 81 ms, and was parsed by three.js in 43 ms. Four barrel variants took 37 to 62 ms each to generate |
| The page while the worker ran | The longest gap between frames was 31 ms |
| Loading the result with artgen's existing runtime | `riggedModel` from `artgen-runtime` 1.5.0 played the generated warden unchanged. It was given a loader that parses bytes in place of one that fetches a file |
| Size of the library | 272 KB minified, 103 KB compressed with gzip, for the model path |

Time to generate in Node, for scale: the warden took about 0.98 s (0.09 s to draw the bind cell, 0.89 s to mesh, skin
and sample). Props took 41 to 134 ms each.

## 3. What artgen has today, and what has to move

### 3.1. The pieces that exist

| Piece | Where | State |
|---|---|---|
| Draw a recipe over its states, facings and frames | `renderAsset` in [`artgen-core/src/render.ts` line 308](https://github.com/rstevenson1237/image_tools/blob/c659e8f/packages/artgen-core/src/render.ts#L308) | Browser-safe |
| The recipe contract: `meta`, `params`, `render`, `anchors` | `AssetModule` in [`render.ts` line 78](https://github.com/rstevenson1237/image_tools/blob/c659e8f/packages/artgen-core/src/render.ts#L78) | Browser-safe |
| Turn a variant number into parameter values | `resolveParams` in [`artgen-scene/src/t2/params.ts` line 34](https://github.com/rstevenson1237/image_tools/blob/c659e8f/packages/artgen-scene/src/t2/params.ts#L34) | Browser-safe |
| Mesh, skin, sample clips, write `.glb` | `buildMesh`, `buildRigged`, `meshGlb`, `riggedGlb` in `artgen-scene/src/mesh/` | Browser-safe, released |
| Pack sprite frames into atlases | `buildPack` in [`artgen-core/src/w2/pack.ts` line 201](https://github.com/rstevenson1237/image_tools/blob/c659e8f/packages/artgen-core/src/w2/pack.ts#L201) | Browser-safe, not released |
| Apply a finish pass to sprites | `applyFinish` in [`artgen-core/src/finish.ts` line 360](https://github.com/rstevenson1237/image_tools/blob/c659e8f/packages/artgen-core/src/finish.ts#L360) | Browser-safe, not released |

### 3.2. Four things in the way

1. **`artgen-core` is not released as a library.** Recipes need it, because it builds the `ctx.lib` they draw with.
2. **The steps from recipe to model live in the command line**, in
   [`artgen-cli/src/mesh.ts`](https://github.com/rstevenson1237/image_tools/blob/c659e8f/packages/artgen-cli/src/mesh.ts),
   mixed with file writing. They are: finding the 3D scene of the bind cell (`scene3dOf`, line 28, and the bind brief
   at line 66), the scale in metres (`metresPerUnit`, line 42), the clip lengths, including the rule that a
   one-frame state lasts 2 seconds (line 72), and the mesh settings from the brief (`settingsOf`, line 90).
3. **The 3D scene is only reachable by drawing a sprite.** A recipe's `render` returns a picture and builds its
   `Scene3D` on the way. The model path draws the bind cell's picture and throws it away. That cost 93 ms of the
   warden's time in Node.
4. **`artgen-core` imports `yaml`** for reading briefs
   ([`w2/briefs.ts` line 5](https://github.com/rstevenson1237/image_tools/blob/c659e8f/packages/artgen-core/src/w2/briefs.ts#L5)).
   A bundle that reaches it drags in a Node-style package. The generation path does not need it.

A1 removes the first, second and fourth. The third is left as it is and listed in section 8.

## 4. A1: the generation library

### 4.1. The package

- **Name:** `artgen-generate`, in `packages/artgen-generate`.
- **Form:** one ES module with no dependencies and no imports, built by bundling. It is attached to each release as
  `artgen-generate-<version>.tgz`, like the other library packages.
- **Limits:** no Node built-ins, no page objects (`window`, `document`), no `yaml`, no file or network access. It may
  use `performance.now` only for the timings it reports.
- **Size budget:** 350 KB minified for the model path and 500 KB with sprites. A test fails the build above that.
- **Version constants:** `VERSION` is the artgen release. `GENERATION` is a whole number that goes up only when the
  bytes produced for an unchanged request can change.

### 4.2. The calls

```ts
/** One recipe as the bundle lists it (section 5.2), with its module loaded by the caller. */
export interface LoadedRecipe { entry: RecipeEntry; module: AssetModule; finish?: FinishModule }

/** What to generate. Every field is optional. */
export interface GenerateRequest {
  /** 0 is the recipe's defaults. A number above 0 samples every declared parameter from that number. */
  variant?: number;
  /** Fixed values for named parameters. These win over the variant's sampled values. */
  params?: Record<string, unknown>;
  /** Seed for noise the recipe draws from `ctx.rng`, and for texture grain. Default 1. */
  seed?: number;
}

/** Mesh settings. Each defaults to the brief's `mesh` block, then to artgen's default for the asset kind. */
export interface ModelOptions { budget?: number; mode?: MeshMode; texels?: number; look?: MeshLook; lods?: number; keys?: number }

export interface ModelResult {
  /** The bytes `artgen mesh` or `artgen rig` would write for the same request. */
  glb: Uint8Array;
  rigged: boolean;
  /** The same data the file carries in `extras.artgen`, so the caller need not parse the file to read it. */
  info: { metresPerUnit: number; height: number; bounds: [number, number, number, number, number, number];
          lods: { triangles: number; budget: number; distance: number }[];
          clips: Record<string, { duration: number; loop: boolean }> };
  checks: Check[];
  key: string;
  ms: { scene: number; build: number; encode: number };
}

export interface SpritesResult {
  /** A pack holding this one asset, as `createPack` in `artgen-runtime` takes it. */
  manifest: PackManifest;
  atlases: AtlasImage[];
  normals?: AtlasImage[];
  key: string;
}

export function generateModel(recipe: LoadedRecipe, direction: Direction, request?: GenerateRequest, options?: ModelOptions): ModelResult;
export function generateSprites(recipe: LoadedRecipe, direction: Direction, request?: GenerateRequest, options?: { normals?: boolean }): SpritesResult;

/** The cache key for a request. Needs only the bundle's data, not the loaded module. */
export function requestKey(entry: RecipeEntry, directionHash: string, what: 'model' | 'sprites', request?: GenerateRequest, options?: object): string;
```

Rules for the calls:

- **They are plain functions that return when done.** A model takes from 40 ms to about a second. The caller runs them
  in a worker. The library contains no worker code.
- **The caller loads the recipe module** with `import()`, because loading differs between Node, a worker and a
  bundler. The library never loads anything.
- **`generateModel` uses the latest base version of the recipe.** A finish pass changes sprite pixels and has no
  effect on a model. This matches `artgen mesh` and `artgen rig` today.
- **`generateSprites` applies the finish pass** when the bundle lists one, so the sprites match what was approved.
- **A rigged recipe gives a rigged model.** A recipe is rigged when its bind-cell scene carries a pose, as
  `artgen rig` decides today.

### 4.3. Why the output is `.glb` bytes

The alternative was raw vertex arrays. The file bytes were chosen for four reasons:

- The game loads a generated model with exactly the code that loads a build-time file, including `riggedModel`.
- The bytes can go straight into a browser cache.
- One set of expected hashes covers the command line and the library, because they produce the same bytes.
- There is no second format to version.

The cost was measured: writing and parsing added 124 ms to the warden's 550 ms, and 13 to 33 ms to a barrel. If that
proves too much, a raw-array result can be added later without changing this one.

### 4.4. The key

`requestKey` returns 16 hexadecimal digits: the FNV-1a 64-bit hash of these parts, joined by a zero byte.

1. `GENERATION`
2. `'model'` or `'sprites'`
3. the entry's `sourceHash`
4. the direction's hash
5. the variant number
6. the `params` overrides as JSON with keys sorted
7. the seed
8. the options as JSON with keys sorted, after defaults are filled in

Two requests with the same key produce the same bytes. A new artgen release that leaves `GENERATION` alone keeps
every cached result valid.

### 4.5. What the library promises

- **Repeatable.** The same key gives byte-identical output on the same JavaScript engine.
- **Same as the command line.** For the same recipe and settings, `generateModel` returns the bytes `artgen rig` or
  `artgen mesh` writes. A1 achieves this by making the command line call the library.
- **Not promised:** identical bytes across JavaScript engines, or across versions of one engine. The test in section
  2 found a rigged model that differed between Node 22 and Chromium, while props matched. So expected hashes are
  checked in Node only, a cache belongs to one browser, and nothing may compare generated bytes between machines.

### 4.6. What moves

| From `artgen-cli/src/mesh.ts` | To |
|---|---|
| Finding the bind cell's scene (`scene3dOf`, and the bind brief in `rigExport`) | `artgen-core`, as `modelScene(module, direction, brief, request)` |
| `metresPerUnit` | `artgen-scene`, beside `pxPerMetre` |
| Clip lengths from the brief's states, with the 2-second rule | `artgen-core`, as `clipSpecs(brief, direction)` |
| `settingsOf` | `artgen-core`, as `meshSettings(brief, options)` |

After the move, `rigExport` and `meshExport` call `generateModel` and write its bytes. Their outputs must not change:
the existing bench hashes guard this.

The `artgen-generate` bundle's entry imports only what the two calls need. It must not reach `w2/briefs.ts`.

### 4.7. Accept for A1

1. `npm run check`, `npm test`, `check:packages`, `test:packages`, the boundary lint, the bench hashes and the
   consumer fixtures stay green. No existing output changes.
2. A bundle test: the built file has no `node:` import, no `yaml`, no `require`, and is under the size budget.
3. For every 3D-mode bench asset, `generateModel` returns exactly the bytes `artgen mesh` or `artgen rig` writes.
4. `requestKey` is stable: a stored table of requests and keys does not change.
5. A headless Chromium test runs the library in a worker, generates the bench rigged hero and one prop, and loads
   both through `artgen-runtime`. It generates each twice and requires identical bytes within the browser. It also
   records whether the bytes match Node's, without requiring it.
6. `generateSprites` on one bench sprite asset gives a pack that `createPack` loads, with the same frames as
   `artgen export` gives for that asset alone. Its time is recorded in this note.

## 5. A2: exporting recipes and models

### 5.1. The command

`artgen export --recipes` writes a bundle the library can read. It is added to the existing `export` operation.

```text
public/assets/recipes/
  recipes.json                       the bundle (5.2)
  modules/<id>.<hash8>.js            each recipe's module, copied unchanged
  modules/<id>.finish.<hash8>.js     its finish module, when it has one
src/art/recipes.ts                   typed ids, and each recipe's parameter names
```

- Paths come from `art/artgen.config.json`, under a new `export.recipeDir` and `export.recipesTs`.
- Only approved assets are written. `--include-drafts` adds the rest, with their status in the bundle, as the pack
  export does today.
- **A recipe module must have no `import` statement.** Recipes get everything through `ctx.lib`, and the two checked
  for this note have none. The export refuses a module that imports, and says which line.
- File names carry the first 8 digits of the source hash, so a changed recipe has a new address and old caches
  cannot serve it.

### 5.2. The bundle

```ts
export interface RecipeBundle {
  format: 1;
  generator: string;                 // the artgen release that wrote it
  generation: number;                // GENERATION of that release
  direction: Direction;              // the locked art direction, in full
  directionHash: string;
  recipes: Record<string, RecipeEntry>;
}

export interface RecipeEntry {
  id: string;
  kind: string;
  brief: Brief;                      // as JSON
  module: string;                    // path relative to recipes.json
  finish?: string;
  version: string;                   // for example 'base.v3'
  sourceHash: string;                // over the module and the finish
  status: 'approved' | 'final' | 'draft';
  can: ('model' | 'rigged' | 'sprites')[];
  params: Record<string, ParamSpec>; // the declared parameters and their ranges
  vary?: VariantApproval;            // section 6.3. Absent until A3
  /** For variant 0, so a game can know sizes and clip lengths without generating. Present when `can` has a model. */
  model?: { height: number; bounds: [number, number, number, number, number, number];
            clips: Record<string, { duration: number; loop: boolean }>; lods: number[] };
}
```

### 5.3. Models in the build-time export

`artgen export --models` does at build time what the plan's `art:sync` script works around today:

- For every asset whose latest base builds a 3D scene, write `<id>.rigged.glb` or `<id>.mesh.glb` to
  `export.modelDir`, by calling `generateModel` with variant 0.
- Add a `Models` object to the generated `src/art/assets.ts`, with each model's id, address, whether it is rigged,
  its clips with lengths, its height, its bounds and its levels of detail.

### 5.4. Accept for A2

1. The three example games export recipes. Each bundle validates against a published JSON schema
   (`artgen schema recipes`).
2. For each exported recipe, loading it from the bundle and calling `generateModel` gives the same bytes as
   `artgen rig` or `artgen mesh` in that game.
3. A recipe with an `import` statement is refused, with its line.
4. `--models` writes a model per 3D asset in the billboard-crawler example. The example's `main.ts` then loads the
   warden through `Models.warden` in place of a hand-written path.
5. Running the export twice writes identical files.
6. The operation appears in the operation registry, `OPERATIONS.md` and the Python client, as other operations do.

## 6. A3: reviewed variants

### 6.1. The problem

artgen's promise is that a person approves each asset. A variant generated while a game loads has never been seen by
anyone. A3 keeps the promise by approving a *counted set* of variants, each of which has passed the automated checks.

This works because of how artgen already samples. Variant number *n* fixes every declared parameter, from a hash of
the parameter's name and *n* (`resolveParams`, cited in 3.1). So "variants 1 to 64" is a finite list of exact
assets. It can be generated, checked and looked at.

### 6.2. What a brief declares

```yaml
- id: barrel
  kind: prop
  vary:
    variants: 64            # variant numbers 1..64 may be used by a game
    open:                   # parameters a game may set itself, and the values allowed
      height: { min: 0.8, max: 1.2 }
      lid: [ true, false ]
```

- `variants` is how many variant numbers a game may use.
- `open` lists parameters a game may pass in `params`. Use it for a value the game's rules decide, such as a
  character's height, so that the model matches the game's own data. A parameter not listed in `open` may not be
  overridden by a game.
- A brief with no `vary` allows variant 0 only. That is every brief today.

### 6.3. The sweep and the approval

`artgen sweep <asset> [--sample 16] [--sheet out.png]`:

1. Generates every variant from 1 to `variants`. For each `open` parameter it also generates the lowest, the highest
   and one middle value, on a sample of variants.
2. Runs the mesh and skin checks on each, and the sprite checks where the asset has sprites.
3. Reports each failure with its variant number and parameter values.
4. Draws a contact sheet of `--sample` variants, always including the ones nearest the extremes of each parameter.
5. Exits with status 1 if any variant fails a check.

`artgen approve <asset>` then records the approval of the set. It refuses unless the latest sweep passed for the
current source hash and direction version. The approval goes in the ledger and is exported in the bundle:

```ts
export interface VariantApproval {
  variants: number;
  open: Record<string, { min: number; max: number } | unknown[]>;
  sourceHash: string;        // the approval lapses when the recipe changes
  direction: number;         // and when the direction version changes
  swept: number;             // how many variants the sweep generated
}
```

### 6.4. What the library enforces

`generateModel` and `generateSprites` check the request against the entry:

| Request | Without `vary` | With `vary` |
|---|---|---|
| Variant 0, no overrides | Allowed | Allowed |
| Variant 1 to `variants` | Refused | Allowed |
| A variant above `variants` | Refused | Refused |
| An override of an `open` parameter, inside its allowed values | Refused | Allowed |
| Any other override | Refused | Refused |

A refusal throws an error that names the asset, the request and the limit. Passing `{ unreviewed: true }` in the
options turns a refusal into a console warning. That switch is for development. A game's release build must not set it.

### 6.5. Accept for A3

1. A brief's `vary` block validates, and `brief add --vary 64` writes one.
2. `artgen sweep` on a bench prop with two parameters generates every variant, and its sheet shows the extremes.
3. A recipe built to fail a check at one variant makes the sweep exit 1 and name that variant.
4. `artgen approve` refuses after the recipe changes, until the sweep is run again.
5. Each row of the table in 6.4 has a test.
6. The Asset Review tool shows the sweep sheet beside the asset. This may be a follow-up if the web work is large.

## 7. The engine side, in phase E7

### 7.1. How a content file asks for a generated asset

```kdl
prefab "barrel" {
    art-model "barrel" bake="load" vary=#true
}

prefab "warden" {
    art-model "warden" bake="load" vary=#true {
        param "height" 1.8
    }
    character-body radius=0.3 height=1.8
}
```

| Property | Meaning |
|---|---|
| `bake="build"` (the default) | Use the file written by `artgen export --models`. Variant 0 only |
| `bake="load"` | Generate while the level loads |
| `vary=#true` | Pick a variant number from the entity's seed: `1 + (seed mod variants)`. Without it the variant is 0 |
| `param "<name>" <value>` | An override for an `open` parameter |

The entity's seed is `Origin.seed` from the data contract. It is fixed when the entity is created and saved with it,
so an entity keeps its look across save and load.

### 7.2. The simulation never depends on a generated asset

This rule protects repeatability. A generated mesh can differ between machines (section 2 found one that did), and it
arrives late.

- Collider sizes, heights and speeds come from content files, never from a generated model's bounds.
- Clip lengths come from the bundle's `model.clips`, which describe variant 0 and are the same for every variant of a
  recipe, because clips come from the brief's states.
- When a parameter matters to the rules, such as height, the content file states it once and passes it to the recipe
  as an `open` parameter, as the warden above does.
- Until a model arrives, the entity shows a greybox shape of the content's size. Play does not wait for art after the
  level's loading screen ends.

### 7.3. The worker pool and the cache

1. When a level loads, the engine collects every `bake="load"` request and works out each key with `requestKey`.
2. It looks each key up in the cache. Hits are loaded at once.
3. Misses are sent to a pool of module workers. The pool has one worker fewer than the processor's cores, at least 1
   and at most 4. Each worker imports `artgen-generate`, imports the recipe module, calls the library, and transfers
   the bytes back.
4. Results are parsed on the main thread with three.js's loader, spread across frames, and written to the cache.
5. The loading screen shows progress by request count.

| Part | Choice |
|---|---|
| Message to a worker | `{ key, recipe id, what, request, options }` |
| Message from a worker | `{ key, bytes, info, checks, ms }`, or `{ key, error }` |
| Cache | The browser's Cache Storage, one entry per key, under a cache named for the bundle's `generation` |
| Cache limit | 200 MB, oldest entries removed first. The number is a starting point for E7 to tune |
| A failed request | A greybox shape, a console error naming the recipe and request, and no retry in that session |
| Identical requests | Generated once and shared by every entity that asked |

### 7.4. Accept for the artgen part of E7

1. Twenty entities from one varied prefab load with at least ten distinct models.
2. Reloading the page generates nothing and reads every model from the cache.
3. Saving and restoring a game gives each entity the same variant as before.
4. A request the library refuses shows a greybox shape and one clear error.
5. No frame takes longer than 50 ms while generation runs.
6. Generation in the browser and the build-time file for variant 0 show the same model.

## 8. Order of work and open points

**Order.** A1, then A2, then A3. The engine's E7 can start before any of them, on its own generators and on dungeons
from `artgen-mapgen`. The artgen part of E7 needs A1 and A2. `vary=#true` needs A3, and until then the engine sets
`unreviewed` in development builds only.

**Open points**, none of which blocks the work:

| Point | Where it is decided |
|---|---|
| Sprite generation cost at load time is unknown | A1 accept step 6 measures it |
| The model path draws a sprite it throws away | Later: an optional `scene(ctx)` export on a recipe, so the library can skip the drawing. Recipes without it keep working |
| Why the rigged model differs between Node and Chromium, and whether it can be made identical | A1 accept step 5 records it. Making it identical is optional, and would mean replacing the engine's trigonometry in the rig code |
| How big a sweep can be before it needs sampling | A3 build: generate all up to 256, sample above |
| Recipes are code, so a mod that ships a recipe runs code | Out of scope. Version 1 loads only the game's own bundle |
