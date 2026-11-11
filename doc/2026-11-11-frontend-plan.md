# it-ticket-admin 前端实现计划

> **面向 AI 代理的工作者：** 必需子技能：使用 subagent-driven-development（推荐）或 executing-plans 逐任务实现此计划。步骤使用复选框（`- [ ]`）语法来跟踪进度。

**目标：** 实现 Vue 3 后台界面：登录、工单列表/详情/创建、用户管理，操作按钮按角色与状态显隐。

**架构：** Vite + Vue 3 单页应用；Pinia 管登录态，`utils/permissions.ts` 为纯函数复刻后端可见性/操作规则（仅决定界面显隐，权限以后端为准）；Element Plus 提供表格/表单/弹窗；API 层 fetch 封装统一注入 Bearer token。

**技术栈：** Vue >= 3.4、Vite >= 6、TypeScript >= 5.4、Pinia >= 3、vue-router 4、Element Plus >= 2.8、Vitest + @vue/test-utils + jsdom。

**规格：** `doc/2026-11-11-it-ticket-admin-design.md`（第 5–8 节）、`doc/2026-11-11-backend-plan.md`（API 路由与响应结构）。计划的论证依据来自规格，执行者两份都读。

## 全局约束

- 界面文案用中文（我的决定）；代码、注释用英文；注释只解释「为什么」。
- 五个页面：登录、工单列表、工单详情、创建工单、用户管理（设计 §7）。状态标签：提交/受理/处理/完成/关闭。
- 禁止 `v-html`；用户输入（标题/描述/备注）一律按纯文本渲染。
- token 存 `localStorage`；API base 取 `import.meta.env.VITE_API_BASE_URL`，禁止硬编码 URL。
- 工单输入限制与后端一致（backend-plan 全局约束）：标题 1–100、描述 1–2000、备注 1–1000。
- 命名：目录 kebab-case、组件 PascalCase、函数 camelCase、命名导出、`const` 优先、禁 `var`、early return、单文件 ≤ 300 行。
- 每任务：从 `develop` 切 `feature/<task-slug>`，测试通过且 `pnpm --filter web build && pnpm --filter web lint` 无报错后 `git merge --no-ff` 回 `develop`；提交用 Conventional Commits。

## 审查重点（Review Focus）

1. 未登录访问受保护路由 → 重定向 `/login`，且登录后回跳原页面（钉在任务 1）。
2. token 过期后接口返回 401 → 清除登录态并跳 `/login`，不卡在错误页（钉在任务 1）。
3. 三种角色看到的列表范围与按钮不同 → 由 `permissions.ts` 纯函数测试钉住（钉在任务 2）。
4. 标题为空或超长 → 表单阻止提交并给出中文提示（钉在任务 5）。
5. 用户输入含 HTML/脚本片段 → 列表与时间线按纯文本展示（钉在任务 4）。

## 文件结构

- 创建 `web/src/utils/permissions.ts` + `permissions.spec.ts`：角色/状态 → 可见操作纯函数
- 创建 `web/src/types/ticket.ts`、`types/user.ts`：与后端共享类型对应
- 创建 `web/src/api/client.ts` + `auth.ts` + `tickets.ts` + `users.ts`：fetch 封装与接口
- 创建 `web/src/stores/auth.ts`：Pinia 登录态
- 创建 `web/src/router/index.ts`：路由与守卫
- 创建 `web/src/views/*`：LoginView、TicketListView、TicketDetailView、TicketCreateView、UsersView
- 创建 `web/src/components/*`：TicketStatusTag、TicketTimeline、TicketActionButtons

## 共享类型（跨任务引用，只在此定义一次）

```ts
// web/src/types/ticket.ts — 与 backend-plan「共享类型」对齐
export type Role = 'employee' | 'agent' | 'admin';
export type TicketStatus = 'submitted' | 'accepted' | 'processing' | 'resolved' | 'closed';
export type TicketAction = 'claim' | 'start' | 'resolve' | 'close' | 'comment' | 'assign' | 'clearAssignee';
export interface ActorView { id: number; role: Role }
export interface TicketStateView { id: number; status: TicketStatus; creatorId: number; assigneeId: number | null }
export interface TicketSummary extends TicketStateView { title: string; updatedAt: string }
export interface TicketDetail extends TicketSummary { description: string; events: TicketEventView[] }
export interface TicketEventView { id: number; type: string; actorName: string; content: string; createdAt: string }
```

```ts
// web/src/utils/permissions.ts
export function canView(ticket: TicketStateView, actor: ActorView): boolean;
export function visibleActions(ticket: TicketStateView, actor: ActorView): TicketAction[];
```

---

### 任务 1：骨架、登录与路由守卫

