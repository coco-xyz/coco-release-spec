---
name: coco-release-spec
version: 0.1.0
description: >
  Dev-facing release engineering specification for Coco projects. Defines how Dev agents
  should prepare releases, write design documents with deployment plans, maintain config
  schemas, provide release checklists, and write deployment scripts. Use when: (1) preparing
  a new release or version tag, (2) writing or updating a design document, (3) adding or
  changing configuration items (schema.json), (4) creating release checklists or manifests,
  (5) writing deploy/upgrade/rollback scripts, (6) reviewing code repo structure for
  deployment readiness.
type: capability

lifecycle:
  npm: false
  data_dir: ~/zylos/components/coco-release-spec
  preserve:
    - config.json

upgrade:
  repo: coco-xyz/coco-release-spec
  branch: main

config:
  required: []
  optional: []

dependencies: []
---

# Dev 发布工程规范

> 提取自《开发与运维协同规范》v0.7，面向 Dev Agent 视角重写。

本文档定义你（Dev Agent）在发布、设计文档、配置 Schema、部署脚本等方面的职责与规范。Ops 侧的执行细节已省略——你只需关注"交付什么"和"怎么交付"。

---

## 1. 基本原则

以下三条原则是强制性的（MUST），违反任何一条都会导致发版流程被阻断：

### 【MUST】设计文档含部署/升级方案 + Ops 评审

你的设计文档中**必须**包含"生产部署方案"和"升级方案"章节。这些章节必须通过 Ops 评审后方可实施。不能先写代码再补方案——方案评审是开工的前提。

### 【MUST】版本变更清单作为 Dev/Ops 合约

每次发版**必须**提供完整的版本变更清单（环境变量、数据库变更、构建参数、部署顺序等），作为你和 Ops 之间的正式合约。Ops 最怕的不是操作复杂，而是信息缺失。

### 【MUST】配置 Schema 机器可读

你维护的 `config/schema.json` 是配置合约的机器可读形式。配置必须符合 schema 校验——Ops 依赖它来准确配置每个环境。

---

## 2. 代码仓库结构

你的每个服务仓库必须包含以下与部署相关的目录：

```
code-repo/
├── src/                        # 业务代码
├── config/
│   └── schema.json             # 配置合约（见第 9 节）
├── deploy/                     # 部署知识（你编写，Ops 审查）
│   ├── deploy.sh               # 首次部署 / 全新部署
│   ├── upgrade.sh              # 从上一版本升级
│   ├── rollback.sh             # 回滚到上一版本
│   ├── healthcheck.sh          # 部署后验证
│   └── docker-compose.yml      # 或 PM2 ecosystem / K8s manifest
├── docs/
│   └── design.md               # 设计文档（含部署方案和升级方案章节）
├── Dockerfile
└── CHANGELOG.md
```

**部署脚本约束：**
- 脚本只封装"怎么部署"的步骤逻辑，**不硬编码任何环境值**
- 环境值由 Ops 从部署仓库的 `config.env` / `secrets.env` 注入
- 脚本接受标准参数：`--env <环境名> --version <版本号> --config <配置文件路径>`
- 示例：`deploy.sh --env production --version 1.2.3 --config /path/to/overlays/production/`

---

## 3. 三层合约

你和 Ops 之间通过三层合约连接。**你负责编写全部三层**，Ops 负责评审和使用：

| 层次 | 解决什么问题 | 你交付的载体 |
|------|------------|-------------|
| **设计方案** | 为什么这样部署、整体架构是什么 | 设计文档中的"生产部署方案"和"升级方案"章节 |
| **配置合约** | 需要哪些配置项、类型和约束 | `config/schema.json` |
| **部署脚本** | 具体的部署/升级/回滚步骤 | `deploy/deploy.sh`、`upgrade.sh`、`rollback.sh` |

**合约成立信号：** PR 合入 = Dev 和 Ops 双方达成一致。后续变更需要新 PR 和重新评审。

**为什么用脚本而非自然语言：** Shell 脚本是确定性的、可测试的、可 diff 的——同一个输入永远产生同一个输出。自然语言有不确定性，执行结果依赖解读。

