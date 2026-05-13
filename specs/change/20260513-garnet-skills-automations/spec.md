---
id: 20260513-garnet-skills-automations
name: 接手 garnet-hemisphere 上 Skills + Automations 两块空缺
status: proposed
created: '2026-05-13'
---

## 背景

同事在 `origin/garnet-hemisphere` 上把原本散在 SettingsDialog 弹窗里的几条产品线
重构成了两条顶级路由：

- `/integrations` → `IntegrationsView.tsx`，四个 tab：MCP / Connectors / **Skills (Coming soon)** / Use everywhere
- `/automations` → `TasksView.tsx`，四张 Primitive 卡片：**Orbit** / **Routines** / **Schedules** / **Live artifacts**

两个页面的 React 壳和 CSS 已经写好，但其中：

1. `/integrations` 的 **Skills tab** 是占位 (`SkillsComingSoonPanel`，11 行硬编码文案)
2. `/automations` 的 **Routines / Schedules / Live artifacts 三张卡**只是装饰；中部
   列表和右侧详情用的是文件顶部硬编码的 `BASE_TASKS` mock，"New automation" 按钮只
   会打开 Orbit 设置面板

需要我们接手把这两块从占位变成真实功能。

## 分支拓扑

| 分支 | 当前 HEAD | 与 main 关系 | 关键点 |
|---|---|---|---|
| `origin/main` | `03ed3960` (2026-05-13 11:25) | — | 功能实现齐全；UI 还挂在 SettingsDialog 弹窗 |
| `origin/garnet-hemisphere` | `9e196d34` (2026-05-13 14:35) | 在 `83ddf760` 处分叉，main 之后又跑了 190 个提交 | 两个顶级路由壳已就位；但缺 main 上后续合入的 Skills/Routines 功能 |
| `local-pr255` (本地接手分支) | `9cdfd2f3` | 从更早的 `816cc1d1` 分叉 | 两套都没有 |

garnet 分叉点之后、被它落下的关键 main commit：

| Commit | PR | 内容 |
|---|---|---|
| `b5eb8c16` | #955 | feat: generic skills + split skills/design-templates + finalize-design API（Skills 体系大改） |
| `49b55761` | #1003 | feat(daemon+web): add install/uninstall for skills & design systems |
| `643d0cf6` | #1033 | feat: add scheduled routines for unattended agent runs（Routines 完整功能） |
| `b1d440d2` | #1043 | refactor(daemon): split route registration |
| `9db5dd87` | #1490 | fix: close Settings modal when opening project from Routines |

## 各模块现状对比

### Skills

| 维度 | `origin/main` | `origin/garnet-hemisphere` | 差异说明 |
|---|---|---|---|
| Daemon 实现 | `apps/daemon/src/skills.ts` **962 行** | 同名文件 **535 行** | PR #955 在 main 上扩出 `POST /api/skills/import`、`DELETE /api/skills/:id`、多 skill root 扫描、user-skill 可覆盖内置、`ChatRequest.skillIds` 等 |
| Web UI | `apps/web/src/components/SkillsSection.tsx` **836 行**（@-mention 选择器、列表、内联编辑表单等） | 不存在 | 这就是同事 placeholder 里说的 "another branch" |
| 测试 | `skills.test.ts`、`skills-delete-route.test.ts`、`skill-asset-rewrite.test.ts`、`skill-id-aliases.test.ts` | 仅 `skills.test.ts`（旧）、`skill-asset-rewrite.test.ts`、`skill-id-aliases.test.ts` | main 多一套 delete-route 测试 |
| 周边 | — | 多出 `apps/daemon/src/plugins/local-skill.ts` + `plugins-local-skill.test.ts` | garnet 上自己长出来的 plugin↔skill 桥；main 上不存在，迁移时要决定怎么并 |

### Automations 四张卡

