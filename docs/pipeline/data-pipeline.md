# Data Pipeline Architecture

## Overview

The PFSRD2 data pipeline transforms hand-maintained HTML from Archives of Nethys (AoN) into structured, searchable JSON served by a production API. The pipeline is mostly manual and batch-oriented — content updates from AoN happen periodically, not continuously.

```
Archives of Nethys (2e.aonprd.com)
       ↓ wget (manual, periodic)
pfsrd2-web (normalized, patched HTML)
       ↓ pfsrd2-parser (Python)
pfsrd2-data (structured JSON, sorted keys)
       ↓ pfsrd2-automation (indexer + uploader)
S3 (JSON files + SQLite index DB)
       ↓ ETag watcher (hourly, automatic)
pfsrd2-data-api (Go Lambda, serves to clients)
```

Side channel:
```
~/.pfsrd2/*.db (enrichment SQLite DBs) + Qwen (LLM-assisted)
       ↓ post-parse enrichment
pfsrd2-data (updated entries)
```

## Stage 1: HTML Acquisition (pfsrd2-web)

**Repo:** `devonjones/PFSRD2-Web` (MasterworkTools org)

AoN's HTML is hand-maintained and contains bugs. The acquisition process is designed to make this HTML diffable and patchable:

### Download Process

1. Check out `upstream` branch
2. Run `download.sh` — full `wget` of 2e.aonprd.com (~2.9 GB)
3. Commit as `download/<date>` branch, create zip backup
4. Check out `main`, `rm -rf 2e.aonprd.com`, unzip fresh download
5. Run numbered normalization scripts (`1.start.sh` through `4.everything_else.sh`)
6. Commit normalized HTML to `main`
7. Check out `modified`, merge `main` — retains hand-curated bug fixes

### Normalization Philosophy

The key insight: **whitespace isn't meaningful in HTML, but it's meaningful in diffs.**

- `xmllint --format --html` pretty-prints the HTML
- Then strip all newlines and re-add one after every `>` — one tag per line
- This means patches to fix AoN bugs produce clean, reviewable diffs
- When a new download arrives, the normalization is reapplied and `modified` is merged — most bug-fix patches survive the merge automatically

### Branch Strategy

| Branch | Purpose |
|--------|---------|
| `upstream` | Receives raw wget downloads |
| `download/<date>` | Snapshot branches for each download |
| `main` | Latest normalized HTML (no bug fixes) |
| `modified` | Normalized + hand-curated HTML bug fixes — **parser reads from here** |

### Common HTML Fixes

- Reorder incorrectly placed weakness/resistance/immunity blocks
- Fix trait name typos
- Normalize encoding issues (bulk values, HTML entities)
- Merge split HTML spans that should be atomic
- Remove malformed/duplicate closing tags

## Stage 2: Parsing (pfsrd2-parser)

**Repo:** `devonjones/PFSRD2-Parser` (521Studios local)

Python parser with dedicated modules for 14+ content types. Uses BeautifulSoup4 + lxml.

### Multi-Pass Architecture

Every parser follows the same 10-14 pass pipeline:

1. `parse_universal()` — Initial HTML parsing
2. `entity_pass()` — Normalize HTML entities
3. `restructure_<type>_pass()` — Transform to domain structure
4. `aon_pass()` — Extract AoN metadata (aonid, game-obj)
5. `section_pass()` — Extract structured fields
6. Type-specific passes — Game mechanics extraction
7. `trait_db_pass()` — Enrich traits from database
8. `game_id_pass()` — Generate content-addressable IDs
9. `license_pass()` — Add license information
10. `edition_pass()` — Determine legacy vs. remastered (must run before cleanup)
11. `remove_empty_sections_pass()` — Cleanup
12. `markdown_pass()` — Convert HTML to markdown
13. Schema validation — Validate against JSON schema
14. Write output — Write JSON files

### Design Principles

- **Fail fast on unknowns** — exceptions propagate, never silently swallowed
- **Zero errors policy** — only acceptable state is 100% of files parsing clean
- **Structured data over text blocks** — extract game mechanics into queryable fields
- **Error seed workflow** — failed files can be re-run individually for fast iteration

### Game ID System

Content-addressable: `MD5("{source_name}: {page}: {name}")`. Enables reconciliation across parsing sources (AoN HTML, PDFs, etc.). Not tied to Archives of Nethys IDs.

## Stage 3: Data Output (pfsrd2-data)

**Repo:** `devonjones/PFSRD2-Data`

21,000+ JSON files organized by content type and source book.

### Diffability by Design

JSON is written with **keys sorted alphabetically**. Same philosophy as pfsrd2-web's HTML normalization — the output format is optimized for `git diff` review, not for machine consumption. When a parser run produces changes, the diff shows actual data changes, not random key reordering.

### Schema Versioning

