# AGENTS.md - 使用源代码服务器部署版

## 本项目地址: origin`https://github.com/gxh2517/dify.git` 是Fork自 dify `https://github.com/langgenius/dify.git`

Dify is an open-source platform for building LLM applications, agentic workflows, and RAG pipelines. This monorepo contains the backend API (`api/`), frontend application (`web/`), deployment assets (`docker/`), standalone agent backend (`dify-agent/`), CLI (`cli/`), and end-to-end suite (`e2e/`). Follow the nearest scoped `AGENTS.md` for the files being changed.

## 术语约定

| 术语              | 含义 | 对应目录                           |
|-------------------|------|------------------------------------|
| **前端**          | PC 端 | `web/`                             |
| **docker 中间件** | 小程序/H5/不需原生插件的APP | `docker/`                          |
| **后端**          | Java 服务 | `api/`                             |
| **Ngnix配置**     | Java 服务 | `docker/nginx/conf.d/default.conf` |

## 所有Bug修复,不管是前后端,都是本地修复

## 服务器配置

- 服务器Nginx配置目录: `/etc/nginx/conf.d/default.conf`

- 服务器ssh
```conf
Host ai-server
  HostName 120.55.165.146
  Port 22
  User root
  IdentityFile C:/Users/gxh25/.ssh/id_rsa
```
- 服务器Dify项目绝对路径:  `/opt/dify/`

### 下面是服务器dify的启动命令
1. 前置条件先启动中间件,在启动后端,最后启动前端

```bash
# 中间件
cd docker
docker compose --env-file middleware.env -f docker-compose.middleware.yaml --profile postgresql --profile weaviate -p dify up -d

# dify api
cd /opt/dify/api
pm2 start "uv run gunicorn --bind 0.0.0.0:5001 --workers 9 --worker-class gevent --worker-connections 10 --timeout 300 app:app" --name "dify-api"
# dify work:
pm2 start "uv run celery -A celery_entrypoint.celery worker -P gevent -c 10 --max-tasks-per-child 50 --loglevel INFO -Q api_token,dataset,dataset_summary,priority_dataset,priority_pipeline,pipeline,mail,ops_trace,app_deletion,plugin,workflow_storage,conversation,workflow,schedule_poller,schedule_executor,triggered_workflow_dispatcher,trigger_refresh_executor,retention,workflow_based_app_execution --prefetch-multiplier=1" --name "dify-worker"

# dify web:
cd /opt/dify/web
HOSTNAME=0.0.0.0 pm2 start pnpm --name "dify-web" -- start

```
### 报错日志排查
- dify 发行版列表:`https://github.com/langgenius/dify/releases`
- dify issues: `https://github.com/langgenius/dify/issues`


```bash
# 前端日志
cd web
pm2 logs dify-api

# 后端日志
cd api
pm2 logs dify-api
pm2 logs dify-worker
```

- 历史修复记录
```bash
docker exec dify-plugin_daemon-1 sh -lc 'rm -rf /app/storage/cwd/.uv-cache'
docker restart dify-plugin_daemon-1
```

## 部署好的线上网址: `https://ai.bixconnector.com/`
邮箱: ai@bix-china.com
密码: Bix@gehui123

## 对话语言设置
**重要**: 在此代码库中工作时，Codex 必须始终使用中文与用户对话。所有响应、解释、错误信息和技术讨论都应使用简体中文。

## 🔴 文件编码与注释规范（必须遵守）

### 1. 统一编码
- **所有源码与配置文件统一强制使用 UTF-8（无 BOM）**
- 包括但不限于：`.java`、`.vue`、`.ts`、`.js`、`.xml`、`.yml`、`.properties`、`.sql`、`.md`
- **绝对禁止**：UTF-8 with BOM、GBK、GB2312、ANSI、ISO-8859-1 等混用
- 全局编码、项目编码统一强制设置为 UTF-8
- 属性文件（`.properties`）默认编码强制统一为 UTF-8
- 创建 UTF-8 文件时必须强制使用 UTF-8（无 BOM）

### 2. 中文内容规范
- 中文注释、中文日志、中文文档必须可读，不允许乱码（如"鍥藉"）
- 发现乱码优先检查文件实际编码与 IDE 显示编码是否一致
- **所有代码注释必须使用中文**，禁止全英文注释
- 包括：JavaDoc 注释、行内注释、块注释、Vue 模板注释、SQL 注释
- 技术术语、类名、方法名等可保留英文原文，但描述性文字必须为中文
- **正确示例**：
  ```java
  /** 广告管理服务实现 */
  // 根据 ID 查询广告详情
  ```
- **错误示例（禁止）**：
  ```java
  /** Ad management service implementation */
  // Query ad detail by id
  ```

