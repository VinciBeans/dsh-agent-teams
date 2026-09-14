# AgentTeams 0.1.18 对 DeepSeek Harness 0.1.5-rc.2 的兼容性检查

检查日期：2026-09-14。范围：只做检查，不修改运行时代码、peer 范围、支持矩阵或用户 profile。

**结论：代码层面兼容，声明层面不支持。** 从 `dsh-v0.1.5-rc.1` 到 `dsh-v0.1.5-rc.2`，AgentTeams 实际消费的宿主 API 没有一处变化；插件产物在真实 rc.2 安装上可以加载。但 `compatibility.json`、22 条 DSH peer 范围、devDependencies baseline 与 pnpm overrides 都只枚举到 `0.1.5-rc.1`，因此 `doctor`、`verify:compatibility` 和 npm 安装路径仍会把 rc.2 判为不支持。本报告不改这些声明，也不把 rc.2 提升为受支持目标。

## 1. 区间、纯净性与证据来源

- 本次开发者环境：`E:\Works\deepseek-harness`，HEAD `185168df676ee31c2eb0241d212116a0fcc36614`（`dsh-v0.1.5-rc.2-141-g185168df67`，工作树仅有未跟踪的 `.zcode/`）。只用 `git show` / `git diff <tag> <tag>` 只读读取目标 tag，没有 checkout、fetch 或改写宿主工作树。
- From：`dsh-v0.1.5-rc.1` / `183f08e9c6dde7e36cd2318eaee70b0da08fb35e`。
- To：`dsh-v0.1.5-rc.2` / `fb2c4b9e698e30edb738bca4cf0618587db7d203`。
- `git merge-base` 等于 From 本身，区间无基线漂移。
- 规模：4 个提交（2 个非合并）、334 文件、+1050 / −1050 行。其中 272 个是 `package.json` 版本号改写，30 个是源码（`.ts`/`.tsx`/`.css`），其余为 README、i18n、`.agents/notes` 与测试。
- 用户实际运行的是 npm 发布的 `0.1.5-rc.2`：`C:\Users\Vinci\.dsh\profiles\node_modules\@deepseek-ai` 下 231 个 DSH 包全部解析为 `0.1.5-rc.2`，宿主 `@deepseek-ai/dsh` 亦为 `0.1.5-rc.2`。以下判定以该已安装闭包和 tag 源码为准。

## 2. 区间内到底改了什么

两个非合并提交：`060323d8e2 feat(web): backport feedback and file refinements to 0.1.5`、`a305303422 release(dsh): 0.1.5-rc.2`。实质源码改动只落在 6 个包：

| 包 | 源码变化 | 是否触及 AgentTeams |
| --- | --- | --- |
| `client/ui-message-feedback` | 反馈对话框/动作控制器重构，slots.ts、controller.ts、dialog.ts、locales.ts 等 | 否（仅 `dsh.client.inject` 元数据里出现，运行时不用） |
| `client/ui-deliverables` | `Deliverables`/`PresentedFileCard` 与 CSS | 否 |
| `client/ui-primitives` | `CodeFileIcon.tsx` 拆出 `code-file-icon-artwork.ts` + manifest（−477/+145） | 包级被导入，但导出面与所用符号不变，见 §3 |
| `client/ui-chat` | `TurnTailNodeView.module.css` 增加 3 行 `margin-top` | 否（无类型/槽契约变化） |
| `feedback/command-feedback`、`feedback/message-feedback` | 4 文件 + `types.ts` 一行注释 | 否 |
| `apps/web/tests` | 4 个 e2e 测试 | 否 |

其余 272 个 `package.json` 全是 `0.1.5-rc.1` → `0.1.5-rc.2` 版本号改写；唯一例外是 `client/ui-message-feedback/package.json` 同时改了 description 文案。`dsh-base` 与 `dsh-web-app` 两个 bundle 的 `cordis.patch.yml` 在两端完全一致，因此插件挂载的配置层没有变化。

## 3. 插件消费面逐项核对

下表按 AgentTeams 源码里的真实导入面（`src/**`，25 个不同包）逐个比较两端源码树 hash。`src` 树 hash 相同即该包的源码在区间内逐字节一致。

| 消费的包 | 用途 | rc.1 → rc.2 源码树 | 判定 |
| --- | --- | --- | --- |
| `subagent/subagent` | `queueMemberPrompt`/`steerMemberPrompt`/`guardSubagentDelivery` 依赖 `Symbol.for('dsh.subagent.deliverPrompt')`、`queuePrompt`、`sendMessage` | 一致（`4ed05739a782…`） | 保留 |
| `core/agent` | `Agent` 类型、`agent/session-start` 显式 payload | 一致 | 保留 |
| `core/tools`、`core/session`、`core/system-prompt` | 工具注册、`ownEvents()`、system prompt 段 | 一致 | 保留 |
| `llm/llm` | `createUserMessage`、ContentBlock/MessageSource | 一致 | 保留 |
| `interaction/commands` | `/agent-teams` 命令注册 | 一致 | 保留 |
| `workspace/workspace`、`api/session-controller` | 工作区与 session 列表 | 一致 | 保留 |
| `host/webserver` | `/plugins/dsh-agent-teams/state` 原始路由 | 一致 | 保留 |
| `util/values` | 值工具 | 一致 | 保留 |
| `client/ui-conversation`、`ui-layout`、`ui-slots`、`ui-model-selection`、`ui-session`、`connection`、`store`、`locale`、`ui-renderer` | 客户端服务与槽 | 一致 | 保留 |
| `client/ui-chat` | `conversation.chat.node` / `commandview` 槽 | **不同**：仅 `TurnTailNodeView.module.css` +3 行 | 契约不变 |
| `client/ui-primitives` | `IconBranchOutline16`、`IconChevronDownOutline14`、`IconPanelLeftOutline16`、`IconStopFill16`、`IconWarningOutline16`、`Modal` | **不同**：`CodeFileIcon.tsx` 拆分 | 所用 6 个符号不受影响 |

