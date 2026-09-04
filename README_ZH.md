# dsh-agent-teams（你的 fork）

基于 `@nanmicoder/dsh-agent-teams` v0.1.15 的**私有多 Agent 协作插件**，移植并适配到 **DeepSeek Harness 0.1.2-alpha.4**，并新增了「新会话接管旧团队」「队长代做账」等增强。

一句话：把当前 DSH 会话变成队长，创建可续聊的子 Agent 成员、把目标拆成带依赖的任务、自动调度、成员直达消息、持久化状态 + Web 活动面板。无需额外 Workflow 引擎。

---

## 与上游 v0.1.15 的差异（本 fork）

### 修复的 Alpha.4 兼容问题
| 上游问题 | 本 fork 做法 |
| --- | --- |
| `ctx.subagents.registerContinuableSetup(...)` 不存在（启动崩溃根因） | 删除该 per-child 钩子；成员 persona / toolFilter / agentOptions / reasoningEffort 直接塞进 `ctx.subagents.startContinuable(request)` |
| `ctx.subagents.followup(...)` 不存在 | 改用 `ctx.subagents.sendMessage(captain, childId, content, { signal })` 唤醒成员 |
| 成员 `reasoningEffort` 丢失 | 在 `startContinuable` 的 `agentOptions` 里补传 `ReasoningEffortId(...)` |
| `scripts/clean-build.mjs` 在 Windows 崩溃 | 路径判定改用 `basename(...)==='lib'`（跨平台） |
| 后端成员 preset 继承导致缺 `agent_teams_*` 工具 | 增加 **member preset 守卫**（默认只允许 `standard`；**开 bookkeeping 后放宽**，允许梁神/极简 preset） |
| 客户端 bundle 加载报错（`without registering "dsh-agent-teams"`） | 三处名字统一为 `dsh-agent-teams`（`package.json.name`、`node_modules` 副本、`lib/client.js` 的 `__ModuleLoader__.load({id})`）；构建前同步 build 目录的 `package.json` |

### 新增功能
1. **`agent_teams_adopt`**：**新会话接管已有团队**。把 `captainSessionId` 改到当前会话、重排队长名下开放任务，并（默认）**自动补位交接**：
   - 为每个还有未完成任务（pending/claimed/in_progress）的旧成员**自动新建同角色新成员**（父代理=新队长，可唤醒）；
   - 自动把这些任务 **reassign 给新成员**（旧 attempt 失效、依赖/输出上下文通过任务记录继承）；
   - 旧成员标记 `removed`（其任务/角色/产出保留）。
   - 返回：`Team <id> adopted; you are now the captain. Auto-rehomed N member(s) and reassigned M unfinished task(s).`
2. **`agent_teams_list`**：任意会话列出当前工作区所有团队（id/name/phase/成员数/任务数），便于新队长发现并 `adopt`。
3. **`memberBookkeepingByCaptain`（队长代做账）**：`true` 时成员**不需要 `agent_teams_*` 工具**也能干活——派工提示改为“不用 claim/update_task，做完把结果写到 `<stateDir>/<teamId>/results/<taskId>.json`”；调度器在成员 idle 时读该文件自动 `completed/failed`。这样即使成员 preset 只有 bash/str_replace（梁神/极简）也能闭环。
4. **`autoReplaceEnabled` / `autoReplaceThreshold`**（70% 上下文自动接管，默认关）：成员 idle 时读 `contextPressure`，≥ 阈值打预警；开启后自动建替代成员并转派其开放任务。
5. **队长/成员 context-pressure 只读预警**：占用 ≥ 阈值时打日志，提示“考虑交接/新会话接管”。
6. **包名统一为 `dsh-agent-teams`**，客户端 bundle id 与之严格一致。

---

## 工具一览（15 个）
`agent_teams_create`、`edit_plan`、`approve`、`add_member`、`remove_member`、`create_task`、`reassign_task`、`claim_task`、`update_task`、`send_message`、`status`、`resume`、`delete` + **`list`** + **`adopt`**。

---

## 安装 / 构建

```sh
pnpm install
pnpm build          # 产出 lib/（含 lib/client.js 客户端面板）
```

装入 DSH profile 有两种方式：
- 本地目录：编辑 profile 的 `package.json` 依赖 + `dsh.profile.bundles` 加 `dsh-agent-teams`；
- 或用 `dsh plugin --profile <name> add <路径>`。

> 注意：本 fork 以 `dsh-agent-teams`（无 scope）为包名。装成 `file:` 本地目录时，`node_modules/dsh-agent-teams` 是**复制**（非符号链接）；若手动改包名/安装方式，必须保证 `package.json.name`、`node_modules` 副本 `package.json.name`、`lib/client.js` 的 `__ModuleLoader__.load({id})` **三处一致**，否则客户端会报 `without registering`。

---

## 配置（`cordis.patch.yml` 的 `dsh-agent-teams` 段）

```yaml
- id: agent-teams
  name: dsh-agent-teams
  config:
    stateDir: .agent-teams                 # 团队状态目录
    memberProvider: spawn                  # 子代理后端（spawn/fork），不是 LLM provider
    memberBookkeepingByCaptain: true       # 队长代做账：成员无需 agent_teams 工具也能闭环
    memberPresets: [standard]              # 允许生成成员的 preset（默认 standard）
    autoReplaceEnabled: false              # 70% 上下文自动接管（默认关）
    autoReplaceThreshold: 0.7              # 上下文阈值
    memberMaxDepth: 1
    maxMembers: 8
```

---

## 使用示例

### 建队 + 跑任务（自然语言）
> 用 AgentTeams 审查 X，从性能/安全/产品分工，最后汇总

### 新会话接管旧团队（队长上下文满 / 换人）
在新会话（建议先看 `agent_teams_list`）：
1. `agent_teams_list` → 找到团队 id；
2. `agent_teams_adopt <teamId>` → 自动补位+交接（返回 Auto-rehomed N / reassigned M）；
3. `agent_teams_status` → 看新成员 + 任务快照，继续派活。

> 说明：`adopt` 后旧成员是旧队长的子代理，新队长**无法唤醒它们**；所以插件自动生成新成员（新队长的子代理）并转派任务。若需要手动：`agent_teams_add_member` → `agent_teams_reassign_task` →（可选）`agent_teams_remove_member` 清旧成员。

---

## 已知注意点
- **一个队长同一时间只能带一个团队**。
- 团队状态为文件级持久化（单进程内锁串行）；多进程同团队不保证一致。
- **`escalated`**（自动 review/repair 循环到上限）≠ **`halted`**（人工停止）；前者需人工决策，后者需 `agent_teams_resume`。
- 成员（模型）不总严格走工具“仪式”；面板/状态以磁盘真相为准。
- 自动功能（`adopt` 自动补位、`autoReplace`）已通过 typecheck/组成校验/apply，**建议在实际团队上先验证再用于关键生产**。

---

## 许可证
MIT（继承上游）。
