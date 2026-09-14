# 本地源码安装与实测记录（web profile，Harness 0.1.5-rc.2）

日期：2026-09-14。目标：把 `web` profile 的 AgentTeams 从 npm 发布版换成工作区源码，并确认源码在真实宿主里的行为。

## 1. 卸载 npm 版并改为源码链接

npm 版（`@nanmicoder/dsh-agent-teams@0.1.18` 精确版本）已被替换，profile 不再引用任何 registry 版本：

```sh
node "C:\Users\Vinci\.dsh\profiles\node_modules\@deepseek-ai\dsh\lib\bin.js" \
  plugin --profile web add link:E:/Works/dsh-plugin/dsh-agent-teams
```

结果（`profiles/web/package.json`）：

- `dependencies["@nanmicoder/dsh-agent-teams"]`：`0.1.18` → `link:E:/Works/dsh-plugin/dsh-agent-teams`
- `dsh.profile.bundles` 仍含 `@nanmicoder/dsh-agent-teams`（reconcile 按真实包名对账，名称未变）
- `profiles/web/node_modules/@nanmicoder/dsh-agent-teams` 现在是指向工作区的链接，`lib/index.js` 与 `lib/client.js` 的时间戳等于本地构建时间

构建前提：该 checkout 此前从未安装依赖或完整构建（`lib/` 只有 34 个文件、没有 client bundle）。本次执行 `pnpm install`（成功；`ERR_PNPM_IGNORED_BUILDS` 仅列出被 pnpm 策略拦下的原生安装脚本，不影响 TypeScript 构建）与 `pnpm build`（exit 0，产物 `lib/index.js` + 197 kB 的 `lib/client.js` 及其 sourcemap）。

## 2. 实测：真实 rc.2 宿主 + 本地构建

### 2.1 隔离环境

为不改动用户正在运行的 GUI 进程，另建了 `tmp/scratch-home`：其 `profiles/node_modules` 是指向真实 `C:\Users\Vinci\.dsh\profiles\node_modules` 的 junction，因此宿主闭包就是用户实际安装的 231 个 `0.1.5-rc.2` 包，插件则来自工作区链接。模型用项目自带的确定性 fixture（`scripts/fixtures/harness-runtime-llm.mjs`），不调用真实 API。

### 2.2 Headless 运行（通过）

```
node <rc.2 dsh bin> --profile scratch 'Run the authorized deterministic AgentTeams fixture immediately.'
```

- stdout 出现 `AGENTTEAMS_PRODUCT_TURN_OK`，退出码 0。
- trace 正常结束（52 个事件：16 次模型请求 + 16 次响应），captain 实际收到的工具清单是全部 13 个 `agent_teams_*` 工具：`add_member, approve, claim_task, create, create_task, delete, edit_plan, reassign_task, remove_member, resume, send_message, status, update_task`。
- 该次运行同时证明了投递契约在 rc.2 生效：`agent/session-start` + `deliverPrompt` 路径下队员被唤醒并回报，任务以 `completed` 收尾。

### 2.3 Web profile 宿主（通过）

另起一个只装了 base + web-app + 本地插件的 scratch web profile，监听 3099（避开正在运行的 3080）：

- 启动成功并打印带 token 的 URL；`http://127.0.0.1:3099/` 返回 200、28 KB、含 `window.__DSH_BOOT__`。
- boot manifest 的 application batch 里包含 `@nanmicoder/dsh-agent-teams/client.js&rev=357548d52065158a-52`，`entries` 中该插件带 7 个 inject 项 —— 浏览器端产物在真实 Web 宿主里被正常下发。
- 带 token 与 cookie 请求 `/plugins/dsh-agent-teams/state` 返回 200，正文是活的团队快照：

```json
{"teams":[{"workspace":"dsh-agent-teams","teamId":"runtime-lab","name":"runtime-lab",
"phase":"running","members":[{"name":"worker","status":"idle","progress":100,"done":1,"total":1}],
"tasks":[{"id":"t1","status":"completed","assignee":"worker","model":"runtime-lab/fixture-model"}]}]}
```

- 无凭据访问同一路由返回 401（不是 404），说明路由确由插件注册并受鉴权保护，与 rc.1 审查记录的 401/403 语义一致。

结论：宿主入口、工具注册、子代理投递、HTTP 路由、以及浏览器端 bundle 下发都在真实 rc.2 上工作。

## 3. 观察到的测试稳定性限制（非本次改动引入）

- 复用同一 workspace 或短时间内重复跑 lifecycle 场景时，fixture 会陷入 captain 反复轮询 `agent_teams_status` 的循环（trace 涨到 18348 次请求直到超时）。这与 `docs/releases/v0.1.18/README.md` 记录的会话收尾竞态属于同一类问题；换干净 workspace 的首次运行稳定通过。
- 运行时 lab（`scripts/harness-runtime-verify.mjs`）在 npm 建出的 rc.2 闭包里缺少 `@deepseek-ai/dsh-code-runtime`、`@deepseek-ai/cordis-plugin-group` 等只在 pnpm hoisted 布局下可见的传递依赖，因此 8 场景无法在该闭包上跑完。这是 npm 闭包与 pnpm 闭包的差异，不是插件问题；用户实际运行的是 pnpm 安装的闭包，本节 2.2/2.3 就是在这个闭包上测的。

## 4. 尚未生效的一步

用户当前 GUI（PID 16852，`127.0.0.1:3080`）仍在用启动时加载的旧构建；`package.json` 的依赖变化不会热更新。要让本地源码在 GUI 里生效，需要重启该进程：

```powershell
cd E:\Works\dsh-harness   # 实际为 E:\Works\deepseek-harness
pnpm dsh web
```

重启后建议用 `/agent-teams` 建一个真实团队，确认面板、成员跳转与任务 DAG。浏览器端 UI 交互本身尚未在本次验证中覆盖（本次只验证了 bundle 下发与宿主路由）。

## 5. 本次新增/修改的文件

| 文件 | 说明 |
| --- | --- |
| `scripts/harness-runtime-verify.mjs` | 新增 `--plugin-dir`（可直接用已构建的 checkout，校验 exports 指向的入口存在）、`--skip-install`；子进程输出写文件；Windows 下 `PATHEXT`/`COMSPEC` 与 npm-cli 直调；`junction` 代替 symlink；`npm_config_userconfig` 只在 POSIX 设置 |
| `docs/local-source-install-2026-09-14/README.md` | 本记录 |
| `tmp/`（gitignore） | 隔离 DSH_HOME、workspace、运行时闭包与 trace，均未入库 |

未修改运行时代码、`compatibility.json`、peer 范围或任何发布元数据。

`pnpm install` 的副作用已清理并还原：它在仓库内生成 `.pnpm-store/`（283 MB）与带 `allowBuilds` 占位符的 `pnpm-workspace.yaml`，并对 `pnpm-lock.yaml`、`scripts/doctor.mjs` 造成纯行尾（LF→CRLF）改动。这三类都已删除/还原，因为本仓库的依赖管理与原生构建策略是既有约定，不应由一次本地验证改写。下次在此目录执行 `pnpm install` 时会重新生成它们；如需长期固定原生构建策略，应在项目自己的配置里显式声明。