**文件：**
- 创建：`web/`（tech-init 第 4 节脚手架）、`web/src/api/client.ts`、`api/auth.ts`、`stores/auth.ts`、`router/index.ts`、`views/LoginView.vue`、`types/user.ts`
- 创建：`web/src/router/router.spec.ts`、`web/src/stores/auth.spec.ts`

- [ ] **步骤 1：编写失败的测试**

```ts
// router.spec.ts
it('redirects unauthenticated users to /login', ...);          // to.meta.requiresAuth 且无 token
it('returns to the original page after login', ...);            // redirect query 保留
it('keeps regular routes open', ...);                           // /login 不重定向
// auth.spec.ts
it('login stores token and user in localStorage', ...);
it('401 response clears session', ...);                         // client.ts on401 → clearSession + throw
it('logout clears token and user', ...);
```

- [ ] **步骤 2：运行测试验证失败**
运行 `pnpm --filter web test -- router` 与 `pnpm --filter web test -- auth`；预期 FAIL（模块不存在）。
- [ ] **步骤 3：实现登录链路**
- `client.ts`：`request<T>(path: string, options?: RequestInit): Promise<T>`，自动加 Bearer、JSON 编解码；401 时 `clearSession()` 并抛 `UnauthorizedError`。
- `stores/auth.ts`：`login(username, password)`、`logout()`、`restore()`，token 与 user 持久化到 `localStorage`。
- `router/index.ts`：`/login`、`/tickets`、`/tickets/new`、`/tickets/:id`、`/users`（`meta: { requiresAuth: true, roles?: Role[] }`）；守卫未登录跳 `/login?redirect=`，角色不符跳 `/tickets`。
- `LoginView.vue`：Element Plus 表单，登录失败显示「用户名或密码错误」，成功跳 `redirect` 或 `/tickets`。
- [ ] **步骤 4：运行测试验证通过**
运行两个测试命令（PASS）+ `pnpm --filter web build && pnpm --filter web lint` 无报错。
- [ ] **步骤 5：Commit** 运行 `git add web && git commit -m "feat(web): add login, auth store and route guards"`，然后 `git checkout develop && git merge --no-ff feature/01-login-guards`

### 任务 2：权限纯函数

**文件：**
- 创建：`web/src/utils/permissions.ts`、`web/src/utils/permissions.spec.ts`

- [ ] **步骤 1：编写失败的测试**

```ts
// permissions.spec.ts — 与 backend-plan 任务 3 的规则一一对应
it('canView scopes tickets by role', ...);                      // employee 仅自己创建；agent 加待接单与自己处理；admin 全部
it('visibleActions gives agent claim on unassigned submitted', ...);
it('visibleActions gives agent start/resolve on own ticket in order', ...);
it('visibleActions gives creator close on own resolved', ...);
it('visibleActions gives admin assign on non-closed and close on resolved', ...);
it('visibleActions returns empty on closed ticket', ...);
```

- [ ] **步骤 2：运行测试验证失败**
运行 `pnpm --filter web test -- permissions`；预期 FAIL。
- [ ] **步骤 3：实现 `permissions.ts`**
按设计 §6.1/§8 与 backend-plan 任务 3 的规则实现两个函数；可见范围与按钮映射只算显隐，不构成权限。
- [ ] **步骤 4：运行测试验证通过**
运行 `pnpm --filter web test -- permissions`（PASS）+ build/lint 无报错。
- [ ] **步骤 5：Commit** 运行 `git add web/src/utils && git commit -m "feat(web): add role and action visibility rules"`，然后 `git checkout develop && git merge --no-ff feature/02-permissions`

### 任务 3：工单列表页

**文件：**
- 创建：`web/src/types/ticket.ts`、`api/tickets.ts`、`views/TicketListView.vue`、`components/TicketStatusTag.vue`
- 创建：`web/src/components/TicketStatusTag.spec.ts`、`web/src/views/TicketListView.spec.ts`

- [ ] **步骤 1：编写失败的测试**

```ts
// TicketStatusTag.spec.ts
it('renders Chinese label for each status', ...);               // 提交/受理/处理/完成/关闭
// TicketListView.spec.ts
it('renders only visible tickets for the actor', ...);          // mock API 返回越权数据也过滤
it('filters by status via query param', ...);
it('shows create button for all and users entry for admin only', ...);
```

