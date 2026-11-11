# it-ticket-admin 后端实现计划

> **面向 AI 代理的工作者：** 必需子技能：使用 subagent-driven-development（推荐）或 executing-plans 逐任务实现此计划。步骤使用复选框（`- [ ]`）语法来跟踪进度。

**目标：** 实现 NestJS + MySQL 后端：JWT 登录、员工/处理人/管理员三级权限、工单全生命周期与用户管理。

**架构：** 单体 NestJS 应用，TypeORM 三张表（users、tickets、ticket_events）；工单状态与权限判定收敛到纯函数 `ticket-state.ts`，服务层只负责持久化与守卫；REST + JWT Bearer；工单操作全部写入只增不改的操作记录。

**技术栈：** NestJS >= 10、TypeORM >= 0.3、MySQL 8、@nestjs/jwt + passport-jwt + bcrypt、Jest + supertest。

**规格：** `doc/2026-11-11-it-ticket-admin-design.md`（第 5–9 节）与 `doc/2026-11-11-it-ticket-admin-tech-init.md`（初始化命令、环境变量）。计划的论证依据来自规格，执行者两份都读。

## 全局约束

- 角色枚举 `employee | agent | admin`；状态枚举 `submitted | accepted | processing | resolved | closed`，单向流转，关闭只能从 `resolved` 发起，管理员也不能跳过状态（设计 §6、§10）。
- 「待接单」= `submitted` 且处理人为空；接单后不允许清空处理人，只能改派（§6.1）。
- `ticket_events` 只增不改不删（§6）；用户软删除 `isActive=false`，禁止硬删除（§5）。
- 作为处理人名下有非 `closed` 工单的用户，停用或降级为非 `agent` 角色 → 409（§5）。
- 错误模型：400 校验失败/非法输入、401 未认证、403 越权、404 不存在或不可见、409 状态冲突或保护规则。
- 工单输入限制（我的决定，前后端一致）：标题 1–100 字符，描述 1–2000 字符，备注 1–1000 字符。
- 环境变量只从 `api/.env` 读取，禁止硬编码；`.env` 禁止提交。
- 命名：目录 kebab-case、函数 camelCase、命名导出、`const` 优先、禁 `var`、early return、单文件 ≤ 300 行。
- 每任务：从 `develop` 切 `feature/<task-slug>`，测试通过且 `pnpm --filter api build && pnpm --filter api lint` 无报错后 `git merge --no-ff` 回 `develop`；提交信息用 Conventional Commits。

## 审查重点（Review Focus）

1. 空/错误的用户名或密码登录 → 401，且不区分「用户不存在」与「密码错误」（钉在任务 2）。
2. 访问不属于自己的或不可见的工单 → 404，不泄露工单是否存在（钉在任务 4）。
3. 两个处理人抢同一张「待接单」→ 先到者成功，后到者 409，状态不被破坏（钉在任务 5）。
4. 停用/降级名下有在途工单的处理人 → 409，其工单保持原状（钉在任务 7）。
5. 非法状态跳转（如 `submitted` 直接 `resolve`、对 `closed` 工单操作）→ 409（钉在任务 3、5）。

## 文件结构

- 创建 `api/src/tickets/ticket-state.ts`：状态机与可见性纯函数，全后端唯一规则来源
- 创建 `api/src/tickets/ticket.entity.ts`、`ticket-event.entity.ts`：TypeORM 实体
- 创建 `api/src/tickets/tickets.service.ts`、`tickets.controller.ts`：工单用例与 REST
- 创建 `api/src/users/user.entity.ts`、`users.service.ts`、`users.controller.ts`：用户与保护规则
- 创建 `api/src/auth/*`：登录、JWT 策略、`JwtAuthGuard`、`RolesGuard`、`Roles` 装饰器
- 创建 `api/src/health.controller.ts`：健康检查
- 创建 `api/src/seed/seed.ts`：演示数据
- 创建 `api/test/*.e2e-spec.ts`：e2e；单测与被测文件同目录（`*.spec.ts`）

