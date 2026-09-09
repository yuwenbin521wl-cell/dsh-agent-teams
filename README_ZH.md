# dsh-agent-teams（你的 fork）

基于 `@nanmicoder/dsh-agent-teams` v0.1.15 的**私有多 Agent 协作插件**，移植并适配到 **DeepSeek Harness 0.1.2-alpha.4**，并新增 / 修复了大量围绕「防记忆丢失、自动补位、任务正确闭环、并行调度、新会话接管」的能力。

一句话：把当前 DSH 会话变成队长，创建可续聊的子 Agent 成员、把目标拆成带依赖的任务、自动调度、成员直达消息、持久化状态 + Web 活动面板。无需额外 Workflow 引擎。

> 本文列出**相对上游 v0.1.15 的全部改动**，方便你 / 协作者 / 看到这个仓库的人快速理解。

---

## 一、Alpha.4 兼容修复（上游会崩，这里已修）
| 上游问题 | 本 fork 做法 |
| --- | --- |
| `ctx.subagents.registerContinuableSetup(...)` 不存在（启动崩溃根因） | 删除该 per-child 钩子；成员 persona / toolFilter / agentOptions / reasoningEffort 直接塞进 `ctx.subagents.startContinuable(request)` |
| `ctx.subagents.followup(...)` 不存在 | 改用 `ctx.subagents.sendMessage(captain, childId, content, { signal })` |
| 成员 `reasoningEffort` 丢失 | `startContinuable` 的 `agentOptions` 补传 `ReasoningEffortId(...)` |
| `scripts/clean-build.mjs` 在 Windows 崩溃 | 路径判定改用 `basename(...)==='lib'`（跨平台） |
| 客户端 bundle 加载报错（`without registering "dsh-agent-teams"`） | 三处名字统一（`package.json.name`、`node_modules` 副本、`lib/client.js` 的 `__ModuleLoader__.load({id})`）；构建前同步 build 目录的 package.json |

---

## 二、任务正确闭环（修“报告完成但状态卡住”）
- **`in_progress` 不终态化**：成员阶段交回写 `{"status":"in_progress","output":"..."}` 时保持进行中（只更新 output），只有明确 `completed`/`failed` 才终态。
- **每个 `kickTeam` 都读结果文件自动闭环**：不再只依赖成员 idle 边沿；任何状态查看/调度都会把“报告完成/失败”的任务置为对应状态，避免卡住下游。
- **级联取消**：上游任务被取消时，自动取消其（传递）下游的非终态任务，避免“永不满足依赖”的死链永久挂起。
- **remove_member 重排任务**：移除成员时把它名下的非终态任务重置为 `pending`+`assignee=undefined`（回共享池），不再挂在被移除成员上。
- 状态一致性：`agent_teams_status` 支持 **`team_id` 只读查看**任意团队（不 adopt、不改队长），避免“查状态误接管”。

---

## 三、新会话接管 / 自动补位（防记忆丢失）
- **`agent_teams_adopt <teamId>`**：新会话接管已有团队（改 `captainSessionId`、重排队长任务），并**自动补位**：为每个还有未完成任务的旧成员建同角色新成员（父代理=新队长，可唤醒）并交接任务；**无开放任务的旧成员也标记 removed（不留占位）；**接管会清退所有旧成员（含前几批 -r2 替补成员）**，多次接管只留新队长名下的一套新成员**。
- **`agent_teams_rehome`**：对已接管但旧成员还是旧队长子代理的团队，一键补位（幂等，只补不是当前队长已生成的成员）。
- **`agent_teams_list`**：任意会话列出团队，便于发现并 adopt。

---

## 四、上下文管理（不压缩、走交接，防记忆丢失）
- 协议第 11 条：**不要压缩上下文硬撑**。
- 成员到 ~70%：优先**新建成员**（`add_member`）或 `agent_teams_rehome` 交接其开放任务。
- 队长到 ~70%：不要压缩、不要自驱；收尾后用 **`agent_teams_adopt` 交给新会话队长**（团队状态持久化，自动补位后继续）。
- **交接前把关键结论/决策写进任务 `output` 或 results 文件**，让新的成员/队长读到真实状态（把“会丢的对话推理”转成“不丢的持久产出”）。
- 并发/并行协议（step 4）：多个无依赖任务分给**不同空闲成员**或留空，**不要全排到一个忙碌成员后面**（否则串行）。

