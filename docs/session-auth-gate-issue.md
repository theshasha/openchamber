# Issue draft — 上游 openchamber/openchamber

状态：**已提交为 #3832（2026-09-22）**
https://github.com/openchamber/openchamber/issues/3832

实际提交正文 = 本文件从 "### What happened instead" 起的内容
（头部元信息未包含在提交正文里）。以下保留原稿备查。

---

## Title

Desktop: password unlock screen on a passwordless remote instance after a
first-connect race over relay

## Body

### What happened instead

The Windows desktop client connected to a remote instance over the private
relay. The instance has no UI password set. On the first connect of the day,
the app showed the "this session is password protected" unlock screen.

No password can ever succeed there. With no password configured, the server
answers `POST /auth/session` with 400 "UI password not configured", so the
user is stuck on a dead-end screen. Switching the runtime to local and back
made it disappear and the app went straight in.

### Steps to reproduce

1. Desktop app on Windows, remote instance over the private relay, no UI
   password configured on the server.
2. First connect of the day, while the relay tunnel is still establishing.
   The first `GET /auth/session` fails at the network layer (the known race
   between this request and the tunnel's initial WebSocket handshake).
3. `checkStatus` in `packages/ui/src/components/auth/SessionAuthGate.tsx`
   lands in its catch branch. `resolveStatusCheckFailureState`
   (`sessionAuthGateState.ts`) returns `'locked'` whenever
   `shouldUseDesktopShellPasswordLogin()` is true, which is always the case
   for a desktop shell pointing at a non-local runtime. The gate renders the
   password screen and never retries.
4. Switching the runtime away and back triggers the endpoint-change
   subscription, which re-runs `checkStatus`. By then the tunnel is up, the
   request reaches the server, and the server answers
   200 `{authenticated: true, disabled: true}`.

Browser clients survive the same race through the bounded retry
(`TRANSIENT_RETRY_MAX_ATTEMPTS = 4`). The desktop remote path checks
`resolveStatusCheckFailureState` before that retry and returns early, so
the retry never runs for exactly the client that needs it.

### What you expected

A transport-layer failure of one fetch says nothing about whether the server
wants a password. The desktop remote path should use the same bounded
transient retry the browser path already has, and if the budget runs out,
an error screen with a retry button is the honest state. A real password
requirement always arrives as an explicit 401, and 401 is already handled
in the branch before the catch.

### Why a 401 was impossible on this server

All observed on the affected instance, each item independently:

- `run/openchamber-10085.json` has `hasUiPassword: false`.
- No `OPENCHAMBER_UI_PASSWORD` in the server process environment; no
  password field in `settings.json` or `preferences.json`.
- With no password and `requireClientAuth` false, the session status
  handler always answers 200.
- The client reaches the server through the relay, which arrives as
  127.0.0.1 and is classified as `local` scope, so the tunnel-lock 401 path
  does not apply.

The only way into the locked screen on this instance is the network-failure
catch branch.

### Where does it happen?

Desktop (Electron) client, remote runtime over private relay. The browser
path over the same relay retries and recovers.

Relevant code:

- `packages/ui/src/components/auth/SessionAuthGate.tsx` — `checkStatus`
  catch branch
- `packages/ui/src/components/auth/sessionAuthGateState.ts` —
  `resolveStatusCheckFailureState`
- `packages/ui/src/components/auth/SessionAuthGate.test.ts` — currently
  pins the `'locked'` mapping

### OpenChamber version

Observed on 1.23.x; the catch branch is unchanged on current main.
