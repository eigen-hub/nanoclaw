Here is the honest code-quality assessment:

---

# NanoClaw — Code Quality Assessment

## Overall Score: 7/10

Solid, well-above-average personal project code. The security model is thoughtfully designed, the channel abstraction is real, and the test isolation is excellent. The weaknesses are concentrated in a few specific areas rather than being systemic.

---

## 1. Code Quality — **7.5/10**

**What's good:**
- Naming is consistently clear and accurate throughout — `processGroupMessages`, `drainGroup`, `runForGroup`, `routeOutbound`. Nothing cryptic.
- `router.ts`, `group-folder.ts`, `container-runtime.ts`, and `types.ts` are textbook small, focused modules — 3–8 line functions, single responsibilities.
- Error paths are mostly consistent: try/catch + pino logging, resolving (never rejecting) the container Promise.
- Non-obvious logic is commented. `container-runner.ts:434` explains "timeout after output = idle cleanup, not failure". `index.ts:164` explains the cursor-advance timing. The agent-runner has a thorough header explaining the full IPC protocol.

**What's bad:**
- **`runContainerAgent` (`container-runner.ts:284–622`) is a 380-line single Promise constructor.** Streaming output parsing, timeout management, log writing, success/error branching — all nested inside one callback. McCabe complexity ~15–20. This is the single worst function in the codebase.
- **`buildVolumeMounts` (`container-runner.ts:57–200`)** has 7 distinct responsibilities: main/non-main branching, session directory creation, `settings.json` writing, skills syncing, IPC directory setup, agent-runner source copying, and additional mount validation. It's a 143-line setup script masquerading as a helper function.
- **Trigger check is copy-pasted verbatim** between `processGroupMessages` (~L155) and `startMessageLoop` (~L380) in the same file. A 2-line `hasTriggerMessage(messages)` helper would eliminate both.
- **`writeTasksSnapshot` pattern is duplicated** between `index.ts:258–271` and `task-scheduler.ts:99–111` — both build the same task-transform inline.
- **`OUTPUT_START_MARKER` / `OUTPUT_END_MARKER`** are defined independently in both `container-runner.ts:30–31` and `container/agent-runner/src/index.ts:108–109`. These are a load-bearing binary protocol — if one side changes, the other silently breaks.
- **`mount-security.ts` instantiates its own independent `pino` logger** instead of importing the shared one from `logger.ts`. This creates a leaked worker thread and a divergent log configuration.
- DB query results are cast with bare `as NewMessage[]`, `as ScheduledTask[]` — no runtime schema validation, type safety is illusory at the DB boundary.
- `processTaskIpc`'s `data` parameter (`ipc.ts:156`) is a massive loosely-typed object with all possible IPC payload fields optional. A discriminated union would catch whole classes of bugs at compile time.

---

## 2. Modularity — **7/10**

The module boundaries are mostly sensible:

| Module | Concern | Verdict |
|--------|---------|---------|
| `router.ts` | Message formatting + channel dispatch | ✅ Clean |
| `group-queue.ts` | Per-group concurrency | ✅ Clean |
| `container-runtime.ts` | Runtime abstraction | ✅ Clean |
| `group-folder.ts` | Path validation | ✅ Clean |
| `db.ts` | Data layer | ✅ Acceptable (no DAO interface) |
| `task-scheduler.ts` | Scheduled task execution | ⚠️ Duplicates container-prep logic |
| `ipc.ts` | IPC file processing | ⚠️ Knows about the IPC filesystem layout AND the queue |
| `container-runner.ts` | Container lifecycle | ❌ Does everything |
| `whatsapp.ts` | WhatsApp transport | ❌ Directly coupled to `db.ts` |

**The worst coupling problem:** `whatsapp.ts` directly calls `getLastGroupSync()`, `setLastGroupSync()`, and `updateChatName()`. A channel implementation is reaching into the database layer. The channel should emit events (`onGroupMetadata`), and the orchestrator should persist them.

**Adding a new channel (e.g. Telegram):** Mostly possible without touching core code. The `Channel` interface in `types.ts` is clean and complete. However:
1. Channel instantiation in `index.ts:479` is imperative and hardcoded — you'd add lines there.
2. The `syncGroupMetadata` callback wired in `startIpcWatcher` is WhatsApp-specific and passed by direct reference — Telegram groups don't have the same sync model.
3. `ownsJid` returning true for `@g.us`/`@s.whatsapp.net` suffixes bleeds WhatsApp addressing into routing logic.

**Swapping container runtimes (Docker → Podman → Apple Container):** Low difficulty. `container-runtime.ts` is correctly designed for this. Change `CONTAINER_RUNTIME_BIN` and the three helper functions. The `--rm` flag and `docker ps --filter` syntax in `cleanupOrphans` are the only Docker-specific leaks.

---

## 3. Hackability / Extensibility — **6.5/10**

| Task | Difficulty | Blocker |
|------|-----------|---------|
| Add new channel | Medium | Channel registration in `index.ts` is hardcoded; `syncGroupMetadata` is WhatsApp-specific |
| Swap container runtime | Low | `container-runtime.ts` handles this cleanly |
| Add new scheduler type | **High** | Requires changes in 5 files: `ipc.ts`, `task-scheduler.ts`, `db.ts`, `ipc-mcp-stdio.ts`, `types.ts` |
| Add new IPC task type | Medium | Add a `case` in `ipc.ts` and a tool in `ipc-mcp-stdio.ts`; no type safety |
| Swap database backend | **Very High** | 20+ functions exported from `db.ts` with no DAO interface; every import site changes |
| Add per-group config | Good | `RegisteredGroup.containerConfig` is clean and already used |
| Understand before changing | Variable | Skills/CLAUDE.md = trivial; container behavior = steep (`container-runner.ts` is dense) |