插件注册的三个槽在 rc.2 全部存在且 kind 与传入参数一致：

| 槽 | rc.2 契约 | 插件传参 |
| --- | --- | --- |
| `shell.overlay` | `{ kind: 'list'; scope: 'root' }` | `id`/`order`/`label`/`locale`（`src/client/index.tsx:79`） |
| `conversation.chat.node` | `{ kind: 'keyed'; scope: 'session' }` | `key: 'agent-teams'`（`src/client/index.tsx:96`） |
| `conversation.chat.commandview` | `{ kind: 'keyed'; scope: 'session' }` | `key: 'agent-teams'`（`src/client/index.tsx:90`） |

rc.1 审查（[0.1.5-rc.1 适配报告](../harness-0.1.5-rc.1-audit-2026-09-10/README.md) §1、§2）标记为“直接阻塞”的两处已被 0.1.17/0.1.18 适配层解决，且承载它们的 `subagent/subagent` 与 `core/agent` 源码在 rc.2 未变，所以适配结论对 rc.2 同样成立。

## 4. 加载与解析实测

在用户真实安装上执行探针（`tmp/compat-rc2/probe.mjs`，工作区临时目录，不入库）：

- 宿主入口 `lib/index.js` 在 rc.2 闭包下**加载成功**，导出 `Config`、`apply`、`inject`、`name`、`usageSectionText`。宿主入口不含任何 `@deepseek-ai/*` 值导入，因此不存在宿主侧模块解析风险。
- 22 个非 shell 拥有的 peer 在该 profile 内全部解析到 `0.1.5-rc.2`（`@deepseek-ai/cordis` 4.0.2、`@deepseek-ai/schemastery` 3.18.2、`react` 18.2.0）。
- `@deepseek-ai/dsh-client-store`、`dsh-client-ui-slots`、`dsh-client-ui-primitives` 在 profile 的 node_modules 里不存在，因为它们是浏览器 shell 的 platform module（`packages/client/web/src/platform.ts` 的 `PLATFORM_MODULES`），由模块表下发而不是靠 node 解析。这属正常形态，不是缺失依赖；探针对这三项不做断言。

## 5. 声明层面的三个门槛

1. **`compatibility.json` 与 peer 范围只枚举到 rc.1。** `supportedHosts` 为 `0.1.5-rc.1`(recommended)、`0.1.2-rc.1`、`0.1.2-alpha.5`、`0.1.2-alpha.2`；`scripts/doctor.mjs` 因此对 rc.2 明确报错：`Unsupported host 0.1.5-rc.2; recommended target is 0.1.5-rc.1`（该次运行同时确认 231 个 DSH 包身份无混合、无缺失包）。
2. **`scripts/compatibility.mjs` 要求 peer 范围逐一枚举宿主目标。** 第 73–80 行规定每个 `@deepseek-ai/dsh*` peer 的 `||` 分段数必须等于 `supportedHosts` 数量且逐一对应。因此“加宽范围”这种最小改法无法通过 `verify:compatibility`，加入 rc.2 必须同步改 22 条 peer、`devDependencies` baseline 和 `pnpm.overrides`。
3. **npm 安装路径会 ERESOLVE。** 探针环境用 npm 安装 `0.1.18` + rc.2 闭包时被拒：

   ```
   Found: @deepseek-ai/dsh-util-values@0.1.5-rc.2
   Could not resolve dependency:
   peerOptional @deepseek-ai/dsh-util-values@"0.1.5-rc.1 || 0.1.2-rc.1 || 0.1.2-alpha.5 || 0.1.2-alpha.2"
   from @nanmicoder/dsh-agent-teams@0.1.18
   ```

   这条**只影响 npm 消费者**。`dsh plugin` 是 pnpm 转发层（`apps/cli/src/plugin.ts:3`、`:134`），而 profile 的 `pnpm-workspace.yaml` 设 `autoInstallPeers: false`、`nodeLinker: hoisted`，peer 从宿主 profile 的 node_modules 直接解析，pnpm 不会用这组范围去否决安装。这也是为什么用户侧安装成功而探针侧失败。

## 6. 已确认保留（兼容性成立的部分）

