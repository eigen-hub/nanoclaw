Here is the complete production readiness report.

---

# NanoClaw — Production Readiness Review

**Verdict: NOT production-ready in its current state.** The codebase has 3 critical security vulnerabilities, 20+ high-severity issues, and numerous medium/low issues that need resolution before deployment in any multi-tenant or internet-facing context.

---

## 🔴 CRITICAL — Must Fix Before Any Deployment

### C-1: `sourceGroup` identity derived from untrusted filesystem directory names (`src/ipc.ts` ~L48–81)
**Category: Privilege Escalation**

The IPC system determines which group is "speaking" by reading directory names under `data/ipc/`. This `sourceGroup` string is used as the authorization identity — including the `isMain` check (`sourceGroup === MAIN_GROUP_FOLDER`). Any container (or any process with write access to `data/ipc/`) that creates a directory named `main` can impersonate the main group and gain full `register_group`, `refresh_groups`, and all-task-management privileges.

**Fix:** Call `isValidGroupFolder(sourceGroup)` on every directory name read from the filesystem, and additionally verify the folder corresponds to a registered group in SQLite before processing any IPC from it.

---

### C-2: IPC files read into memory without size limits (`src/ipc.ts` ~L74)
**Category: OOM / Process Crash (DoS)**

```ts
const data = JSON.parse(fs.readFileSync(filePath, 'utf-8'));
```
No size check before the read. A container that writes a multi-GB JSON file to its IPC directory will load it entirely into the host Node.js heap, crashing the main process.

**Fix:** `fs.statSync(filePath).size` check before reading; reject files above ~1 MB.

---

### C-3: Unsanitised `TZ` environment variable injected into Docker args (`src/container-runner.ts` ~L217)
**Category: Command/Argument Injection**

```ts
args.push('-e', `TZ=${TIMEZONE}`);
```
`TIMEZONE` is user-controllable via the `TZ` env var and is never validated as a legal IANA timezone string. Values containing `=`, newlines, or null bytes can manipulate the Docker argument vector.

**Fix:** Validate `TIMEZONE` against a known IANA timezone list or strip invalid characters before use.

---

## 🟠 HIGH — Fix Before Production

### H-1: `RESERVED_FOLDERS` is missing `main`, `errors`, `input`, `messages`, `tasks` (`src/group-folder.ts` ~L6)
A non-main group can register with `folder: 'main'` and gain main-group privileges, since `isMain = folder === MAIN_GROUP_FOLDER`. The reserved set only blocks `'global'`.

### H-2: `JSON.parse(container_config)` is unguarded in `db.ts` (~L550, L599)
A malformed row (DB corruption, manual edit) throws an uncaught `SyntaxError` in `loadState()` at startup or in the IPC poll loop, crashing the entire process. Wrap in try-catch, log a warning, return `undefined`.

### H-3: `registerGroup` leaves in-memory state inconsistent on DB/filesystem error (`src/index.ts` ~L96)
`registeredGroups[jid] = group` runs before `setRegisteredGroup()` and `mkdirSync()`. If either throws, in-memory and DB state diverge. `mkdirSync` is unguarded and will propagate to the IPC caller.

### H-4: Shutdown signal handler is fire-and-forget (`src/index.ts` ~L455–462)
```ts
process.on('SIGTERM', () => shutdown('SIGTERM')); // async fn, never awaited
```
If `ch.disconnect()` throws, `process.exit(0)` is never called. Systemd/launchd timeout-kills the process instead of clean exit. Also no guard against double-SIGTERM. The scheduler loop has no shutdown hook at all — it can write mid-task state on SIGTERM.

### H-5: `@whiskeysockets/baileys` pinned to a pre-release RC (`package.json` L22)
`"^7.0.0-rc.9"` will auto-upgrade to any new RC on `npm install`. Pre-release versions can have breaking changes with no semver guarantees. Pin to an exact version or a stable release.

### H-6: WhatsApp reconnect has no exponential backoff (`src/channels/whatsapp.ts` ~L112–119)
On disconnect, the channel reconnects immediately in a tight loop with no backoff, no max-retries cap, and no jitter. A prolonged server-side issue will result in rapid hammering of WhatsApp servers, risking account ban.

### H-7: Outgoing WhatsApp message queue is unbounded and not persisted (`src/channels/whatsapp.ts` ~L42)
Messages queued during disconnect are held in a plain in-memory array with no cap. A long disconnect accumulates all pending messages in RAM with no persistence; they are lost on process restart.

