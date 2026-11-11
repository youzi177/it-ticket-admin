# it-ticket-admin 技术初始化文档

用途：从零初始化项目仓库。命令按顺序执行即可得到可运行、可测试的骨架。设计依据见 `doc/2026-11-11-it-ticket-admin-design.md`。

## 1. 环境要求

- Node.js >= 20，pnpm >= 9
- MySQL 8.0（本地用 Docker 启动）
- Git（Git Flow 分支模型）

## 2. 技术选型（我的决定）

| 位置 | 选型 | 版本下限 |
|---|---|---|
| 后端 | NestJS + TypeScript | NestJS >= 10，TS >= 5.4 |
| ORM | TypeORM + mysql2 | TypeORM >= 0.3 |
| 认证 | @nestjs/jwt + passport-jwt + bcrypt | — |
| 校验 | class-validator + class-transformer | — |
| 后端测试 | Jest + supertest | — |
| 前端 | Vue 3 + Vite + TypeScript | Vue >= 3.4，Vite >= 6 |
| 状态/路由 | Pinia + vue-router | Pinia >= 3 |
| UI 组件库 | Element Plus | >= 2.8 |
| 前端测试 | Vitest + @vue/test-utils + jsdom | — |
| 规范 | ESLint 9（typescript-eslint + eslint-plugin-vue）+ Prettier 3 | — |

工程形态：pnpm workspace 单仓库，`api/` 与 `web/` 两个包（设计文档第 11 节留给建造者决定的项）。

## 3. 目录结构

```text
it-ticket-admin/
├── api/                        # NestJS 后端
│   ├── src/
│   │   ├── main.ts
│   │   ├── app.module.ts
│   │   ├── health.controller.ts
│   │   ├── auth/               # 登录、JWT、守卫
│   │   ├── users/              # 用户实体、用户管理
│   │   ├── tickets/            # 工单实体、状态机、流转 API
│   │   └── seed/               # 演示数据
│   ├── test/                   # e2e 测试
│   ├── .env.example
│   └── package.json
├── web/                        # Vue 3 前端
│   ├── src/
│   │   ├── api/                # fetch 封装与接口
│   │   ├── components/         # PascalCase 组件
│   │   ├── router/
│   │   ├── stores/             # Pinia
│   │   ├── types/
│   │   ├── utils/              # 纯函数（权限判定）
│   │   └── views/              # 页面
│   ├── .env.example
│   └── package.json
├── doc/                        # 全部 Markdown 文档
├── docker-compose.yml          # 本地 MySQL 8
├── pnpm-workspace.yaml
├── package.json
└── .gitignore
```

## 4. 初始化命令

```bash
# 0. 仓库与工作区
mkdir it-ticket-admin && cd it-ticket-admin
git init -b main
mkdir -p doc
pnpm init                       # 根 package.json 加 "private": true
# pnpm-workspace.yaml 内容：packages: [api, web]
# .gitignore 内容：node_modules、dist、coverage、.env、*.log、.DS_Store

# 1. 后端脚手架
pnpm dlx @nestjs/cli@latest new api --package-manager pnpm --skip-git --strict
cd api
pnpm add @nestjs/config @nestjs/jwt @nestjs/passport passport passport-jwt bcrypt typeorm mysql2 class-validator class-transformer
pnpm add -D @types/passport-jwt @types/bcrypt
cd ..

# 2. 前端脚手架
pnpm create vite web --template vue-ts
cd web
pnpm add vue-router pinia element-plus @element-plus/icons-vue
pnpm add -D vitest @vue/test-utils jsdom
cd ..

# 3. 数据库
docker compose up -d            # 启动本地 MySQL 8（配置见第 5 节）
```

根 `package.json` scripts（npm 包名用 `it-ticket-admin`）：

```json
{
  "dev:api": "pnpm --filter api start:dev",
  "dev:web": "pnpm --filter web dev",
  "build": "pnpm --filter api build && pnpm --filter web build",
  "lint": "pnpm --filter api lint && pnpm --filter web lint",
  "test": "pnpm --filter api test && pnpm --filter web test",
  "test:api:e2e": "pnpm --filter api test:e2e",
  "seed": "pnpm --filter api seed"
}
```

## 5. 数据库（docker-compose.yml）

```yaml
services:
  mysql:
    image: mysql:8.0
    ports: ["3306:3306"]
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:-local-dev-only}
      MYSQL_DATABASE: it_ticket_admin
      MYSQL_USER: it_ticket
      MYSQL_PASSWORD: ${MYSQL_PASSWORD:-local-dev-only}
    volumes: [mysql_data:/var/lib/mysql]
volumes:
  mysql_data:
```

以上口令仅为本地开发占位符，真实值一律放 `.env`，禁止提交（`.gitignore` 必须包含 `.env`）。

## 6. 环境变量

`api/.env.example`（TypeORM 启动时读取，不写死在代码里）：

```bash
DB_HOST=127.0.0.1
DB_PORT=3306
DB_USERNAME=it_ticket
DB_PASSWORD=local-dev-only
DB_DATABASE=it_ticket_admin
JWT_SECRET=replace-me-outside-dev
JWT_EXPIRES_IN=2h
API_PORT=3000
SEED_DEMO_PASSWORD=change-me-demo
```

`web/.env.example`：

```bash
VITE_API_BASE_URL=http://localhost:3000
```

## 7. Git Flow 约定

- 常驻分支：`main`（随时可发布）、`develop`（集成）。
- 每个任务一条 `feature/<task-name>`，从 `develop` 切出，完成后 `git merge --no-ff` 回 `develop`。
- 发布：`release/vX.Y.Z` 从 `develop` 切出，验收后合入 `main` 打 tag，再回灌 `develop`。
- 线上修复：`hotfix/<name>` 从 `main` 切出，修完合回 `main` 与 `develop`。
- 提交用 Conventional Commits；禁止 `--no-verify`；未经明确要求不 push。

## 8. 工程约定（来自 AGENTS.md，全部任务适用）

- 目录 kebab-case、组件 PascalCase、函数 camelCase；命名导出优于默认导出；`const` 优于 `let`，禁止 `var`。
- early return，减少嵌套；单文件 ≤ 300 行；注释只解释「为什么」。
- 密钥统一走环境变量，禁止硬编码；禁止提交密钥、凭证、token。
- 每次修改后验证：`pnpm build` + `pnpm lint` 无报错，测试通过后才提交。

## 9. 国内镜像与代理（用户环境补充）

- npm 走 npmmirror：根目录 `.npmrc` 写入 `registry=https://registry.npmmirror.com`。
- Docker 镜像拉不动时换镜像源：`docker pull docker.m.daocloud.io/library/mysql:8.0`，再 `docker tag docker.m.daocloud.io/library/mysql:8.0 mysql:8.0`，compose 文件不用改。
- 镜像源也慢时走本地代理：`HTTP_PROXY=http://127.0.0.1:10808 HTTPS_PROXY=http://127.0.0.1:10808 pnpm install`（代理只对当前命令生效）。
- pnpm 12 默认拒绝依赖构建脚本，报 `ERR_PNPM_IGNORED_BUILDS` 时在 `pnpm-workspace.yaml` 的 `allowBuilds` 加映射（如 `bcrypt: true`），再 `pnpm install`。

