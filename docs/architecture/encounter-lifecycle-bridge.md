# Encounter Lifecycle and the lets-roll Combat Bridge

Status: ACCEPTED (Devon, 2026-11-10). The conflict rule is a default that Devon has not yet confirmed.
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

The beads must still settle these details:
- How lets-roll (Rails on ECS) signs its pull and its kill reports to encounter-builder-api's Function URL. This needs an ECS task role with a grant like `SERVICE_RELEASE_ROLE_ARN`.
- How encounter-builder-api authenticates its push into lets-roll. lets-roll does not verify SigV4 today.
- How lets-roll links each of its `encounter_monsters` rows back to the EB encounter and monster, so kills can be matched.

## Conflicts

**Default (pending Devon's confirmation):** the latest change wins, and EB is the record.
If lets-roll and EB disagree about a kill, the most recent change applies, and EB's stored state is what everything else reads.

## Related

- [content-ref.md](content-ref.md): live state is never a ContentRef and never released.
- [encounter-treasure-cluster.md](encounter-treasure-cluster.md): the EB and party-treasure cluster plan.
- [ROADMAP.md](../../ROADMAP.md): M1 (treasure loop) and M3 (run-time view).
