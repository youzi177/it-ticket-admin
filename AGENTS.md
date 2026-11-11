# AGENTS.md

项目：`it-ticket-admin` —— 内部 IT 报修工单 admin 系统（Vue 3 + NestJS + MySQL），用于验证 AI 开发流程。

## 先读文档（doc/）

- `2026-11-11-it-ticket-admin-design.md`：需求与设计（目标/非目标/状态机/权限矩阵），一切行为以此为准
- `2026-11-11-it-ticket-admin-tech-init.md`：技术选型、目录结构、初始化命令、环境变量、Git Flow
- `2026-11-11-backend-plan.md`、`2026-11-11-frontend-plan.md`：实现计划（TDD 任务分解），动手前先读对应计划
- `api/AGENTS.md`：NestJS 12 专属约定（ESM、相对导入带 `.js` 后缀、Vitest 用 `vi.fn()`）

## 常用命令

| 命令 | 作用 |
|---|---|
| `pnpm install` | 安装全部依赖（workspace） |
| `pnpm build` | 构建 api + web |
| `pnpm lint` | oxlint 检查 api + web |
| `pnpm test` | 单元测试 api + web |
| `pnpm test:api:e2e` | api e2e 测试 |
| `pnpm dev:api` / `pnpm dev:web` | 启动开发服务 |
| `docker compose up -d` | 启动本地 MySQL 8 |

## 技术栈

- api：NestJS 12 + TypeORM + MySQL 8；ESM（相对导入带 `.js` 后缀）；测试 Vitest + supertest；lint oxlint
- web：Vue 3 + Vite + TypeScript + Pinia + vue-router + Element Plus；测试 Vitest + @vue/test-utils
- 认证：@nestjs/jwt + passport-jwt + bcrypt

## 工程约定

- 目录 kebab-case、组件 PascalCase、函数 camelCase；命名导出优于默认导出；`const` 优于 `let`，禁止 `var`
- early return，减少嵌套；单文件 ≤ 300 行；注释只解释「为什么」，代码自解释
- 界面文案中文，代码和注释英文
- 密钥一律走环境变量（`api/.env`、`web/.env`，模板见 `.env.example`），禁止硬编码或提交
- 修改后立即验证：`pnpm build && pnpm lint && pnpm test` 全绿才提交；修复 bug 先写重现测试

## Git Flow

- 常驻分支：`main`（随时可发布）、`develop`（集成）；任务 `feature/*`、发布 `release/*`、修复 `hotfix/*`
- 每个任务从 `develop` 切 `feature/<task-slug>`，完成后 `git merge --no-ff` 回 `develop`
- 提交用 Conventional Commits，小步提交；禁止 `--no-verify`；未经明确要求不 push

## 国内网络环境（重要）

- npm 走 npmmirror（`.npmrc` 已配置）；Docker 镜像优先 `docker.m.daocloud.io`，拉取后 `docker tag` 回原镜像名
- 两者都慢时走本地代理：`HTTP_PROXY=http://127.0.0.1:10808 HTTPS_PROXY=http://127.0.0.1:10808 pnpm ...`（代理只对当前命令生效）
- 不修改系统级配置；项目级配置写入 `.npmrc` / `docker-compose.yml`
- pnpm 12 默认禁止依赖构建脚本：报 `ERR_PNPM_IGNORED_BUILDS` 时在 `pnpm-workspace.yaml` 的 `allowBuilds` 加映射（如 `bcrypt: true`），再 `pnpm install`

## 安全底线

- 禁止删除或修改 `.git`、`.env`、`node_modules`、`.pnpm-store`；禁止 `rm -rf`、`git reset --hard`、`git push --force`
- 禁止提交密钥、凭证、token 等敏感信息
