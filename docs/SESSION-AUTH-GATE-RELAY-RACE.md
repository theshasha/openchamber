# SessionAuthGate 在无密码的远程实例上误弹密码界面

状态（2026-09-23 更新）：issue #3832 已认领（评论
https://github.com/openchamber/openchamber/issues/3832#issuecomment-5779315422）。
**开发已迁移到 Win11**，本机（无头 Linux）今后只扮演两个角色：
远程实例本体（relay 测试配合见"明天从这里继续"）和文档存档。
接手的会话先读"开发环境迁移"一节。

状态（2026-09-22）：issue 已提交：#3832
（https://github.com/openchamber/openchamber/issues/3832）。修复已在
本地实施并验证（见"验证记录"），PR 草稿在
docs/session-auth-gate-pr.md，等维护者过目后提交，正文写
`Fixes #3832`。根因基于 main 7be5f80bb 逐行核实，传输层一侧
（runtime-fetch.ts、tunnel-client.ts、main.mjs）也已读通，链条见
"根因"和"背景事实"。

## 开发环境迁移（2026-09-23）

维护者决定日常开发放在 Win11（本地克隆 theshasha/openchamber），
这台 Linux 机器只作为远程实例和文档存档。分支布局：

- `fix/session-auth-gate-relay-race`：**干净的 PR 分支**，单提交
  d01e591a7，只含四个代码文件。向上游提 PR 用它，别把中文文档
  提交进来。
- `worknotes`：当前分支，= 修复分支 + 全部中文工作文档。Win11 上
  `git fetch && git checkout worknotes` 即得到代码 + 文档的完整
  工作环境，日常在这里开发。
- 代码改动的流向：在 worknotes 上改完后，把代码文件的提交整理到
  fix 分支再推（cherry-pick 代码提交，或对 fix 分支单独提交代码
  文件），PR 分支始终保持只含代码。

测试场景需要 Linux 侧配合（停/启 relay、设/删 UI 密码）：维护者
的 OpenChamber 桌面端（正式版）仍连着这台机器的 port 10085，在这
边的工作区开会话即可操作，无需在 Win11 之间传文件。

## 现象

Win11 桌面端通过私有 relay 连远程实例，实例没设 UI 密码。当天首次
连接时应用显示"此会话受密码保护"的解锁界面。把 runtime 切到本地再
切回来，界面消失，直接进入了应用。

在那个界面输密码不可能成功。服务端没配密码时对 `POST /auth/session`
返回 400 "UI password not configured"，用户被卡在死胡同里，直到碰巧
切换 runtime 才解脱。

## 根因

`packages/ui/src/components/auth/SessionAuthGate.tsx` 里 `checkStatus`
的 catch 分支。首次 `GET /auth/session` 在网络层失败时，gate 调
`resolveStatusCheckFailureState`（sessionAuthGateState.ts）决定显示
什么。只要 `shouldUseDesktopShellPasswordLogin()` 为 true（桌面端连
远程时恒为 true），它就返回 'locked'，界面直接变成密码解锁页，停在
那里，不再重试。

网络失败本身是已知的瞬时现象，机制已从传输层代码读通。"竞争"不
是两个请求在网上抢道：relay 模式下 HTTP 请求根本不走网络，而是被
停进隧道客户端里等通道。链条是：

1. `runtimeFetch` 发现活跃 relay 隧道，把 `/auth/session` 交给
   `relay.fetch`（runtime-fetch.ts 的 relay 路由分支）。
2. 隧道还没建好，`waitForChannel`（tunnel-client.ts）把请求挂为
   waiter 排队，此刻是 pending，不失败。
3. 当天首次连接，隧道的第一次 WebSocket 尝试失败（host 未在 relay
   注册、网络冷启动等任何非致命原因）。`failAttemptLocal` 里一行
   `rejectWaiters(error)` 把队列里所有 waiter 全部拒绝，停在里面
   的 `/auth/session` 在这里收到异常，抛回 checkStatus 的 catch。
4. `scheduleReconnect` 一秒起步退避重连，第二次尝试通常成功。但
   被拒的请求不复活：rejectWaiters 已经清空队列，传输层不跨重连
   保留请求。这是有意设计，tunnel-client.ts 文件头写明"一次隧道
   重连会杀掉所有打开的流，靠应用层已有的重试机器恢复"。

所以请求的生死绑定在隧道的第一次尝试上，而隧道的健康度由最新一
次尝试定义。catch 收到错误时隧道已经在恢复路上，错误滞后于现实。
浏览器客户端靠 1.5 秒后的自动重试落在已恢复的隧道上，撑过这个阶
段；桌面端连远程却在轮到重试之前就落进了密码界面。

