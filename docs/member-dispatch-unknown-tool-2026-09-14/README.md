# Web 组合下队员无法生成：未知工具名导致 `tools.restrict()` 抛错

发现日期：2026-09-14。环境：Harness `0.1.5-rc.2`（源码运行 `pnpm dsh web`），AgentTeams `0.1.18` 以 `link:` 方式从本工作区加载，`web` profile。

## 症状

在 Web GUI 里走正常流程（`/agent-teams` 规划 → Approve & Run）后：

- 面板显示 `t1` "就绪待开工"，队员状态 `idle/unspawned`，看起来像在等人工操作；
- 实际上调度器**反复派发并回滚**：`t1` 的 `attempt` 从 1 一路涨到 4，每次回滚后 `status` 回到 `pending`、`attemptId` 清空、队员状态置回 `idle`（`src/scheduler.ts:438-449` 的回滚分支）；
- 队员的 `id` 始终是空字符串 —— `spawnMember` 从未走到 `member.id = start.childId`；
- 宿主终端没有任何 `agent-teams:` 输出，面板也没有错误提示。

headless 组合下同样的代码路径完全正常（运行时 lab 的 8 个场景、以及本次 rc.2 headless 实测均通过），所以问题只在 Web 组合出现。

## 根因

子代理组合时，`tools.restrict()` 对**未注册的工具名**是硬错误：

```
tools.restrict() names unknown global tool "subagent"; known global tools: … subagent_fork …
  at packages/core/tools/src/index.ts:1081
  at applyChildComposition (packages/subagent/subagent/src/child-agent.ts:217)
  at setup (packages/subagent/subagent/src/continuation-activation.ts:586)
```

而插件的 deny 列表里确实带着一个在本组合不存在的名字：

- `src/members.ts` 原先传 `toolFilter: { deny: [...MEMBER_DENIED_TOOLS, ...(maxDepth === 0 ? ['subagent', 'send_message'] : [])] }`；
- `dsh-web-app` 的 bundle patch **整行禁用**了基础委派工具行 —— `tool-subagent-control`、`tool-subagent-list-agents`、`tool-subagent`、`tool-subagent-fork`（第 436-453 行，注释写明"subagent registry 与后端留在 host plane，preset 决定 agent 看到哪些委派工具"）；
- 因此 `subagent` 这个名字没有被注册，`restrict()` 抛错 → `startContinuable` 失败 → `spawnMember` 中断 → 派发失败 → 任务回滚。

`send_message`（`maxDepth === 0` 时的另一项）在本组合存在，所以只有 `subagent` 触发。

### 为什么既有测试没抓到

四个宿主版本的 CI 矩阵与运行时 lab 都使用 headless 式组合（`dsh-base` + `dsh-headless`），那里 `tool-subagent` 是挂载的，`subagent` 名字存在，路径畅通。缺陷只在**浏览器面**的组合出现，而那恰恰是主要使用场景。`compatibility.json` 的版本枚举与本缺陷无关。

## 修复

**1. deny 列表只保留真实存在的全局工具名。**

- `src/members.ts`：新增 `memberToolDenyList(ctx, maxDepth)`，用 `ctx.tools.schemas()` 取当前可见的全局工具名，与待拒绝列表求交集后再传给 `toolFilter`。不存在的名字本来也不可调用，拒绝它没有意义，写上它却会让整条派发失败。
- `src/capabilities.ts`：队员会话启动时对 AgentTeams 工具名做 restrict 的同类调用（原第 90 行）也加了相同过滤，避免同一模式下再次崩溃。

**2. 让派发失败可被看见。**

`src/tools.ts` 的 `dispatchMember` 原本在所有静默 `false` 分支（团队状态守卫、队员守卫、attemptId 能力守卫）以及 catch 分支只写 `ctx.logger.warn`。现在每个分支都会把结构化原因追加写入 `.agent-teams/<team>/dispatch-failure.jsonl`（守卫名 + 当时的 phase/halted/captainMatches/队员状态/reassigning 任务/磁盘上各任务的 status·assignee·attemptId），catch 分支额外记录报错与堆栈。写盘失败被吞掉，**不改变派发行为**。