- 子代理投递符号与签名：`Symbol.for('dsh.subagent.deliverPrompt')`、`queueHostSubagentPrompt`、`steerHostSubagentPrompt`、`sendMessage` 在 rc.2 源码中存在且树 hash 与 rc.1 相同。
- `agent/session-start` 显式 Agent payload、`Agent.followup/steer/inject/whenIdle`。
- `Session.ownEvents()`、工具注册表、system prompt 段、命令注册。
- 原始 HTTP 路由 `webServer.register` 与真实路由包 `host/webserver/src`（两端一致）。
- 三个客户端槽的名称、kind 与 scope。
- 客户端 bundle 的导入纯度：只用 platform module（`react`、`react/jsx-runtime`、`@deepseek-ai/dsh-client-ui-primitives`）加已豁免的 `@deepseek-ai/dsh-client-runtime`。

## 7. 验证范围与未验证项

已做：tag 级源码树比对、插件导入面映射、槽契约核对、真实 rc.2 闭包上的宿主入口加载与 peer 解析、`doctor` 实测。

未做（不要据本报告推断已通过）：

- **真实宿主运行未完成。** 项目的 `scripts/harness-runtime-verify.mjs` 8 个场景（lifecycle/fallback/failure/captain-idle-wakeup/progressive-entry/web-approval/protocol-compatibility/stability）在本次 Windows 环境下未能跑通，原因是 lab 自身的 npm 安装在 `npm_config_userconfig=/dev/null` 与 `@deepseek-ai/cordis-plugin-group` peer 上失败；这是 lab 环境问题，不是插件结论。修复过程中已把该脚本的 Windows 兼容问题（`PATHEXT`/`COMSPEC`、npm-cli 直调、`junction` 代替 symlink、子进程输出改文件、新增 `--skip-install`）一并处理，见 §9。
- 浏览器内真实 UI 未验证：活动面板渲染、成员跳转、刷新与会话切换、探针 HTTP 路由的真实响应，都需要在受支持宿主上按项目规则用 Ego Lite 复核。
- 模型/推理力度继承、冷恢复、会话迁移与归档面板未在 rc.2 上复跑。
- pack 产物问题（与兼容性无关）：本次 `pnpm pack` 出来的 tarball 缺 `lib/`，因为该 checkout 从未安装依赖或构建（`pnpm build` 因缺 node_modules 失败）。npm 上的 0.1.18 是完整的。

## 8. 要把 rc.2 纳入受支持矩阵需要做什么

1. 把 `compatibility.json` 的 `supportedHosts` 加入 `0.1.5-rc.2` 并选定 track；若改为 recommended，需同时更新 `recommendedHost`。
2. 同步 22 条 DSH peer 范围、`devDependencies` baseline 与 `pnpm.overrides`（`scripts/compatibility.mjs:73-80`、`:55-72` 强制三者一致）。
3. 在真实 rc.2 宿主上跑完 §7 的未完成项，尤其是 8 个运行时场景和 Web UI。
4. 更新 `README.md` 第 52–80 行的推荐组合表与安装命令（当前仍写 `0.1.17`，而 npm `latest` 已是 `0.1.18`）。
5. 若还希望 npm 消费者能直接安装，需要让 peer 范围覆盖实际宿主（或在文档中明确只支持经 `dsh plugin`/pnpm 安装）。

在此之前，rc.2 的正确状态仍是“已安装、可加载、未获支持声明”：`doctor` 报 unsupported 是当前策略下的正确输出，不应绕过。

## 9. 本次检查附带产生的改动

| 文件 | 改动 | 原因 |
| --- | --- | --- |
| `scripts/harness-runtime-verify.mjs` | 子进程输出写文件而非管道；Windows 下传 `PATHEXT`/`COMSPEC` 并经 `process.execPath` 直调 `npm-cli.js`；用 `junction` 代替 symlink；新增 `--skip-install`；`npm_config_userconfig` 仅在 POSIX 设置 | 该脚本在 Windows 上原本无法启动，与 rc.2 无关 |
| `.gitignore` | 增加 `tmp/` | lab 的 runtime 闭包与报告目录落在 `tmp/` 下 |
| `docs/install-logs/` | 安装记录与 `--dump-config` 前后快照 | 安装验收证据 |

这些都未触碰运行时代码、`compatibility.json`、peer 范围或用户 profile。`scripts/harness-runtime-verify.mjs` 已通过 `node --check` 与参数守卫检查，且没有被 `pnpm verify` 或其他脚本导入（只作为独立 CLI 使用）；但它尚未在 Linux/CI 上验证，也未跑通完整 8 场景，提交前应补一次真实运行。

## 10. 证据文件位置

- 本次临时证据：`tmp/compat-rc2/`（已 gitignore）——探针脚本、lab runtime 闭包、`result.json`、各场景日志。
- 安装记录：[install-agent-teams-web.md](../install-logs/install-agent-teams-web.md)。
- 上游参考：[0.1.5-rc.1 发布](https://github.com/deepseek-ai/deepseek-harness/releases/tag/dsh-v0.1.5-rc.1)、[0.1.5-rc.2 发布](https://github.com/deepseek-ai/deepseek-harness/releases/tag/dsh-v0.1.5-rc.2)。
