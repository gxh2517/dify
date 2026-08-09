---
name: cmd-framework-sync
description: |
  /framework-sync 命令 - 从框架原仓库同步更新到子项目。

  触发场景：
  - 用户执行 /framework-sync 命令
  - 需要同步框架最新代码到子项目
  - 需要查看框架更新并选择性合并

  触发词：/framework-sync、框架同步命令
---

# /framework-sync - 框架更新同步助手

从 ruoyi-pro 框架仓库同步最新更新到子项目。支持多分支、逐提交分析、选择性合并。

---

## 核心原则

1. **选择性合并**：不是所有更新都需要，逐个分析后用户确认
2. **标识符自动替换**：合并时自动将框架标识符替换为子项目标识符
3. **历史可追溯**：每次同步记录到 `.framework-sync.json`
4. **不用 git merge**：使用 Edit 工具手动应用改动，避免标识符冲突

---

## 执行流程

### 第零步：环境检查

```bash
# 1. 检查 upstream remote
git remote -v | grep upstream

# 2. 检查 .framework-sync.json
cat .framework-sync.json
```

**如果 upstream 不存在**：

不要直接报错，**询问用户提供框架仓库地址**：

```
未检测到 upstream remote。请提供框架仓库地址，我来帮你配置：
- 远程地址示例：https://gitcode.com/yechaoa/ruoyi-pro.git
- 本地路径示例：D:/projects/ruoyi-pro
```

用户提供后执行：`git remote add upstream <地址>` + `git fetch upstream`

**如果 `.framework-sync.json` 不存在**：

进入首次配置流程，**逐步询问用户**（先尝试自动检测，检测不到再问）：

1. **框架仓库地址** — 如果 upstream 已配置则展示确认，否则询问用户提供
2. **基于哪个分支** — 先从 `application.yml` 的 `app.id` 推断，推断不了再问用户：
   - master（多租户）
   - single（单租户）
   - workflow（工作流）
   - 如果用户也不确定，可以描述业务场景帮助判断
3. **项目标识符** — 从 `application.yml` 读取，向用户确认
4. **创建时的 commit** — 尝试从第一条 git log 推断，推断不了可输入 "auto"
5. **保留的子项目** — 检查实际存在的目录自动勾选，向用户确认

收集完信息后生成 `.framework-sync.json`。

**环境就绪后**：

```bash
# 拉取 upstream 最新
git fetch upstream <branch>
```

---

### 第一步：提交数量检测

```bash
LAST_COMMIT=$(从 .framework-sync.json 读取 lastCommit)
BRANCH=$(从 .framework-sync.json 读取 upstream.branch)

# 统计待同步提交
git log ${LAST_COMMIT}..upstream/${BRANCH} --oneline | wc -l

# 查看提交列表
git log ${LAST_COMMIT}..upstream/${BRANCH} --oneline --date-order
```

**根据数量输出**：

```markdown
## 框架更新检测

从上次同步到最新，共有 **X** 个新提交。

请选择：
1. **一次性处理**（推荐）
2. **分批处理** — 按时间段分批
3. **只看重点** — 只显示 fix/feat/update
4. **按模块筛选** — 只看特定目录（backend/frontend/mobile）
5. **指定范围** — 输入起止 commit
```

---

### 第二步：智能分类

对每个提交自动分类：

**建议合并**（框架核心）：
- `ruoyi-common/*` — 工具类、通用模块
- `ruoyi-modules/ruoyi-system/*` — 系统模块
- `ruoyi-admin/*` — 启动模块
- `plus-ui/src/components/*`, `plus-ui/src/utils/*` — 前端框架
- `plus-uniapp/src/wd/*`, `plus-uniapp/src/composables/*` — 移动端框架
- `.claude/skills/*`, `.claude/commands/*` — AI 技能/命令
- `pom.xml` — 依赖管理
- `script/*` — 部署脚本

**建议跳过**（业务示例）：
- `ruoyi-modules/ruoyi-business/*`
- `plus-ui/src/views/business/*`
- `plus-uniapp/src/pages-sub/business/*`
- 用户未保留的子项目（根据 `retainedProjects`）

