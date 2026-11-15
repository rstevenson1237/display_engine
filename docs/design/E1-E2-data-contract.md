# The data contract: design for phases E1 and E2

- **Purpose:** Build the engine core (E1) and the KDL content module (E2) against one fixed contract for how game data is defined, stored, written in content files, and saved.
- **For:** The Sonnet or Opus sessions that build E1 and E2, and later sessions that add modules on top.
- **Version:** 1, 2026-11-15. Status: design, not built.
- **Instead of this:** For why the engine is shaped this way, read [the engine plan](../PLAN.md). For the artgen generation interface, read [the artgen generation design](A1-A3-artgen-generation.md).

## Contents

- [1. Summary](#1-summary)
- [2. What was tested](#2-what-was-tested)
- [3. One schema, four uses](#3-one-schema-four-uses)
- [4. The world](#4-the-world)
- [5. Content files](#5-content-files)
- [6. Files to create](#6-files-to-create)
- [7. Tests that define done](#7-tests-that-define-done)
- [8. Left open for the build sessions](#8-left-open-for-the-build-sessions)

## 1. Summary

Every piece of game data is described once, by a schema written in TypeScript. That one schema gives the type in
code, the way the data is written in a KDL file, the way it is saved, and the way it reloads during development.
Nothing else may define a data shape.

This note fixes the parts that are expensive to change once content and modules exist:

- the closed list of value types (section 3.1);
- the difference between a *config* field and a *state* field (section 3.2);
- how entities are numbered, and the promise that a restored game continues exactly as an unbroken one (section 4);
- when a change to the world takes effect, and how events and commands are delivered (section 4);
- the rules that turn a KDL node into data, and how prefabs combine (section 5).

**What you do with it.** Build E1 from sections 3, 4, 6 and 7.1. Build E2 from sections 3, 5, 6 and 7.2. Where this
note gives a rule, follow it. Where it gives a code sketch, the names and behaviour are fixed and the typing details
are yours. Section 8 lists what is deliberately left to you.

**Qualifications.**

- The rules in sections 3 to 5 were tested in a throwaway project (section 2). That code is not in the repo. Do not
  look for it; build from this note.
- Speed was not tested. The 10,000-entity target in section 7.1 is a requirement, not a measurement.
- Level generators and anchors are only reserved here. Phases E7 and E12 design them.

## 2. What was tested

A 200-line test implemented the rules below with the `@bgotink/kdl` parser and checked them.

| Rule | Result |
|---|---|
| A component schema decodes its KDL node, with arguments, properties and child nodes | Passed for scalars, vectors, lists, lists of structs, entity names and content references |
| Mistakes are reported with file, line, column, an error code and a suggestion | 8 of 8 deliberate mistakes gave the expected code and position |
| `use` and `without` combine prefabs field by field | Passed. A list replaced the inherited list whole |
| Queries visit entities in ascending index order | Passed with entity slots reused 600 times |
| Save at tick 120, restore, run to tick 300: same state as an unbroken run to 300 | Passed. The state hashes were equal |
| A system sees each event exactly once, whatever the system order | Passed: 621 events raised, 621 seen by a system that runs before and after their source |
| Reload changes config fields and keeps state fields | Passed, and found a fault: reload overwrote an entity reference with its unresolved name. Section 3.2 now makes entity references state |

## 3. One schema, four uses

### 3.1. The value types

This list is closed. A field has exactly one of these types. There is no "any" type, no map type and no union type.
A shape that seems to need one is a list of structs.

| Type | Builder | In memory | In a KDL file | In a save |
|---|---|---|---|---|
| Number | `t.num(1.5)` | `number` | `1.5` | number |
| Whole number | `t.int(3)` | `number` | `3` | number |
| Yes or no | `t.bool(false)` | `boolean` | `#true` or `#false` | boolean |
| Text | `t.str('')` | `string` | `"door"` | string |
| One of a set | `t.oneOf('open', 'shut')` | `string` | `"open"` | string |
| Angle | `t.angle(0)` | radians | degrees | radians |
| Colour | `t.color('#ffffff')` | `number` (0xRRGGBB) | `"#ffffff"` | number |
| 2D vector | `t.vec2(0, 0)` | three.js `Vector2` | 2 numbers | array of 2 |
| 3D vector | `t.vec3(0, 0, 0)` | three.js `Vector3` | 3 numbers | array of 3 |
| Rotation | `t.rot()` | three.js `Quaternion` | 3 angles in degrees: about x, about y, about z | array of 4 |
| Entity | `t.entity()` | entity id, or 0 for none | the entity's name | entity id |
| Content reference | `t.ref('cue')` | the record's id, a `string` | `"grunt"` | string |
| List | `t.list(t.str())` | array | several values, or one child node per item | array |
| Struct | `t.struct({ … })` | plain object | a child node | object |
| Optional | `t.maybe(t.num())` | the value or `null` | the value or `#null` | the value or `null` |

Rules that go with the table:

- **Units.** A KDL file uses metres, seconds and degrees. Memory uses metres, seconds and radians.
- **Rotation.** The three angles are applied in the order y, then x, then z (three.js Euler order `YXZ`).
- **Numbers.** `NaN` and infinity are never valid. A save that meets one throws and names the component and field.
  Negative zero is saved as zero.
- **Ownership.** Each entity owns its values. Creating an entity deep-copies defaults, so no two entities share a
  vector or a list.
- **Names.** A component named `CharacterBody` is the KDL node `character-body`. A field named `stepHeight` is the
  KDL property `step-height`. The conversion is automatic and has no exceptions.

### 3.2. Config fields and state fields

Every field is one of two kinds.

- A **config** field comes from content and does not change during play: a maximum health, a radius, a model id.
- A **state** field changes during play: current health, position, ammunition left.

Mark a state field with `.state()`. A field is config unless marked. A field of type `t.entity()` is always state,
whether marked or not, because it points at one particular entity.

The kind matters in one place today: reload during development replaces config fields and leaves state fields alone
(section 5.8). Saves store both kinds in full.

### 3.3. How things are defined

All definitions use the same types. The second argument to a builder is its default.

```ts
import { defineComponent, defineEvent, defineCommand, defineResource, defineKind, t } from 'display-engine/core';

// A component: data on an entity.
export const Health = defineComponent('Health', {
  hp: t.num(100).state(),
  max: t.num(100),
});

// Fields named in `args` may be written as arguments, in this order: `team "wardens"`.
export const Team = defineComponent('Team', { id: t.str() }, { args: ['id'] });

// An event: a fact raised during a tick.
export const Damaged = defineEvent('damaged', { target: t.entity(), amount: t.num(), by: t.entity() });

// A command: a player's intent.
export const MoveOrder = defineCommand('move-order', { units: t.list(t.entity()), to: t.vec3() });

// A resource: one shared value per world. With a schema it is saved.
export const Score = defineResource('Score', { kills: t.int(0).state() });

// A content kind: named records in KDL files. `cue "grunt" bus="effects"`.
export const Cue = defineKind('cue', { bus: t.oneOf('effects', 'music', 'interface'), spatial: t.bool(false) });
```

A component may set `saved: false`. Use that only for data rebuilt from other data, such as a cached child list.

## 4. The world

### 4.1. Entities are numbers

An entity id is a 32-bit unsigned whole number. The low 20 bits are an *index*. The high 12 bits are a
*generation*.

- Index 0 is never used. The id 0 means "no entity".
- Despawning an entity adds 1 to its index's generation and puts the index on a free list. The next spawn takes the
  most recently freed index. An old id then fails `isAlive`, because its generation no longer matches.
- At most 1,048,575 entities can be alive at once.
- The free list and every generation are part of a save. A restored world therefore hands out the same ids as the
  world it was saved from. This is what makes a restored game continue exactly.

### 4.2. Reading and changing the world

```ts
const e = world.spawn(Health.with({ hp: 60, max: 60 }), Team.with({ id: 'wardens' }));
world.add(e, Regen, { rate: 2 });        // adding a component the entity has replaces it
world.remove(e, Regen);
world.despawn(e);                         // also despawns every entity whose Parent is e

world.isAlive(e);                         // false for id 0 and for stale ids
world.has(e, Health);
const h = world.get(e, Health);           // the live object, or undefined
h.hp -= 10;                               // field writes take effect at once

for (const e of world.query(Health, Team).without(Dead)) { … }
const boss = world.instantiate('warden', Health.with({ max: 200 }));   // from a prefab, with overrides
```

- **Order.** A query visits entities in ascending index order. This is a promise, and tests check it.
- **Caching.** `world.query(...)` with the same components returns the same cached query.
- **Missing data.** `world.get` returns `undefined` for a dead entity or a missing component. `world.need` throws
  with the entity and component named. Use `need` where absence is a bug.

### 4.3. When a change takes effect

Inside a running system:

1. Writing a field changes it at once.
2. `spawn`, `despawn`, `add`, `remove` and `instantiate` are queued. They are applied when the system returns, in the
   order they were called.
3. `spawn` and `instantiate` return the new id at once. Until the system returns, `get` on that id gives `undefined`
   and queries do not include it.

Outside a system (set-up, tests, loading a level), every change applies at once.

Add and remove hooks run while the queue is applied:

```ts
world.onAdd(Health, (e, data) => { … });      // just after the component is attached
world.onRemove(Health, (e, data) => { … });   // just before it is dropped
```

A despawn removes the entity's components in the order the components were registered, and calls each remove hook.
Children are despawned before their parent, in ascending index order.

### 4.4. Systems, stages and their order

A system is a named function in one stage.

```ts
export const regen = defineSystem({
  name: 'regen',
  stage: 'step',
  after: ['damage'],
  run(world, dt) { … },
});
```

Stages of one fixed tick, in order:

| Stage | Used for |
|---|---|
| `commands` | Make this tick's commands readable. Engine only |
| `before` | Decide: controllers and AI turn commands and state into intentions |
| `step` | Act: movement, combat, timers. Most gameplay goes here |
| `physics` | Step the physics world and copy results to components |
| `after` | React to results: damage from collisions, deaths, scoring |
| `rules` | Run level rules against this tick's events (section 5.5) |
| `cleanup` | Drop old events, advance the tick number. Engine only |

Stages of one frame, in order: `input`, then the fixed ticks, then `update`, `camera`, `sync`, `render`, `ui`. Frame
stages run only in the browser. They may read the world. They must not change it.

Order inside a stage: a system runs after every system it names in `after` and before every one it names in
`before`. Systems with no ordering between them run in plugin order, then in the order listed in their plugin. A name
that matches no system, or an ordering cycle, stops start-up with an error that names both systems. To order against
another module, use stages, not names.

### 4.5. Events

```ts
world.emit(Damaged, { target, amount: 30, by });

// In a system:
for (const ev of world.events(Damaged)) { … }

// In browser code, for sound, effects and the 2D layer:
sim.on(Damaged, ev => { … });
```

- **Each system sees each event exactly once**, in the order the events were raised. A system that runs before the
  event's source sees it on the next tick. A system that runs after sees it on the same tick. No ordering mistake can
  lose an event.
- An event is kept for two ticks, then dropped.
- `sim.on` listeners are called after each tick, with that tick's events, in order. A listener must not change the
  world.
- An event is data. Treat it as read-only. An entity named in an event may be dead by the time the event is read, so
  check `isAlive`.
- `world.events` throws outside a system. Outside one, use `sim.on`.

### 4.6. Commands

A command is the only way a player changes the game.

```ts
sim.send(MoveOrder, { units, to }, { player: 1 });      // browser side, any time

for (const c of world.commands(MoveOrder)) { c.player; c.data; }   // sim side, in a system
```

- A command sent during tick N is delivered in tick N + 1. It is readable for that one tick.
- Commands for a tick are ordered by player number, then by the order they were sent.
- Simulation code never reads the keyboard, mouse or gamepad. The input module turns devices into commands.
- The list of `[tick, player, command name, data]` is the recording of a session. Replaying it into a world with the
  same seed and content gives the same state.

### 4.7. Random numbers and seeds

- The generator is `sfc32`. Its four words of state are seeded from a 32-bit seed with `splitmix32`.
- `world.random()` returns a number from 0 up to but not including 1. Inside a system it draws from that system's
  own stream. Adding a random call to one system therefore never changes the numbers another system gets.
- `world.stream('loot')` gives a named stream for use across systems.
- Each stream is seeded from `hash32(gameSeed, streamName)`. All stream states are part of a save.
- `seedFor(gameSeed, ...parts)` is a pure hash for seeds that must not depend on call order, such as the seed of a
  level by its id.
- `hash32` is FNV-1a over the UTF-8 bytes of its parts joined by a zero byte.

### 4.8. Components the core defines

| Component | Fields | Notes |
|---|---|---|
| `Transform` | `position` vec3, `rotation` rot, `scale` vec3 (1, 1, 1). All state | Local to the parent when there is one |
| `Parent` | `entity` | A cycle throws. Despawning the parent despawns the entity |
| `Name` | `value` text | Unique within a level. Level rules find entities by it |
| `Origin` | `prefab` content reference, `spawn` whole number, `seed` whole number | Set by `instantiate`. `spawn` is the entity's place in the level file, or −1. `seed` is drawn from the `spawn` stream and never changes |

`worldTransform(world, e, out)` returns an entity's transform in world space. The core computes it on demand.

### 4.9. Saving, restoring and the state hash

`sim.snapshot()` returns a plain object that `JSON.stringify` can write. It holds:

1. the format number, and the hash of the content the game was running;
2. the tick number;
3. every random stream's state;
4. the entity generations and the free list;
5. every saved component, as rows of entity id and data, in ascending entity order, with components in name order;
6. every saved resource;
7. the events still being kept, and each system's place in them;
8. commands not yet delivered;
9. one entry per plugin that has a `save` function, such as the physics world.

`sim.restore(snapshot)` removes everything, calling remove hooks, then loads the snapshot, calling add hooks for
every component in ascending entity order. The browser side therefore rebuilds its views with no special case.

`sim.hash()` is the FNV-1a 64-bit hash of the snapshot written as canonical JSON, given as 16 hexadecimal digits.

**The promise.** Run to tick A, save, restore, run to tick B. The hash equals that of an unbroken run to tick B. Every
module must keep this promise, and the tests in section 7.1 enforce it for the core.

### 4.10. Plugins, and the two hosts

```ts
export interface Plugin {
  name: string;
  requires?: string[];
  components?: Component[]; events?: EventType[]; commands?: CommandType[]; resources?: Resource[];
  kinds?: Kind[]; actions?: Action[]; systems?: System[];
  setup?(sim: Sim): void;
  save?(sim: Sim): unknown;
  load?(sim: Sim, data: unknown): void;
}
```

Every name is unique across all plugins, or start-up fails and names both owners.

There are two hosts:

- `createSim({ plugins, content, seed, tickRate })` runs in Node and in the browser. It has `tick(n)`, `send`, `on`,
  `snapshot`, `restore`, `hash` and `world`. Tests use this.
- `createEngine({ canvas, … })` is the browser host built in E3. It owns a sim, turns real time into ticks (at most 5
  ticks a frame, then it drops the backlog), and runs the frame stages.

### 4.11. Rules for simulation code

These apply to `core`, `content`, `physics`, `nav`, `ai`, and every `sim/` folder in a kit or game. The boundary lint
from E0 enforces them.

- No `Math.random`, `Date`, `performance`, `setTimeout`, `setInterval` or `requestAnimationFrame`.
- No `window`, `document`, `navigator` or any other browser object.
- From three.js, only the maths classes.
- No state outside the world. A module-level variable that changes during play is a bug, because a save misses it.

## 5. Content files

### 5.1. Everything is a kind

A top-level KDL node is a record of some *kind*. The node's name is the kind. Its first argument is the record's id.

```kdl
cue "grunt" bus="effects" spatial=#true
```

- An id is lower-case letters, digits and hyphens. It is unique within its kind across all files.
- Plugins register kinds with `defineKind`. The core registers four: `prefab`, `level`, `language` and `game`.
- `game` has no id, and there is exactly one.
- Which file a record sits in does not matter. The folder layout in the plan is a convention.
- Content is read-only at run time. `content.get(Cue, 'grunt')` returns frozen data. `content.ids(Cue)` lists ids.

### 5.2. How a node is read

These rules apply to every kind, every component and every struct. They are the whole mapping.

1. **One value is a property.** `health hp=60 max=60`.
2. **More than one value is a child node** named after the field. Vectors, rotations, lists and structs are written
   this way: `position 1 0 3`.
3. **Fields listed in `args` may be arguments**, in order. A vector takes as many arguments as it has numbers. A list
   may be last in `args` and takes all the arguments left: `tags "guard" "night"`.
4. **A list of structs is one child node per item.** The node name is the field name unless the list sets `each`:
   `t.list(Slot, { each: 'slot' })` reads `slot "hand" item="keys"`.
5. **An entity field is a name.** It is resolved when the entity is created (section 5.3).
6. **A content reference is an id.** It must exist, or the file fails the check.
7. **A missing field takes its default.**
8. **Nothing is converted.** A string is never read as a number. A whole-number field rejects `1.5`.
9. **A node may appear once in a body.** Writing `health` twice in one prefab is an error.

```kdl
prefab "warden" {
    health hp=60 max=60                 // rule 1
    transform {
        position 1 0 3                  // rule 2
        rotation 0 90 0
    }
    team "wardens"                      // rule 3
    loadout {
        slot "hand" item="keys"         // rule 4
        slot "belt" item="lamp" count=2
    }
    follow target="leader" distance=3   // rule 5
    sound-on "damaged" cue="grunt"      // rule 6
}
```

### 5.3. Prefabs

A prefab body holds component nodes and four special nodes.

| Node | Meaning |
|---|---|
| `use "a" "b"` | Start from prefabs `a` then `b`. Later ones override earlier ones |
| `without "regen"` | Drop a component that came from `use` |
| `child "lamp" { … }` | A child entity with `Parent` set to this one. Its body is a prefab body |
| a component node | Set fields of that component |

How an entity's data is built, lowest priority first:

1. the schema defaults;
2. each prefab in `use`, in order, applied the same way;
3. the prefab's own nodes;
4. overrides from the level's `spawn` node;
5. overrides passed to `world.instantiate` in code.

A later step replaces only the **fields** it names. A list is replaced whole. There is no merging inside a list.

Entity names in a prefab are resolved when it is instantiated. The names available are `self`, `parent`, the name
of each `child`, and any names the spawn supplies. A name that matches nothing becomes 0.

`use` may not form a cycle. The check reports the cycle.

### 5.4. Levels

```kdl
level "crypt-1" {
    environment {
        fog "#06070c" near=2.5 far=9
    }
    generate "dungeon" seed=42 rooms=8
    spawn "warden" name="boss" {
        at 4 0 -2
        facing 90
        health max=200
    }
    spawn "rat" count=6 anchor="room-3"
    on "trigger-entered" zone="vault" once=#true {
        spawn "warden" count=2 anchor="room-5"
        play-sound "alarm"
    }
}
```

| Node | Meaning |
|---|---|
| a settings block | A struct that a plugin registers with `defineLevelBlock`, such as `environment`. At most one of each |
| `generate "<name>" …` | A request to a generator. E2 checks it against the schema the generator registered. E7 runs it |
| `spawn "<prefab>" …` | Create entities. Its body takes `at`, `facing` and component nodes as overrides |
| `on "<event>" … { … }` | A level rule (section 5.5) |

`spawn` takes these properties: `name` (sets the `Name` component), `count` (default 1), and `anchor`. `at x y z` and
`facing degrees` in the body are short forms for the `Transform` position and a rotation about y.

An **anchor** is a named place that a generator publishes, such as `room-3`. Anchor names cannot be checked when the
file is compiled, because a generator may run later. A spawn at an unknown anchor logs a warning at run time and uses
the level origin. E12 designs anchors in full.

### 5.5. Level rules

A rule says: when this event happens, do these actions.

```kdl
on "damaged" target="boss" name="boss-hurt" once=#true {
    emit "boss-angry"
    after 2.0 {
        spawn "rat" count=4 anchor="boss-room"
    }
    set "gate" {
        door open=#true
    }
    despawn "@by"
}
```

- **Filters.** A property that names an event field must equal that field for the rule to fire. An entity field is
  compared with a level name. There are no other comparisons.
- **Rule properties.** `once` (fire one time only), `name` (so other rules can enable and disable it), `enabled`
  (default true).
- **Entities in actions.** An entity argument is a level name, or `@field` for an entity field of the event that
  fired the rule.
- **Actions run in the `rules` stage**, in the order written. They follow section 4.3.
- **Rule state is game state.** Which rules have fired, which are enabled, and pending `after` timers live in a
  saved resource.

Actions the core provides:

| Action | Effect |
|---|---|
| `spawn "<prefab>" …` | As the level node |
| `despawn "<entity>"` | Despawn it |
| `set "<entity>" { component nodes }` | Write the named fields |
| `add "<entity>" { component nodes }` and `remove "<entity>" "<component>"` | Attach or drop components |
| `emit "<event>" field=value …` | Raise an event |
| `after <seconds> { actions }` | Run the actions later |
| `enable "<rule>"` and `disable "<rule>"` | Switch a rule on or off |
| `load-level "<level>"` | Request a level change |

Modules add actions with `defineAction(name, schema, run)`. An action that only affects what the player sees or hears,
such as `play-sound`, raises an event. The browser side listens for it.

**Where rules stop.** A rule has no variables, no arithmetic, no conditions beyond its filters, and no loops. When a
level needs any of those, write a system and have it raise an event that a rule can react to. Do not extend the rule
syntax.

### 5.6. From files to the game

`npm run content:check` and the build both run the same compiler:

1. Read every `.kdl` file under each content layer, in layer order: engine, then kits, then the game.
2. Parse each file as KDL version 2 with source positions kept.
3. Decode each top-level node with the rules in 5.2. Collect errors and carry on, up to 50.
4. Resolve references. Flatten `use`. Check rules against the registered events and actions.
5. If there are errors, print them and exit with status 1.
6. Write `src/content.gen.json` and `src/content.gen.ts`.

A record in a later layer replaces a record of the same kind and id in an earlier layer, whole.

`content.gen.json` has this shape. Vectors are arrays, angles are radians, and prefabs are already flattened.

```json
{
  "format": 1,
  "hash": "9f2c…",
  "kinds": {
    "prefab": { "warden": { "components": { "Health": { "hp": 60, "max": 60 } }, "children": [] } },
    "cue": { "grunt": { "bus": "effects", "spatial": true } }
  },
  "text": { "en": { "vault-alarm": "The vault is open. Something heard it." } }
}
```

`content.gen.ts` exports one `as const` object of ids per kind (`Prefabs`, `Levels`, `Cues`, …) and the loaded
content, so a wrong id in code is a compile error.

### 5.7. Errors

Every error is one line: `file:line:column CODE message`. The message says what was found, what was expected, and the
nearest known name when there is one within three edits.

```text
content/prefabs/warden.kdl:4:12 E-UNKNOWN-PROP unknown property "hp-max" on health. Did you mean "max"? Known: hp, max.
```

| Code | Raised when |
|---|---|
| `E-SYNTAX` | The file is not valid KDL version 2 |
| `E-UNKNOWN-KIND` | A top-level node names no registered kind |
| `E-UNKNOWN-NODE` | A node in a prefab or level body names no component, block or special node |
| `E-UNKNOWN-PROP` | A property names no field |
| `E-UNKNOWN-FIELD` | A child node names no field |
| `E-TYPE` | A value has the wrong type |
| `E-ARITY` | A vector or single-value field got the wrong number of values |
| `E-EXTRA-ARG` | A node has more arguments than its `args` allow |
| `E-NOT-PROP` | A field with more than one value was written as a property |
| `E-DUPLICATE` | Two records of one kind in one layer share an id, or a node appears twice in a body |
| `E-ID` | An id has characters other than lower-case letters, digits and hyphens |
| `E-MISSING-REF` | A content reference or `use` names a record that does not exist |
| `E-CYCLE` | Prefabs `use` each other in a loop |
| `E-UNKNOWN-EVENT`, `E-UNKNOWN-ACTION` | A rule names an event or action that is not registered |
| `E-FILTER` | A rule filter names a field the event does not have |

### 5.8. Reload during development

The Vite plugin watches the content folders, recompiles on a change, and sends the changed records to the page. The
engine applies them between ticks.

| What changed | What happens in the running game |
|---|---|
| A record of an ordinary kind (a cue, a table row, a text) | The content store takes the new record. The next read sees it |
| A config field in a prefab | Written to every live entity whose `Origin` names that prefab, or a prefab that uses it. Overrides from the level's `spawn` node are applied again on top |
| The default of a state field | New entities only |
| An entity-reference field | Never applied to live entities |
| A component added to or removed from a prefab | Added to or removed from live entities |
| A level file | The level restarts |
| `game.kdl`, a schema, or system code | The page reloads |

Two limits. Overrides passed to `world.instantiate` in code are lost for config fields on reload. A session that has
reloaded content can no longer be replayed from its recording, and the recording says so.

### 5.9. Text and languages

```kdl
language "en" {
    text "vault-alarm" "The vault is open. Something heard it."
    text "ammo-left" "{count} rounds left"
}
```

- A field of type `t.ref('text')` holds a text id. The 2D layer looks the id up in the current language.
- `{name}` in a text is replaced by a value the caller supplies. There is no plural or gender logic in version 1.
- `game.kdl` names the default language. Another language that lacks an id falls back to the default, and the check
  prints a warning.

## 6. Files to create

```text
packages/engine/src/core/
  schema.ts       the value types and the five define functions
  entity.ts       ids, generations, the free list
  world.ts        component stores, queries, queued changes, hooks
  schedule.ts     stages and system order
  events.ts       events and per-system positions
  commands.ts     commands and the recording
  random.ts       sfc32, streams, hash32, seedFor
  snapshot.ts     snapshot, restore, canonical JSON, the state hash
  components.ts   Transform, Parent, Name, Origin, worldTransform
  plugin.ts       the Plugin type and registration checks
  sim.ts          createSim
packages/engine/src/content/
  kinds.ts        the kind registry and the four core kinds
  decode.ts       KDL node to data: the rules in 5.2
  prefab.ts       use, without, child, flattening
  level.ts        blocks, generate, spawn
  rules.ts        on, filters, the core actions, the rules system
  compile.ts      the six steps in 5.6
  errors.ts       codes, positions, suggestions
  store.ts        the run-time content store and world.instantiate
  reload.ts       the table in 5.8
  vite.ts         the Vite plugin
  cli.ts          content:check and the generated content reference
```

Each file has a test file beside it.

## 7. Tests that define done

### 7.1. E1

1. Each value type round-trips: default, then snapshot, then restore gives equal data, and vectors are new objects.
2. A stale id fails `isAlive` after its index is reused. Id 0 is never alive.
3. A query visits entities in ascending index order after 1,000 mixed spawns and despawns.
4. Inside a system, a queued `add` is not visible until the system returns. Outside a system it is visible at once.
5. Add and remove hooks fire in the order given in 4.3, and children are despawned before parents.
6. An ordering cycle and an unknown system name each stop start-up with both names in the message.
7. A reader that runs before an event's source and a reader that runs after both see each of 1,000 events once.
8. Commands sent in tick N arrive in tick N + 1, ordered by player then send order, and are gone in tick N + 2.
9. Adding a `world.random()` call to system A does not change the numbers system B draws.
10. **Repeatability.** Two sims with the same seed and command recording have equal hashes after 1,000 ticks.
11. **The promise in 4.9.** Save at tick 400, restore into a new sim, run to 1,000: the hash equals the unbroken run.
    The test game must spawn and despawn entities, use random numbers, raise events and take commands.
12. A save that meets `NaN` throws and names the component and field.
13. The lint in 4.11 fails a file that calls `Math.random` in a simulation folder.
14. **Speed.** With 10,000 entities and 20 queries, one tick of empty systems costs under 1 ms in Node.

### 7.2. E2

1. The five files in Appendix B of the plan check and compile. The test registers stand-in components and kinds for
   the ones later modules will own, such as `art-model`, `cue` and `action`.
2. Each rule in 5.2 has a test that decodes a node and compares the data.
3. Each code in 5.7 has a broken file that raises it at the right line and column. There are at least 20 broken
   files in all.
4. One run reports several errors from several files, not only the first.
5. The priority order in 5.3 holds: defaults, `use` in order, own nodes, spawn overrides, code overrides.
6. `without` removes an inherited component. A list in a later step replaces the earlier list whole.
7. A prefab instantiated from KDL equals the same entity built in code with `world.spawn`.
8. Entity names resolve to `self`, `parent`, a `child` and a level name. An unknown name becomes 0.
9. A rule with a filter fires only on a matching event. `once` fires one time. `after` fires at the right tick.
   `@field` names the event's entity.
10. Rule state survives save and restore: a rule that fired `once` before the save does not fire after the restore.
11. A game record in a later layer replaces the engine's record of the same kind and id.
12. Reload: each row of the table in 5.8 has a test. The entity-reference row is the fault found in section 2.
13. In a browser test, saving a KDL file changes a live entity's config field without a page reload.
14. `docs/content-reference.md` is generated and lists every registered kind, component, block, event and action.

## 8. Left open for the build sessions

These are yours to decide. None of them changes the contract.

- How component stores and query caches are built, as long as the order promise and the speed test hold.
- The exact TypeScript generics, as long as `world.get(e, Health)` is typed as the component's data.
- How the Vite plugin sends changes to the page.
- The layout of the generated content reference.
- Whether `worldTransform` caches within a tick.

Decided against, so do not add them: change tracking on fields, map and union value types, more than one way to
write a value in KDL, expressions in rules, and patching part of a record from a later layer.