---

## 4. 设计文档要求

你的技术设计文档必须包含以下与运维相关的章节：

### "生产部署方案"章节（首次部署）

- 服务架构（哪些进程、端口、进程间依赖）
- 部署拓扑（单机 / 多机 / 容器 / 混合）
- 数据库初始化方案（schema、seed data、索引）
- 外部依赖（第三方 API、DNS 记录、证书）
- 部署顺序和依赖（先后端再前端？需要停机吗？）
- 监控指标和健康检查端点
- 回滚方案

### "升级方案"章节（已有生产版本的迭代）

- 支持的升级路径（从哪些版本可以直升，哪些需要中间步骤）
- DB migration 步骤和回滚方式
- 是否需要停机（能否滚动升级）
- 配置项的增删改（与 schema `config_changes` 对应）
- 对在线用户的影响评估
- 升级失败的回滚路径

---

## 5. 评审流程

```
你编写设计文档（含部署/升级方案 + deploy/ 脚本）
    │
    ▼
提交 PR 到代码仓库
    │
    ▼
Ops 评审 ◄── Ops 关注：
    │         - 方案在当前基础设施上是否可行
    │         - 对在线用户的影响
    │         - 监控和告警覆盖是否充分
    │         - 回滚路径是否可靠
    │         - 部署脚本是否安全
    │
    ├── 有意见 → 你修改 → Ops 重新评审
    │
    ▼
PR 合入 = Dev/Ops 合约成立
```

**你的职责：**
- 提交完整的设计文档（含部署/升级方案章节）和 `deploy/` 脚本
- 响应 Ops 的评审意见并修改
- PR 合入后，后续变更需新 PR

**触发重新评审的情况：**
- 部署方式变更（如从 PM2 迁移到 Docker、从单机到集群）
- 新增外部依赖（如新增 Redis、新增第三方 API）
- 数据库 schema 变更（涉及 migration）
- 部署脚本的逻辑变更

---

## 6. 部署脚本规范

你在代码仓库的 `deploy/` 目录下维护以下脚本：

| 脚本 | 用途 | 调用时机 |
|------|------|----------|
| `deploy.sh` | 全新部署（从零开始） | 首次上线或新环境搭建 |
| `upgrade.sh` | 从上一版本升级 | 常规版本迭代 |
| `rollback.sh` | 回滚到指定版本 | 部署失败或紧急回退 |
| `healthcheck.sh` | 部署后验证 | 每次部署/升级后 |

### 脚本接口标准

```bash
# 所有脚本接受统一参数格式
deploy.sh   --env <环境> --version <版本> --config <配置目录>
upgrade.sh  --env <环境> --from <旧版本> --to <新版本> --config <配置目录>
rollback.sh --env <环境> --to <目标版本> --config <配置目录>
healthcheck.sh --env <环境> --version <版本>
```

### 脚本质量要求

- **幂等性**：重复执行不产生副作用
- **错误处理**：任何步骤失败立即退出并报告
- **日志输出**：关键步骤输出可读日志
- **无硬编码**：环境值全部从 `--config` 目录注入
- **可测试**：在 test 环境可完整运行

---

## 7. 版本变更清单

每次发版时，你**必须**提供一份版本变更清单。清单可以是 PR description、独立 markdown 文件或固定模板，但必须包含以下内容。

### 7.1 必填项

**环境变量变更表**

| 操作 | 变量名 | 服务 | 值/说明 | 密钥？ | 构建时？ |
|------|--------|------|---------|--------|---------|
| 新增 | `ROUTE_DOMAIN` | api, web | 路由域名（如 `aidep.online`） | 否 | 是（web） |
| 修改 | `STRIPE_PRICE_BASIC_MONTHLY` → `STRIPE_PRICE_AIR_MONTHLY` | api | 套餐名变更 | 否 | 否 |
| 删除 | `OLD_WEBHOOK_URL` | api | 已迁移至 `CLAWMARK_WEBHOOK_URL` | 否 | 否 |

**数据库变更**