## 五、自动补位 / 自动接管
- **`autoReplaceEnabled`（默认 true）** + **`autoReplaceThreshold`（默认 0.7）**：成员空闲且上下文 ≥ 阈值时，自动建替代成员（`-r3`）并交接其开放任务；**在 `kickTeam` 也扫描**（覆盖冷恢复的已耗尽团队），不再只等 idle 边沿。

---

## 六、成员 preset 守卫 + 客户端包名
- **member preset 守卫**：默认只允许 `standard`（避免生成只拿 bash/str_replace 的“缺 agent_teams 工具”成员）；**开启 bookkeeping 后放宽**（允许梁神/极简 preset，因为成员不再需要 agent_teams 工具也能闭环）。
- **`memberBookkeepingByCaptain`（队长代做账，默认 true）**：成员无需 `agent_teams_*` 工具，只写 `results/<taskId>.json`，插件/队长闭环。
- **团队契约自动注入**：用 `agent_teams_set_contract` 写入硬规则，插件会自动把它注入到每个成员 persona 和每次任务派工提示（不再只出现在早期聊天里被遗忘）。
- **用法（不用记命令）**：你直接用自然语言说“写进团队契约：必须复用 v39 的 GraphRunner，不得自实现执行器”，队长会**自动**调用 `agent_teams_set_contract` 写入；无需你打工具指令。
- **追加 vs 覆盖**：`mode=append`（默认）追加新规则保留旧规则；`mode=replace` 整体覆盖。
- **怎么写好契约**：具体可核对（复用 X / 禁止 Y / 必须引用 Z）、显式负向约束（不得…）、给出权威来源（以 v39 源码/反编译为准）、给出“不一致时怎么办”（先停下向用户确认）、给出验收证据形式（引用 v39 代码/日志而非口头承诺）。
- **配合验收兜底**：任务用 `objective/acceptance/inScope/outOfScope/verify` 把约束写进合同（如 `outOfScope=["自实现执行器"]`、`acceptance=["复用 v39 GraphRunner"]`），并建 `kind=review` 门禁，不合规 `needs_revision` 打回。\n- **包名统一为 `dsh-agent-teams`**：`package.json.name`、`node_modules` 副本、`lib/client.js` 的 `__ModuleLoader__ id` 三处一致（避免 `without registering`）。

---

## 七、UI：任务依赖图「显示已完成」开关
- 活动面板任务依赖图新增 **「显示已完成任务」** 开关（**默认关**）：
  - 关：只显示进行中/待命任务 + 作为进行中任务前置依赖的已完成任务（依赖链不断开）；
  - 开：显示全部（含已完成/已终止），便于回溯。

---

## 八、工具一览（16 个）
`agent_teams_create`、`edit_plan`、`approve`、`add_member`、`remove_member`、`create_task`、`reassign_task`、`claim_task`、`update_task`、`send_message`、`status`、`resume`、`delete`、`list`、`adopt`、`rehome`。

---

## 九、安装 / 构建

```sh
pnpm install
pnpm build          # 产出 lib/（含 lib/client.js 客户端面板）
```

装入 DSH profile：本地目录（`package.json` 依赖 + `dsh.profile.bundles` 加 `dsh-agent-teams`），或 `dsh plugin --profile <name> add <路径>`。

> 注意：`file:` 本地目录装的是**复制**（非符号链接）；改包名/安装方式时必须保证 `package.json.name`、`node_modules` 副本、`lib/client.js` 的 `__ModuleLoader__ id` 三处一致，且**重建前把构建目录的 package.json 同步**，否则会回退到旧 scope 名。

---

## 十、配置（`cordis.patch.yml` 的 `dsh-agent-teams` 段）

```yaml
- id: agent-teams
  name: dsh-agent-teams
  config:
    stateDir: .agent-teams
    memberProvider: spawn
    memberBookkeepingByCaptain: true      # 队长代做账：成员无需 agent_teams 工具
    memberPresets: [standard]             # 允许生成成员的 preset（开 bookkeeping 后放宽）
    autoReplaceEnabled: true              # 上下文 70% 自动建替代成员并交接
    autoReplaceThreshold: 0.7
    memberMaxDepth: 1
    maxMembers: 8
```

---

## 十一、使用示例

