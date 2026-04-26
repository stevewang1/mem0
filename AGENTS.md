# AGENTS.md

Mem0 仓库的 AI 编码助手上下文指南。

## Monorepo 结构

多语言 monorepo — Python 和 TypeScript 包各自拥有独立的构建系统、linter 和 CI 流水线。

| 目录 | 包名 | 发布源 | 说明 |
|------|------|--------|------|
| `mem0/` | `mem0ai` | PyPI | 核心 Python SDK — memory、LLMs、embeddings、vector stores、rerankers |
| `mem0-ts/` | `mem0ai` | npm | TypeScript SDK — `src/client/`（托管版）+ `src/oss/`（自托管版） |
| `cli/python/` | `mem0-cli` | PyPI | Typer CLI，入口 `mem0 = "mem0_cli.app:main"`，源码在 `src/mem0_cli/` |
| `cli/node/` | `@mem0/cli` | npm | Commander CLI |
| `vercel-ai-sdk/` | `@mem0/vercel-ai-provider` | npm | Vercel AI SDK memory provider |
| `openclaw/` | `@mem0/openclaw-mem0` | npm | OpenClaw 插件（Claude Code / AI 编辑器） |
| `server/` | — | Docker | FastAPI + PostgreSQL/pgvector + Neo4j（自托管 REST 服务器） |
| `openmemory/` | — | Docker | 自托管平台 — `api/`（FastAPI + Alembic + MCP）+ `ui/`（Next.js 15） |
| `embedchain/` | — | PyPI（独立） | 遗留 RAG 框架 — **有自己的 Poetry 构建系统，不要混用** |
| `docs/` | — | Mintlify | 文档站点；API 规范在 `docs/openapi.json` |
| `evaluation/` | — | — | 基准测试框架（LOCOMO evals） |

## 环境要求

- **Python**: 3.10+（3.9 **不支持** — `pyproject.toml` 写的是 `>=3.10,<4.0`；CI 测 3.10、3.11、3.12）
- **Node.js**: v18+（CI 测 20、22）
- **pnpm**: v10+ — **所有** TypeScript 包统一使用（禁用 npm / yarn）
- **Hatch**: Python 构建/环境工具（`pip install hatch`）— **不用** pip 或 conda 管理依赖
- **Docker**: `server/` 和 `openmemory/` 开发必需

## 常用命令

### Python SDK (`mem0/`)

```bash
hatch shell dev_py_3_11        # 激活环境（3.10、3.11、3.12 均可）
make lint                      # ruff check
make format                    # ruff format
make sort                      # isort mem0/
make test                      # pytest tests/
make test-py-3.12              # 指定 Python 版本测试
```

### TypeScript SDK (`mem0-ts/`)

```bash
cd mem0-ts && pnpm install
pnpm run build                 # tsup (CJS + ESM)
pnpm run test                  # jest
pnpm run test:unit             # jest --coverage（仅单元测试）
pnpm run test:integration      # 需要 MEM0_API_KEY
```

### Python CLI (`cli/python/`)

```bash
cd cli/python && pip install -e ".[dev]"
ruff check . && ruff format .  # lint + format
pytest                         # 测试
hatch build                    # 构建
```

### Node CLI (`cli/node/`)

```bash
cd cli/node && pnpm install
pnpm run build                 # tsup (ESM)
pnpm run lint                  # biome check src/
pnpm run typecheck             # tsc --noEmit
pnpm run test                  # vitest
```

### Vercel AI SDK / OpenClaw

```bash
cd <package> && pnpm install && pnpm run build && pnpm run test
```

### Server (`server/`)

```bash
cd server && make bootstrap    # 推荐：启动全栈 + 创建管理员 + 签发 API key
# 或：docker compose up -d     # http://localhost:3000
```

### Docs (`docs/`)

```bash
make docs                      # mintlify dev
```

## 关键陷阱

### 行长度和 Linter 配置因包而异

| 包 | 行长度 | Linter | Formatter | 测试框架 |
|----|--------|--------|-----------|---------|
| `mem0/`（根 SDK） | **120** | Ruff | Ruff | pytest |
| `cli/python/` | **100** | Ruff（规则: E,F,I,W,UP,B,SIM,RUF） | Ruff | pytest |
| `mem0-ts/` | — | — | Prettier | jest |
| `cli/node/` | — | **Biome**（不是 ESLint） | Biome | **vitest**（不是 jest） |
| `vercel-ai-sdk/` | — | ESLint | Prettier | jest + vitest |
| `openclaw/` | — | — | — | vitest |