**The scheduler fan-out problem** is the worst hackability issue. The `schedule_type` discriminated union is spread across 5 files with no strategy pattern. Adding a `'webhook'` trigger type is a 5-file change where each file independently switches on the same string.

**The skill system** is functional but rudimentary — markdown files in `container/skills/` are `cpSync`'d into each group's Claude sessions directory at container startup. There's no versioning, no per-group skill assignment, and skills are global to all groups. It works but doesn't scale as the skill count grows.

**Dead code in the critical path:** `container-runner.ts:560–606` is a "legacy non-streaming" output parsing path that is unreachable in practice — both callers (`index.ts` and `task-scheduler.ts`) always pass an `onOutput` callback, routing to the streaming path. This ~50 lines of dead code adds cognitive load for anyone trying to understand how container output works.

---

## 4. Test Quality — **7/10**

**What's well-tested:**
- `GroupQueue` — excellent state machine coverage: serialization, global concurrency limit, task/message priority, retry backoff, idle preemption (8 focused tests)
- `ipc-auth.test.ts` — comprehensive authorization matrix covering all IPC operations with all permission combinations
- `container-runner.test.ts` — timeout behavior across three scenarios (timeout before output = error; timeout after output = success; normal exit = success)
- `db.test.ts` — CRUD, upsert behavior, bot message filtering

**What's NOT tested:**
- `task-scheduler.ts` — `startSchedulerLoop` and `runTask` have **zero tests**
- `mount-security.ts` — **zero tests** on the security-critical path validation and blocked-pattern matching
- `whatsapp.ts` — **zero tests** (understandable but risky)
- The main orchestration in `index.ts` — cursor management, recovery, trigger detection are entirely untested
- `container/agent-runner/src/` — the entire agent-side code is untested

**Test isolation is excellent.** In-memory SQLite via `_initTestDatabase()`, fake `ChildProcess` with `EventEmitter` + `PassThrough` streams, `vi.useFakeTimers()` for timeout tests, `vi.mock('fs')` where needed. Tests don't pollute the filesystem or require external services.

**One design smell:** `ipc-auth.test.ts:389–396` defines a local `isMessageAuthorized` function that *replicates* the authorization logic from `ipc.ts` rather than calling through it. It's testing a copy, not the real code — if the authorization logic in `ipc.ts` changes, this test won't catch the regression.

---

## 5. Technical Debt — **7/10** *(lower = more debt)*

**Notable debt items:**

1. **`input.secrets` mutation pattern** (`container-runner.ts:297–301`): `input.secrets = readSecrets(); ... delete input.secrets;` — mutates a passed-in object, then deletes the field. Works, but fragile and surprising to read.

2. **Silent catch-all DB migrations** (`db.ts:88–128`): `try { ALTER TABLE } catch { /* column exists */ }` swallows all errors including disk-full and corruption. No migration version tracking. When a third migration is needed for the same table, ordering is entirely implicit.

3. **`_close` sentinel constant duplication**: Defined independently in `group-queue.ts:184` (writer) and `agent-runner/src/index.ts:59` (reader). Protocol constants defined in two places silently diverge.

4. **`// Re-export for backwards compatibility during refactor`** at `index.ts:46` — a refactor marker that was never cleaned up.

5. **DB migration hardcodes future channel JIDs** (`db.ts:113–126`): Backfill migration uses `@g.us`, `@s.whatsapp.net`, `dc:`, `tg:` patterns — archaeological data from a schema evolution that bakes WhatsApp/Discord/Telegram knowledge into migration code.

6. **Dead "legacy" output parsing path** in `container-runner.ts:560–606`: ~50 lines of non-streaming output handling that is unreachable in the current call graph.

7. **No explicit TODO/FIXME comments** anywhere — either the code is genuinely clean of known debt, or debt is untracked. The security review found 3 critical issues with no inline markers.

---

## Scorecard Summary

| Dimension | Score | Primary Bottleneck |
|-----------|-------|--------------------|
| Code Quality | 7.5/10 | `runContainerAgent` (380 lines, complexity ~18); duplicate trigger check; duplicate logger |
| Modularity | 7/10 | `buildVolumeMounts` (7 responsibilities); `WhatsAppChannel` coupled to `db.ts` |
| Hackability | 6.5/10 | Scheduler strategy in 5 files; no DB abstraction; dead legacy output path |
| Test Quality | 7/10 | `mount-security.ts` and `task-scheduler.ts` have zero coverage; one test tests a copy |
| Technical Debt | 7/10 | Mutated `input.secrets`; silent migration catch-all; dead code; protocol constant duplication |
| **Overall** | **7/10** | |

---

## Verdict

This is good code for what it is — a personal-scale WhatsApp agent bridge. The fundamental architecture is sound: the `Channel` interface is real, the container isolation model is coherent, `GroupQueue` is well-engineered, and the test isolation technique is excellent. A new developer could navigate the codebase in a day.

The main problems are concentrated in `container-runner.ts` (too large, too many responsibilities, dead code inside the critical path) and the scheduler fan-out (a change to scheduling types touches 5 files). Neither is catastrophic — both are refactorable. The codebase reads like something that grew organically and hasn't yet had the "second pass" cleanup that would push it from 7 to 9.