这次排查的最大障碍正是"失败没有可见面"：面板、终端、团队状态三处都没有原因，只能靠给源码加埋点才拿到那条 `restrict()` 报错。

## 验证

修复前后在**同一个真实 Web profile** 上对照：

| 指标 | 修复前 | 修复后 |
| --- | --- | --- |
| 队员 `inspector` | `idle/unspawned`，`id` 为空 | `working/running`，`id=97e632e1-904…` |
| 任务 `t1` | `pending`，attempt 反复递增后回滚 | `in_progress`（attempt 5） |
| `dispatch-failure.jsonl` | 每次派发新增一条 `spawn-threw` | 无新增（只留修复前那条记录） |
| 依赖门禁 | — | `t2` 保持 `pending`，等 `t1` 完成 |

修复后队员正常执行并回报：`t1` 由 `inspector` 完成，产出对本地 link、导出闭包、双产物平面、名称一致性的四项核对结果。

### 端到端生命周期（同一真实宿主）

修复后一轮完整周期全部走通，并在过程中由队员独立取证：

| 环节 | 证据 |
| --- | --- |
| 暂存 | `agent_teams_create` 带 `approval: "required"` → "It is staged"，队长未自行批准 |
| 审批 | `approvedAt` 由 UI 触发，与暂存相隔 43.4 s；审批前不存在任何队员会话 |
| 派发 | `t1` attempt 5 成功生成 `inspector`（`id` 非空），`t2` 受依赖门禁保持 pending |
| 执行与回报 | `t1` 完成写入 output/acceptanceResults/commandsRun，报告落 `inbox/captain.jsonl` |
| 依赖解锁 | `t1` 完成后调度器自动把 `t2` 派给 `runner` |
| 能力隔离 | 队员会话恰有 4 个 `agent_teams_*` 工具（`MEMBER_TOOL_NAMES`），队长 13 个；`create/approve/delete` 对队员不可见 |
| 归档 | `agent_teams_delete` → `.agent-teams/archive/local-build-smoke/`，`team.json`（28630 B）、`inbox/captain.jsonl`、任务依赖图全部保留，成员标记 `removed`，团队从 `agent_teams_status` 消失 |

### 任务书写的教训（非产品缺陷）

本轮 `t2` 被标为 `failed`，原因是我在任务书里要求**队员**去"创建再归档一个临时团队"，而 `create/approve/delete` 属队长专属工具。队员按设计拒绝了这两步并在报告里指出"重派给队员无法修复，只有队长权限可以"。这不是插件问题，反而验证了能力隔离。教训：给队员的任务只能要求其权限内的动作（claim/update/report/status 与其普通编码工具）；需要队长权限的验证必须由队长自己执行。

## 遗留

1. **缺少 Web 组合的回归覆盖。** 运行时 lab 场景集应加入"委派工具行被禁用"的组合（复现本次条件），否则同类缺陷仍会只在浏览器面暴露。本次未实现。
2. **`compatibility.json` 与 peer 范围仍不含 `0.1.5-rc.2`** —— 与本次缺陷无关，属既有声明策略，见 [0.1.5-rc.2 兼容性检查](../harness-0.1.5-rc.2-compat-2026-09-14/README.md)。队员 `inspector` 独立复核时也报告了同一漂移。
3. **`lib/` 由未提交的工作树构建**（HEAD `3b95edbe04aacc4a27cc91ba41430c1f69e47739`，改动了 `src/capabilities.ts`、`src/members.ts`、`src/tools.ts`）。发布或提交前需按项目流程走静态检查与验证矩阵。
4. **`.agent-teams/runtime-lab/` 是 lab 夹具留下的孤儿团队目录**（`captainSessionId` 属于早前进程的夹具会话 `session-a665cc6d…`，`phase` 为空、`worker` 仍 `idle`）。当前队长会话无权归档他人的团队，需要按仓库规则单独清理。
