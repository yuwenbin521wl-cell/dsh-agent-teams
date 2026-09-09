# Changelog

All notable changes to this fork (based on `@nanmicoder/dsh-agent-teams` v0.1.15, ported for DeepSeek Harness 0.1.2-alpha.4).

## Unreleased / current fork

### Added
- Ported to DSH **0.1.2-alpha.4**: removed `registerContinuableSetup`, inject member persona/toolFilter/agentOptions/`ReasoningEffortId` via `ctx.subagents.startContinuable`; wake via `ctx.subagents.sendMessage`.
- Windows-safe `clean-build.mjs` (`basename` instead of `endsWith('/lib')`).
- **Tools (20+):** `agent_teams_list`, `adopt`, `rehome`, `help`, `set_contract`, `set_decision`, `clear_decision`, `cleanup`, + readiness/state tools.
- **Read-only status** via `agent_teams_status team_id=<id>` (no auto-adopt).
- **Auto-governance config:** `maxMembers=10`, `failureThreshold=2`, `maxAutoReplace=2`, `autoReplaceThreshold=0.7`, `autoReplaceEnabled`.
- **Replacement logic with cap:** context-full (`readMemberContextPressure`) OR failed-many-times (`failures`/`failureThreshold`) → auto-spawn fresh member + transfer open tasks (1-for-1). `replacementDepth` counts generations; **auto-replace stops at `maxAutoReplace`** and defers to captain/user. **Explicit captain/user replacement is not capped.**
- **Zombie cleanup:** `cancelZombieTasks` cancels non-terminal tasks whose dep chain contains failed/cancelled; run on `adopt`/`rehome` (plus cascade-cancel of cancelled deps on every kick).
- **Archive history:** terminal tasks pruned to `archive-tasks.json` (outputs preserved); team.json only keeps active tasks + needed deps. `handoff.md` generated for takeover with state + archive pointer.
- **Orphan-subagent cleanup:** `agent_teams_cleanup` drains the captain's non-active continuable children and prunes removed-member records; `adopt` auto-cleans orphans.
- **Context management protocol:** no compaction; prefer fresh member / `agent_teams_adopt` handoff; write key conclusions/decisions into task `output` before handoff; persist pending decisions (`set_decision`/team.pendingDecision).
- **Team contract auto-inject:** `agent_teams_set_contract` (mode=append default) injects hard rules into every member persona + task assignment.
- **Member no-spam rule:** persona forbids gratuitous captain messages (上线/待命/无分配/状态报告).
- **Progress visibility:** members refresh `in_progress` result with %/milestone; status shows `⏱Nmin` + `⚠️no-progress-30min`.
- **Parallel/verify protocol:** don't queue independent tasks on one busy member; captain must `status`-verify after create/assign.
- **UI:** task DAG "show-completed" toggle (default off); id/status consistency (package name = bundle id = `__ModuleLoader__` id).

### Fixed
- `registerContinuableSetup is not a function` boot crash.
- Client bundle load error (`without registering "dsh-agent-teams"`) — three-name consistency.
- "Reported completed but status stuck": auto-close result files on every `kickTeam`.
- `remove_member` requeues its open tasks (no hanging on retired member).
- Cascade-cancel dead dependents of cancelled tasks.
- Adopt rehomes/retires **every** old member (incl previous `-r2` replacements); unique `-rN` names.
- Captain can take over/cancel zombie tasks despite unfinished deps.
- Removed the 4s scheduler heartbeat (caused process saturation on large teams).
- Disabled memory-evolve error churn (updated to latest commit independently).

### Config (`cordis.patch.yml`)
```yaml
memberProvider: spawn
memberBookkeepingByCaptain: true
memberPresets: [standard]      # relaxed when bookkeeping=true
autoReplaceEnabled: true
autoReplaceThreshold: 0.7
failureThreshold: 2
maxAutoReplace: 2
maxMembers: 10
stateDir: .agent-teams
```
