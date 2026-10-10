# 562 — Missions as containers: missions without a crew, roles added and removed while they run

> Tracking issue: [#562](https://github.com/yicheng47/runner/issues/562). Priority: P1.
> Status: spec under review on `feat/562-missions-as-containers`. Code follows once the open decisions below are settled; the app phase is designed first.
> Design: `design/specs/562-mission-spawn.pen`, drawn on `main` before Phase 4, together with the Start mission dialog of [849](./849-start-mission-modal.md).
> Identity: a caller is the person at the app or a roster handle, never a location ([648](./archive/648-runner-cli.md) decision 7).
> Related: [704](./704-session-send.md), `runner session send`, the terminal layer beside this coordination layer. Direct chats stay off the bus.
> Rewritten 2026-10-10 against the `runnerd` code, after Jason set the direction: a crew is not a prerequisite for a mission, a mission can start from a single role, and roles can be added to or removed from a running mission when needed. The 2026-09-18 draft's lead verbs (`runner spawn`, `ps`, `stop`) and outside seats move to Follow-ups; they build on this.

## Motivation

A mission can only start from a crew today. `ops::mission::start` refuses a crew with no slots or no lead, every session is spawned from one of the crew's slots, and the roster cannot change until the mission ends. Two everyday needs fall through:

- **One agent handing work to another.** Jason's daily pattern (a Claude Code session plans, a Codex agent implements) needs a one-slot crew ("codex solo") just to start a mission. A crew of one is setup in front of a single role.
- **A mission that needs another pair of hands.** When the task splits three ways, or a reviewer is needed that the crew never had, the only move is to stop the mission, edit the crew and start over, losing every conversation. The reverse, letting an agent go once its part is done, is not possible either.

Crews stay. A repeatable shape, such as coder plus reviewer, needs a fixed template so runs can be compared. The crew becomes one way to fill a mission, not the only way in.

## How it works today

Most of the runtime machinery already works per session and per handle. What ties a mission to its crew is the data model, the log location, and two lists fixed when the mission mounts.

| Layer | Today | Ready for a changing roster? |
| --- | --- | --- |
| Database | `missions.crew_id` is `NOT NULL` with `ON DELETE CASCADE` (`0001_init.sql`). `slots.crew_id` is `NOT NULL`, so a mission has no slots of its own; `sessions.slot_id` points at a crew slot. | No: the mission has no roster of its own. |
| Start | `ops::mission::start` copies nothing. It checks the crew's slots, writes `roster.json` from them, and spawns one session per slot. | No: needs a crew with a lead. |
| Re-mount and app | After a daemon restart, `ensure_mission_router_mounted` rebuilds the router from the crew's *current* slots (`slot::list(crew_id)`), and so do `validate_roster_handle`, `post_message` and the app's `mission_workspace/attach.rs`. | No, and it is a latent bug (below). |
| Event log | `crews/<crew_id>/missions/<mission_id>/events.ndjson` (`event_log::mission_dir`). The mission's scratch directory with per-slot `runner` shims is already `missions/<mission_id>/`. | No: the path needs a crew. |
| Events and env | Every event envelope carries `crew_id`; every mission session gets `RUNNER_CREW_ID`. | No: both assume a crew. |
| Session spawn | `register_mission_session` then `complete_mission_session_spawn`, once per slot; inbox nudges queue until the new PTY is live. | Yes: works for one new slot. |
| Router routing | `session_by_handle` is a map under the router's state lock; `register_sessions` adds entries at any time, which Resume already uses. | Yes. |
| Router roster | `LaunchInputs.roster` is fixed at mount. Broadcast nudges walk it, and the lead's Restart prompt lists it. | No: fixed list. |
| Bus | `BusState.handles` is fixed at mount ("Adding handles after mount is not supported in MVP"); it decides whose inbox each message is projected into. | No: fixed list. |
| CLI | Rereads `roster.json` on every call to validate `--to`. | Yes, once the file is rewritten on change. |
| Per-session lifecycle | #542's Stop, Resume and Restart work on one session. | Yes. |
| Prompts | The lead's prompt names the crew and splices the crew's `system_prompt_addendum`, both read from the crew row again at re-mount. | No: needs a crew row that still exists. |

**Latent bug.** Because the running roster is read from the crew's current slots, editing a crew while one of its missions runs changes that mission's roster at the next re-mount, while `roster.json` and the live sessions keep the old one. Deleting a crew slot leaves its mission session pointing at a slot that no longer exists. Mission-owned slots fix this as a side effect.

## Model

- **Mission**: a container with one event log, a cwd, a goal, a roster of mission slots, and exactly one lead. The mission owns its roster.
- **Crew**: a template. Starting a mission from a crew copies its slots into mission slots. Editing or deleting the crew afterwards never touches the mission, which keeps the crew's id only as a record of where it came from.
- **Starting without a crew**: pick one or more roles and mark one as lead. A single role is a one-agent mission.
- **Add a role**: insert a mission slot into a running mission and start its session. An added role is always a worker.
- **Remove a role**: stop the slot's session and take it off the roster. Its messages stay in the feed. The lead cannot be removed.

Vocabulary: **crew slot** (a row the crew page edits) and **mission slot** (a row one mission owns). No new nouns. The vision's "Mission: one live activation of a crew" becomes "one live run of a roster, seeded from a crew or from roles".

## Scope

### In scope

- **Mission-owned slots.** `slots` gains a nullable `mission_id` and `crew_id` becomes nullable, with a check that exactly one is set. Crew slots keep `mission_id IS NULL` and are all the crew page lists. Mission slots keep the runtime, model, effort and speed overrides and gain `added_by` (`NULL` when seeded at start, otherwise `human`) and `removed_at`. Handles and positions are unique per crew among crew slots and per mission among mission slots, as partial unique indexes.
- **Missions without a crew.** `missions.crew_id` becomes nullable with `ON DELETE SET NULL`, so deleting a crew keeps its missions. The lead prompt's crew name and addendum are snapshotted onto the mission at start (`missions.roster_name`, `missions.prompt_addendum`): a crew-seeded mission takes the crew's name and addendum, a crewless one takes its title and none. Re-mount and Restart read the snapshot, never the crew row.
- **Migration and backfill.** One migration rebuilds `slots` and `missions`. For each mission it creates one mission slot per mission session, copied from the crew slot the session points at, or from the session's role and its `roster.json` handle when that crew slot is gone, and repoints `sessions.slot_id`. It fills the prompt snapshot from the crew row.
- **One log location.** Every mission keeps its log and `roster.json` in `missions/<mission_id>/`, next to its shims. `event_log::mission_dir` drops its crew argument. Existing directories under `crews/<crew_id>/missions/` are moved once, idempotently, at daemon start before any router mounts. A daemon update already restarts every session, so resumed sessions get shims with the new `RUNNER_EVENT_LOG`.
- **Events and env without a crew.** The envelope's `crew_id` becomes optional: omitted for crewless missions, and old logs parse unchanged. `RUNNER_CREW_ID` is set only for crew-seeded missions, and the CLI stops requiring it.
- **Start from a crew or from roles.** `mission_start` takes exactly one of `crew_id` or `roles: [{ role, handle?, lead, runtime?, model?, effort? }]` with exactly one lead, and copies either into mission slots in the start transaction. Everything after the copy (prompt composition, spawn loop, router and bus mount) reads mission slots.
- **Add a role.** `mission_add_role(mission_id, { role, handle?, runtime?, model?, effort?, task? })`. It refuses unless the mission is running and not archived. It picks the handle with the crew page's rule unless one is given (`suggest_slot_handle` and `slot_handle_error` move from `runner-app`'s `surfaces/crews/logic.rs` into `runner-core`, so the daemon and the modal share them), inserts the mission slot, adds the handle to the router's roster and session map and to the bus with an inbox that starts at its join event, rewrites `roster.json`, appends a `role_joined` signal and a broadcast `message` from `runner` that introduces the newcomer to the team and the team to the newcomer, adds `task` as a directed message, then spawns through the existing per-slot path with the worker first turn.
- **Remove a role.** `mission_remove_role(mission_id, handle)`. It refuses the lead and an unknown or already-removed handle. It stops the slot's session with #542's Stop, sets `removed_at`, removes the handle from the router's roster (broadcasts skip it, `--to` refuses it) and from the bus's projection, rewrites `roster.json`, and appends a `role_removed` signal and a `runner` broadcast that the agent left the mission.
- **Router and bus.** `LaunchInputs.roster` moves behind the router's state lock with add and remove; broadcast nudges and the lead's Restart prompt read the current roster. The bus gains add-handle and remove-handle control messages handled on its consumer thread, so a join or leave is ordered exactly against the log it projects.
- **Protocol and CLI.** The client protocol carries the start, add and remove operations. The `runner` CLI gains `runner mission start --role <role>` (repeatable, `--lead <role>` when more than one), `runner mission add-role <mission> <role> [--as <handle>] [--runtime …] [--model …] [--effort …] [--task …]` and `runner mission remove-role <mission> <handle>`. They act as the person, the identity of a caller without a handle, and are refused from inside a mission session until the lead's verbs exist (Follow-ups).
- **App.** The Start mission dialog offers a crew or a set of roles (with 849). The mission rail gains **+ Add role** under its sessions while the mission runs, opening a modal with the role picker from the crew page's add-slot form, runtime, model and effort, a handle pre-filled with the suggested unique handle, and an optional task. A worker card's menu gains **Remove from mission**, with a confirmation. The feed renders `role_joined` and `role_removed` rows. The mission header shows a crew only when one seeded the mission.
- **Docs.** `docs/arch/arch.md` §8 (the roster is mission-owned; `role_joined` and `role_removed`) and §10.2 (the log layout); `docs/product/vision.md` §3 definitions, as in Model.