切换 runtime 能恢复，是因为 `activateRelayTunnel` 对相同描述符复用
已有隧道、`adoptRelayTunnel` 直接收编已打开的探测隧道
（runtime-tunnel.ts）。切回来时通道是活的，`waitForChannel` 立即返
回，请求正常到达，服务端没有密码，返回 200。

代码里其实备好了重试机器（`TRANSIENT_RETRY_MAX_ATTEMPTS = 4`，
1.5 秒起步逐次加长），catch 分支后半段就在用它。问题只是顺序：密码
界面的判断排在重试之前，桌面远程路径提前 return，永远走不到重试。

## 为什么这台服务端不可能真的返回 401

受影响实例上的证据，互相印证：

- run/openchamber-10085.json 里 `hasUiPassword: false`。
- 服务端进程环境变量没有 `OPENCHAMBER_UI_PASSWORD`，settings.json
  和 preferences.json 里也没有密码字段。
- 无密码且 `requireClientAuth` 为 false 时，`handleSessionStatus`
  永远返回 200。
- 客户端走私有 relay，流量以 127.0.0.1 到达服务端，被判为 local
  而非 tunnel scope，tunnel 锁定 401 的路径不适用。

结论：这台服务端上唯一能进密码界面的路径就是网络失败的 catch
分支，401 不可能出现。

## 背景事实（2026-09-22 复查）

- catch 分支代码没变过，行为与根因描述一致。
- `SessionAuthGate.test.ts` 里的测试 "keeps the desktop-shell
  password login fallback intact" 钉死了当前行为（桌面远程网络
  失败 → 'locked'）。
- `resolveStatusCheckFailureState` 自 2026-07-06 引入
  （a53e991eb，"narrow mobile auth fallback"）没改过。09-13 后动过
  SessionAuthGate.tsx 的三个提交（窗口活动项目互抢、设置保存回显、
  启动 splash 渐变）都与此无关。
- 上游用多组关键词搜过 issue 和 PR，无重复报告。近亲但不同的：
  #3778（有密码场景 Web SPA 会话过期 401 死锁，open）、#2325（WSL
  连接配置问题，closed）。#2046/#2027 是引入
  `resolveStatusCheckFailureState` 的那个 PR，修浏览器端时把桌面
  路径留在了原地。
- `resolveStatusCheckFailureState` 全仓库只有一个运行时调用点
  （SessionAuthGate.tsx 的 catch 分支）。
  `shouldUseDesktopShellPasswordLogin` 另有两处用途
  （handlePasswordUnlock 里的 shell 密码登录），与本次改动无关，
  保持不动。
- waiter 被 `rejectWaiters` 拒绝时拿到的是原始错误，不带
  `dispatchedFailure` 的"已发出、结果未知"标记（那个标记只用于已
  上通道的流）。也就是说 catch 里连"服务器可能处理过"的余地都没
  有，请求确实从未上网线。这加固了"catch 里不该猜密码"的论点。
- 主进程 shell 登录（main.mjs 的 `loginRemoteAndIssueClientToken`，
  约第 1566 行起）用主进程自己的一次 fetch 直发服务器，与渲染进程
  的网络路径相互独立。推演其在无密码服务器上的结局：用户在误弹的
  密码框输密码，主进程拿到 400 "UI password not configured"，
  401/429 分支都不匹配，最终照样落错误屏。即旧代码那条"主进程代发"
  通道在无密码服务器上是双重死胡同，它只在"渲染进程网络坏 + 主进
  程网络好 + 服务器真有密码"三者同时成立时才有用，而没有任何一行
  代码检测过这些条件，通不通全凭巧合。
- 会话中途过期路径（`authSessionState === 'reauthenticating'` 触发
  的 checkStatus）网络失败时行为跟着变：旧行为桌面远程落密码框，
  新行为重试后错误屏。与非桌面端今天的行为一致，同一权衡的自然延
 伸，PR 里一句话交代即可。

## 修复方案（2026-09-22 已定）

catch 分支删掉密码界面判断，塌缩成其他端已经在用的流程：先自动
重试，预算耗尽就显示带重试按钮的网络错误界面。不新增任何机制，
纯复用现有重试机器。改动清单：

1. SessionAuthGate.tsx：catch 分支删掉
   `resolveStatusCheckFailureState` 的判断和 'locked' 分支，只留
   `scheduleTransientRetry()`，预算耗尽就 `setState('error')`。
   第 20 行 import 里的 `resolveStatusCheckFailureState` 同步删掉。
   改完的 catch 与上面 5xx 分支完全同形。
2. sessionAuthGateState.ts：`resolveStatusCheckFailureState` 失去
   唯一调用点，删掉函数本体。
3. SessionAuthGate.test.ts：删掉它的两个测试；describe 名从
   `resolveStatusCheckFailureState` 改掉（只剩
   `runtimeIdentityMatches` 的两个测试），import 行同步删符号。
