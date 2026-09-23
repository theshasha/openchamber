# PR 草稿 — 上游 openchamber/openchamber

状态（2026-09-23）：开发已迁移 Win11（分支布局见
SESSION-AUTH-GATE-RELAY-RACE.md 的"开发环境迁移"）。代码已实施，
Before Submitting 四件套已跑完（test 有 31 个存量失败，已核实与
本次改动无关，证据见 SESSION-AUTH-GATE-RELAY-RACE.md 的"验证记
录"）。live run 与截图经维护者确认由 Win11 真机测试补上（见
"明天从这里继续"），PR 按声明版写明缺口和替代证据，测试后更新。

标题建议：`fix(ui): stop guessing a password requirement from a
failed session check`

以下为 PR 正文。

---

## What and why

A Windows desktop client connected to a remote instance over the private
relay showed the "this session is password protected" unlock screen on
the first connect of the day — on an instance with no UI password. No
password can succeed there: the server answers the unlock POST with 400
"UI password not configured". Switching the runtime to local and back
made the screen disappear and the app opened straight in.

What was actually happening: over the relay, HTTP rides the encrypted
tunnel. The first session status check was parked inside the tunnel
client while the tunnel's first WebSocket attempt was still
handshaking. That first attempt failed (a normal first-connect
transient) and the tunnel rejects every parked request on a failed
attempt, so the check died before ever reaching the wire. The tunnel
reconnects a second later by design and expects callers to retry. But
the gate's failure branch had already decided what to show: on desktop
remote it mapped any network failure straight to the password screen,
before the bounded transient retry could run. On a passwordless server
that screen is a dead end.

A transport failure is not evidence that the server wants a password.
The only legitimate evidence is an explicit 401, and 401 is handled in
the branch above the failure path. After this change the failure branch
does what every other runtime already did: the same bounded retry the
5xx path uses, then an honest network error screen with a retry button
and the desktop host switcher. The first-connect race now self-heals
into the app; the password prompt appears only when the server actually
answers 401, and the desktop main-process password login flow behind it
is untouched.

Two tests pinned the old mapping and were updated with it. What they
protected is the desktop password-login flow; that flow triggers on
explicit 401s and is unchanged. What the tests actually froze was
guessing "locked" from a network failure, which is the bug.

One trade-off, accepted knowingly: in theory the renderer can fail to
reach a server the main process can still reach, and on a
password-protected server the old path let a user type the password and
log in through the main process anyway. Nothing ever detected those
conditions, and on a passwordless server the same path dead-ends the
same way (the main process gets the same 400), so the guess is not
kept. Mid-session reauth checks that fail at the network layer now
retry and land on the error screen too, which matches what non-desktop
runtimes already did.

Fixes #3832

## Affected surfaces

| Runtime | Behavior after this change |
|---|---|
| Web | No change; the non-desktop failure path already retried then showed the error screen |
| Desktop (Electron) | The fix itself: a network-failed session check now retries then shows the network error screen with host switcher, instead of a password prompt that can never succeed on a passwordless server |
| VS Code | Not applicable; auth is skipped entirely in this runtime |
| Hosted mobile | No change; non-desktop failure path |
| Capacitor mobile | No change; the removed desktop-shell branch requires the Electron shell, which is always false here |

No persisted data, external contracts, or package exports change.

## Validation

| Check | Result |
|---|---|
| `bun test packages/ui/src/components/auth/SessionAuthGate.test.ts packages/ui/src/components/auth/SessionAuthGate.behavior.test.tsx` | 5 tests pass (updated desktop assertion: error screen + host switcher, no password prompt) |
| `bun run --cwd packages/ui test` (full UI suite) | 552/552 test files pass |
| `bun run type-check` (root, all workspaces) | Pass |
| `bun run lint` (root, all workspaces) | Pass |
| `bun run test` (full repo) | 3302 pass / 31 fail — every failure is in `packages/web/server/lib` (cron computation, PR status, skills, project-config); verified unrelated: none of the failing files import `@openchamber/ui` or the auth components, the UI suite is fully green, and the cron tests pin 2026-09-18 dates with UTC/Kyiv expectations on an Asia/Shanghai machine (fail when run standalone too) |
| `bun run build` | Pass (~100s; chunk-size notes are advisory warnings) |
| `bunx oxlint` on the three changed source/test files | No findings on changed lines; pre-existing findings elsewhere in the file left as-is per repo backlog policy |
| `bun run dead-code` | No entries for the changed files |

**Live run:** not performed, honestly. The changed branch requires the
Electron desktop shell connecting to a remote runtime; the development
machine is a headless Linux server (the remote instance itself), and
the maintainer's Windows desktop runs the released build, so neither
can exercise the fixed code path as-is. What covers the branch instead:
the behavior test renders the full gate with a mocked desktop shell and
a failing status check and asserts the rendered tree shows the network
error screen with the host switcher and no password prompt. A real
first-connect check will happen naturally once the fix ships in a
release and the desktop connects to this instance again.

## Visual evidence

No screenshots from this environment, with the reason: the changed
branch only renders inside the Electron desktop shell, which cannot run
on the headless Linux development machine. The before state is the
reported bug itself (the impossible password screen on the passwordless
instance, described in #3832 and seen daily by the maintainer before
switching to manual host selection). The after state is pinned by
`SessionAuthGate.behavior.test.tsx`, which renders the complete gate
with `isDesktopShell` mocked true and a network-failed status check,
and asserts the error screen, host switcher, and absence of the
password prompt. A real screenshot can be attached after the fix ships
and the maintainer reconnects from the Windows desktop.

## Risks

Small and contained: only the network-failure branch of the initial
session check changed; the 200/401/429 branches and the desktop
password login flow are untouched, and no other runtime took the
removed branch. Rollback is the plain revert with no data or persisted
state involved. The real 401 path (server actually requires a password)
was exercised by the existing behavior test that drives the locked
screen through a 401 and asserts the host-switch discard of an in-flight
login.
