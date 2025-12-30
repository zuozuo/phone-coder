# Phone Coder 设计文档

## 概述

Phone Coder 是一个手机 App，用于在手机上远程操作电脑上的 Claude Code。

**使用场景：** 移动办公，不在电脑前时用手机继续编程任务。

## 技术架构

```
┌─────────────────┐     HTTPS/WSS      ┌─────────────────┐
│   Ionic App     │ ←───────────────→  │   Go Server     │
│   (手机端)       │   Cloudflare       │   (本地 Mac)     │
└─────────────────┘     Tunnel         └────────┬────────┘
                                                │ subprocess
                                       ┌────────┴────────┐
                                       │   Claude Code   │
                                       └─────────────────┘
```

**技术栈：**
- 前端：Ionic + Capacitor
- 后端：Go
- CLI：Claude Code（MVP 只支持 Claude）
- 部署：本地 Mac + Cloudflare Tunnel

## 通信协议

### 客户端 → Server

```typescript
// 认证
{ type: "auth", password: "xxx" }

// 创建会话
{ type: "session.create", workDir: "/path/to/project" }

// 获取会话列表
{ type: "session.list" }

// 停止任务
{ type: "session.stop", sessionId: "xxx" }

// 发送消息
{ type: "message.send", sessionId: "xxx", content: "..." }

// 心跳
{ type: "ping" }
```

### Server → 客户端

```typescript
// 认证结果
{ type: "auth.result", success: true, error?: "wrong password" }

// 会话创建成功
{ type: "session.created", sessionId: "xxx", workDir: "/path" }

// 会话列表
{ type: "session.list", sessions: [{id, workDir, status}] }

// CLI 输出（流式）
{ type: "message.stream", sessionId: "xxx", content: "..." }

// 消息完成
{ type: "message.done", sessionId: "xxx" }

// 错误
{ type: "error", message: "xxx" }

// 心跳响应
{ type: "pong" }
```

## 数据存储

### 目录结构

```
~/.phone-coder/
├── config.yaml      # 配置文件
├── data.db          # SQLite 数据库
└── logs/            # 日志目录
```

### 数据库表

```sql
-- 会话表
CREATE TABLE sessions (
    id TEXT PRIMARY KEY,              -- 我们的 session ID
    claude_session_id TEXT,           -- Claude 的 session ID（用于 resume）
    work_dir TEXT NOT NULL,
    status TEXT DEFAULT 'idle',       -- idle/running/stopped
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

### 配置文件

```yaml
password: "your-password"
port: 8080
workdirs:
  - /Users/xxx/project1
  - /Users/xxx/project2
```

## Claude Code 集成

### Session ID 获取

Claude Code 的 session 数据存储在：
```
~/.claude/projects/{project-hash}/{sessionId}.jsonl
```

**project-hash：** 工作目录路径，`/` 替换为 `-`

**获取流程：**
1. 计算 project hash
2. 扫描目录 `~/.claude/projects/{project-hash}/*.jsonl`
3. 找到最新的 .jsonl 文件
4. 文件名（去掉 .jsonl）就是 session ID

### 命令行调用

```bash
# 新会话（非交互模式）
claude -p "用户输入"

# 恢复会话
claude --resume <sessionId> -p "用户输入"
```

## 前端 UI

### 页面结构

1. **首页** - 会话列表 + 新建会话按钮
2. **聊天页** - 对话界面 + 停止按钮
3. **设置页** - Server 地址、密码配置

### 首页布局

```
┌─────────────────────────────────────┐
│  Phone Coder                    ≡   │
├─────────────────────────────────────┤
│  ┌─────────────────────────────┐   │
│  │ 📁 项目A          running   │   │
│  └─────────────────────────────┘   │
│  ┌─────────────────────────────┐   │
│  │ 📁 项目B          idle      │   │
│  └─────────────────────────────┘   │
│              [ + 新建会话 ]          │
├─────────────────────────────────────┤
│  ⚙️ 设置                            │
└─────────────────────────────────────┘
```

### 聊天页布局

```
┌─────────────────────────────────────┐
│  ← 项目A                    ⏹ 停止  │
├─────────────────────────────────────┤
│  用户消息和 AI 回复（Markdown 渲染） │
├─────────────────────────────────────┤
│  [继续] [取消] [/help]              │  ← 快捷指令
├─────────────────────────────────────┤
│  ┌─────────────────────┐  🎤  ➤    │  ← 输入框
└─────────────────────────────────────┘
```

### 功能特性

- **输入方式：** 文本 + 语音 + 快捷指令
- **输出显示：** 现代聊天风格，Markdown 渲染 + 代码高亮
- **交互控制：** 发送消息 + 停止任务

## MVP 简化点

- 只支持 Claude Code（不支持 Gemini/Codex）
- 密码直接验证，不做 token
- 无断点续传（断了重连看 session.list）
- 预设目录手动配置 config.yaml，不做 UI 管理
- 错误信息纯文本

## 后续迭代（V2+）

- 支持更多 CLI（Gemini、Codex）
- Token 认证 + 自动刷新
- 断点续传 + 消息序号
- 预设目录 UI 管理
- CLI 确认交互支持