4. SessionAuthGate.behavior.test.tsx：计划最初漏了这个文件。它的
   测试 "keeps desktop-shell status-check rejection on the locked
   password prompt" 用 mock 渲染整个 gate，钉死了桌面远程网络失败
   → 密码框的旧行为（ui 全量套件跑出来的）。改成断言新契约：错误
   屏出现、密码框不出现、桌面主机切换器在。同文件另两个测试不动：
   非桌面错误屏测试与新行为一致；切主机丢弃登录的测试走 401 分支
   不受影响。

页面行为对应变化（原 bug 场景，relay 首连竞争，无密码服务器）：
重试期间停在启动 logo（pending 态），隧道秒级自愈，一般第一次或
第二次重试拿到 200，直接进应用，密码界面根本不出现。重试预算是
1.5/3/4.5/6 秒递增，共四次。网络真坏则显示错误屏：网络错误标题、
描述、重试按钮，桌面端下面挂着主机切换器。服务器真有密码时与以
前一模一样：请求到达，401 分支弹密码框，主进程登录照旧。

为什么可以删 locked 映射：密码框的含义是"服务器说要密码"。这个
信息只能由一次成功到达的 401 传达，而 401 在 catch 之前的分支已
处理，根本进不了 catch。进了 catch 就说明请求没到服务器，手里没有
任何来自服务器的信息，在这里显示密码框等于没问到就替服务器回答
"要密码"。构造不出这个映射正确的场景，而误报的代价已实证（无密码
实例上的死胡同）。

桌面密码登录流程本体不受影响：真的要密码时，服务器回 401，走 401
分支进密码界面，`desktop_remote_password_login` 主进程登录照旧。
删掉的只是"没问到就猜要密码"这一条路。

## PR 怎么论证（不能默默改测试）

主线一句话：网络失败不构成密码要求的证据，401 才是。论证素材直接
用上面"为什么可以删 locked 映射"那段，外加传输层复核的一个新论据：
waiter 被拒时拿到的是未上过网线的原始错误（见"背景事实"），连"服
务器可能处理过"的余地都没有。

现有两个测试钉死了旧行为，PR 必须正面回答它们保的是什么。一个是
SessionAuthGate.test.ts 的 "keeps the desktop-shell password login
fallback intact"，一个是 SessionAuthGate.behavior.test.tsx 的
"keeps desktop-shell status-check rejection on the locked password
prompt"。它们真正保的是桌面密码登录流程的出现，而那条流程由明确
401 触发，不由网络失败触发，修改后依然完整（401 分支、主进程登录
一行没动）。两个测试实际冻结的是"从网络失败猜要密码"这个映射本身，
那正是 bug。

脚注两句，带过即可，不展开：理论上存在"渲染进程发不出请求、但主
进程能发"的环境，旧代码下用户还能在密码框输密码、靠主进程代发登录
自救，新代码下只见网络错误屏。但无任何代码检测过那些条件，且在无
密码服务器上那条通道同样是死胡同（见"背景事实"里 400 的推演），
不构成保留理由。错误屏有重试按钮和主机切换器兜底。

## 验证计划（对照 CONTRIBUTING.md）

改完代码后按顺序跑：

```bash
bun test packages/ui/src/components/auth/SessionAuthGate.test.ts  # 聚焦测试
bun run type-check:ui
bunx oxlint packages/ui/src/components/auth/SessionAuthGate.tsx \
  packages/ui/src/components/auth/sessionAuthGateState.ts \
  packages/ui/src/components/auth/SessionAuthGate.test.ts
bun run dead-code   # 删了导出，仓库规则要求；报告是非阻塞的，要看内容
```

提交 PR 前过 CONTRIBUTING.md 的 Before Submitting 全套：
`bun run type-check`、`bun run lint`、`bun run test`、`bun run build`。

Live run 与截图不做了（原因和替代证据见"验证记录"）。真机验证
顺其自然：修复随版本发布、维护者 Win11 桌面端升级后，把这里的
工作区设回默认再首连，不弹密码框即修复生效；这也是对 issue #3832
场景的直接复检。

## PR 模板预填

| Runtime | Behavior after this change |
|---|---|
| Web | 无变化，非桌面路径本来就重试 |
| Desktop (Electron) | 修复本体：网络失败先重试后错误屏，不再误弹密码框 |
| VS Code | Not applicable，skipAuth 不经过这个 gate |
| Hosted mobile | 无变化，非桌面路径 |
| Capacitor mobile | 无变化，`shouldUseDesktopShellPasswordLogin` 需要 `isDesktopShell()`，移动端恒为 false |

风险一栏的素材：改动只影响 catch 分支的桌面远程路径；401/200/429
分支未动；删掉的行为是三重巧合下的意外通道（见 PR 论证脚注）；回
滚即恢复原状，无数据或持久化影响。

## 修复涉及的文件