- 13 JSON schema files define the output contract
- Breaking changes → new git branch in pfsrd2-data (e.g., `schema1.0`, `schema1.3`)
- Non-breaking additions → stay on `main`
- Multiple schema versions can coexist in S3 via `entry_versions` table

## Stage 4: Enrichment Pipeline

A secondary pipeline that updates data after initial parsing:

- SQLite databases at `~/.pfsrd2/*.db` hold enrichment data
- Some enrichments are LLM-assisted (Qwen)
- Used for data that can't be extracted from HTML alone or needs semantic understanding
- Updates flow back into pfsrd2-data before indexing

## Stage 5: Indexing & Upload (pfsrd2-automation)

**Repo:** `521-studios/pfsrd2-automation`

Python package (uv, boto3 as the only dependency) that bridges parsed JSON to the
production API. `bin/pf2_build_index` runs the whole thing:

**Build**
1. Walk pfsrd2-data; a per-type `TypeExtractor` pulls core columns plus a
   type-specific `attrs` JSON blob and a flat `search_text` blob. Unknown types
   fall through to a `GenericExtractor` rather than failing the build.
2. Schema version per type is inferred from the root `*.schema.N.N.json`
   filenames, with aliases for types that reuse another type's schema
   (npcs→creature, weatherhazards→hazard, curses/diseases→affliction).
3. Upsert `entries` + `entry_versions`, resolve image filenames to S3 keys.
4. **Prune ghosts** — rows whose `game_id` is no longer in the data. Renames
   shift the content-addressable game_id, and an upsert-only builder left the
   old row searchable forever (`Arms Of Nature` → `Arms of Nature`). Pruning is
   scoped to the types actually built, so a `--type` run can't wipe other
   content. Corollary: a *whole type* vanishing from the data is not pruned.
5. Rebuild the FTS5 and trigram indexes, then resolve `alternates`
   (legacy ↔ remastered, via game-obj + aonid) and the hand-curated
   cross-type `equivalents`.

**Upload** to `521studios-{env}-pfsrd2-data`
6. JSON files, schema files, images, and the SQLite DB. Uploads are parallel
   (process pool) and **ETag-skipped** — a local MD5 matching the S3 ETag means
   no transfer.
7. **Parity check (hard failure).** ETag-skipping is what makes repeat runs
   fast, but a sync launched from a wrong-branch clone leaves *mixed* old/new
   objects that skipping then never repairs — silently serving stale rules.
   After the JSON upload, every entry's S3 ETag is compared against the local
   file's MD5 in one paginated listing; any drift exits non-zero naming the
   drifted keys. This exists because that failure mode produced two false
   defect reports.
8. **Reconcile** — list everything under `json/`, parse each key back to its
   `base_path`, and upsert `entry_versions` so the API can serve any schema
   version present in the bucket. Keys that map to no current entry are logged
   as ORPHAN. The DB is re-uploaded afterward with the reconciled table.

**CI** — staging auto-builds on pushes to `main`; production is manual
`workflow_dispatch` with a main-branch guard. Both authenticate via GitHub OIDC.

Note the trigger scope: the push trigger watches *indexer code only*. Content
changes in pfsrd2-data do **not** currently fire a build — data ships are a
deliberate manual dispatch, so a human decides when new rules go live, matching
the batch cadence of upstream AoN releases. This is expected to change: once
data quality is gated automatically, a `repository_dispatch` from pfsrd2-data
should drive staging builds on merge.

## Stage 6: API Serving (pfsrd2-data-api)

**Repo:** `521-studios/pfsrd2-data-api`

Go Lambda that serves the indexed data:

- **Cold start:** Downloads SQLite DB from S3 (ETag-based caching)
- **Background watcher:** Checks S3 ETag hourly, atomically swaps DB if changed
- **Endpoints:** Search (FTS5), suggest (trigram), entries, types, sources, templates, images
- **Template engine:** RFC 6902 JSON Patch for monster variants (Elite, Weak, etc.)
- **Edition auto-resolution:** Automatically maps between legacy and remastered content

The API is stateless — the SQLite DB is the only state, and it's downloaded fresh. No managed database dependency beyond what `infra` provides for lets-roll.

## Key Design Decisions

### Why SQLite for the API?

No managed database overhead. The entire game rules corpus fits in a single SQLite file (~5-10 MB). Cold start includes the download, but ETag caching means subsequent invocations skip it. Read-only, single-connection pool — perfect for Lambda.

### Why content-addressable game IDs?

`MD5("{source}: {page}: {name}")` gives a stable identifier that works across parsing sources. If content is re-parsed from a different HTML structure, the game ID stays the same as long as the citation is the same.

### Why sorted JSON keys?

Same reason as one-tag-per-line HTML: optimize for human review via `git diff`. The data pipeline is manually operated — every step involves Devon inspecting output. The format serves the operator, not the machine.

### Why batch, not streaming?

AoN publishes content in book-sized batches. There's no real-time feed. The manual batch process matches the source cadence and gives full control over quality — every parser run is reviewed before data ships.