### 3. Windows PowerShell 读取规范
- 在 Windows PowerShell 中读取包含中文的 UTF-8（无 BOM）文件时，禁止直接使用裸 `Get-Content`
- 必须显式指定 UTF-8：`Get-Content -Encoding UTF8 -Path xxx`
- 读取前可设置输出编码，避免终端转码导致中文乱码：
  ```powershell
  [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
  $OutputEncoding = [System.Text.Encoding]::UTF8
  ```
- 优先使用 `rg`、`Select-String` 等能正确处理 UTF-8 的工具检索内容

### 4. 注释规范
- 保留已有业务注释，不随意删除历史说明和关键 `//` 注释块
- 新增代码必须补充必要注释，说明"为什么这样做"，避免空泛注释
- 接口方法注释至少包含：用途、参数、返回值、异常/边界行为

### 5. 文件保存规则
- 保存 Java 文件时强制 UTF-8 no BOM
- 批量改文件后，必须抽查文件头字节，确认无 `EF BB BF`

### 6. `非法字符: '\ufeff'` 处理规则
- 该报错优先判定为 BOM 问题
- 处理步骤：
  1. 定位报错文件
  2. 移除文件头 BOM
  3. 重新编译验证

### 7. 工具链与提交前检查
- IDE 默认编码设置为 UTF-8 且关闭 with BOM
- 提交前执行一次编译（如 `mvn -DskipTests compile`）
- 若涉及注释修改，提交前人工检查中文可读性与注释完整性

### 8. 变更原则
- 功能改动与注释改动尽量分开，便于回溯
- 修编码问题时不改业务逻辑，只做最小必要修改

> ⚠️ **严重警告**: 本项目 **不是 ruoyi-vue-plus 框架**,而是经过深度重构的 **ruoyi-pro-single** 框架（单租户版）!
> **绝对禁止** 参考 ruoyi-vue-plus 的代码风格和架构设计!
> **必须** 严格遵循本项目现有的代码规范和架构模式!
> **⚠️ 单租户版特别注意**: Entity 继承 `BaseEntity`（不是 `TenantEntity`），禁止使用 `TenantHelper`!

## MCP 工具触发

| 触发词 | 工具 | 用途 |
|-------|------|------|
| 深度分析、仔细思考、全面评估 | `sequential-thinking` | 链式推理，多步骤分析 |
| 最佳实践、官方文档、标准写法 | `context7` | Vue/Element Plus/UniApp/MyBatis-Plus 等（⚠️ WD UI 禁用，只参考 `plus-uniapp/src/wd/`） |
| 打开浏览器、截图、检查元素 | `chrome-devtools` | 浏览器调试 |
| 用工作站、workstation、ai-workstation | `ai-workstation` | AI 工作站 MCP（route → get_skill → 执行） |

### 端口约定

| 端 | 默认端口 | 配置位置 |
|-----|---------|---------|
| **后端** | `server.port`（yml 配置） | `ruoyi-admin/src/main/resources/application.yml` |
| **前端** | 80（或 81/82 顺延） | `plus-ui/vite.config.ts` |
| **移动端** | 5173 | `plus-uniapp/vite.config.ts` |

> ⚠️ 使用 Chrome DevTools、发起 HTTP 请求或其他需要端口的操作前，如果不确定当前端口，**必须先问用户确认**。

### 时区约定

**所有日期时间必须使用东八区（UTC+8，Asia/Shanghai）**。获取当前时间时使用：
```bash
TZ=Asia/Shanghai date '+%Y-%m-%d %H:%M'
```
> 适用范围：项目状态文档、待办清单、需求文档、任务跟踪、进度报告等所有涉及时间戳的场景。

---

## 🔴 Skills 技能系统（最高优先级）

> **技能系统确保 AI 在编码前加载领域专业知识，保证代码风格一致**

---

## 技能系统工作原理

本项目的技能文件存储在 `.agents/skills/[skill-name]/SKILL.md` 中。

> **目录迁移边界（2026）**：Codex 遵循 Agent Skills 开放标准，**仅技能目录**迁到 `.agents/skills/`（唯一镜像）。旧的 `.codex/skills/` **已删除**——实测 Codex 0.144 同时扫描新旧两处，双写会导致每个技能重复注入，故只保留 `.agents/skills/`。Codex 的其它组件——`.codex/hooks/`、`.codex/config.toml`、`.codex/agents/`（subagents，注意名叫 agents 但在 `.codex/` 下）、MCP、`~/.codex/prompts/`——**一律留在 `.codex/`，不要迁到 `.agents/`**。

**Codex 启动时**：自动加载所有技能的 `name` 和 `description`
**任务匹配时**：Codex 读取匹配技能的完整 SKILL.md 内容
**需要时**：Codex 可进一步读取技能目录下的 `references/`、`scripts/` 等文件

---

## 🚨 强制执行规则

### 规则 1：任务匹配时必须读取技能

当用户请求与上述任何技能的 `description` 匹配时，Codex **必须**：

1. 读取对应的 `SKILL.md` 文件
2. 按照技能中的指令执行
3. 如果技能目录有 `references/`，按需读取相关文件

