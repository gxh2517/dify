# Dify 源码部署与旧模型类型迁移

## 源码、数据库迁移链必须一致

**问题**：源码切换到不包含数据库当前 Alembic 修订的发布分支后，`flask db upgrade` 会报“找不到修订版本”。

**做法**：先读取数据库 `alembic_version`，再确认目标 Git 分支包含该迁移文件；若数据库版本领先源码，应恢复到包含该修订的源码分支，而不是伪造或手动修改 Alembic 版本。

**原因**：强行标记迁移版本会让数据库结构与源码迁移图脱节，后续升级无法可靠判断需要执行的 DDL。

## 用内置迁移规范化旧模型类型

**问题**：升级后遗留的 `text-generation` 模型类型会被当前 `ModelType` 枚举拒绝，导致模型供应商页或相关 API 报错。

**做法**：先执行默认预演，再应用仅针对 LLM 的内置迁移：

```bash
cd /opt/dify/api
uv run --project /opt/dify/api flask data-migrate legacy-model-types --model-types llm --concurrency 1
uv run --project /opt/dify/api flask data-migrate legacy-model-types --model-types llm --concurrency 1 --apply
```

随后重启 `dify-api` 与 `dify-worker`。

**原因**：该迁移会把 `text-generation` 转为 `llm`，处理唯一键冲突、关联凭据与缓存失效；直接执行 SQL 容易遗漏这些关联处理。

## 线上数据迁移的验证闭环

**做法**：先运行只读预演确认受影响的表与记录，再执行 `--apply`，重启依赖进程，最后同时验证 API 健康检查和已登录控制台中的受影响页面/交互。

**原因**：命令成功不代表缓存和运行进程已加载新数据；API、后台任务与前端交互均通过后，才能确认用户可见问题已经解决。