### Decisions to confirm

Each has a recommendation; the spec assumes it until Jason decides otherwise.

1. **A removed role's handle stays reserved in the mission.** Reusing it would merge two agents' history under one name in the feed and in inbox projection. Adding the same role again gets `coder-2`.
2. **A mission cannot start empty.** It needs a lead to receive the goal. "Create a mission, then assign roles" is: pick the lead role at start, then add the rest.
3. **Deleting a crew keeps its missions.** Today it deletes the rows of its archived missions and refuses while any is unarchived; afterwards neither applies.
4. **The lead is fixed for the mission's life.** Removing or changing the lead is not in this delivery.
5. **No roster cap for the person.** A cap arrives with the lead's spawn verb, where an agent could loop.
6. **The add and remove commands are for the person only** until the lead's verbs land; a mission agent calling them is refused.

### Out of scope

- **The lead's verbs** (Follow-ups): `runner spawn`, `runner ps` and `runner stop` from the lead, idle and crash notices to the lead (`slot_exited`), and outside seats, where an agent outside Runner holds the lead seat. A separate issue once this lands.
- Saving a mission's roster as a crew.
- A direct chat joining a mission as a slot. A chat's `RUNNER_*` environment is fixed at spawn; it drives a mission from outside instead.
- Workers adding or removing roles.
- Adding to or removing from a completed, aborted or archived mission, or while the mission's cold-start spawn queue is still running.

