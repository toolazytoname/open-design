---
id: 20260513-garnet-skills-automations / track-A-skills
name: 把 /integrations 的 Skills tab 从占位换成真实功能
status: proposed
created: '2026-05-13'
---

## 目标

`/integrations` 路由下 Skills tab 当前是占位（`SkillsComingSoonPanel`）。本 track
负责把它接上 `apps/daemon/src/skills.ts` 提供的实际能力，让用户能在这个 tab 内：

- 看到全部技能列表（built-in + user-imported），支持搜索/source/mode/category 过滤
- 展开任意一条查看 `SKILL.md` 预览和文件树
- 内联编辑或新建 user 技能
- 删除 user 技能（built-in 不可删）

## 交付物

| # | 文件 | 操作 |
|---|---|---|
| 1 | `apps/web/src/components/SkillsSection.tsx` | 新增。来源：`origin/main:apps/web/src/components/SkillsSection.tsx`（836 行） |
| 2 | `apps/web/src/components/IntegrationsView.tsx` | 修改：移除 `SkillsComingSoonPanel`，`activeTab === 'skills'` 改为渲染 `<SkillsSection cfg={...} setCfg={...} />`；同时移除 `INTEGRATION_TABS` 中 Skills 项的 `hint: 'Coming soon'`，改为合适的描述（参考 main 上的 i18n 文案，或固定为 `'Project skills'` / `'技能管理'`） |
| 3 | `apps/daemon/src/skills.ts` | 升级。从当前 base（535 行 / 旧版）替换为 `origin/main:apps/daemon/src/skills.ts`（962 行）。带来的新能力：多 skill root 扫描、user skill 可 shadow 内置、新增导入/更新/删除入口的服务层支持 |
| 4 | `apps/daemon/src/static-resource-routes.ts` | 升级 skills 相关路由。从 main 拉取以下端点的完整实现（如果当前 base 上还没有）：`GET /api/skills`、`GET /api/skills/:id`、`POST /api/skills/import`、`PUT /api/skills/:id`、`GET /api/skills/:id/files`、`GET /api/skills/:id/example`、`GET /api/skills/:id/assets/*`、`POST /api/skills/install`、`DELETE /api/skills/:id` |
| 5 | `apps/web/src/providers/registry.ts` | 补齐 `fetchSkills`、`fetchSkill`、`fetchSkillFiles`、`importSkill`、`updateSkill`、`deleteSkill`、`SkillFileEntry` 等导出（SkillsSection 依赖；以 main 上的实现为准） |
| 6 | `packages/contracts/src/api/registry.ts` 或对应 contracts 文件 | 补齐 `SkillSummary` 等类型（以 main 为准） |
| 7 | `apps/daemon/tests/skills.test.ts`、`skills-delete-route.test.ts`、`skill-id-aliases.test.ts`、`skill-asset-rewrite.test.ts` | 与 main 对齐 |
| 8 | `apps/web/src/i18n/locales/*.ts` + `apps/web/src/i18n/types.ts` | 新增 `integrations.skills.*` 命名空间下需要的 string id（按 main 上的 SkillsSection 实际引用补齐；其它语言用 `*_IDS_WITH_EN_FALLBACK` 兜底） |

## 前置依赖

本 track 假设 spec.md 描述的 **Phase 0（shell 移植）已经完成**，即
`IntegrationsView.tsx`、新 router、EntryShell 挂载都已落在当前分支上。

如果 Phase 0 尚未落地，先按 spec.md `Phase 0` 章节做一次性移植，再开始本 track。

## 实现步骤

### Step 1 · 同步 daemon 端 skills 实现

1. 把 `origin/main:apps/daemon/src/skills.ts` 完整替换当前文件。
2. 解决与 `apps/daemon/src/plugins/local-skill.ts` 的并存问题：
   - `local-skill.ts` 是 garnet 上自己长出来的 plugin↔skill 桥，main 上没有。
   - 检查它现在引用 `skills.ts` 的哪些导出（grep `from '../skills'` / `from './skills.js'`）。
   - 如果新 `skills.ts` 已能覆盖其逻辑，删除 `local-skill.ts`；否则保留并把它需要的导出补回到新 `skills.ts` 末尾，原则上不修改 main 已有导出签名。
   - 同步看 `apps/daemon/tests/plugins-local-skill.test.ts` 是否需要调整。
3. 把 main 上 `static-resource-routes.ts` 中所有 `/api/skills/*` 端点同步过来；若当前 base 上 `static-resource-routes.ts` 不存在，看 server.ts 里 skills 路由的当前注册位置，把端点平移过去，然后在 server.ts 里调用 `registerStaticResourceRoutes(app, {...})`。
4. 同步 tests 目录下的 skills 相关测试文件。
5. 运行 `pnpm --filter @open-design/daemon test`，应当全绿。

### Step 2 · 同步 contracts

1. 对照 `origin/main:packages/contracts/src/api/registry.ts`（或 main 上声明 `SkillSummary` 的实际文件），补齐 `SkillSummary`、`SkillImportRequest`、`SkillUpdateRequest` 等类型。
2. 在 `packages/contracts/src/index.ts` 加 export（**注意：barrel 文件是 Track A 与 Track B 共享触点**，按"共享文件协议"约定，只追加自己一行 export，不改其它行的顺序）。
3. 运行 `pnpm --filter @open-design/contracts typecheck` 通过。

### Step 3 · 引入 SkillsSection.tsx

1. 把 `origin/main:apps/web/src/components/SkillsSection.tsx` 整文件复制到当前分支同名路径下。
2. 解决可能的 import 失配：
   - `useT()` —— 确认 `apps/web/src/i18n/index.tsx` 上有这个导出；没有就用当前分支的 i18n hook 等价物。
   - `Icon` —— 确认当前分支 `Icon.tsx` 支持 SkillsSection 用到的所有图标名；缺的图标补到 Icon 集里。
   - `../providers/registry` —— Step 2 已补齐。
