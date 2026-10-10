# 562 — Missions as containers: missions without a crew, roles added and removed while they run

> Tracking issue: [#562](https://github.com/yicheng47/runner/issues/562). Priority: P1.
> Status: spec under review on `feat/562-missions-as-containers`. Code follows once the open decisions below are settled; the app phase is designed first.
> Design: `design/specs/562-mission-spawn.pen`, drawn on `main` before Phase 4, together with the Start mission dialog of [849](./849-start-mission-modal.md).
> Identity: a caller is the person at the app or a roster handle, never a location ([648](./archive/648-runner-cli.md) decision 7).
> Related: [704](./704-session-send.md), `runner session send`, the terminal layer beside this coordination layer. Direct chats stay off the bus. [#863](https://github.com/yicheng47/runner/issues/863), the crew-edit roster bug this design fixes for new missions.
> Rewritten 2026-10-10 against the `runnerd` code, after Jason set the direction: a crew is not a prerequisite for a mission, a mission can start from a single role, and roles can be added to or removed from a running mission when needed. Jason chose the same day to give each mission its own crew row (one `crews` table, kind `template` or `mission`, no UI rename) over a separate mission-owned slot model. The 2026-09-18 draft's lead verbs (`runner spawn`, `ps`, `stop`) and outside seats move to Follow-ups; they build on this.

## Motivation

A mission can only start from a crew today. `ops::mission::start` refuses a crew with no slots or no lead, every session is spawned from one of the crew's slots, and the roster cannot change until the mission ends. Two everyday needs fall through:

- **One agent handing work to another.** Jason's daily pattern (a Claude Code session plans, a Codex agent implements) needs a one-slot crew ("codex solo") just to start a mission. A crew of one is setup in front of a single role.
- **A mission that needs another pair of hands.** When the task splits three ways, or a reviewer is needed that the crew never had, the only move is to stop the mission, edit the crew and start over, losing every conversation. The reverse, letting an agent go once its part is done, is not possible either.

Crews stay. A repeatable shape, such as coder plus reviewer, needs a fixed template so runs can be compared. The crew becomes one way to fill a mission, not the only way in.

## Model

There are two kinds of crew, stored in the same `crews` table:

| Kind | What it is | Who edits it | Where it shows |
| --- | --- | --- | --- |
| **Template crew** (`kind = template`) | Every crew that exists today: a named, reusable set of slots with one lead. | The person, on the crew page. | The crew page, crew pickers, `runner crew list`. |
| **Mission crew** (`kind = mission`) | The roster of exactly one mission, created when the mission starts: a copy of a template crew, keeping its name, or built from roles the person picked, with no name. | Only the mission's add-role and remove-role operations. | Nowhere as a crew. The person sees it as the mission's roster in the mission rail. |

- **Mission**: a container with one event log, a cwd, a goal, and one mission crew whose slots are its roster, with exactly one lead. `missions.crew_id` points at that mission crew.
- **Start from a crew**: copy the template crew's name, prompt addendum and slots into a new mission crew. Editing or deleting the template afterwards never touches the mission.
- **Start from roles**: create a mission crew from one or more picked roles, one marked as lead. A single role is a one-agent mission.
- **Add a role**: add a slot to the mission crew of a running mission and start its session. An added role is always a worker.
- **Remove a role**: stop the slot's session and mark the slot removed. Its messages stay in the feed and its handle stays taken. The lead cannot be removed.

**Why one table.** A mission reaches its roster, log, events and prompts through `crew_id`: `slot::list(mission.crew_id)` is the roster at start, at re-mount and in the app; the event log lives at `crews/<crew_id>/missions/<mission_id>/`; every event carries `crew_id`; every session gets `RUNNER_CREW_ID`; and the lead's prompt reads the crew's name and addendum. Giving each mission a crew of its own keeps all of that working unchanged, where a separate roster model would rewire each of those paths. The cost is two kinds of crew in the code, kept apart by a `kind` filter in the queries that list crews.

The pair mirrors role and slot. A role is reusable and a slot places it in one crew; a template crew is reusable and a mission crew places a copy of it in one mission. Unlike a slot, which points at its role so a role edit reaches every slot, a mission crew is a copy, so a template edit never reaches a running mission. The mission crew's slots still point at their roles, so a role edit reaches running missions as it does today.

UI copy keeps "Crew" for template crews, the only kind a person picks or edits. In code and docs the two are a *template crew* and a *mission crew*. The vision's "Mission: one live activation of a crew" becomes "one live run of its own mission crew, seeded from a crew or from roles".

## How it works today

Most of the runtime machinery already works per session and per handle; a mission crew leaves most of the rest unchanged.

| Layer | Today | With a mission crew |
| --- | --- | --- |
| Database | `missions.crew_id` and `slots.crew_id` are `NOT NULL`; a mission uses its crew's slots directly, and `sessions.slot_id` points at one of them. | Unchanged shape. `crews` gains `kind`; `slots` gains `added_by` and `removed_at`. |
| Start | `ops::mission::start` requires a crew with a lead, writes `roster.json` from its slots and spawns one session per slot. | Copies the template, or the picked roles, into a new mission crew in the start transaction; everything after reads the mission crew as today. |
| Re-mount and app | `ensure_mission_router_mounted`, `validate_roster_handle`, `post_message` and the app's `mission_workspace/attach.rs` read `slot::list(mission.crew_id)` as it is now, so a template edit mid-mission changes the mission at the next re-mount (#863). | Unchanged code, now correct: only the mission's own operations edit a mission crew. |
| Event log, events, env | `crews/<crew_id>/missions/<mission_id>/events.ndjson`; `crew_id` on every envelope; `RUNNER_CREW_ID` on every session. | Unchanged; the id is the mission crew's. |
| Prompts | The lead prompt names the crew and splices its `system_prompt_addendum`, read from the crew row again at re-mount. | Unchanged; the mission crew holds its own copy, so a later template edit does not change a running lead's prompt. |
| Crew lists | Every crew is listed. | Lists, counts and member previews filter to `kind = template`. |
| Missions of a crew | `missions.crew_id = <crew>`. | `runner mission list --crew <name>`, the only caller that filters by crew, matches the mission crew's name, copied from the template at start, plus missions started before this change that still point at the template directly. |
| Session spawn | `register_mission_session` then `complete_mission_session_spawn`, once per slot; nudges queue until the PTY is live. | Ready for one new slot. |
| Router routing | `session_by_handle` is a map under the router's state lock; `register_sessions` adds entries at any time. | Ready. |
| Router roster | `LaunchInputs.roster` is fixed at mount; broadcast nudges walk it and the lead's Restart prompt lists it. | Needs add and remove. |
| Bus | `BusState.handles` is fixed at mount ("Adding handles after mount is not supported in MVP") and decides whose inbox each message lands in. | Needs add and remove. |
| CLI | Rereads `roster.json` on every call to validate `--to`. | Ready once the file is rewritten on each change. |
| Per-session lifecycle | #542's Stop, Resume and Restart work on one session. | Ready; Remove uses Stop. |

## Scope

### In scope

- **Schema.** `crews.kind` (`template` or `mission`, default `template`, a free string checked in code). `slots.added_by` (`NULL` when seeded at start, `human` for the person) and `slots.removed_at`. Existing rows are template crews and seeded slots; nothing is backfilled.
- **Start from a crew.** In the start transaction, `mission_start` creates a mission crew with the template's name and `system_prompt_addendum`, copies each slot with its role, handle, position, lead flag and runtime, model, effort and speed overrides, and points the mission at it. The spawn loop, prompts, router, bus and `roster.json` then read the mission crew exactly as they read a crew today.
- **Start from roles.** `mission_start` takes exactly one of `crew_id` or `roles: [{ role, handle?, lead, runtime?, model?, effort? }]` with exactly one lead. A roles start creates a mission crew with an empty name and no addendum; the mission header, summaries and the lead prompt leave an empty crew name out.
- **Keeping the kinds apart.** `repo::crew` listing, counting and member-preview queries filter to `kind = template`, which covers the crew page, the pickers, `crew_list_all` and `runner crew list`. The crew page's slot operations (`ops::slot::create`, `update`, `delete`, `set_lead`, `reorder`) refuse a mission crew. `runner mission list --crew` matches by name as described above.
- **Deletion.** Deleting a mission deletes its mission crew and that crew's slots and directory. Deleting a template crew never touches the mission crews copied from it, so those missions survive; missions from before this change that point at the template directly keep today's rule.
- **Missions started before this change.** They keep pointing at their template crew and are not migrated. #863's guard, refusing slot edits on a template crew while one of its missions is running, protects them.
- **Add a role.** `mission_add_role(mission_id, { role, handle?, runtime?, model?, effort?, task? })`. It refuses unless the mission is running, not archived and has a mission crew. It picks the handle with the crew page's rule unless one is given (`suggest_slot_handle` and `slot_handle_error` move from `runner-app`'s `surfaces/crews/logic.rs` into `runner-core` so the daemon and the modal share them), inserts the slot into the mission crew with `added_by = human`, adds the handle to the router's roster and session map and to the bus with an inbox that starts at its join event, rewrites `roster.json`, appends a `role_joined` signal and a broadcast `message` from `runner` that introduces the newcomer to the team and the team to the newcomer, adds `task` as a directed message, then spawns through the existing per-slot path with the worker first turn.
- **Remove a role.** `mission_remove_role(mission_id, handle)`. It refuses the lead and an unknown or already-removed handle. It stops the slot's session with #542's Stop, sets `removed_at`, removes the handle from the router's roster (broadcasts skip it, `--to` refuses it) and from the bus's projection, rewrites `roster.json`, and appends a `role_removed` signal and a `runner` broadcast that the agent left the mission. `slot::list` skips removed slots, so a re-mount rebuilds the reduced roster; the row stays so its handle is never reused in that mission.
- **Router and bus.** `LaunchInputs.roster` moves behind the router's state lock with add and remove; broadcast nudges and the lead's Restart prompt read the current roster. The bus gains add-handle and remove-handle control messages handled on its consumer thread, so a join or leave is ordered exactly against the log it projects.
- **Protocol and CLI.** The client protocol carries the start, add and remove operations. The `runner` CLI gains `runner mission start --role <role>` (repeatable, `--lead <role>` when more than one) beside `--crew`, `runner mission add-role <mission> <role> [--as <handle>] [--runtime …] [--model …] [--effort …] [--task …]` and `runner mission remove-role <mission> <handle>`. They act as the person, the identity of a caller without a handle, and are refused from inside a mission session until the lead's verbs exist (Follow-ups).
- **App.** The Start mission dialog offers a crew or a set of roles (with 849). The mission rail gains **+ Add role** under its sessions while the mission runs, opening a modal with the role picker from the crew page's add-slot form, runtime, model and effort, a handle pre-filled with the suggested unique handle, and an optional task. A worker card's menu gains **Remove from mission**, with a confirmation. The feed renders `role_joined` and `role_removed` rows.
- **Docs.** `AGENTS.md` and `docs/product/vision.md` core vocabulary as in Model; `docs/arch/arch.md` §8 (`role_joined`, `role_removed`, the roster changes while a mission runs).

### Decisions to confirm

Each has a recommendation; the spec assumes it until Jason decides otherwise.

1. **A removed role's handle stays taken in the mission.** Reusing it would merge two agents' history under one name in the feed and in inbox projection. Adding the same role again gets `coder-2`.
2. **A mission cannot start empty.** It needs a lead to receive the goal. "Create a mission, then assign roles" is: pick the lead role at start, then add the rest.
3. **The lead is fixed for the mission's life.** Removing or changing the lead is not in this delivery.
4. **No roster cap for the person.** A cap arrives with the lead's spawn verb, where an agent could loop.
5. **Add and remove are for the person only** until the lead's verbs land; a mission agent calling them is refused.
6. **Missions started before this change are not migrated** to mission crews; #863's guard covers them until they end.

### Out of scope

- **The lead's verbs** (Follow-ups): `runner spawn`, `runner ps` and `runner stop` from the lead, idle and crash notices to the lead (`slot_exited`), and outside seats, where an agent outside Runner holds the lead seat. A separate issue once this lands.
- Saving a mission's roster as a crew. With mission crews this later becomes turning a mission crew into a template.
- A direct chat joining a mission as a slot. A chat's `RUNNER_*` environment is fixed at spawn; it drives a mission from outside instead.
- Workers adding or removing roles.
- Adding to or removing from a completed, aborted or archived mission, or while the mission's cold-start spawn queue is still running.

## Implementation Phases

Phases 1 to 3 are daemon and CLI work and can land before any UI. Each phase is a commit that builds and passes its tests. Phase 1 changes nothing visible and fixes #863 for every new mission, so it can ship on its own.

### Phase 1 — mission crews

- Migration `0026_mission_crews.sql`: `crews.kind`, `slots.added_by`, `slots.removed_at`.
- `mission_start` from a crew creates and fills the mission crew; `repo::crew` listing queries filter by kind; slot operations refuse mission crews; `mission list --crew` matches by name; mission and crew deletion as above; #863's guard for missions that still point at a template.
- Tests: a started mission's crew is a mission crew with the template's name, addendum and slots, overrides included; the crew page, pickers, counts and `runner crew list` never show it; editing or deleting a template changes no running mission, also after a re-mount; listing a template's missions finds both new and pre-change missions; deleting a template keeps its new missions; deleting a mission removes its mission crew; a slot edit on a template with a running pre-change mission is refused.

### Phase 2 — add and remove while running

- `ops::mission::{mission_add_role, mission_remove_role}`, the shared handle rule in `runner-core`, the router's mutable roster, the bus's add-handle and remove-handle, the `roster.json` rewrite, and the `role_joined` and `role_removed` events.
- Client protocol operations and the `runner mission add-role` and `remove-role` commands.
- Tests: join events land in order with the documented payloads; the newcomer's inbox holds the broadcast and the task and nothing earlier; `--to` the newcomer validates right after the join and broadcasts reach it; `--to` a removed handle is refused and broadcasts skip it; removing the lead is refused; a re-mount after a daemon restart rebuilds both changes; two missions of one template can each add `@coder-2`.

### Phase 3 — start from roles

- `mission_start` with `roles`, the empty-name rule in the lead prompt and the mission header, and `runner mission start --role`.
- Tests: a one-role mission runs end to end and its lead's `runner msg post` works; a several-role start has exactly one lead and the picked roster; a start with zero or two leads is refused.

### Phase 4 — design, then app

- Draw the frames in `design/specs/562-mission-spawn.pen` on `main` and stop for sign-off: the Start mission dialog's crew-or-roles choice (with 849), **+ Add role** and its modal, **Remove from mission** and its confirmation, a card added mid-mission, and the two feed rows.
- `mission_workspace`: the rail row, the modal (reusing the crew page's add-slot pieces and the shared handle rule), the card menu item, and the feed rows; `app_store.rs` refreshes on `role_joined` and `role_removed`; the Start mission dialog.
- Tests: the modal validates handles against the mission roster, including removed ones; the rail shows a slot added after `session/spawned` and hides a removed one; feed row copy.

## Verification

- [ ] A mission started from a crew runs as before, on its own mission crew, and missions started before the change resume, restart and replay unchanged.
- [ ] Editing or deleting a template crew never changes a running mission, including after a daemon restart, and mission crews never appear on the crew page, in pickers or in `runner crew list`.
- [ ] A mission started from one role runs end to end: the lead gets the goal and posts with `runner msg post`.
- [ ] A mission started from several roles has exactly one lead and the roster that was picked.
- [ ] An added role boots with the join broadcast and its task, is reachable with `--to` at once, receives later broadcasts, and survives a daemon restart.
- [ ] A removed role's session stops, `--to` it is refused, broadcasts skip it, its history stays in the feed, its handle is not reused, and the removal survives a daemon restart.
- [ ] The lead cannot be removed, and add and remove are refused on a mission that is not running.
- [ ] Every join and leave on the feed is attributed to the person.
- [ ] Live checks: the affected `MIS-*` cases in [`docs/tests/regression/missions.md`](../tests/regression/missions.md) plus new cases for a roles start, an add and a remove, on macOS and Windows.