## 共享类型（跨任务引用，只在此定义一次）

```ts
// api/src/tickets/ticket-state.ts
export type Role = 'employee' | 'agent' | 'admin';
export type TicketStatus = 'submitted' | 'accepted' | 'processing' | 'resolved' | 'closed';
export type TicketAction = 'claim' | 'start' | 'resolve' | 'close' | 'comment' | 'assign' | 'clearAssignee';
export interface ActorView { id: number; role: Role }
export interface TicketStateView { id: number; status: TicketStatus; creatorId: number; assigneeId: number | null }
export interface ActionResult { allowed: boolean; nextStatus: TicketStatus | null; reason?: string }
export function evaluateAction(action: TicketAction, ticket: TicketStateView, actor: ActorView): ActionResult;
export function canView(ticket: TicketStateView, actor: ActorView): boolean;
```

```ts
// api/src/users/user.entity.ts 导出
export interface UserView { id: number; username: string; role: Role; isActive: boolean }
```

---

### 任务 1：骨架与健康检查

**文件：**
- 创建：根 `package.json`、`pnpm-workspace.yaml`、`docker-compose.yml`、`api/`（`nest new` 产物）
- 创建：`api/src/health.controller.ts`、`api/test/health.e2e-spec.ts`
- 修改：`api/src/app.module.ts`、`api/src/main.ts`

- [ ] **步骤 1：搭建骨架**
执行 `doc/2026-11-11-it-ticket-admin-tech-init.md` 第 4 节全部命令（含 `git init -b main`、创建 `develop`），然后 `git checkout -b feature/01-skeleton-health`。确认 `docker compose up -d` 后 MySQL 可连接。
- [ ] **步骤 2：编写失败的测试**

```ts
// api/test/health.e2e-spec.ts
it('GET /health returns ok', async () => {
  const res = await request(app.getHttpServer()).get('/health').expect(200);
  expect(res.body).toEqual({ status: 'ok' });
});
```

- [ ] **步骤 3：运行测试验证失败**
运行 `pnpm --filter api test:e2e -- health`；预期 FAIL（404）。
- [ ] **步骤 4：实现 `HealthController`**
`@Get('health')` 的 `getHealth(): { status: string }` 返回 `{ status: 'ok' }`，注册进 `AppModule`。
- [ ] **步骤 5：运行测试验证通过**
运行 `pnpm --filter api test:e2e -- health`（PASS）+ `pnpm --filter api build && pnpm --filter api lint`（无报错）。
- [ ] **步骤 6：Commit** 运行 `git add . && git commit -m "feat: scaffold api workspace with health check"`，然后 `git checkout develop && git merge --no-ff feature/01-skeleton-health`

### 任务 2：用户实体与 JWT 认证

**文件：**
- 创建：`api/src/users/user.entity.ts`、`users.module.ts`、`users.service.ts`
- 创建：`api/src/auth/auth.module.ts`、`auth.service.ts`、`auth.controller.ts`、`jwt.strategy.ts`、`jwt-auth.guard.ts`
- 创建：`api/test/auth.e2e-spec.ts`

- [ ] **步骤 1：编写失败的测试**

```ts
// api/test/auth.e2e-spec.ts（beforeAll 用 UsersService.createUser 建 staff1/employee）
it('login rejects an empty body with 400', ...);
it('login rejects an unknown user with 401', ...);
it('login rejects a wrong password with 401', ...);   // 与上一条响应体一致，不泄露原因
it('login returns token and user for valid credentials', async () => {
  // expect 201/200, body: { token: expect.any(String), user: { username: 'staff1', role: 'employee', isActive: true } }
});
it('me rejects a missing token with 401', ...);
it('me returns the current user', ...);
```