- [ ] **步骤 2：运行测试验证失败**
运行 `pnpm --filter web test -- TicketList`；预期 FAIL。
- [ ] **步骤 3：实现列表页**
- `api/tickets.ts`：`listTickets(status?: TicketStatus): Promise<TicketSummary[]>`、`getTicket(id: number): Promise<TicketDetail>`。
- `TicketListView.vue`：Element Plus 表格（标题/状态/处理人/更新时间），状态筛选、行点击进详情；列表用 `canView` 二次过滤（后端已过滤，前端兜底）。
- [ ] **步骤 4：运行测试验证通过**
运行 `pnpm --filter web test -- TicketList`（PASS）+ build/lint 无报错。
- [ ] **步骤 5：Commit** 运行 `git add web && git commit -m "feat(web): add ticket list view"`，然后 `git checkout develop && git merge --no-ff feature/03-ticket-list`

### 任务 4：工单详情页与时间线

**文件：**
- 创建：`web/src/views/TicketDetailView.vue`、`components/TicketTimeline.vue`、`components/TicketActionButtons.vue`
- 创建：`web/src/components/TicketTimeline.spec.ts`、`web/src/components/TicketActionButtons.spec.ts`

- [ ] **步骤 1：编写失败的测试**

```ts
// TicketTimeline.spec.ts
it('renders events in creation order with actor and time', ...);
it('renders HTML-like content as plain text', ...);             // content 含 <script> 必须转义展示（审查重点 5）
// TicketActionButtons.spec.ts
it('renders claim button for agent on claimable ticket', ...);
it('renders start and resolve only for the assignee in order', ...);
it('renders close for creator or admin on resolved only', ...);
it('renders no action buttons on closed ticket', ...);
```

- [ ] **步骤 2：运行测试验证失败**
运行 `pnpm --filter web test -- Ticket`；预期 FAIL。
- [ ] **步骤 3：实现详情页**
- `TicketDetailView.vue`：字段卡片 + `TicketTimeline` + `TicketActionButtons` + 备注输入框（1–1000 字）。
- `TicketActionButtons.vue`：props `{ ticket: TicketDetail; actor: ActorView }`，按钮集来自 `visibleActions`，点击 emit 对应 action，成功后刷新详情。
- [ ] **步骤 4：运行测试验证通过**
运行 `pnpm --filter web test -- Ticket`（PASS）+ build/lint 无报错。
- [ ] **步骤 5：Commit** 运行 `git add web && git commit -m "feat(web): add ticket detail with timeline and actions"`，然后 `git checkout develop && git merge --no-ff feature/04-ticket-detail`

### 任务 5：创建工单页

**文件：**
- 创建：`web/src/views/TicketCreateView.vue`
- 创建：`web/src/views/TicketCreateView.spec.ts`

- [ ] **步骤 1：编写失败的测试**

```ts
it('submits title and description then navigates to detail', ...);
it('blocks empty title with Chinese message', ...);             // 「请输入标题」（审查重点 4）
it('blocks title over 100 chars and description over 2000 chars', ...);
```

- [ ] **步骤 2：运行测试验证失败**
运行 `pnpm --filter web test -- TicketCreate`；预期 FAIL。
- [ ] **步骤 3：实现 `TicketCreateView.vue`**
Element Plus 表单 + 校验规则（长度与全局约束一致）；`api/tickets.ts` 补 `createTicket(input: CreateTicketInput): Promise<TicketDetail>`，成功跳 `/tickets/:id`。
- [ ] **步骤 4：运行测试验证通过**
运行 `pnpm --filter web test -- TicketCreate`（PASS）+ build/lint 无报错。
- [ ] **步骤 5：Commit** 运行 `git add web && git commit -m "feat(web): add ticket create view"`，然后 `git checkout develop && git merge --no-ff feature/05-ticket-create`

### 任务 6：用户管理页与收尾

**文件：**
- 创建：`web/src/types/user.ts`、`api/users.ts`、`views/UsersView.vue`
- 创建：`web/src/views/UsersView.spec.ts`

- [ ] **步骤 1：编写失败的测试**

```ts
it('renders user table with role and active state', ...);
it('create and edit dialogs submit expected payloads', ...);
it('shows backend 409 message when deactivation is blocked', ...); // 名下在途工单 → 展示原因，不静默失败
```

- [ ] **步骤 2：运行测试验证失败**
运行 `pnpm --filter web test -- UsersView`；预期 FAIL。
- [ ] **步骤 3：实现用户管理页**
`api/users.ts`：`listUsers()`、`createUser(input)`、`updateUser(id, input)`；表格 + 创建/编辑弹窗（角色下拉、重置密码、停用/恢复）；409 时展示后端错误信息。
- [ ] **步骤 4：全量验证**
运行 `pnpm --filter web test && pnpm --filter web build && pnpm --filter web lint`；全绿后合回 `develop`，与后端一起按 tech-init §7 打 `release/v0.1.0`。
- [ ] **步骤 5：Commit** 运行 `git add web && git commit -m "feat(web): add admin user management view"`，然后 `git checkout develop && git merge --no-ff feature/06-users-view`

