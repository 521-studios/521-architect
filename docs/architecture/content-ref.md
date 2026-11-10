# ContentRef and `modifications[]` — cross-service contract

Status: **DRAFT** (2026-11-10). Owner: Abadar (521-architect). Agreed with
521-engine, encounter-builder-api, party-treasure-api. Supersedes the informal
shape in [encounter-treasure-cluster.md §4](encounter-treasure-cluster.md).

ContentRef is how every service in the encounter/treasure cluster holds "a
monster" or "an item". It is stored by encounter-builder-api, copied verbatim
to party-treasure-api on release, and resolved by clients against
pfsrd2-data-api (and later 521-engine). Because three services touch it, its
shape is a contract, not an app detail.

## Shape

```
ContentRef :=
  { game_id, version }                       # pristine dataset entry
| { base: ContentRef, modifications: [Mod],  # derived
    json: {...} }
| { json: {...} }                            # fully custom
  (+ price_cp, optional, on any variant)

Mod := { kind, id, params, v, engine_version? }
```

### The wrapper is closed

The wrapper fields are exactly `game_id`, `version`, `base`, `modifications`,
`json` and `price_cp`. Both encounter-builder-api and party-treasure-api decode
it into a typed Go struct, so **any new wrapper-level field is silently
dropped**. Adding one means a one-line change in both repos, landed before any
client sends it. New information goes inside `Mod` elements instead.

### `Mod` elements are open

Both APIs hold each element as `json.RawMessage` and never read it. The only
check is that a derived ref has a non-empty list. Unknown keys inside an
element survive every round trip.

| Field | Meaning |
|---|---|
| `kind` | Discriminator. Decides how `params` is read. |
| `id` | Unique per application (ULID). Two "+2 AC (GM)" overrides must not collide. |
| `params` | Kind-specific. Every game object it references carries a pinned version. |
| `v` | Schema version of this element's `params`. Not the same as `version` inside `params`, which is the pfsrd2 schema version of a referenced object. |
| `engine_version` | On kinds that 521-engine resolves: the engine semantics the snapshot was computed under, so a stored snapshot can be recomputed when stacking rules change. |

## Truth, cache and replay

- **`modifications` is the source of truth; `json` is a cache.** `json` is the
  full resolution, GM overrides included. Otherwise overrides never reach a
  released item.
- **Release freezes `json`.** party-treasure-api stores what it was sent.
- **Replay is exact only if inputs are pinned.** A pristine ref or `base`
  carries `version` (the pfsrd2 schema version). Every object referenced from
  `params` (templates, runes, spells) carries its own version. An unpinned
  reference replays against whatever the data says today, which is a silent
  change after errata or a schema bump.
- **"Refresh to latest" is an explicit user action.** It rewrites the pins and
  `json`. Replay never does it implicitly.
- **Pinned versions are never pruned.** The data-api serves old schema
  versions through the reconciled `entry_versions` table
  ([data-pipeline.md](../pipeline/data-pipeline.md)), so pinned JSON must stay
  in S3. Do not publish a DB built with `--skip-reconcile`.

## Ordering

Elements apply in array order, and the order is semantic. Numeric bonus
stacking does not depend on order (stacking rules are data per bonus type).
Order matters for list/set operations and last-wins strings.

## Security

(Devon, 2026-11-10.)

- **GM-authored kinds are declarative only:** target, value, bonus_type,
  simple ops. No string in a `Mod` is ever evaluated as code.
- **Rule logic is referenced, never embedded.** Vetted registry compute is
  compiled to ES modules at build time and referenced by a **content-hashed
  id**, the same convention as `game_id`. A changed body gets a new id, so a
  stored element cannot pick up new behaviour. Registry build output is
  append-only, and old hashes stay loadable.
- **Compiled compute must be pure:** no `Date`, `Math.random` or I/O, and the same output for the same inputs. Client and server must agree for replay to be exact.
- There is no runtime `eval` or `new Function` in the browser path, so the
  SPAs can run under a CSP without `unsafe-eval`. Client (browser) and server
  (goja) load the same built modules.

## What is not a ContentRef

**Play-time state is not a ContentRef.** HP, conditions and initiative belong
to live combat, which **lets-roll** owns (Devon, 2026-11-10). Encounter
Builder is prep-only: prep, release, then run the encounter in lets-roll.
Conditions never enter `modifications`, so released snapshots never carry
stale temporary state.

lets-roll keeps live state per encounter monster in its own
`combat_mods jsonb` list (lets-roll-jlm). That list reuses the `Mod` element
shape (`condition` with a pinned `{game_id, version, value}`, and later
`override`), so the engine resolves both lists the same way. It is never
copied into a ContentRef and never released. Templates applied at the table,
such as elite/weak, are ContentRef `template` Mods in lets-roll's
`content_ref` column, not live state. Initiative is a plain column in
lets-roll's turn order, not a Mod.

## Validation

encounter-builder-api and party-treasure-api keep elements opaque. Validating
`kind`, `params` and pins is the resolver's job (pfsrd2-data-api today,
521-engine later). If an API ever needs to validate, it is scoped as API work
in that repo.

## Kinds

| `kind` | Status | Notes |
|---|---|---|
| `template` | **Shipped, pre-spec shape** | Legacy elements are `{template_game_id, template_name}`, with no `kind`, `id`, `v` or pin (`encounter-builder-web/src/monsterRef.js`). Writers move to `{kind:"template", id, params:{game_id, version}, v:1}`. |
| `item_effect` | **Shipped, pre-spec shape** | Legacy elements are `{effect_game_id, effect_name, grade, price_mode:'add'\|'set'}`, where a missing `price_mode` means `'add'` (`encounter-builder-web/src/itemRef.js`). Writers move to `{kind:"item_effect", id, params:{game_id, version, grade, price_cp, price_mode}, v:1}`. |
| GM override | Planned (hzg8/kdv8) | Declarative effect. Replaces the proposed per-instance overrides field on MonsterEntry. |
| Item construction (runes, spell in scroll) | Planned | Owned by pfsrd2-data-api itemapply. |
| Engine game objects | Planned | 521-engine Phase 5. (Conditions are a Phase 4 consumer in lets-roll's `combat_mods`, not a ContentRef.) |

**Reading legacy elements.** An element without `kind` is classified by its
keys: `template_game_id` means a legacy `template`, and `effect_game_id`
means a legacy `item_effect`. Anything else is an error.

**`MonsterEntry.adjustment`** (`none|elite|weak`) is a legacy field on
encounter-builder's monster line, outside the ContentRef. Readers honour it
only on pristine refs, and a derived ref's template Mods take precedence (as
`encounter-builder-web/src/budget.js` already does). New writes express
elite/weak as `template` Mods, matching lets-roll's `content_ref`.

## Open

- Template stacking (`4zx0`) waits for 521-engine Phase 5 (templates as
  engine game objects). pfsrd2-data-api's `internal/template/engine.go` is the
  regression oracle (Devon, 2026-11-10).
- Migrating stored legacy template elements: read-compatible forever, or a
  one-time rewrite.
