<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# AGENTS.md — Auto-Ontology 代理入口

这里收录教会代理如何**搭建**当前 Auto-Ontology 实现、并**使用**其本体层（ontology layer）的技能。它们位于仓库根目录的 **`skills/`** 下，每个技能一个目录，统一使用固定的 `nvidia-ontology*` 命名空间。

本文件面向将 Auto-Ontology 跑起来的开发者，以及把它嵌入代理框架（agent harness）的合作伙伴。Auto-Ontology 的**维护者**内容（构建、测试、schema、前端约定）见 [`CLAUDE.md`](./CLAUDE.md)；不要把这些约定混入技能文档。

## 开发命令

后端命令在仓库根目录执行；依赖由 `uv` 管理（用 `uv add` 加依赖，不要直接改 `pyproject.toml`）：

| 任务 | 命令 |
| --- | --- |
| 启动 FastAPI（热重载，:3001） | `uv run uvicorn auto_ontology.server.__main__:create_app --factory --reload --host 127.0.0.1 --port 3001` |
| 启动 ingestion 服务（:3002） | `uv run python -m auto_ontology.ingestion_service` |
| 运行全部后端测试 | `uv run pytest auto_ontology dev_tools -q` |
| 运行单个测试 | `uv run pytest auto_ontology/tests/test_foo.py::test_name -q` |
| Lint + 格式检查 | `uv run ruff check auto_ontology/ && uv run ruff format --check auto_ontology/` |
| 应用迁移 | `make migrate` |
| 自动生成迁移 | `make migrate-revision m="what changed"` |
| 检查 schema 漂移 | `make migrate-check` |
| 重新生成已提交的 OpenAPI 规范 | `uv run python -m dev_tools.generate_backend_openapi` |

前端在 `frontend/` 目录（pnpm）下运行：`pnpm lint`、`pnpm build`、`pnpm format`（Prettier —— 每次编辑前端文件后都要执行；CI 会拒绝未格式化的代码）。只要某个路由的代码**或 docstring** 发生变化，就必须重新生成 OpenAPI 规范 —— 已提交的规范内嵌了 docstring，规范过期会导致 CI 失败。

应用工厂是 `auto_ontology/server/__main__.py` 中的 `create_app()` —— 项目里**没有** `auto_ontology/server/main.py`。

## 架构

全栈组成：Next.js 前端（`frontend/`）+ FastAPI 后端（`auto_ontology/`）+ 独立的 ingestion 服务和 MCP 服务器，全部以 Postgres 17 + pgvector 作为底座。

- **前端（:3000）是认证与网关边界。** Better Auth + Prisma 拥有 Postgres 的 `public` schema。前端通过 Next.js rewrites 代理到后端。
- **后端（:3001）。** `server/<feature>/router.py` 存放路由（全部在 `/api/` 前缀下），`server/<feature>/service.py` 负责业务编排。后端信任前端转发的 `x-auto-ontology-user-id` 身份头 —— 3001/3002 端口必须保持在私有网络内。
- **DAL。** `auto_ontology/dal/` 使用 SQLAlchemy Core，不用 ORM。`dal/schema.py` 是 schema 的唯一事实来源（后端拥有 Postgres 的 `auto_ontology` schema；Prisma 拥有 `public` —— 两者相互独立、分别迁移）。schema 变更只能通过 Alembic autogenerate 进行。
- **连接器**（`auto_ontology/connectors/`）适配十种源引擎（Postgres、MySQL、Snowflake、Databricks、ClickHouse、Trino、Kyuubi、HeavyDB、DuckDB、SQLite）；DuckDB/SQLite 仅支持连接字符串方式。
- **Ingestion 服务**（`auto_ontology/ingestion_service/`，:3002）是第二个 FastAPI 应用：读取源数据、抽取/画像，并执行语义编译。它的 ingest/delete/reset/compile 端点没有认证。
- **本体与检索。** 本体由三部分组成：terms（映射到列的业务概念）、SQL attributes（经认证的表达式）和 relationships（join / 桥接表），存储在 Postgres 中，pgvector 存放嵌入向量（`retrieval/`、`vdb/`）。问答流程：意图分类 → 嵌入检索候选 terms/列/SQL attributes → 生成 SQL → 对过滤值和查询做校验、失败则修复 → 执行并返回答案 + SQL + 步骤。若候选内容不足以支撑问题，则直接报告“无法回答”。
- **模型角色**通过环境变量三元组配置（`<PREFIX>_ENDPOINT/_API_KEY/_MODEL`，前缀为 `REASONING`、`NON_REASONING`、`EMBED`、`RERANK`），未设置的字段回退到 `DEFAULT_MODELS_*`；ingest 与 query 必须使用相同的嵌入模型。
- **MCP 服务器**（`mcp/`，FastMCP，:3003）是**独立项目**，有自己独立的 venv 和 CI —— 它被排除在后端 pytest 套件之外，且不得导入后端代码。它只暴露对公共 API 的只读工具；写操作只能走 Next.js 的 `/api/...` 接口。

