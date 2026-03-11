# 配置说明

本文档详细介绍 Symphony 的配置选项。

## 配置文件

Symphony 的配置通过 `WORKFLOW.md` 文件定义，该文件应放在你的代码库根目录。

### 文件结构

```markdown
---
# YAML Front Matter（配置区域）
tracker:
  kind: linear
  project_slug: "your-project"
# ... 更多配置
---

# Markdown Body（提示词模板）
You are working on issue {{ issue.identifier }}...
```

## 配置参考

### tracker（跟踪器）

配置问题跟踪器连接。

```yaml
tracker:
  kind: linear                                    # 必填，目前只支持 linear
  endpoint: https://api.linear.app/graphql        # 可选，Linear API 端点
  api_key: $LINEAR_API_KEY                        # 可选，默认从环境变量读取
  project_slug: "your-project-slug"               # 必填，Linear 项目标识
  active_states:                                  # 可选，活跃状态列表
    - Todo
    - In Progress
    - Merging
    - Rework
  terminal_states:                                # 可选，终态列表
    - Closed
    - Cancelled
    - Done
    - Duplicate
```

**获取 project_slug**：
1. 在 Linear 中右键点击项目
2. 复制项目 URL
3. URL 中的最后部分就是 slug

### polling（轮询）

配置轮询行为。

```yaml
polling:
  interval_ms: 5000    # 轮询间隔（毫秒），默认 30000
```

### workspace（工作空间）

配置工作空间存储。

```yaml
workspace:
  root: ~/code/workspaces    # 工作空间根目录
                             # 支持 ~ 和 $VAR 扩展
                             # 默认：系统临时目录/symphony_workspaces
```

### hooks（钩子）

配置工作空间生命周期钩子。

```yaml
hooks:
  # 工作空间首次创建后执行
  after_create: |
    git clone git@github.com:your-org/your-repo.git .
    npm install

  # 每次代理运行前执行
  before_run: |
    git pull origin main

  # 每次代理运行后执行（失败也会执行）
  after_run: |
    echo "Run completed"

  # 工作空间删除前执行
  before_remove: |
    echo "Cleaning up..."

  timeout_ms: 60000    # 钩子超时时间，默认 60000
```

**钩子执行规则**：
- 在工作空间目录中执行
- 使用 `bash -lc <script>` 运行
- `after_create` 失败会阻止工作空间创建
- `before_run` 失败会阻止当前运行
- `after_run` 和 `before_remove` 失败仅记录日志

### agent（代理）

配置代理执行参数。

```yaml
agent:
  max_concurrent_agents: 10           # 最大并发代理数，默认 10
  max_turns: 20                       # 单次运行最大 Turn 数，默认 20
  max_retry_backoff_ms: 300000        # 最大重试退避时间（5分钟），默认 300000
  max_concurrent_agents_by_state:     # 按状态限制并发
    "in progress": 5
    "merging": 2
```

### codex（Codex 配置）

配置 Codex 代理行为。

```yaml
codex:
  command: codex app-server           # Codex 启动命令，默认 codex app-server

  # 审批策略（Codex 参数）
  approval_policy: never              # 可选值：untrusted, on-failure, on-request, never
                                      # 或对象形式：{"reject": {"sandbox_approval": true}}

  # 沙箱模式
  thread_sandbox: workspace-write     # 可选值：read-only, workspace-write, danger-full-access

  # Turn 级别沙箱策略
  turn_sandbox_policy:
    type: workspaceWrite

  # 超时配置
  turn_timeout_ms: 3600000            # Turn 超时（1小时），默认 3600000
  read_timeout_ms: 5000               # 读取超时，默认 5000
  stall_timeout_ms: 300000            # 停滞检测超时（5分钟），默认 300000
                                      # 设为 0 或负数禁用停滞检测
```

**安全建议**：
- 生产环境推荐 `approval_policy: never` 配合 `thread_sandbox: workspace-write`
- 避免使用 `danger-full-access` 除非完全信任环境

### server（服务器扩展）