- [ ] **步骤 2：运行测试验证失败**
运行 `pnpm --filter api test:e2e -- auth`；预期 FAIL（路由不存在）。
- [ ] **步骤 3：实现认证**
- `UserEntity`：`id, username(unique), passwordHash, role, isActive, createdAt`。
- `UsersService.createUser(dto: CreateUserDto): Promise<UserView>`：bcrypt cost 10 哈希；`findByUsername(username: string): Promise<User | null>`。
- `AuthService.validateUser(username: string, password: string): Promise<UserView>`：用户不存在、密码错误、`isActive=false` 均抛 `UnauthorizedException`（响应一致）。
- `POST /auth/login` → `{ token, user }`，payload `{ sub, username, role }`，过期取 `JWT_EXPIRES_IN`；`GET /auth/me` 走 `JwtAuthGuard`。
- [ ] **步骤 4：运行测试验证通过**
运行 `pnpm --filter api test:e2e -- auth`（PASS）+ build/lint 无报错。
- [ ] **步骤 5：Commit** 运行 `git add api/src api/test && git commit -m "feat(auth): add JWT login and current-user endpoint"`，然后 `git checkout develop && git merge --no-ff feature/02-auth`

### 任务 3：工单实体与状态机纯函数

**文件：**
- 创建：`api/src/tickets/ticket.entity.ts`、`ticket-event.entity.ts`
- 创建：`api/src/tickets/ticket-state.ts`、`ticket-state.spec.ts`

- [ ] **步骤 1：编写失败的测试**

```ts
// api/src/tickets/ticket-state.spec.ts
it('lets an agent claim a submitted unassigned ticket', () => {
  // evaluateAction('claim', {status:'submitted', assigneeId:null, ...}, {id:3, role:'agent'})
  // → { allowed: true, nextStatus: 'accepted' }
});
it('rejects claim by an employee', ...);                  // allowed: false
it('rejects claim when another agent already claimed', ...); // status 'accepted', assigneeId: 2, actor 3
it('lets the assigned agent start then resolve in order', ...); // nextStatus 'processing' 再 'resolved'
it('rejects resolve before processing', ...);             // 'accepted' 直接 resolve → false
it('lets the creator close own resolved ticket', ...);
it('lets an admin close any resolved ticket', ...);
it('rejects closing a processing ticket', ...);
it('rejects clearAssignee on an accepted ticket', ...);
it('canView hides other creators tickets from employees', ...);
it('canView shows unassigned submitted tickets to agents', ...);
```

- [ ] **步骤 2：运行测试验证失败**
运行 `pnpm --filter api test -- ticket-state`；预期 FAIL（模块不存在）。
- [ ] **步骤 3：实现 `ticket-state.ts`**
按「共享类型」实现 `evaluateAction` 与 `canView`；规则按设计 §6/§6.1/§8 逐条落地：`claim` 仅 agent 且（待接单或分派给自己）；`start/resolve` 仅当前处理人按序推进；`close` 仅 `resolved` 状态（本人或任意 admin）；`assign` 仅 admin 且非 `closed`；`clearAssignee` 仅 admin 且 `submitted`；`comment` 权限同 §8 可见范围。`canView`：employee 看自己创建的，agent 看自己创建的 + 待接单的 + 自己处理的，admin 全部。实体只映射表结构（`users`/`tickets`/`ticket_events`）。
- [ ] **步骤 4：运行测试验证通过**
运行 `pnpm --filter api test -- ticket-state`（PASS）+ build/lint 无报错。
- [ ] **步骤 5：Commit** 运行 `git add api/src/tickets && git commit -m "feat(tickets): add entities and pure state machine"`，然后 `git checkout develop && git merge --no-ff feature/03-ticket-state`

### 任务 4：工单创建、列表与详情

**文件：**
- 创建：`api/src/tickets/tickets.service.ts`、`tickets.controller.ts`、`dto/create-ticket.dto.ts`
- 创建：`api/test/tickets.e2e-spec.ts`

- [ ] **步骤 1：编写失败的测试**

```ts
it('create sets status submitted and writes a created event', ...);   // body.status === 'submitted'
it('create rejects empty title with 400', ...);                       // 含 101 字符标题
it('list scopes tickets by role', ...);                               // 员工只看自己的；agent 看待接单+自己的；admin 全部
it('list filters by status query', ...);
it('getOne returns 404 for a ticket the actor cannot view', ...);     // 不泄露存在性
it('getOne returns events timeline in creation order', ...);
```

