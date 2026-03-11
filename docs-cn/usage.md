# 使用指南

本文档介绍 Symphony 的完整使用流程和最佳实践。

## 准备工作

### 1. 代码库准备

Symphony 在采用了 [Harness Engineering](https://openai.com/index/harness-engineering/) 最佳实践的代码库中效果最好：

- ✅ 清晰的项目结构
- ✅ 完善的测试覆盖
- ✅ 代码风格规范
- ✅ CI/CD 配置
- ✅ 良好的文档

### 2. 获取 Linear API Token

1. 登录 Linear
2. 进入 Settings → Security & access → Personal API keys
3. 创建新的 API Key
4. 设置环境变量：

```bash
export LINEAR_API_KEY=lin_api_your_token_here
```

### 3. 配置 Linear 工作流状态

Symphony 使用一些非标准状态。在 Linear 的 Team Settings → Workflow 中添加：

- **Rework**：需要返工
- **Human Review**：等待人工审核
- **Merging**：正在合并

## 安装和启动

### 使用 Elixir 参考实现

```bash
# 克隆仓库
git clone https://github.com/openai/symphony
cd symphony/elixir

# 安装 mise（如果未安装）
curl https://mise.run | sh

# 安装依赖
mise trust
mise install
mise exec -- mix setup
mise exec -- mix build

# 启动服务
mise exec -- ./bin/symphony ./WORKFLOW.md
```

### 命令行选项

```bash
# 使用默认 WORKFLOW.md
./bin/symphony

# 指定配置文件
./bin/symphony /path/to/custom/WORKFLOW.md

# 启用仪表盘
./bin/symphony --port 4000 ./WORKFLOW.md

# 自定义日志目录
./bin/symphony --logs-root /var/log/symphony ./WORKFLOW.md
```

## 配置你的项目

### 步骤 1：复制工作流模板

将 Symphony 的 `WORKFLOW.md` 复制到你的项目：

```bash
cp symphony/elixir/WORKFLOW.md your-project/WORKFLOW.md
```

### 步骤 2：修改配置

编辑 `WORKFLOW.md`，更新以下配置：

```yaml
---
tracker:
  kind: linear
  project_slug: "your-project-slug"    # 改成你的项目

workspace:
  root: ~/code/your-workspaces         # 改成你的目录

hooks:
  after_create: |
    git clone git@github.com:your-org/your-repo.git .    # 改成你的仓库
    # 添加其他初始化命令
---
```

### 步骤 3：复制技能（可选）

如果需要使用内置技能：

```bash
cp -r symphony/.codex/skills your-project/.codex/
```

可用的技能：
- `commit`：生成规范的提交
- `push`：推送分支到远程
- `pull`：同步主分支更新
- `land`：安全合并 PR
- `linear`：Linear 操作
- `debug`：调试辅助

## 日常使用

### 创建任务

1. 在 Linear 中创建 Issue
2. 填写清晰的标题和描述
3. 包含验收标准
4. 将状态设为 `Todo`

**好的任务描述示例**：

```markdown
## 背景
用户反馈在移动端无法正常显示表格。

## 需求
- 修复表格在移动端的显示问题
- 确保响应式布局正常工作

## 验收标准
- [ ] 移动端表格可正常滚动
- [ ] 表头保持可见
- [ ] 单元格内容不截断

## 测试计划
1. 在 Chrome 移动模拟器测试
2. 在真实移动设备测试
```

### 监控进度

#### 使用仪表盘

启动时添加 `--port 4000`，然后访问 `http://localhost:4000`

仪表盘显示：
- 正在运行的任务
- 重试队列
- Token 消耗
- 运行时间统计

#### 查看日志

```bash
# 查看实时日志
tail -f log/symphony.log

# 搜索特定任务的日志
grep "ABC-123" log/symphony.log
```

### 人工审核

当代理完成任务后，Issue 会进入 `Human Review` 状态：

1. 检查关联的 PR
2. 审查代码变更
3. 运行本地测试（如需要）
4. 决定：
   - **批准**：将状态改为 `Merging`，代理会自动合并
   - **要求修改**：将状态改为 `Rework`，添加评论说明需要修改的内容

### 处理阻塞

如果任务被阻塞（缺少权限、密钥等）：

1. 检查代理在工作 pad 中的说明
2. 解决阻塞问题
3. Issue 会自动重试

## 工作流最佳实践

### 提示词模板设计

```markdown
---
# ... 配置 ...
---

You are working on Linear issue `{{ issue.identifier }}`

{% if attempt %}
**This is retry attempt #{{ attempt }}**
Resume from current state, don't restart from scratch.
{% endif %}

## Task
{{ issue.description }}

## Workflow
1. Create a workpad comment with your plan
2. Implement the changes
3. Run tests: `make test`
4. Create PR with description following template
5. Move to Human Review when ready

## Constraints
- Only modify files in this workspace
- Follow existing code style
- Add tests for new functionality
```

### 钩子脚本示例

#### 初始化 Node.js 项目

```yaml
hooks:
  after_create: |
    git clone git@github.com:org/repo.git .
    npm ci
  before_run: |
    git fetch origin main
    git merge origin/main --no-edit
```

#### 初始化 Elixir 项目

```yaml
hooks:
  after_create: |
    git clone git@github.com:org/repo.git .
    if command -v mise >/dev/null 2>&1; then
      mise trust
      mise exec -- mix deps.get
    fi
  before_run: |
    mise exec -- mix deps.get
```

#### 清理大型文件

```yaml
hooks:
  before_remove: |
    rm -rf node_modules _build deps .elixir_ls
```

## 故障排除

### 常见问题

#### 1. 任务一直在重试

**可能原因**：
- Codex 超时
- 工作空间权限问题
- 网络问题

**解决方法**：
```bash
# 检查日志
grep -A 10 "retry" log/symphony.log

# 手动清理工作空间
rm -rf ~/code/workspaces/ABC-123
```

#### 2. 代理无法克隆仓库

**可能原因**：
- SSH 密钥未配置
- 仓库权限不足

**解决方法**：
```bash
# 在 hooks.after_create 中使用 HTTPS
git clone https://github.com/org/repo.git .

# 或确保 SSH 密钥已添加到 GitHub
ssh -T git@github.com
```

#### 3. 工作空间清理失败

**解决方法**：
```bash
# 手动清理
rm -rf ~/code/workspaces/*

# 或检查 before_remove 钩子是否有问题
```

### 调试技巧

#### 启用详细日志

```bash
# 设置日志级别
export SYMPHONY_LOG_LEVEL=debug
./bin/symphony ./WORKFLOW.md
```

#### 检查工作空间状态

```bash
# 进入工作空间
cd ~/code/workspaces/ABC-123

# 检查 Git 状态
git status
git log --oneline -5

# 检查最近的变更
git diff HEAD~1
```

#### 使用 debug 技能

代理可以使用 `.codex/skills/debug/SKILL.md` 中的调试技能来帮助诊断问题。

## 高级用法

### 按状态控制并发

```yaml
agent:
  max_concurrent_agents: 10
  max_concurrent_agents_by_state:
    "in progress": 5     # 执行中的任务最多 5 个
    "merging": 2         # 合并中的任务最多 2 个
    "rework": 3          # 返工的任务最多 3 个
```

### 使用不同的 Codex 模型

```yaml
codex:
  command: codex --model gpt-5.3-codex app-server
```

### 自定义审批策略

```yaml
codex:
  # 拒绝所有需要人工审批的操作
  approval_policy:
    reject:
      sandbox_approval: true
      rules: true
      mcp_elicitations: true
```

### 禁用停滞检测

```yaml
codex:
  stall_timeout_ms: 0    # 设为 0 或负数
```

## 安全建议

1. **限制沙箱权限**：使用 `workspace-write` 而非 `danger-full-access`
2. **审计钩子脚本**：确保钩子脚本不会泄露敏感信息
3. **保护 API 密钥**：使用环境变量而非硬编码
4. **定期清理**：定期检查和清理工作空间
5. **监控日志**：关注异常活动和错误

## 下一步

- 阅读 [核心概念](concepts.md) 深入理解系统
- 参考 [配置说明](configuration.md) 优化你的设置
- 查看 [架构设计](architecture.md) 了解内部实现
