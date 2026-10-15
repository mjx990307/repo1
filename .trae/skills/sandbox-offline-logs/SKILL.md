---
name: "sandbox-offline-logs"
description: "下载并解压沙箱容器的离线归档日志。当需要拉取已终止/离线沙箱的容器日志、从归档 pod 诊断问题、或用户提供 sessionId/podName/sandboxName 要求获取日志时调用。"
---

# 沙箱离线日志下载

通过 `trae-remote-dashboard` API 下载沙箱容器的归档（离线）日志，base64 解码后解压到本地进行分析。

## 使用场景

- 需要下载已终止或离线沙箱的容器日志
- 用户提供了 sessionId、podName 或 sandboxName，要求获取归档日志
- 诊断问题时需要历史容器日志，但 pod 已不存在无法通过 kubectl 获取

## 前置条件

- 有效的 JWT token（`x-jwt-token`），用于 trae-remote-dashboard 鉴权
- 目标沙箱的 `sessionId`、`podName` 和 `sandboxName`

## API 接口

```
GET `https://trae-remote-dashboard.gf.bytedance.net/api/logs/download-offline`
```

### 请求参数

| 参数 | 必填 | 说明 |
|------|------|------|
| `sessionId` | 是 | 会话 ID（如 `6a0f28433d8ab5c8ee81b41c`） |
| `podName` | 是 | K8s Pod 名称（如 `work-32-vci-sdlrb`） |
| `sandboxName` | 是 | Sandbox Claim 名称（如 `run-agent-6a0f28433d8ab5c8ee81b41c-mpjk33m8`） |

### 请求头

| Header | 必填 | 说明 |
|--------|------|------|
| `x-jwt-token` | 是 | JWT 鉴权 token（从用户获取或复用现有 session） |

### 响应格式

```json
{
  "data": "<base64 编码的 zip 内容>",
  "filename": "run-agent-xxx-offline-logs-timestamp.zip"
}
```

## 操作步骤

### 第 1 步：准备本地目录

创建 `.local` 目录（已在 `.gitignore` 中）存放临时数据：

```bash
mkdir -p .local
```

### 第 2 步：下载日志

```bash
curl -s \
  -H "x-jwt-token: <TOKEN>" \
  "`https://trae-remote-dashboard.gf.bytedance.net/api/logs/download-offline?sessionId=<SESSION_ID>&podName=<POD_NAME>&sandboxName=<SANDBOX_NAME>`" \
  -o .local/response.json
```

### 第 3 步：base64 解码并保存为 zip

```python
import json, base64

with open('.local/response.json', 'r') as f:
    resp = json.load(f)

filename = resp.get('filename', 'output.zip')
data = resp['data']

decoded = base64.b64decode(data)
output_path = '.local/' + filename
with open(output_path, 'wb') as f:
    f.write(decoded)

print(f'Saved: {output_path}')
print(f'Size: {len(decoded)} bytes')
```

### 第 4 步：解压 zip

```bash
unzip -o .local/<filename>.zip -d .local/logs
```

## 解压后的日志结构

```
.local/logs/
├── agent/                      # AI Agent 服务日志
│   ├── ai-agent.stdout.log    # 当前 ai-agent 日志（最新）
│   ├── ai-agent.stdout.log.1  # 轮转日志（较旧）
│   ├── ai-agent.stdout.log.2
│   ├── ai-agent.stdout.log.3  # 最旧的轮转日志
│   ├── ai-agent.stderr.log
│   ├── sentinel.stdout.log    # Sentinel（快照/hook 管理器）
│   ├── fs-server.stdout.log   # 文件服务
│   ├── ckg.stdout.log         # 代码知识图谱
│   ├── ssh-agent.stdout.log
│   └── supervisord.log        # 进程管理器
├── tool/                       # Tool Host 日志
│   ├── agent-tool-host.stdout.log
│   └── jobs/                   # 各任务执行记录
│       ├── job-N/output.log    # 命令输出
│       └── job-N/state.json    # 任务元数据（command, pid, 时间戳, exit_code）
├── apt/                        # 系统包管理日志
├── dpkg.log
└── ...
```

## 关键文件用途

| 文件 | 用途 |
|------|------|
| `agent/ai-agent.stdout.log*` | LLM 调用、工具执行、SSE 心跳、错误信息 |
| `agent/sentinel.stdout.log` | TurnStart/TurnEnd hook、restic 备份、commit_session、termination |
| `agent/fs-server.stdout.log` | 文件操作、stat 错误 |
| `agent/supervisord.log` | 进程生命周期（启动/重启/崩溃） |
| `tool/jobs/job-N/state.json` | 任务元数据：command、created_at、finished_at、exit_code |
| `tool/jobs/job-N/output.log` | 命令实际输出（stdout/stderr） |

## 常用诊断命令

### 检查错误
```bash
grep -i "error\|fail\|panic\|timeout\|401\|403" agent/sentinel.stdout.log
grep -i "error\|fail" agent/ai-agent.stdout.log | grep -v heartbeat
```

### 检查 commit_session 失败
```bash
grep -n "commit_session\|991303\|403 Forbidden\|401 Unauthorized" agent/sentinel.stdout.log
```

### 检查 turn 生命周期
```bash
grep -n "TurnStart\|TurnEnd\|dispatch completed\|dispatch failed" agent/sentinel.stdout.log
```

### 检查任务执行情况
```bash
cat tool/jobs/job-N/state.json   # 查看命令、时间戳、退出码
tail tool/jobs/job-N/output.log  # 查看输出
```

### 检查 restic 备份状态
```bash
grep -n "snapshot\|restic\|backup\|commit_session" agent/sentinel.stdout.log | tail -30
```

## 注意事项

- 日志中的时间戳均为 **UTC**，用户侧时间通常为 **UTC+8**（北京时间）。
- `ai-agent.stdout.log` 每个文件约 50MB 自动轮转。编号规则：`.log` 是最新的，`.log.3` 是最旧的。
- `.local/` 目录已在 `.gitignore` 中，下载的文件不会污染 git 树。
- 如果 API 返回鉴权错误，说明 JWT token 已过期 — 需要向用户索要新的 token。