- [ ] **步骤 2：运行测试验证失败**
运行 `pnpm --filter api test:e2e -- tickets`；预期 FAIL（404 路由）。
- [ ] **步骤 3：实现 `TicketsService` 与 `TicketsController`**
- `create(actor: ActorView, dto: CreateTicketDto): Promise<TicketDetail>`：写 `created` 事件；DTO 校验标题 1–100、描述 1–2000。
- `list(actor: ActorView, status?: TicketStatus): Promise<TicketSummary[]>`：用 `canView` 过滤。
- `getOne(actor: ActorView, id: number): Promise<TicketDetail>`：不可见或不存在都 404。
- 路由：`POST /tickets`、`GET /tickets?status=`、`GET /tickets/:id`，全部 `JwtAuthGuard`。
- [ ] **步骤 4：运行测试验证通过**
运行 `pnpm --filter api test:e2e -- tickets`（PASS）+ build/lint 无报错。
- [ ] **步骤 5：Commit** 运行 `git add api/src api/test && git commit -m "feat(tickets): add create, list and detail endpoints"`，然后 `git checkout develop && git merge --no-ff feature/04-ticket-crud`

### 任务 5：工单流转与备注

**文件：**
- 修改：`api/src/tickets/tickets.service.ts`、`tickets.controller.ts`
- 修改：`api/test/tickets.e2e-spec.ts`

- [ ] **步骤 1：编写失败的测试**

```ts
it('agent claims a ticket then start then resolve in order', ...);    // status: submitted→accepted→processing→resolved
it('second agent claiming the same ticket gets 409', ...);            // 并发抢占，状态不被破坏
it('illegal transition returns 409', ...);                            // submitted 直接 resolve、对 closed 操作
it('employee closes own resolved ticket', ...);
it('admin closes any resolved ticket', ...);
it('comment writes a commented event without changing status', ...);
```

- [ ] **步骤 2：运行测试验证失败**
运行 `pnpm --filter api test:e2e -- tickets`；预期 FAIL（新路由 404）。
- [ ] **步骤 3：实现流转**
`claim|start|resolve|close(actor: ActorView, id: number): Promise<TicketDetail>` 与 `addComment(actor: ActorView, id: number, content: string): Promise<TicketDetail>`：先 `evaluateAction` 判定，`allowed=false` 抛 409（不可见仍 404）；状态更新与事件写入在同一事务；claim 用条件更新（`update ... where status='submitted' and assigneeId is null`）保证后到者 409。
- [ ] **步骤 4：运行测试验证通过**
运行 `pnpm --filter api test:e2e -- tickets`（PASS）+ build/lint 无报错。
- [ ] **步骤 5：Commit** 运行 `git add api/src api/test && git commit -m "feat(tickets): add claim, progress, close and comment actions"`，然后 `git checkout develop && git merge --no-ff feature/05-ticket-actions`

### 任务 6：分派、改派与角色守卫

**文件：**
- 创建：`api/src/auth/roles.decorator.ts`、`roles.guard.ts`
- 修改：`api/src/tickets/tickets.service.ts`、`tickets.controller.ts`
- 修改：`api/test/tickets.e2e-spec.ts`

- [ ] **步骤 1：编写失败的测试**

```ts
it('admin assigns a submitted ticket to an agent', ...);              // 事件 type 'assigned'，状态不变
it('admin reassigns an accepted ticket', ...);                        // 操作权立即切换（原处理人操作 409）
it('clearAssignee only works on submitted tickets', ...);             // accepted 清空 → 409
it('assign to a non-agent or inactive user returns 400', ...);
it('non-admin calling assign gets 403', ...);
```