### H-8: Dockerfile base image and global npm installs are unpinned (`container/Dockerfile`)
- `FROM node:22-slim` — should be `FROM node:22-slim@sha256:<digest>` for reproducible builds.
- `RUN npm install -g agent-browser @anthropic-ai/claude-code` — no version pins; breaks silently when upstream releases incompatible versions.

### H-9: No container resource limits (`src/container-runtime.ts` / `container-runner.ts`)
`docker run` is invoked with no `--memory`, `--cpus`, `--pids-limit`, or `--ulimit` flags. Five concurrent Chromium-powered containers can exhaust all host RAM and OOM-kill the main process.

### H-10: Agent-runner source at `/app/src` is mounted read-write into containers (`src/container-runner.ts` ~L182)
A prompt injection attack or compromised agent can modify TypeScript source on the host filesystem, which persists across container restarts for that group. This is an intentional extensibility feature but constitutes a persistent code-execution vector.

### H-11: `pino-pretty` used unconditionally in production logger (`src/logger.ts` / `src/mount-security.ts`)
`pino-pretty` is in `dependencies` (not `devDependencies`) and is instantiated at module load time with no guard. It is also independently instantiated in `mount-security.ts` (creating a leaked worker thread). In high-throughput scenarios, `pino-pretty` has significant overhead vs. raw JSON logging.

### H-12: IPC error recovery has no inner error handling (`src/ipc.ts` ~L99–106)
The `errors/` directory creation and file rename in the `catch` block are both unguarded. If `mkdirSync` fails (permissions, full disk), the exception escapes, and the failed IPC file is neither consumed nor moved — causing infinite reprocessing and log spam on every 1-second poll.

### H-13: Credentials may appear in error log files (`src/container-runner.ts` ~L297–300, L487)
On error exits in verbose/debug mode, `JSON.stringify(input, ...)` is written to a log file. Although `delete input.secrets` runs before the `close` event fires, the pattern is fragile. If the stdin write throws before `delete` executes (stream closed, write error), the secrets remain on the object and are written to disk in cleartext.

**Fix:** Never attach secrets to the shared `input` object. Construct the secrets-containing payload inline: `JSON.stringify({ ...input, secrets: readSecrets() })`.

### H-14: MCP server has no startup validation for required env vars (`container/agent-runner/src/ipc-mcp-stdio.ts` ~L19–21)
```ts
const chatJid = process.env.NANOCLAW_CHAT_JID!;  // ! assertion, no check
```
If the container is started without these vars, all tool calls silently write IPC files with `undefined` values.

### H-15: `.env.example` is completely empty
Not a single environment variable is documented in the example file. `CLAUDE_CODE_OAUTH_TOKEN`, `ANTHROPIC_API_KEY`, and all tunables are undocumented for operators. This is a first-day operational failure.

---

## 🟡 MEDIUM — Fix Before Long-Running Production Use

### M-1: `matchesBlockedPattern` is case-sensitive on potentially case-insensitive filesystems (`src/mount-security.ts` ~L155–170)
Path component comparison is case-sensitive. On macOS (default HFS+ case-insensitive), `.SSH` would not match the blocked pattern `.ssh`, creating a bypass vector.

### M-2: Interval schedule has no minimum or maximum bounds (`src/ipc.ts` ~L228–229)
`interval: 1` creates a task firing every millisecond. `interval: 9007199254740991` creates a task with a `next_run` ~285,000 years in the future. Minimum: 60,000 ms. Maximum: 1 year in ms.

### M-3: `once` schedule accepts past timestamps (`src/ipc.ts` ~L237–246)
Tasks scheduled in the past fire immediately on the next scheduler poll. Should reject `scheduled.getTime() <= Date.now()`.

### M-4: `containerName` uses `Date.now()` with no collision protection (`src/container-runner.ts` ~L254`)
Two containers started within the same millisecond (during retry) collide names. Use `crypto.randomBytes(4).toString('hex')` suffix.

### M-5: `register_group` IPC does not validate `name`, `trigger`, or `containerConfig.timeout` (`src/ipc.ts` ~L360–381)
Arbitrary-length strings stored to DB. `containerConfig.timeout: 0` bypasses `CONTAINER_TIMEOUT` entirely.