**需要评估**（用户可能已定制）：
- `*/env/.env*` — 环境配置
- `application*.yml` — 后端配置
- `CLAUDE.md`, `AGENTS.md` — 项目文档

**输出格式**：

```markdown
## 提交分类分析

### 建议合并（共 X 个）
| 提交ID | 说明 | 涉及文件 | 分类 |
|--------|------|---------|------|
| abc123 | fix: 修复 RedisUtils 问题 | 2 files | 框架核心 |

### 建议跳过（共 Y 个）
| 提交ID | 说明 | 跳过原因 |
|--------|------|---------|
| ghi789 | feat: 广告管理优化 | 业务示例 |

### 需要评估（共 Z 个）
| 提交ID | 说明 | 评估原因 |
|--------|------|---------|
| jkl012 | update: 升级依赖 | 需确认兼容性 |

---

请选择要合并的提交（all / 提交ID列表 / skip）：
```

---

### 第三步：逐个分析选中的提交

对每个选中的提交：

```bash
git show <commit_id> --stat
git diff <commit_id>~1 <commit_id> -- <文件>
```

**输出**：

```markdown
## 提交分析：abc123

- **说明**：fix: 修复 RedisUtils 序列化问题
- **文件**：ruoyi-common/.../RedisUtils.java（+5 -2）
- **标识符替换**：不需要
- **本项目对应**：路径相同

是否合并？（合并 / 跳过 / 详细查看）
```

---

### 第四步：执行合并

**合并策略**：

1. **路径相同、无标识符**：直接用 Edit 工具应用改动行
2. **包含标识符**：先替换标识符再应用
3. **新增文件**：从 upstream 导出到本地
4. **有冲突**：展示两端差异，用户选择处理方式

**标识符替换规则**（从 `identifierMap` 读取）：
- 替换顺序：先长后短（防止部分匹配）
- 端口仅在特定文件中替换

**配置文件特殊处理**：
- `application.yml` 等：只合并新增配置项，不覆盖用户定制
- `pom.xml`：只合并依赖版本变更，保留用户添加的依赖

---

### 第五步：记录同步历史

更新 `.framework-sync.json`：

```json
{
  "lastSync": "2026-03-16T14:30:00+08:00",
  "lastCommit": "新commit_hash",
  "syncs": [
    {
      "date": "...",
      "fromCommit": "旧hash",
      "toCommit": "新hash",
      "totalCommits": 15,
      "merged": [
        { "commit": "abc123", "message": "...", "files": [...], "replaced": false }
      ],
      "skipped": [
        { "commit": "ghi789", "message": "...", "reason": "业务示例" }
      ]
    }
  ]
}
```

---

### 第六步：输出同步报告

```markdown
## 框架同步完成报告

**同步时间**：YYYY-MM-DD HH:MM
**框架分支**：master
**提交范围**：oldHash → newHash

### 统计
- 总提交数：15
- 已合并：10
- 已跳过：5

### 已合并的提交
| 提交ID | 说明 | 标识符替换 |
|--------|------|-----------|
| abc123 | fix: 修复 RedisUtils | 否 |

### 已跳过的提交
| 提交ID | 说明 | 跳过原因 |
|--------|------|---------|
| ghi789 | feat: 广告管理 | 业务示例 |

### 后续操作
1. 编译后端，检查编译错误
2. 运行前端/移动端，确认功能正常
3. 提交同步后的代码

### 下次同步起始点
- commit: newHash
```

---

## 快速命令

### 只看后端更新
```
/framework-sync backend
```

### 只看前端更新
```
/framework-sync frontend
```

### 只看移动端更新
```
/framework-sync mobile
```

### 从特定 commit 开始
```
/framework-sync from abc1234
```

### 查看同步历史
```
/framework-sync history
```

---

## 注意事项

1. **不要使用 git merge 或 cherry-pick** — 因为标识符不同，会导致大量冲突
2. **配置文件只合并新增项** — 不覆盖用户的定制配置
3. **每次同步后更新 .framework-sync.json** — 记录同步点
4. **同步前先提交当前工作** — 方便回滚
5. **同步优先级**：安全修复 > Bug 修复 > 性能优化 > 依赖升级 > 功能增强