| 卡片 | 后端 | UI 数据来源 | 缺口 |
|---|---|---|---|
| **Orbit** | 两边都有 `apps/daemon/src/orbit.ts`（旧版单日 scheduler） | `config.orbit.enabled` / `config.orbit.time`，实数据 | 已工作，可不动 |
| **Routines** | main 有 `routines.ts` (543L) + `routine-routes.ts` + `db.ts` 新表 + `packages/contracts/src/api/routines.ts` (150L)；garnet 完全没有 | `BASE_TASKS` mock | 整笔功能 port (PR #1033) |
| **Schedules** | 同 Routines（Schedule 是 RoutineSchedule 子类型，kind = hourly/daily/weekdays/weekly + IANA tz） | 同 Routines mock | 随 Routines 一起 port |
| **Live artifacts** | 两边后端都有 `apps/daemon/src/live-artifacts/{schema,store,render,refresh,refresh-service}.ts` + `mcp-live-artifacts-server.ts`；garnet **不少于** main | 右侧 artifact 预览是文件硬编码 markdown 字符串 | 主要是 web 端拉一下 `LiveArtifactBadges.tsx` 等展示组件，把 TasksView 右栏接到真 artifact stream |

## 选定基础分支策略

**结论：基于 `origin/main`，把 garnet 的两个路由壳挪过来。**

理由：
- main 已经有 Skills + Routines + Live artifacts 的完整后端与原 UI，迁移 garnet 的
  两个壳只需挪 ≈ 600 LOC（IntegrationsView + TasksView + EntryShell 改动 + 路由 +
  CSS），冲突面最小。
- 反过来从 garnet 拉 PR #955 + #1033 + #1003 + #1043，要落 4000+ LOC 和 DB schema
  改动，且 garnet 上还有 plugins/local-skill.ts 这个自己长出来的桥，与 PR #955 的
  skills.ts 边界需要现场调和，风险显著大。

具体动作：
1. 从 main 切一条新分支（命名见 division 文档）。
2. 把 garnet 上以下文件 cherry-pick 或人工迁移过来：
   - `apps/web/src/router.ts`（新 route `/integrations`、`/automations`/`tasks`）
   - `apps/web/src/components/IntegrationsView.tsx`
   - `apps/web/src/components/TasksView.tsx`
   - `apps/web/src/components/EntryShell.tsx` 中 `view === 'integrations'` 和
     `view === 'tasks'` 两个分支的挂载逻辑
   - `apps/web/src/styles/home/integrations.css` 以及 `tasks-*` 相关样式
   - 任何被 EntryShell 顺带新引的 prop 链（如 `composioConfigLoading`、
     `onPersistComposioKey`、`integrationTab` 等）
3. 在这个 base 上分两条 track 同时推进（见 `division.md`）。

## Phase 0 · 共同前置（shell 移植）

两条 track 都依赖这一步先完成。它本身是一次性的小 PR，无业务逻辑改动，谁先开始
谁就把它落了，**只需做一次**。完成后两条 track 才可以并行开干。

### 改动范围

从 `origin/garnet-hemisphere` 移植到 main 基础上：

| 文件 | 操作 |
|---|---|
| `apps/web/src/router.ts` | 替换为 garnet 版本（新增 `/integrations`、`/automations`、`/tasks` 路由及 `EntryHomeView` 中的对应值） |
| `apps/web/src/components/IntegrationsView.tsx` | 新增整文件（154 行）。Skills tab 内部保留占位 `SkillsComingSoonPanel`，由 Track A 后续接真组件 |
| `apps/web/src/components/TasksView.tsx` | 新增整文件（398 行）。`BASE_TASKS` mock、4 张 PrimitiveCard 卡片样式照搬，由 Track B 后续替换为真数据 |
| `apps/web/src/components/EntryShell.tsx` | 引入 `IntegrationsView` / `TasksView`，挂在 `view === 'integrations'` 和 `view === 'tasks'` 两个分支；新增 prop 链：`integrationInitialTab`、`composioConfigLoading`、`onPersistComposioKey`、`onOpenSettings` 扩展枚举值（`integrations` / `mcpClient` / `composio` / `orbit` 等） |
| `apps/web/src/App.tsx` | 传 `integrationInitialTab`、`composioConfigLoading`、`onPersistComposioKey` 等到 EntryShell |
| `apps/web/src/styles/home/integrations.css` | 新增整文件 |
| `apps/web/src/index.css` | 追加 `tasks-view`、`tasks-primitive`、`task-card`、`task-detail` 等样式块（从 garnet 该文件相关 hunk 摘出） |
| `apps/web/src/components/EntryNavRail.tsx` 或导航项 | 加入 Integrations / Automations 入口（参考 garnet 实现） |

### 验收

- `pnpm typecheck` 通过
- 启动 `pnpm tools-dev run web` 后：
  - `/integrations` 打开能看到 4 个 tab，Skills tab 显示 "Coming soon" 占位
  - `/automations` 打开能看到 4 张 PrimitiveCard + mock 列表/详情
- 现有 SettingsDialog 不受影响（Skills/Routines/Composio 等子页仍可正常打开）

### 不在 Phase 0 范围内

- 不动 `apps/daemon/src/skills.ts`、`apps/daemon/src/routines.ts`、任何 contracts
- 不改 SettingsDialog 的现有挂载
- 不接任何真数据；占位/mock 照搬

## 两条独立 track

本期把工作拆成两条文件级互不重叠的 track，各自有独立的实现文档可以直接交给
执行者：

| Track | 范围 | 实现文档 |
|---|---|---|
| A · Skills | `/integrations` 的 Skills tab + `apps/daemon/src/skills.ts` 升级 + 周边 contracts / 测试 | [`track-skills.md`](./track-skills.md) |
| B · Automations | `/automations` 四张卡接真数据：Routines/Schedules 完整 port + Live artifacts 展示组件接入 | [`track-automations.md`](./track-automations.md) |

少量共享文件（`apps/daemon/src/server.ts`、`packages/contracts/src/index.ts`、
`apps/web/src/i18n/locales/*.ts`、`apps/web/src/index.css`）通过命名空间和分区约定避免
冲突，约定列在每份 track 文档的"共享文件协议"小节。

## 后续延伸（不在本期范围）

- SettingsDialog 里同名的 Skills / Routines / Composio / Orbit 子页，等顶级路由稳定
  后可以考虑保留入口跳转、移除重复内容。本期不动 SettingsDialog 的代码。
- garnet 上的 `apps/daemon/src/plugins/local-skill.ts` 与 PR #955 的 `skills.ts`
  如何融合属于 Skills 这条 track 内部决策，列在 division 文档里。