Agent **几乎必定**猜错这些配置，lint 前务必核对。

### Ruff 排除目录

根 ruff 配置**排除了** `embedchain/` 和 `openmemory/`，不要对它们运行根级 ruff。

### `docs/llms.txt` 同步（CI 拦截项）

任何新增或修改 `docs/**/*.mdx` 的 PR **必须**同步更新 `docs/llms.txt`，否则 CI 报错（`docs-llms-txt-check.yml`）。本地修复：

```bash
python scripts/check-llms-txt-coverage.py --write   # 在 "## Unclassified" 下生成占位条目
# 然后手动：替换 [TODO: ...] 标签、写 "Use when ..." 描述、移到正确分区
```

### Changelog 检查（CI 拦截项）

PR 中若 `pyproject.toml` 版本号变了，`docs/changelog/sdk.mdx` 必须同步更新，否则 CI 报错。在 Python tab 下新增 `<Update>` 条目。

### Pre-commit Hooks

Hooks 自动运行 ruff（带 `--fix`）+ isort（`--profile black`）。首次设置时执行 `pre-commit install`。

### Python 依赖管理

**绝对不要**往根 `pyproject.toml` 的核心 `dependencies` 列表添加依赖，请使用可选依赖组（`[project.optional-dependencies]`）。

### `embedchain/` 完全独立

有自己的 `pyproject.toml`、`poetry.lock` 和 Makefile。不要对它使用根级 hatch 或 ruff。

## 架构

### Provider 模式

所有 provider 继承各自目录下 `base.py` 的抽象基类，配置类在 `configs.py`。新增 provider 的步骤：

1. 创建 `mem0/<category>/<provider>.py`，继承 `mem0/<category>/base.py`
2. 在 `mem0/<category>/configs.py` 添加配置
3. 在 `mem0/<category>/__init__.py` 注册
4. 在 `tests/<category>/` 添加测试
5. 在 `pyproject.toml` 的对应可选组添加依赖
6. 严格遵循同类别现有 provider 的模式

### 入口点

- **Python 自托管**: `from mem0 import Memory, AsyncMemory` → `mem0/memory/main.py`
- **Python 托管平台客户端**: `from mem0 import MemoryClient, AsyncMemoryClient` → `mem0/client/main.py`
- **TypeScript 托管版**: `import { MemoryClient } from 'mem0ai'` → `src/client/`
- **TypeScript OSS 版**: `import { Memory } from 'mem0ai/oss'` → `src/oss/`

### 新记忆算法（v2.0+）

单次 ADD-only 提取 — 一次 LLM 调用，不执行 UPDATE/DELETE。记忆只增不改，实体链接 + 多信号检索（语义 + BM25 + 实体）。`add()` 是主要写入路径。

## CI/CD

### 发布 Tag 前缀

| 包 | Tag 前缀 | 示例 |
|----|----------|------|
| Python SDK | `v*` | `v0.1.31` |
| Python CLI | `cli-v*` | `cli-v0.2.1` |
| TypeScript SDK | `ts-v*` | `ts-v2.4.6` |
| Node CLI | `cli-node-v*` | `cli-node-v0.1.2` |
| Vercel AI SDK | `vercel-ai-v*` | `vercel-ai-v2.0.6` |
| OpenClaw | `openclaw-v*` | `openclaw-v1.0.1` |

所有发布均使用 **OIDC 可信发布** — 无需 token 或密钥。新 npm 包首次发布需手动操作。

### CI 触发

每个包有自己的 CI workflow，仅在该目录变更时触发。根 CI（`ci.yml`）同时覆盖 `embedchain/` 变更。

## 禁止事项

- 用 `pip` 或 `conda` 管理 Python 依赖 — 必须用 `hatch`
- 用 `npm` 或 `yarn` 操作 TypeScript 包 — 必须用 `pnpm`
- 在 TypeScript 中使用 `require()` — 必须用 ES module `import`
- 往核心 `dependencies` 添加 Python 依赖 — 必须用可选组
- 对 `embedchain/` 或 `openmemory/` 运行根级 ruff
- 在不了解迁移链的情况下修改 `openmemory/` 的 Alembic 迁移
- 未经明确批准修改 CI/CD workflow
- 提交 `.env`、API key 或凭证文件
- 跳过 pre-commit hooks
