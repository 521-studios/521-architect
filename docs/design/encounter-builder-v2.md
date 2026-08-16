# Encounter Builder v2 — two-pane UI, chapters, templates, item search

> **Status — 2026-08-14: SHIPPED (staging).** Epic `0ybu` closed. Chapters landed
> as a first-class entity (`be75`); encounter v2 fields + templates via derived
> ContentRef (`sdmq`); the library-first split held (Markdown/CreatureSearch/
> ItemSearch/template helpers exported from pfsrd2-display, hand-rolled
> `MonsterSearch.jsx`/`TreasureLine` deleted); Playwright e2e harness (`r2ex`).
> Treasure budgeting from the "data-api follow-ups" grew into its own shipped epic
> `ncnq` (numeric price, difficulty bands, chapter/campaign rollups). Kept as the
> design-of-record; open decisions below are resolved.

## Guiding principle (non-negotiable)

**pfsrd2-display is THE library for displaying and selecting Pathfinder 2 objects.**
Anything display- or selection-related that's missing gets **added to pfsrd2-display
as a library export** — never reimplemented in encounter-builder-web or worked
around. Data-layer gaps go in **pfsrd2-data-api**. encounter-builder-web stays a
thin consumer that wires library components to its own domain (encounters, chapters).

Corollary: encounter-builder-web currently hand-rolls `MonsterSearch.jsx` + a bare
`TreasureLine` — this v2 **deletes those** in favor of library components.

## UX target

After a campaign is selected, the app is a **two-pane** layout:

- **Sidebar (left):** the selected campaign name at the top (click → switch
  campaigns); below it a list of **chapters**. Click a chapter → it expands to
  list its encounters, **naturally sorted** (so `1. … 2. … 15.` and `B1 … B2 …
  B15` both order correctly, not lexically). A **(+)** at the bottom adds a new
  encounter.
- **Main pane (right):** the selected encounter — a **markdown description** at the
  top (eventually editable), a **monster list**, and a **treasure list**.
- **A monster**, opened, shows its **stat block** (CreatureStatBlock) and can have
  templates applied: elite/weak *and* arbitrary pfsrd2 templates, with change
  highlighting.
- **Treasure** is chosen via an **item search** (not a raw game_id), and rendered
  with **ItemCard** (masked/identify already modeled).

## Library-first split (what goes where)

### pfsrd2-display (add these EXPORTS + wire into the deployed harness)
pfsrd2-display is both a **library** (exports consumed by encounter-builder-web) and
a **deployed validation harness** (the `dev/harness.jsx` app, built to `dist-site`,
served at **display.pfsrd2.staging.521studios.com** via `pfsrd2-display-cf`). The
harness is the reference consumer. So for every library change below:
1. add the export to the library, AND
2. wire it into the harness app so the harness demonstrates it, AND
3. **Playwright-validate it on the deployed harness** (the harness has NO item search
   today — adding item search there is part of the ItemSearch work).

Investigation: only 8 symbols are exported today (`CreatureStatBlock[/V1_2/V1_3]`,
`ItemCard`, `DisplayProvider`, `useDisplay`, `useIsChanged`, `buildChangedPaths`).

- **`Markdown`** — exists (`src/shared/Markdown.js`, markdown-it + PF2 action
  glyphs) but isn't exported. **One-line export.** (encounter description render.)