### M-6: `ALTER TABLE` migrations swallow all errors, not just "duplicate column" (`src/db.ts` ~L88–128)
Disk-full or WAL errors silently succeed, leaving the schema broken. Only swallow `SQLITE_ERROR: duplicate column name`.

### M-7: WhatsApp reconnect `setInterval` for group sync leaks on reconnect (`src/channels/whatsapp.ts` ~L155–160)
Each reconnect adds a new 24-hour interval timer without clearing the old one, leaking timers indefinitely in a flapping connection.

### M-8: `macOS('Chrome')` browser identity hardcoded; `osascript` notification not gated on platform (`src/channels/whatsapp.ts` ~L79, L92)
On Linux, `osascript` silently fails (or errors). The browser fingerprint is hardcoded to macOS Chrome, potentially triggering WhatsApp's anomaly detection on Linux deployments.

### M-9: Queue `shutdown()` resolves instantly without actually waiting for containers (`src/group-queue.ts` ~L339)
`shutdown(timeoutMs)` sets a flag, waits for `timeoutMs` (but internally just does nothing), and resolves. Running containers are not awaited. Calling code in `index.ts` thinks the queue is flushed when it is not.

### M-10: Retry backoff has no jitter (`src/group-queue.ts` ~L14–15)
All groups at the same retry count will retry at exactly the same time, causing thundering-herd behavior when multiple groups fail simultaneously. Add ±20% random jitter.

### M-11: Agent-runner IPC message queue is unbounded (`container/agent-runner/src/index.ts` ~L86–95)
The container-side message queue has no maximum depth. A flood of IPC messages while the agent is running accumulates without limit.

### M-12: Scheduler interval task `next_run` drifts with processing time (`src/task-scheduler.ts` ~L196–203)
`next_run = Date.now() + intervalMs` sets the next run relative to *now* (after processing completes) rather than relative to the *scheduled run time*. Long-running tasks cause accumulating drift. Fix: `next_run = previous_next_run + intervalMs`.

### M-13: No health check anywhere in the stack
No HTTP endpoint, no Docker `HEALTHCHECK` instruction, no systemd `WatchdogSec`. External monitoring tools cannot determine if the service is healthy without WhatsApp-specific hacking.

---

## 🔵 LOW — Technical Debt / Operational Improvements

| ID | Location | Issue |
|----|----------|-------|
| L-1 | `src/container-runner.ts` L421 | Log files accumulate indefinitely with no rotation policy |
| L-2 | `src/index.ts` L337 | Fixed poll interval; add ±200ms jitter to reduce SQLite lock contention |
| L-3 | `group-queue.ts` L271 | Retry timers not cancelled on shutdown — fire after process exits |
| L-4 | `launchd/com.nanoclaw.plist` | No `ThrottleInterval` for crash-loop protection; `ASSISTANT_NAME` hardcoded to "Andy" |
| L-5 | `container/Dockerfile` | apt packages unversioned — `apt-get install chromium` can change behavior on image rebuild |
| L-6 | `src/logger.ts` L9–12 | Pino async transport not flushed before `process.exit(1)` in `uncaughtException` handler |
| L-7 | `src/task-scheduler.ts` L124 | `TASK_CLOSE_DELAY_MS = 10_000` hardcoded, not configurable |
| L-8 | `groups/main/CLAUDE.md` | Documentation references legacy `data/registered_groups.json` (system now uses SQLite) |
| L-9 | `container/agent-runner/src/ipc-mcp-stdio.ts` L26 | `Date.now() + Math.random()` filename — use `crypto.randomUUID()` |
| L-10 | `src/env.ts` L32–37 | Quote stripping silently passes malformed values (e.g. `"token` without closing quote) |

---

## Summary by Severity

| Severity | Count |
|----------|-------|
| 🔴 Critical | 3 |
| 🟠 High | 15 |
| 🟡 Medium | 13 |
| 🔵 Low | 10 |
| **Total** | **41** |

---

## Top 5 Immediate Priorities

1. **Fix `sourceGroup` privilege escalation** (C-1) — any container can impersonate the main group
2. **Add IPC file size limit** (C-2) — crash vector via OOM
3. **Sanitize `TZ` before Docker arg injection** (C-3)
4. **Add `'main'` and `'errors'` to `RESERVED_FOLDERS`** (H-1) — enables main-group privilege escalation via group registration
5. **Fix shutdown handler to await and guard against double-signal** (H-4) — SIGTERM leaves process hanging and data inconsistent