可选的 HTTP 服务器配置。

```yaml
server:
  port: 4000    # 启用 HTTP 服务器和仪表盘
                # 0 表示使用临时端口
```

也可以通过命令行启用：`./bin/symphony --port 4000`

## 环境变量

### 必需的环境变量

```bash
export LINEAR_API_KEY=lin_api_xxxx    # Linear API 密钥
```

### 在配置中引用环境变量

```yaml
tracker:
  api_key: $LINEAR_API_KEY

workspace:
  root: $SYMPHONY_WORKSPACE_ROOT

codex:
  command: "$CODEX_BIN app-server --model gpt-5.3-codex"
```

## 完整配置示例

### 最小配置

```yaml
---
tracker:
  kind: linear
  project_slug: "my-project"
workspace:
  root: ~/workspaces
---

You are working on issue {{ issue.identifier }}.
Title: {{ issue.title }}
Description: {{ issue.description }}
```

### 完整配置

```yaml
---
tracker:
  kind: linear
  api_key: $LINEAR_API_KEY
  project_slug: "my-project"
  active_states:
    - Todo
    - In Progress
    - Merging
    - Rework
  terminal_states:
    - Closed
    - Cancelled
    - Canceled
    - Duplicate
    - Done

polling:
  interval_ms: 5000

workspace:
  root: ~/code/workspaces

hooks:
  after_create: |
    git clone --depth 1 https://github.com/my-org/my-repo.git .
    npm install
  before_run: |
    git fetch origin main
    git merge origin/main
  after_run: |
    echo "Agent run completed at $(date)"
  before_remove: |
    rm -rf node_modules
  timeout_ms: 120000

agent:
  max_concurrent_agents: 5
  max_turns: 15
  max_retry_backoff_ms: 300000
  max_concurrent_agents_by_state:
    "in progress": 3
    "merging": 1

codex:
  command: codex --config model_reasoning_effort=high app-server
  approval_policy: never
  thread_sandbox: workspace-write
  turn_sandbox_policy:
    type: workspaceWrite
  turn_timeout_ms: 3600000
  read_timeout_ms: 5000
  stall_timeout_ms: 300000

server:
  port: 4000
---

You are working on a Linear issue `{{ issue.identifier }}`

{% if attempt %}
**Continuation context:**
- This is retry attempt #{{ attempt }}
- Resume from current workspace state
{% endif %}

## Issue Details
- Identifier: {{ issue.identifier }}
- Title: {{ issue.title }}
- Status: {{ issue.state }}
- URL: {{ issue.url }}

## Description
{{ issue.description }}

## Instructions
1. Start by updating the workpad comment
2. Implement the required changes
3. Run tests and validation
4. Create a PR when ready
```

## 配置热更新

Symphony 支持**配置热更新**：

1. 监控 `WORKFLOW.md` 文件变化
2. 自动重新加载配置
3. 新配置立即生效（未来调度）

**注意事项**：
- 正在运行的代理不受影响
- HTTP 服务器端口变更可能需要重启
- 配置错误不会导致服务崩溃

## 配置验证

### 启动时验证

以下配置必须在启动时有效：
- `tracker.kind` 存在且支持
- `tracker.api_key` 存在（环境变量解析后）
- `tracker.project_slug` 存在
- `codex.command` 非空

### 运行时验证

每个调度周期会重新验证配置，验证失败会：
- 跳过当前周期的调度
- 保持协调功能活跃
- 记录错误日志

## 默认值速查表

| 配置项 | 默认值 |
|--------|--------|
| `polling.interval_ms` | 30000 |
| `workspace.root` | 系统临时目录/symphony_workspaces |
| `hooks.timeout_ms` | 60000 |
| `agent.max_concurrent_agents` | 10 |
| `agent.max_turns` | 20 |
| `agent.max_retry_backoff_ms` | 300000 |
| `codex.command` | codex app-server |
| `codex.turn_timeout_ms` | 3600000 |
| `codex.read_timeout_ms` | 5000 |
| `codex.stall_timeout_ms` | 300000 |