3. 补齐 `apps/web/src/i18n/locales/en.ts` 下 SkillsSection 用到的所有 string id；其它 locale 按"无 TODO 注释"约定，仅在 `*_IDS_WITH_EN_FALLBACK` 列表追加 id，不写 TODO。
4. 运行 `pnpm --filter @open-design/web typecheck` 通过。

### Step 4 · 接到 IntegrationsView

修改 `apps/web/src/components/IntegrationsView.tsx`：

```tsx
// 顶部新增 import
import { SkillsSection } from './SkillsSection';

// INTEGRATION_TABS 改这一项
{
  id: 'skills',
  label: 'Skills',
  hint: 'Project skills', // 或 i18n key，统一与 main 一致
},

// 删掉 SkillsComingSoonPanel 函数

// body 渲染分支
{activeTab === 'skills' ? (
  <SkillsSection cfg={localConfig} setCfg={setLocalConfig} />
) : null}
```

`IntegrationsView` 当前只接收 `config` prop。`SkillsSection` 需要的是 `cfg` + 可写
的 `setCfg`。两种处理方式选其一：

- **A**：把 IntegrationsView 的内部 `localConfig` state 同时提供给 SkillsSection。
  缺点是 SkillsSection 改动不会回写到 App 顶层 config。
- **B（推荐）**：在 IntegrationsView 的 Props 上加 `onConfigChange: (next: AppConfig) => void`，从 EntryShell 一路传下来，最终在 App.tsx 处接到原本同步 config 的入口（参考 main 上 SettingsDialog 处理 cfg/setCfg 的方式）。

选 B 时，注意 EntryShell 的 prop 链是 Phase 0 已建立的；这里只是给 IntegrationsView
增加一个 prop，不会冲到 Track B。

### Step 5 · 验证 + 清理

- `pnpm typecheck && pnpm test` 全绿
- `pnpm --filter @open-design/daemon test` 全绿
- `pnpm tools-dev run web` 手动验：
  - 进 `/integrations` → Skills tab：列表加载、搜索、过滤、展开预览、新建用户技能、删除用户技能、删除内置技能被拒
  - 进首页 `@`-mention 选技能能正常出现（这是 PR #955 的另一面）
- 提交：建议拆 2~3 个 commit
  - `feat(daemon): port skills service + routes from main`
  - `feat(web): wire SkillsSection into /integrations`
  - `chore(i18n): add skill ids for integrations.skills.*`

## 共享文件协议（与 Track B 之间）

为避免两条 track 在并行时反复改到同一 hunk，约定如下：

| 文件 | 约定 |
|---|---|
| `apps/daemon/src/server.ts` | Track A 只在 skills 端点注册块附近（或 `registerStaticResourceRoutes(app, {...})` 调用位置）改动。Track B 的 `registerRoutineRoutes(app, {...})` 调用块独立，两边追加新调用而不重排已有调用顺序 |
| `packages/contracts/src/index.ts` | 只追加自己一行 `export * from './api/...'`，不动其他行 |
| `apps/web/src/i18n/locales/*.ts` | Track A 只在 `integrations.skills.*` 命名空间下追加 key |
| `apps/web/src/i18n/types.ts` | 只追加自己命名空间的 key 类型 |
| `apps/web/src/index.css` | 不要动 `tasks-*` 开头的 class（那是 Track B 的）。本 track 不预期会动 CSS（SkillsSection 自带的样式已在 main 的 index.css 里，按需把相关 hunk 整段拉过来追加到文件末尾） |
| `apps/web/src/components/EntryShell.tsx` | Phase 0 已经搭好骨架，本 track 仅在 IntegrationsView 的 prop 链上加 `onConfigChange`（如选实现 B）；不要动 TasksView 的挂载块 |

## 验收 checklist

- [ ] `pnpm typecheck` 通过
- [ ] `pnpm test` 通过
- [ ] `pnpm --filter @open-design/daemon test` 通过（含新拉过来的 skills 测试）
- [ ] `pnpm check:residual-js` 通过
- [ ] `/integrations` 路由的 Skills tab 显示真实技能列表，能完成"搜索 → 展开 → 编辑/导入/删除"完整流程
- [ ] 删除内置技能被拒（403/BAD_REQUEST），用户技能可被删除
- [ ] Chat 输入框 `@`-mention 中能看到技能项（验证 `ChatRequest.skillIds` 链路未坏）
- [ ] SettingsDialog → Skills 子页（如果当前 base 还保留）行为与 `/integrations` Skills tab 一致或正确跳转
- [ ] git diff 不含 `tasks-*` CSS 改动，不含 `apps/daemon/src/routines.ts`、`routine-routes.ts`、`contracts/src/api/routines.ts` 改动

## 不在范围内

- `/automations` 路由及其 4 张卡：归 Track B
- `apps/daemon/src/routines.ts` 及 routine API：归 Track B
- SettingsDialog 内部结构清理（保留旧 Skills/Routines 子页，等本期后另开 PR）
- Connectors tab、MCP tab、Use everywhere tab 的功能改动（已工作，不动）

## 参考来源（main 上的提交）

| Commit | PR | 用途 |
|---|---|---|
| `b5eb8c16` | #955 | Skills 体系大改：daemon 服务、路由、SkillsSection UI |
| `49b55761` | #1003 | skills & design-systems install/uninstall |
| `b1d440d2` | #1043 | daemon 路由拆分（如果当前 base 还没拆 server.ts，先看是否需要这一笔） |