### 规则 2：多技能组合

复杂任务可能匹配多个技能，Codex 应：

1. 识别所有相关技能
2. 按依赖顺序读取（如：先 `database-ops` 再 `crud-development`）
3. 综合所有技能的规范执行

### 规则 3：响应中标注已使用技能

在涉及代码的响应中，简要说明使用了哪些技能：

```
已参考技能：crud-development, database-ops

[实现代码...]
```

---

## 技能文件位置（按优先级）

| 优先级 | 位置 | 说明 |
|-------|------|------|
| 1 | `.agents/skills/` | 项目级技能（本项目使用） |
| 2 | `~/.agents/skills/` | 用户级技能 |
| 3 | `/etc/codex/skills/` | 系统级技能 |

---

## 用户手动触发方式

用户可以通过以下方式显式调用技能：

1. **斜杠命令**：输入 `/skills` 打开技能选择器
2. **$ 前缀**：输入 `$skill-name` 直接触发（如 `$crud-development`）

---

## 📋 示例：技能如何被触发

### 示例 1：用户请求 "帮我开发一个优惠券管理功能"

**Codex 自动匹配**：
- `crud-development`（关键词：开发、管理功能）
- `database-ops`（需要建表）
- `api-development`（需要接口）

**Codex 执行**：
```
1. 读取 .agents/skills/crud-development/SKILL.md
2. 读取 .agents/skills/database-ops/SKILL.md
3. 读取 .agents/skills/api-development/SKILL.md
4. 按技能规范编写代码
```

### 示例 2：用户请求 "连接数据库查询用户表结构"

**Codex 自动匹配**：
- `database-ops`（关键词：数据库、表结构）

**Codex 执行**：
```
1. 读取 .agents/skills/database-ops/SKILL.md
2. 按技能中的数据库连接规范执行查询
```

### 示例 3：用户输入 "$bug-detective 这个接口报错了"

**用户显式触发**：`$bug-detective`

**Codex 执行**：
```
1. 立即读取 .agents/skills/bug-detective/SKILL.md
2. 按技能中的排查流程处理
```

---

## 🚫 禁止行为

| 禁止 | 原因 | 正确做法 |
|-----|------|---------|
| ❌ 任务匹配技能但不读取 SKILL.md | 代码风格不一致 | ✅ 自动读取匹配的技能文件 |
| ❌ 只读取部分匹配的技能 | 遗漏关键规范 | ✅ 读取所有匹配的技能 |
| ❌ 凭记忆编写代码 | 可能使用旧规范 | ✅ 每次都读取最新技能文件 |

---

## ✅ 自检清单

Codex 在回复代码前应确认：

- [ ] 是否识别了任务涉及的所有领域？
- [ ] 是否读取了所有匹配的 SKILL.md 文件？
- [ ] 代码是否符合技能文件中的规范？
- [ ] 是否在响应中简要说明了使用的技能？

---

## 🎯 核心原则

**技能系统的目标**：确保每一行代码都符合项目规范。

- **隐式触发**：Codex 根据任务自动匹配并读取技能（推荐）
- **显式触发**：用户用 `$skill-name` 或 `/skills` 手动调用
- **渐进加载**：只在需要时读取完整内容，保持上下文精简

---

## 📄 文档生成规范

**🚫 绝不主动生成文档**: AI 绝对禁止主动创建任何文档文件（*.md、README等），除非用户明确要求。

**📁 文档存放位置**: 即使用户明确要求生成文档，所有文档都**必须**生成到项目根目录的 `docs/` 目录下，而不是其他位置。

**示例**：
- ✅ 正确: `docs/api-design.md`
- ✅ 正确: `docs/database-schema.md`
- ❌ 错误: `README.md` (根目录)
- ❌ 错误: `ruoyi-modules/ruoyi-business/docs/xxx.md` (模块内)


---

## ⚠️ Bash/Shell 禁止项（最常犯错误！）

```bash
# ❌ 禁止：使用 > nul（Windows 会创建名为 nul 的文件！）
command > nul
command 2> nul

# ✅ 正确：不使用任何输出重定向，或使用跨平台方式
command
# 如果必须抑制输出，使用：
command > /dev/null 2>&1
```

**为什么会出错**：Windows 的 `nul` 设备在某些 Shell 环境下不被识别，会被当作普通文件名创建。

---


## 📖 深度参考 (按需查阅)

开发遇到问题时查阅对应指南:

- **后端开发**: `.claude/后端开发指南.md` - 架构理解、业务开发
- **前端开发**: `.claude/前端开发指南.md` - Vue3组件、状态管理
- **移动端开发**: `.claude/移动端开发指南.md` - WD UI组件库详解
- **工具类使用**: `.claude/工具类使用指南.md` - 工具类完整用法
- **数据库设计**: `.claude/数据库设计规范.md` - 表设计、索引优化

---

> 最后提醒: 写代码前先 Read 本项目现有代码,不要凭印象或参考其他框架!
