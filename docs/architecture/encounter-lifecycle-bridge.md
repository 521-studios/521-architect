# Encounter Lifecycle and the lets-roll Combat Bridge

Status: ACCEPTED (Devon, 2026-11-10), including the conflict rule.
Beads: epic `bd_521Studios-we5k.2` (under M3). lets-roll prerequisite: `lets-roll-jlm`.

This note answers how Encounter Builder's progress state stays consistent with lets-roll's live combat state.

## Roles

lets-roll is a dice roller plus initiative tracker.
It pulls in an encounter, fills initiative with the encounter's monsters, and records damage.

Encounter Builder (EB) is the record of the encounter.
It owns the encounter's lifecycle, which creatures were killed, and the treasure package.

| Owner | Owns | Flows to the other side |
|---|---|---|
| EB | Lifecycle, kill state, the treasure package, close-out notes | The encounter and its monsters (ContentRefs) |
| lets-roll | Live combat: initiative, damage and HP, conditions (`combat_mods`, `content_ref`, `initiative` on `encounter_monsters`) | Only kills and the outcome |

HP, damage and conditions never flow back to EB.

## Lifecycle

`prepared` → `executing` → `complete`, then the treasure package is released to party-treasure.

1. **Prepared.** The GM has built the encounter in EB.
2. **Executing.** The encounter enters combat either way: lets-roll pulls it (`we5k.2.2`), or EB pushes it to lets-roll (`we5k.2.3`). The GM can drive from either side. An encounter with no combat can skip lets-roll entirely and go straight to complete.
3. **Kills.** After combat, the creatures killed in lets-roll are recorded on the EB encounter. The GM can also mark creatures killed directly in EB. Kill state syncs both ways (`we5k.2.4`).
4. **Complete.** The GM marks the encounter complete in EB and can add close-out notes (`we5k.2.1`).
5. **Package and release.** The package is the encounter's treasure plus the gear of every killed creature (`77l2`). The GM marks anything that doesn't carry over, such as a quaffed potion or a shattered sword, and can add a note to each item as it's closed out. The GM then releases the package to party-treasure (`we5k.2.5`). The release is the last step of this flow, not a standalone button.

## Integration

Both directions are server-to-server, like the existing release push (SigV4; see [service-dependencies.md](service-dependencies.md)).
This adds two server-to-server edges between encounter-builder-api and lets-roll, after the release push.

### EB → lets-roll push (decided, Devon 2026-11-13; revised after lets-roll's review)

**Endpoint.** `POST /api/v1/games/:id/eb_encounters` returns `201` on create and `200` on a replay or sync (`lets-roll-f6e`).
It is an upsert on `(game, eb_encounter_id)` through one shared service, `Encounters::ImportFromEb` (`lets-roll-dur`, after `lets-roll-jlm`).
The pull (`we5k.2.2`) and the push (`we5k.2.3`) both use that service.
The service does its own ActionCable broadcasts, so open tables update.

**Auth: SigV4 plus the GM's bearer.** A push carries two proofs, and lets-roll checks both.
- *Service identity.* lets-roll sits behind an ALB with no IAM authorizer, so EB proves its identity with a presigned STS `GetCallerIdentity` request (the Vault IAM-auth pattern).
  - EB sends the presigned request in an `X-EB-Identity` header.
  - The signed headers include `X-EB-Audience` (`lets-roll-<env>`) and `X-EB-Body-SHA256`, which bind the proof to this audience and this payload.
  - lets-roll checks that the URL is the STS endpoint, then checks the audience and the body hash.
  - lets-roll then forwards the request to STS and requires the returned role ARN to equal `EB_LAMBDA_ROLE_ARN`.
- *User.* EB also forwards the GM's lets-roll OIDC bearer. lets-roll verifies it and requires `current_user` to be GM of game `:id` (`game.user_id`, or a GM `game_role`). The bearer identifies the user, so no separate `acting_sub` is sent.

**Back-link** (`lets-roll-6ru`). These keys are also what kill sync (`we5k.2.4`) matches on.
- `encounters.eb_encounter_id` is unique per game.
- `encounter_monsters.eb_monster_key` is `"<EB content item id>:<n>"`, with one row per creature.
- Both columns have partial unique indexes, which apply only where the column is set, so encounters created in lets-roll itself are unaffected.

**Matching and names.** lets-roll matches on `eb_encounter_id` first. A new encounter whose name is already taken in the game is named `"Name (EB)"`.

**Re-push syncs (lets-roll's split).** A re-push adds, updates and removes monsters, matched by `eb_monster_key`.
- EB's data overwrites `name`, `content_ref` and `max_health`. Elite/weak travel inside `content_ref` as template Mods; there is no separate adjustment field.
- If `max_health` changes, the damage already taken is kept: `health = max(0, new_max - (old_max - old_health))`.
- Apart from that shift, a re-push never touches `health`, and it never touches `nickname`, `combat_mods` or `initiative`.
- A monster missing from the payload is kept, and reported in the response, if it is damaged (`health < max_health`), has non-empty `combat_mods`, or is killed (`health <= 0` until `we5k.2.4` adds kill state). Otherwise it is deleted.
- Monsters arrive without initiative; the GM rolls it.
- A push is allowed only while the encounter is `prepared` or `executing`.

### Still open

- How lets-roll (Rails on ECS) signs its pull and its kill reports to encounter-builder-api's Function URL. This needs an ECS task role with a grant like `SERVICE_RELEASE_ROLE_ARN`.

## Conflicts

**Confirmed (Devon, 2026-11-10):** the latest change wins, and EB is the record.
If lets-roll and EB disagree about a kill, the most recent change applies, and EB's stored state is what everything else reads.

## Related

- [content-ref.md](content-ref.md): live state is never a ContentRef and never released.
- [encounter-treasure-cluster.md](encounter-treasure-cluster.md): the EB and party-treasure cluster plan.
- [ROADMAP.md](../../ROADMAP.md): M1 (treasure loop) and M3 (run-time view).