- 建队：`用 AgentTeams 审查 X...`
- 只看：`agent_teams_list` / `agent_teams_status team_id=<id>`（只读）
- 接管：`agent_teams_adopt <teamId>`（自动补位+交接）→ `agent_teams_status`
- 交给新会话队长：新会话 `agent_teams_adopt <teamId>`；先把它名下的关键结论写进任务 output/results。
- 手动补位：`agent_teams_rehome`；移除旧成员：`agent_teams_remove_member <name>`（自动重排其任务）。

---

## 十二、已知注意点
- 一个队长同一时间只能带一个团队。
- `escalated`（自动 review/repair 到上限）≠ `halted`（人工停止）；前者需人工决策。
- 自动功能（`adopt` 自动补位、`autoReplace`）已通过 typecheck/构建/组成校验，**建议在真实团队先验证再用于关键生产**。
- 旧成员被 rehome 后是旧队长的子代理，新队长无法唤醒它们（所以插件自动生成新队长名下的替代成员）。
- **自动派发心跳**：插件每约 4s 自动 `kickTeam` 一次，把未指派/已就绪任务派给空闲成员——即使没有事件（如建任务没人派、成员空闲没触发），也会被自动领走，不再需要你催。
- **队长验证 + 僵尸清理**：队长建任务后必须 `agent_teams_status` 核实（存在/已派/在跑），不会谎称“已运行”；队长可**接管（reassign assignee=captain）并取消**依赖已 failed/cancelled 的僵尸任务（t184/t185 类），不再被“未完成依赖”卡死。
- **执行进度可见性**：bookkeeping 模式下，成员在任务执行期间会**周期性写进度**（`results/<taskId>.json` 里的 `in_progress` + `output`，如 “40% - 已完成X，下一步Y”）；`agent_teams_status` 会显示每个任务的 **最近更新时间（⏱Nmin）**，且 **in_progress/claimed 超 30 分钟未更新**会标 **⚠️no-progress-30min**（疑似卡住/死循环）——此时用 `agent_teams_rehome` 或 `agent_teams_reassign_task` 把活转给别人。

---

## 十三、自动治理与自愈（Auto-governance）

插件内置一套“自动治理”，防止上下文被撑爆、死链/僵尸堆积、无限换人：

- **成员 / 活子代理上限 `maxMembers=10`**：`agent_teams_add_member` 超 10 抛错；adopt/rehome **1 换 1 替换**不突破上限，只有已超上限才停。
- **自动替换（有开放任务才换）**：成员**上下文满**（≥ `autoReplaceThreshold` 默认 0.7）或**任务失败≥ `failureThreshold` 默认 2**，且**有开放任务** → 自动 spawn 新成员接管、旧成员退役；**无任务不动**（不自动废弃）。
- **替换上限 `maxAutoReplace=2`**：按 `replacementDepth` 计代，**自动替换到 2 代就停**，改由**队长/用户判断**任务是否本身无法完成。
  - **显式替换不受此限**：队长/用户明确要求（`agent_teams_rehome` / `agent_teams_reassign_task` / 让队长换）可**继续替换**，不受自动上限限制。
- **自动清理**：
  - `agent_teams_adopt` / `agent_teams_rehome` 自动**取消僵尸任务**（依赖链含 failed/cancelled，`cancelZombieTasks`）+ **归档历史**（`archive-tasks.json`）+ **清孤儿子代理**。
  - `agent_teams_cleanup`：清除当前队长名下“非活跃成员”的孤儿子代理 + 修剪 removed 成员记录。
- **成员不刷屏**：成员 persona 禁止发无意义消息（上线/待命/无分配任务/状态报告），只在真实结果/阻塞/被问时发。
- **执行进度可见**：成员用结果文件 `in_progress` 刷新进度，`agent_teams_status` 显示 `⏱Nmin` + `⚠️no-progress-30min`。

对应 `cordis.patch.yml` 配置：

```yaml
memberBookkeepingByCaptain: true
autoReplaceEnabled: true
autoReplaceThreshold: 0.7   # 上下文 ≥70% 触发自动换人
failureThreshold: 2          # 失败 ≥2 次触发自动换人
maxAutoReplace: 2            # 自动替换到第 2 代就停（交队长/用户判断）
maxMembers: 10               # 活成员/子代理上限
```
---

## 许可证
MIT（继承上游）。