| 变更 | 文件/语句 | 执行时机 | 可回滚？ |
|------|----------|----------|---------|
| 新增 `enterprise_plans` 表 | `prisma/migrations/20260308_add_enterprise_plans/` | 部署代码前 | 是（drop table） |
| 新增 `route_domain` 字段 | 同上 | 部署代码前 | 是（alter table drop column） |

**构建参数变更**（构建时注入，运行时不可修改）

| 变量名 | 旧值 | 新值 |
|--------|------|------|
| `NEXT_PUBLIC_ROUTE_DOMAIN` | （新增） | `aidep.online` |

**部署顺序**

```
1. DB migration（prisma migrate deploy）
2. API 服务部署（新版本依赖新表）
3. Web 服务重新构建 + 部署（构建参数变更）
4. VM Orchestrator 部署（如有变更）
```

是否需要停机窗口：否 / 是（原因：___）

### 7.2 建议项

**基础设施变更**
- 新增 Secret Manager secret：`VM_ORCH_PROVISION_SERVICE_TOKEN`
- 新增 Cloud Scheduler job：无
- GCP 权限变更：无

**验证方法**
- `GET /healthz` 返回 200
- `GET /healthz/config` 无 missing 项
- Dashboard 首页正常加载，套餐页显示新套餐名

**已知风险和回滚方案**
- DB migration 可回滚（drop table）
- 如 Web 构建参数有误，需要重新构建镜像（约 5 分钟）
- API 可直接回退到上一个 Cloud Run revision

### 7.3 模板位置

版本变更清单模板存放在代码仓库 `docs/templates/release-checklist.md`。发版时复制模板填写，附在 GitHub Release 或 develop→main PR description 中。

---

## 8. 发布流程

**发布 = Git Tag + 制品 + 发布清单**

你的发布步骤：

1. 代码就绪，测试通过
2. 如配置项有变更，更新 `config/schema.json`
3. 如部署方式有变更，更新 `deploy/` 脚本和设计文档中的部署/升级方案
4. 更新 CHANGELOG.md
5. 创建 Git Tag：`v1.2.3`
6. CI 自动触发：
   - 构建 Docker 镜像 → 推送到镜像仓库（`ghcr.io/org/app:1.2.3`）
   - 构建前端制品 → 上传到制品存储
   - 生成 `release-manifest.json` → 附加到 GitHub Release
7. CI 向部署仓库提交 PR：
   - 更新 `apps/{service}/base/desired-version.json` → `"1.2.3"`
   - 从代码仓库同步 `apps/{service}/base/schema.json`
   - 从代码仓库同步 `deploy/` 脚本到 `apps/{service}/base/deploy/`

**你的职责到"制品发布 + 清单附加 + 部署脚本同步"为止。不执行任何部署操作。**

### release-manifest.json

```json
{
  "app": "clawmark",
  "version": "1.2.3",
  "released_at": "2026-03-06T10:00:00Z",
  "artifacts": {
    "server": {"type": "docker", "image": "ghcr.io/coco-xyz/clawmark:1.2.3"},
    "dashboard": {"type": "static", "url": "https://github.com/.../releases/download/v1.2.3/dashboard-dist.tar.gz"}
  },
  "config_schema_version": "1.2.3",
  "config_changes": [
    {"action": "added", "key": "GEMINI_API_KEY", "required": true, "type": "string", "description": "Gemini API key for AI features"},
    {"action": "deprecated", "key": "OLD_WEBHOOK_URL", "migrate_to": "CLAWMARK_WEBHOOK_URL", "remove_in": "2.0.0"}
  ],
  "deployment_changes": [
    {"description": "新增 Redis 依赖，deploy.sh 已更新", "requires_ops_review": true}
  ],
  "breaking_changes": [],
  "min_compatible_config_version": "1.0.0"
}
```

**关键字段说明：**
- `config_changes`：列出本次发版的配置项变更。Ops 依赖此字段更新环境配置。
- `deployment_changes`：列出部署方式变更。标记 `requires_ops_review: true` 时，Ops 需重新评审部署脚本。
- `breaking_changes`：破坏性变更列表。存在时需要人工审批才能部署。