- [ ] **步骤 2：运行测试验证失败**
运行 `pnpm --filter api test:e2e -- tickets`；预期 FAIL。
- [ ] **步骤 3：实现 `assign`/`clearAssignee` 与守卫**
- `assign(actor: ActorView, id: number, assigneeId: number): Promise<TicketDetail>`、`clearAssignee(actor: ActorView, id: number): Promise<TicketDetail>`，规则走 `evaluateAction`。
- `RolesGuard` + `@Roles(...roles: Role[])` 装饰器；分派目标必须是 `isActive` 的 `agent`，否则 400。
- [ ] **步骤 4：运行测试验证通过**
运行 `pnpm --filter api test:e2e -- tickets`（PASS）+ build/lint 无报错。
- [ ] **步骤 5：Commit** 运行 `git add api/src api/test && git commit -m "feat(tickets): add assignment actions with role guard"`，然后 `git checkout develop && git merge --no-ff feature/06-assignment`

### 任务 7：用户管理与停用/降级保护

**文件：**
- 创建：`api/src/users/dto/create-user.dto.ts`、`update-user.dto.ts`
- 修改：`api/src/users/users.service.ts`、`users.controller.ts`
- 创建：`api/src/users/users.service.spec.ts`、`api/test/users.e2e-spec.ts`

- [ ] **步骤 1：编写失败的测试**

```ts
// users.service.spec.ts
it('deactivate rejects when the user has open assigned tickets', ...);   // ConflictException，工单保持原状
it('deactivate allows when all assigned tickets are closed', ...);
it('role change away from agent rejects with open assigned tickets', ...);
it('password reset does not touch ticket data', ...);
// users.e2e-spec.ts
it('admin lists, creates and updates users', ...);
it('non-admin managing users gets 403', ...);
```

- [ ] **步骤 2：运行测试验证失败**
运行 `pnpm --filter api test -- users` 与 `pnpm --filter api test:e2e -- users`；预期 FAIL。
- [ ] **步骤 3：实现用户管理**
- `listUsers(): Promise<UserView[]>`、`updateUser(id: number, dto: UpdateUserDto): Promise<UserView>`；`UpdateUserDto = { role?: Role; isActive?: boolean; password?: string }`。
- 保护规则：停用（`isActive=false`）或降级为非 `agent` 时，若该用户是任何非 `closed` 工单的处理人 → 409；`GET /users`、`POST /users`、`PATCH /users/:id` 全部 `@Roles('admin')`。
- [ ] **步骤 4：运行测试验证通过**
运行两个测试命令（PASS）+ build/lint 无报错。
- [ ] **步骤 5：Commit** 运行 `git add api/src api/test && git commit -m "feat(users): add admin user management with open-ticket protection"`，然后 `git checkout develop && git merge --no-ff feature/07-user-management`

### 任务 8：种子数据、全链路 e2e 与收尾

**文件：**
- 创建：`api/src/seed/seed.ts`、`api/test/lifecycle.e2e-spec.ts`
- 修改：`api/package.json`（`seed` 脚本）

- [ ] **步骤 1：编写失败的测试**

```ts
it('walks one ticket through the full lifecycle across three roles', ...);
// staff1 创建 → agent1 接单/开始/完成 → staff1 关闭；admin 改派场景再走一遍
it('seed refuses to run without SEED_DEMO_PASSWORD', ...);            // 缺环境变量即抛错，不写库
```

- [ ] **步骤 2：运行测试验证失败**
运行 `pnpm --filter api test:e2e -- lifecycle`；预期 FAIL。
- [ ] **步骤 3：实现种子与 e2e**
`seed()` 幂等创建演示账号 `admin / agent1 / staff1`（密码取 `SEED_DEMO_PASSWORD`）与 2 张示例工单；`pnpm seed` 调用它。补齐 `lifecycle.e2e-spec.ts`。
- [ ] **步骤 4：全量验证**
运行 `pnpm --filter api test && pnpm --filter api test:e2e && pnpm --filter api build && pnpm --filter api lint`；全绿后 `git checkout develop && git merge --no-ff feature/08-seed-e2e`，再按技术文档 §7 打 `release/v0.1.0`。
- [ ] **步骤 5：Commit** 运行 `git add api/src api/test api/package.json && git commit -m "feat(seed): add demo seed and full lifecycle e2e"`