- `packages/ui/src/components/auth/SessionAuthGate.tsx`（checkStatus
  的 catch 分支、import 行）
- `packages/ui/src/components/auth/sessionAuthGateState.ts`
  （删 `resolveStatusCheckFailureState`）
- `packages/ui/src/components/auth/SessionAuthGate.test.ts`（删对应
  测试，describe 改名）
- `packages/ui/src/components/auth/SessionAuthGate.behavior.test.tsx`
  （桌面远程断言改为错误屏）

## 验证记录（2026-09-22 实施时实跑）

- `bun test` 聚焦两个测试文件：5 过 0 挂。
- `bun run --cwd packages/ui test` 全量：552/552 文件通过（就是它
  抓出了行为测试文件，见改动清单第 4 条）。
- `bun run lint`（根，五工作区）：全过。
- `bun run type-check`（根，四个工作区）：全过。
- `bun run test`（全仓库）：3302 过 / 31 挂，挂的全部在
  packages/web/server/lib（cron 计算、PR 状态、skills、project-config），
  已核实与本次改动无关：失败文件无一引用 @openchamber/ui 或 auth
  组件；ui 套件 552/552 全绿；cron 测试钉死 2026-09-18 日期、按
  UTC/基辅时区算期望值，本机是 Asia/Shanghai，单跑也挂，属时区/日期
  敏感的存量失败。
- `bun run build`：通过（1m40s，chunk 大小为建议性警告）。
- `bunx oxlint` 三个改动文件：改动行无发现；文件其余部分的
  anti-slop 存量发现属已知积压，未动。
- `bun run dead-code`：报告无本次文件的条目。
- live run 与截图：经维护者确认不做（2026-09-22）。原因：改的是
  Electron 桌面 shell 分支，开发机是无头 Linux（就是远程实例本
  身），维护者 Win11 桌面端跑的是发布版，两边都跑不了修复后的
  代码；在 Win11 搭 dev 环境成本过高。替代证据是
  SessionAuthGate.behavior.test.tsx 的渲染断言（mock 桌面 shell +
  网络失败 → 错误屏 + 主机切换器、无密码框）。PR 按声明版写明，
  预期挂 review:needs-evidence 标签，不阻塞；修复随版本发布后
  维护者重连即自然验证。

动手修改时需要加载的 skills：`openchamber-change-discipline`、
`sync-state-invariants`（runtime 切换行为）、`relay-transport`（relay
首次请求竞争问题）。

## 环境备忘

- 排查在维护者的甲骨文 Ubuntu 实例上进行（run/openchamber-10085.json，
  root 在线服务 port 10085）。这台机器就是"远程实例"本身：维护者从
  Win11 桌面端连过来，bug 场景 = 桌面端 + 这里设为默认工作区 + 当天
  首连竞争。维护者现在手动切主机使用，绕开了触发条件（隧道已建好，
  请求不再失败），所以不复现——不是修好了。
- 本地 main 已同步至 7be5f80bb（2026-09-22）。bun.lock 本地旧版存在
  `stash@{0}`（"local bun.lock before sync"），若需要找回：
  `git stash show -p stash@{0}`。
- 2026-09-22 环境变化：本机（无头 Linux）通过 npm 全局装了 bun
  1.4.2（不在 PATH 的场景用 `npm i -g bun` 重装）；gh 令牌已补
  workflow scope。
- 修复分支已推送：`fork`（theshasha/openchamber）上的
  `fix/session-auth-gate-relay-race`，单提交 d01e591a7，只含四个
  代码文件（中文工作文档不进 PR）。git 用户名 shasha。
- Win11 侧：clone fork → checkout 分支 → `bun install` →
  `bun run electron:dev`（clone 公开仓库无需切换 git 账号）。

## 明天从这里继续（2026-09-23，Win11 侧）

1. Win11 克隆里 `git fetch && git checkout worknotes`（文档 + 代码
   都在），装 bun、`bun install`、`bun run electron:dev`，添加远程
   主机连这台实例（首次可能要重新配对，最可能折腾的一步）。
2. 测试场景一（核心）：这边停 relay → 桌面端触发连接 → 断言错误屏
   而非密码框，截图①；恢复 relay → 点重试 → 进应用，截图②。
   Linux 侧配合在这台机器的工作区开会话即可。
3. 测试场景二（可选）：这边临时设 UI 密码 → 密码框照常出现且能登录，
   截图③；验完删密码。
4. 截图就位后：把 docs/session-auth-gate-pr.md 的 live run 和
   visual evidence 两段替换为实际结果，needs-evidence 预期解除。
5. 提 PR：从 fork 的 fix/session-auth-gate-relay-race 分支向上游，
   正文用 session-auth-gate-pr.md 的英文部分，`Fixes #3832`。
   注意 PR 分支保持只含代码（见"开发环境迁移"）。