---

## 9. 配置 Schema 规范

`config/schema.json` 是三层合约中的配置合约层。它是你（"我的应用需要什么配置"）和 Ops（"每个环境提供什么值"）之间的机器可读桥梁。

### 9.1 设计原则

1. **你生产，Ops 消费。** Schema 由你在代码仓库中编写，每次发版时同步到部署仓库。
2. **声明式，非命令式。** Schema 声明约束；校验工具执行约束。
3. **向后兼容演进。** 可以自由添加新项。删除必填项是破坏性变更，必须遵循弃用周期。
4. **多服务感知。** 单个应用可能包含多个服务。每个配置项声明由哪个服务消费。
5. **密钥/非密钥分离。** Schema 声明哪些项是密钥。Ops 据此将值路由到 `secrets.env.enc` 或 `config.env`。

### 9.2 Schema 格式

根文档：

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://coco.xyz/schemas/config/{app-name}/{version}",
  "title": "应用可读名称",
  "description": "应用功能描述",
  "type": "object",
  "schemaVersion": "1.0.0",
  "appVersion": "1.6.0",
  "services": ["api", "web", "worker"],
  "items": [ ... ]
}
```

**根级字段：**

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `$schema` | string | 是 | 始终为 `https://json-schema.org/draft/2020-12/schema` |
| `$id` | string | 是 | 唯一 schema URI：`https://{org}/schemas/config/{app}/{version}` |
| `title` | string | 是 | 应用可读名称 |
| `description` | string | 是 | 应用功能描述 |
| `schemaVersion` | string | 是 | Schema 格式本身的版本（当前 = `1.0.0`） |
| `appVersion` | string | 是 | 此 schema 对应的应用版本 |
| `services` | string[] | 是 | 消费此 schema 配置的服务名称列表 |
| `items` | object[] | 是 | 配置项声明数组 |

### 9.3 配置项声明

`items` 中的每个元素声明一个环境变量/配置键：

```json
{
  "key": "DATABASE_URL",
  "type": "string",
  "description": "Prisma ORM 使用的 PostgreSQL 连接字符串",
  "required": true,
  "secret": true,
  "category": "database",
  "service": "api",
  "constraints": {
    "pattern": "^postgresql://.+"
  },
  "addedInVersion": "1.0.0"
}
```

**配置项字段：**

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `key` | string | 是 | 环境变量名。必须为 `UPPER_SNAKE_CASE`。 |
| `type` | enum | 是 | 值类型：`string`、`number`、`boolean`、`url`、`enum`、`json` |
| `description` | string | 是 | 配置项的可读用途说明 |
| `required` | boolean | 是 | 应用启动是否必须有此值 |
| `secret` | boolean | 是 | 值是否敏感。密钥进 `secrets.env.enc`，不进 `config.env`。 |
| `default` | any | 否 | 默认值。**密钥不得设置默认值。** |
| `constraints` | object | 否 | 校验规则（见下文） |
| `category` | string | 是 | 逻辑分组：`database`、`auth`、`billing`、`server`、`queue`、`security`、`feature_flags`、`frontend` 等 |
| `service` | string | 是 | 哪个服务消费此项。必须是根级 `services[]` 中的值。 |
| `buildTime` | boolean | 否 | 如为 `true`，此值在构建时烘入制品（如 `NEXT_PUBLIC_*`）。变更需要重新构建。 |
| `addedInVersion` | string | 是 | 引入此项的发布版本（semver） |
| `deprecatedInVersion` | string | 否 | 弃用此项的发布版本。项仍可用但将在未来主版本中移除。 |

### 9.4 约束对象

| 约束 | 适用类型 | 描述 | 示例 |
|------|----------|------|------|
| `pattern` | string、url | 值必须匹配的正则表达式 | `"^sk_(live\|test)_.+"` |
| `minLength` | string | 最小字符长度 | `32` |
| `maxLength` | string | 最大字符长度 | `256` |
| `min` | number | 最小数值（含） | `1` |
| `max` | number | 最大数值（含） | `65535` |
| `enum` | enum | 允许值列表 | `["debug", "info", "warn", "error"]` |