## Implementation Phases

Phases 1 to 3 are daemon and CLI work and can land before any UI. Each phase is a commit that builds and passes its tests; Phase 1 can ship on its own, since it fixes the crew-edit bug with no visible change.

### Phase 1 — mission-owned roster and one log location

- Migration `0026_mission_slots.sql`: rebuild `slots` (nullable `crew_id`, `mission_id`, the check, `added_by`, `removed_at`, partial unique indexes) and `missions` (nullable `crew_id` with `ON DELETE SET NULL`, `roster_name`, `prompt_addendum`), with the backfill.
- `repo::slot`: `list_for_crew` filters `mission_id IS NULL`; add `list_for_mission` and `insert_for_mission`. `ops::mission::start` copies crew slots. Re-mount, `validate_roster_handle`, `post_message`, Resume, Restart and the app's attach read mission slots and the prompt snapshot.
- `event_log::mission_dir(app_data, mission_id)`, the one-time move, and the shim path; `ops::crew::delete` keeps missions.
- Tests: the backfill gives every session a mission slot with the same handle, role and overrides, including a session whose crew slot was deleted; the crew page never lists a mission slot; editing or deleting a crew slot changes no running mission, also after a re-mount; a moved log replays; deleting a crew keeps its missions.

### Phase 2 — add and remove while running

- `ops::mission::{mission_add_role, mission_remove_role}`, the router's mutable roster, the bus's add-handle and remove-handle, the `roster.json` rewrite, and the `role_joined` and `role_removed` events.
- Client protocol operations and the `runner mission add-role` and `remove-role` commands.
- Tests: join events land in order with the documented payloads; the newcomer's inbox holds the broadcast and the task and nothing earlier; `--to` the newcomer validates right after the join and broadcasts reach it; `--to` a removed handle is refused and broadcasts skip it; removing the lead is refused; a re-mount after a daemon restart rebuilds both changes; two missions of one crew can each add `@coder-2`.

### Phase 3 — start without a crew

- `mission_start` with `roles`, the optional envelope `crew_id` and `RUNNER_CREW_ID`, and `runner mission start --role`.
- Tests: a one-role mission runs end to end with its log under `missions/<id>/`; its lead's `runner msg post` works without `RUNNER_CREW_ID`; a log with `crew_id` on every line still replays; deleting a crewless mission removes its directory.

### Phase 4 — design, then app

- Draw the frames in `design/specs/562-mission-spawn.pen` on `main` and stop for sign-off: the Start mission dialog's crew-or-roles choice (with 849), **+ Add role** and its modal, **Remove from mission** and its confirmation, a card added mid-mission, and the two feed rows.
- `mission_workspace`: the rail row, the modal (reusing the crew page's add-slot pieces and the shared handle rule), the card menu item, and the feed rows; `app_store.rs` refreshes on `role_joined` and `role_removed`; the Start mission dialog.
- Tests: the modal validates handles against the mission roster, including removed ones; the rail shows a slot added after `session/spawned` and hides a removed one; feed row copy.

## Verification

- [ ] Pre-migration missions resume, restart and replay unchanged, with their logs moved to `missions/<id>/`.
- [ ] Editing or deleting a crew never changes a running mission, including after a daemon restart, and deleting a crew keeps its missions.
- [ ] A mission started from one role runs end to end: the lead gets the goal, posts with `runner msg post`, and no `RUNNER_CREW_ID` is set.
- [ ] A mission started from several roles without a crew has exactly one lead and the roster that was picked.
- [ ] An added role boots with the join broadcast and its task, is reachable with `--to` at once, receives later broadcasts, and survives a daemon restart.
- [ ] A removed role's session stops, `--to` it is refused, broadcasts skip it, its history stays in the feed, its handle is not reused, and the removal survives a daemon restart.
- [ ] The lead cannot be removed, and add and remove are refused on a mission that is not running.
- [ ] Every join and leave on the feed is attributed to the person.
- [ ] Live checks: the affected `MIS-*` cases in [`docs/tests/regression/missions.md`](../tests/regression/missions.md) plus new cases for a crewless start, an add and a remove, on macOS and Windows.