## 应读哪个技能

- [`skills/nvidia-ontology-setup/`](./skills/nvidia-ontology-setup/) — Helm（官方安装方式）、本地 Compose、开发者 `--dev` / `--ds`、对已运行实例使用 MCP，以及故障排查。当 Auto-Ontology 尚未启动或启动过程中出错时，从这里开始。
- [`skills/nvidia-ontology-management/`](./skills/nvidia-ontology-management/) — 检查、建模、**修改**并通过术语表（glossary terms）、SQL attributes、语义关系、模型 YAML 导入/导出、受治理的结果定义进行安全发布。MCP 不能写；本技能覆盖所有已批准的写路径。
- [`skills/nvidia-ontology-query/`](./skills/nvidia-ontology-query/) — 代理如何调用 Auto-Ontology（优先 MCP，REST 兜底）、如何通过语义层发现含义，以及如何校验 SQL、行结果和答案。当 Auto-Ontology 已部署时，从这里开始。

## 工作流归属与交接

| 用户动作 | 负责技能 |
| --- | --- |
| 安装、配置、连接、ingest、编译、验证就绪 | `nvidia-ontology-setup` |
| 查看含义与语义关系 | `nvidia-ontology-management` |
| 设计概念、映射、关系、度量、单位、粒度和策略 | `nvidia-ontology-management` → `references/modeling.md` |
| 应用、验证、如实认证、发布、回滚 | `nvidia-ontology-management` → `references/publication.md` |
| 提出描述性或诊断性问题，并校验 SQL、行结果和答案文本 | `nvidia-ontology-query` → `references/query-validation.md` |

组合起来的生命周期是：安装/连接/就绪验证 → 检查/建模 → 分阶段应用/读回 → 查询校验 → 受治理的发布 → 全新查询读回。某一步若没有产品支持，必须返回明确的缺口或回执，而不是虚构一个操作。认证、权限、来源（provenance）和错误处理是各技能共享的关注点，而不是各自重复实现的部分。

## 事实来源（不要重复）

技能文档只承载**工作流、顺序和坑**，不得变成第二份 API 目录或第二份安装器。

| 职责 | 归属 | 技能文档应做的 |
| --- | --- | --- |
| 安装命令 | `DEPLOYMENT.md`（Helm，官方）、`docker-compose.yml`；开发者工作流见 `dev_tools/setup_env.sh` | 调用它们；不要重写安装器 |
| 在线读取工具 | `mcp/auto_ontology_mcp/tools.py`（握手时返回的工具列表） | 优先 MCP；编排读取顺序；不要照抄工具表 |
| REST 形状与权限 | `docs/openapi/auto-ontology-api.json` | 指出操作名和所需权限并附链接；不要粘贴 schema |

MCP 设计上就是只读的。写操作走 Next.js 的 `/api/...` 接口。若握手时返回的实时 MCP 工具列表与技能文档不一致，以实时列表为准。

## 目录镜像（Catalog mirror）

以下目录会同步到 [github.com/nvidia/skills](https://github.com/nvidia/skills)（`npx skills add nvidia/skills`）和 build.nvidia.com/skills。该流水线要求每个技能具备：

- `SKILL.md`、`skill-card.md`、`evals/evals.json`（在本仓库维护）
- `BENCHMARK.md` 和 `skill.oms.sig` — 由 NVSkills CI 生成。维护者需要在任何改动 `skills/` 的 PR 下评论 `/nvskills-ci`；`nv-skills-ci[bot]` 会推送一个 "Attach NVSkills validation signatures" 提交，且该提交必须保留在 PR 中。绝不能手改或伪造这些文件；同一 PR 内技能后续再有改动，必须重跑 NVSkills CI。
- **nvidia/skills** 仓库中的 `components.d/auto-ontology.yml` 条目，用于注册：

```yaml
name: Auto-Ontology
repo: NVIDIA/auto-ontology
description: Set up Auto-Ontology, model and publish its semantic layer, and run validated text-to-SQL queries through it.
skills:
  - path: skills/nvidia-ontology-setup/
    catalog_dir: nvidia-ontology-setup
  - path: skills/nvidia-ontology-management/
    catalog_dir: nvidia-ontology-management
  - path: skills/nvidia-ontology-query/
    catalog_dir: nvidia-ontology-query
```

技能文档必须避免包含 staging 主机名、集群名、Vault 路径和任何凭据 —— 即使 Auto-Ontology 本身有访问限制，目录（catalog）也是公开的。