同一项上的多个约束为 AND 关系。

### 9.5 类型语义

| 类型 | 值格式 | 校验 |
|------|--------|------|
| `string` | 任意字符串 | 检查 pattern/length 约束 |
| `number` | 整数或小数 | 必须可解析为数字；检查 min/max |
| `boolean` | `true` 或 `false` | 必须严格为这两个字符串之一 |
| `url` | 有效 URL | 必须可解析为带 scheme 的 URL |
| `enum` | 允许值之一 | 必须匹配 `constraints.enum` 中的值 |
| `json` | 有效 JSON 字符串 | 必须可解析为有效 JSON |

### 9.6 Schema 演进规则

| 变更类型 | 向后兼容？ | 你需要做什么 |
|----------|-----------|-------------|
| 新增带默认值的可选项 | 是 | 添加到 schema 即可 |
| 新增必填项 | **否**（破坏性） | 在 `release-manifest.json` 的 `config_changes` 中标记 `action: "added"` |
| 变更默认值 | 是 | 更新 schema |
| 为已有项添加约束 | 可能破坏 | 通知 Ops 验证已有值 |
| 弃用项 | 是 | 设置 `deprecatedInVersion` |
| 删除项 | **否**（破坏性） | 必须已弃用至少一个 minor 版本。需要 major 版本升级。 |
| 变更类型 | **否**（破坏性） | 需要 major 版本升级 |

---

## 10. 应用编码规范

### 10.1 配置集中管理

所有配置项在单一入口文件中加载和校验：

```javascript
// config/index.js — 唯一的配置入口
const config = {
  port: env('CLAWMARK_PORT', 3458),
  jwtSecret: env('CLAWMARK_JWT_SECRET'),
  googleClientId: env('CLAWMARK_GOOGLE_CLIENT_ID'),
  geminiApiKey: env('GEMINI_API_KEY', null),
};

validate(config);
module.exports = config;
```

**业务代码中不得直接使用 `process.env.XXX`。** 所有配置通过 config 模块访问。

### 10.2 配置命名规范

```
<应用前缀>_<类别>_<名称>

示例：
CLAWMARK_PORT
CLAWMARK_GOOGLE_CLIENT_ID
CLAWMARK_JWT_SECRET
CLAWMARK_DATA_DIR
```

- 全大写 + 下划线
- 必须有应用前缀（避免跨服务冲突）
- 前端构建时变量使用框架前缀（`VITE_`、`NEXT_PUBLIC_` 等）

### 10.3 schema.json 规范

每个字段必须有：
- `type`：string、integer、url、boolean
- `required`：true/false
- `description`：可读的用途说明
- `secret`：如敏感则为 true
- `build_time`：如构建时注入则为 true
- `default`：可选项的默认值

Schema 版本跟随应用版本。

### 10.4 配置变更纪律

- 新增配置项 → 同时更新 `schema.json` + `config/index.js` + `CHANGELOG`
- 新增必填配置项 = **破坏性变更** → 在 release-manifest 的 `config_changes` 中标记
- 删除配置 → 至少弃用一个版本后再移除
- 所有可选配置项必须有合理的默认值
- 不得在源代码中硬编码环境特定的值

### 10.5 健康检查端点

你的应用必须暴露以下端点：

```
GET /healthz        → 简单存活检查
GET /healthz/ready  → 依赖就绪检查（DB、外部 API）
GET /healthz/config → 配置自检（每项的配置/缺失状态，不暴露值）
```

Ops 在部署后通过这些端点验证应用状态。`/healthz/config` 尤其重要——它让 Ops 立即知道哪些配置项缺失或异常。

### 10.6 前端构建时配置

前端配置通过构建时注入（`VITE_xxx`、`NEXT_PUBLIC_xxx`）：
- 必须在 `schema.json` 中标记 `"buildTime": true`
- CI 在构建时从部署仓库读取环境配置
- 构建制品与环境绑定
- 替代方案：运行时配置注入（服务端生成 `/config.js`）实现单次构建多环境部署