- **`<CreatureSearch onPick apiBase types edition>`** — extract the harness
  `SearchPanel` (`dev/harness.jsx:48-126`), folding in the app's monotonic
  `seq`-ref stale-response guard (better than the harness's timer-only version).
  Endpoint `GET /search/suggest/unified?q=&type=monsters&type=npcs` (repeated
  `type`), 250ms debounce, min length 2, `{game_id,name,type,level,edition,
  alternate?}`, `↔` cross-edition twin.
- **`<ItemSearch onPick apiBase types>`** — NEW, same shape as CreatureSearch but
  over item types. `suggest/unified?type=equipment&type=weapons&type=armor&type=
  shields` already returns unified multi-type results (live-confirmed) — MVP needs
  no data-api change.
- **Template helpers/UI** — extract from the harness (none exported today):
  - `listTemplates(apiBase, edition)` → `GET /search?type=monster_templates&
    applicable_to=<edition>` (edition-aware server-side; do NOT reimplement).
  - `applyTemplate(apiBase, creature, templateGameId, selections?)` → `POST
    /templates/apply` **with the `x-amz-content-sha256` body-hash header** (the
    OAC requirement we already implement in the app), parse the **multipart**
    `{patches, creature}` response.
  - `mergePatches(stack)` → flattened `applied_patches` tagged by template for
    `CreatureStatBlock`'s `patches` prop.
  - Optional `<TemplatePicker>` component composing the above. The arbitrary-
    template **SelectionsPanel** (~460 lines) is a later extraction — MVP applies
    with `selections: []`.
- **Consumption hygiene:** the git dep currently **floats on default-branch HEAD**
  (no ref) and `prepare`-builds on install. Pin encounter-builder-web to a commit
  SHA/tag and bump deliberately.

### pfsrd2-data-api (data layer)
MVP works on existing endpoints. Follow-up gaps (own tickets, not blockers):
- **Unified multi-type item search WITH metadata** — `/search?type=` is single-type
  only; `suggest/unified` is multi-type but returns no price/bulk/traits. A treasure
  picker that facets across all item types with metadata needs this.
- **Numeric price** — `attrs.price` is a display string ("135 gp", sometimes null);
  no numeric/copper field, so no price range filter/sort and **no by-level treasure
  budgeting**. Add a normalized numeric price column + range params.
- **Populate `item_category`/`item_subcategory`** — columns exist but the indexer
  leaves them null; blocks category grouping/faceting.
- **Template enhancements (optional):** per-monster validity pre-check (only edition
  filtering exists today), multi-template stacking in one request, `?format=json`
  single-body alternative to multipart.

### encounter-builder-api (backend model)
Encounters are flat under `pk=CAMPAIGN#<gid>, sk=ENCOUNTER#<id>` today. v2 adds:

- **Chapter entity** — `{id, campaign_id, name, order, created_at, updated_at}`,
  stored `pk=CAMPAIGN#<gid>, sk=CHAPTER#<cid>`. CRUD API
  (`/api/app/campaigns/:id/chapters`), incl. reorder. A campaign fetch returns
  chapters + encounters in one `Query` on the campaign pk; the client builds the
  tree.
- **Encounter v2 fields:** `chapter_id` (which chapter), `description` (markdown).
  Existing encounters have no chapter → back-compat: a default **"Unsorted"** chapter
  (nullable chapter_id groups there).
- **Monster templates** via the **derived `ContentRef`** the model already defines:
  `{base:{game_id}, modifications:[{template_game_id, selections}…], json:<resolved
  creature>}`. Store the resolved snapshot (for display) + provenance (for
  re-editing/refresh). Note the DynamoDB 400KB item ceiling — a few templated
  creatures are fine; spill `json` to S3 only if ever hit (already noted in the
  store). Whether elite/weak stays the simple `Adjustment` field or folds into
  `modifications` is an implementation detail for that ticket.
- **Treasure refs** become real item game_ids from ItemSearch (no model change —
  `ref.game_id`); ItemCard renders via `entries/{id}/full`.

### encounter-builder-web (thin consumer)
- Two-pane layout shell + sidebar campaign switcher.
- Chapter tree (expand → encounters), **natural-sort util** (unit-tested against the
  `1..15` and `B1..B15` cases), (+) add encounter, chapter CRUD.
- Encounter main pane v2: `Markdown` description, monster list (`CreatureSearch` +
  `CreatureStatBlock` + `TemplatePicker`), treasure list (`ItemSearch` + `ItemCard`).
- Delete the hand-rolled `MonsterSearch.jsx` + bare `TreasureLine` inputs.

## Testing (build the harness Devon asked for)
Replace the throwaway scratch scripts with **real Playwright functional harnesses**,
one per app that has a UI:
- **encounter-builder-web** `e2e/`: playwright config, an OIDC **login helper**
  (reusable; creds are non-secret staging), feature specs for the v2 flow (chapters,
  natural sort, monster+template, item search, markdown, release) against
  `encounters.staging.521studios.com`.
- **pfsrd2-display** `e2e/`: specs against the deployed harness
  `display.pfsrd2.staging.521studios.com` for creature search, template apply +
  highlighting, **item search** (new), and ItemCard render.

Wire a **post-deploy staging-test job** in each repo (and a local runner). Every
feature ships with a spec. Every PR runs the full pr-review-loop to convergence
before merge, and its feature is Playwright-validated on staging.

## Sequencing
1. **Library exports** (Markdown, CreatureSearch, ItemSearch, template helpers) — unblock everything; each its own PR to pfsrd2-display.
2. **Backend** chapters + encounter v2 (encounter-builder-api) — parallel with (1).
3. **Frontend** two-pane + chapter tree + v2 editor consuming (1)+(2).
4. **Playwright harness** grows alongside — each frontend feature lands with a spec.
5. **Data-api enhancements** (multi-type metadata search, numeric price/budgeting,
   categories, template stacking) — follow-ups after the MVP two-pane works.

## Decisions (resolved 2026-08-14)
- **Chapters as a first-class entity** — resolved: first-class (bead `be75`),
  with an "Unsorted" default for back-compat.
- **Arbitrary-template selections UI** — resolved: elite/weak + no-selection
  templates first; full SelectionsPanel extraction stayed deferred.
- **Treasure budgeting** (award-by-level → gp) — resolved: shipped as its own epic
  `ncnq` (numeric price + Table 5-3 difficulty bands + chapter/campaign rollups),
  which also landed the data-api numeric-price work it depended on.
