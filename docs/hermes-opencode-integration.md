# Hermes + OpenCode 智能体整合方案

## 概述

本实验项目旨在整合 **Hermes Agent**（消息网关型 AI 助手）与 **OpenCode**（软件工程 AI 助手），实现跨平台的智能体协作。

| 智能体 | 核心能力 | 入口 |
|--------|---------|------|
| **Hermes** | 消息网关（微信/Telegram/Discord 等）、75+ 技能、28+ 工具 | `hermes chat` / `hermes gateway` |
| **OpenCode** | 代码编辑、文件操作、Bash 执行、Git 操作、子代理委托 | `opencode` CLI |

## 设计原则

- **MCP 协议**：通过 Model Context Protocol 标准协议互联，固定工具接口
- **无侵入**：不修改 Hermes 或 OpenCode 源码，纯外部集成
- **渐进式**：从单向调用到双向协作，逐步增加集成深度

## 架构总览

```
┌─────────────────────────────────────────────────────────┐
│                    用户交互层                             │
│  微信 / Telegram / Discord / Terminal                     │
└───────────┬─────────────────────────────┬───────────────┘
            │                             │
            ▼                             ▼
┌─────────────────────┐     ┌─────────────────────────────┐
│   Hermes Gateway    │     │     OpenCode CLI            │
│   (消息网关服务)      │     │     (软件工程助手)           │
└──────────┬──────────┘     └──────────────┬──────────────┘
           │                               │
           │     MCP 协议 (stdio)           │
           │  ┌─────────────────────────┐   │
           └──┤  hermes mcp serve       ├───┘
              │  messages_send(target,  │
              │    message)             │
              └─────────────────────────┘
```

---

## 阶段一：Hermes → OpenCode（消息通道桥接，MVP）

### 目标

通过 Hermes 的消息网关，让用户能在微信上与 OpenCode 交互。

### 流程

```
微信消息 → Hermes Gateway → opencode chat -q "..." → 处理结果 → 回复回微信
```

### 实现

在 Hermes 中创建一个 `opencode` 技能，CLI 调用 OpenCode。

```yaml
# Hermes skill 配置
skills:
  - name: opencode
    command: opencode chat -q "{{query}}" --quiet
    platforms: [weixin, telegram, discord]
    prefix: /opencode
    timeout: 300
```

### 示例

```
用户 (微信): /opencode 当前项目结构是怎样的？
    ↓
Hermes gateway: → opencode chat -q "当前项目结构是怎样的？" --quiet
    ↓
OpenCode: 探索目录，返回结果
    ↓
Hermes gateway: 将结果发回用户微信
```

**注意**：Hermes 需配置 `WEIXIN_HOME_CHANNEL` 为自己微信号，并确保 gateway 运行中。

---

## 阶段二：OpenCode → Hermes 消息发送（MCP，已实现 ✅）

### 原理

Hermes 提供 MCP 服务端（`hermes mcp serve`），通过 stdio 通信，暴露 `messages_send` 工具：

```
messages_send(target="weixin:<chat_id>", message="...")
  → send_message_tool → send_weixin_direct → iLink API → 微信
```

### 配置

在 `.opencode/opencode.json` 中声明 MCP 服务器：

```json
{
  "mcp": {
    "hermes": {
      "type": "local",
      "command": ["hermes", "mcp", "serve"],
      "enabled": true
    }
  }
}
```

OpenCode 启动时自动拉起 `hermes mcp serve` 子进程，通过 MCP 协议发现 `messages_send` 等 8 个工具。

### OpenCode 中使用

OpenCode 知道 `messages_send` 工具可用，调用时传递固定参数：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `target` | string | ✅ | `"weixin:<chat_id>"` 格式 |
| `message` | string | ✅ | 消息内容 |

示例：
```
OpenCode 调用 messages_send(target="weixin:o9cq80zB-xxxxx@im.wechat", message="重构完成")
```

### 验证

```bash
# 直接测试 MCP 调用
$ printf '...' | hermes mcp serve
→ {"success": true, "platform": "weixin", "chat_id": "o9cq80zB-xxxxx@im.wechat"}
```

### 可用工具一览

| 工具 | 用途 |
|------|------|
| `messages_send` | 发送消息到任意平台 |
| `channels_list` | 列出可用消息目标 |
| `conversations_list` | 列出活跃会话 |
| `conversation_get` | 查看会话详情 |
| `messages_read` | 读取会话消息 |
| `events_poll` / `events_wait` | 轮询/等待新事件 |
| `permissions_list_open` / `permissions_respond` | 审批管理 |

---

## 阶段三：双向委托协作

### 目标

两个智能体在复杂任务中相互委托，各司其职。

### 分工模型

| 任务类型 | 负责智能体 | 方式 |
|---------|-----------|------|
| 代码工程（编辑/搜索/重构） | **OpenCode** | 直接执行 |
| 消息推送与通知 | **Hermes** | `hermes chat -q "发消息:..."` |
| 用户交互与对话管理 | **Hermes** | 多平台会话管理 |
| 文件系统操作 | **OpenCode** | 直接执行 |
| 外部 API 调用 | **Hermes** | 丰富的技能生态 |
| Git 版本控制 | **OpenCode** | 直接执行 |

### 委托机制

- **Hermes → OpenCode**：Hermes skill 中配置 `opencode run` 命令（built-in skill）
- **OpenCode → Hermes**：OpenCode 通过 MCP 协议调用 `messages_send` 工具

### 流程示例

```
用户 (微信): 帮我重构 src/utils.py，完成后通知我
    ↓
Hermes: 理解意图，调用 opencode skill
    ↓
Hermes → opencode run "重构 src/utils.py，完成后通知我"
    ↓
OpenCode: 执行重构任务
    ↓
OpenCode → messages_send(target="weixin:...", message="重构完成，改动如下：...")
    ↓
Hermes MCP → send_weixin_direct → iLink API → 微信
```

---

## 阶段四（可选）：共享工作区与协议文件

### 目标

实现异步解耦的协作模式。

### 方案

使用共享目录 `agent-inbox/` 作为通信层：

```
agent-inbox/
├── requests/       # Hermes → OpenCode 请求
│   └── 001.json
├── responses/      # OpenCode → Hermes 响应
│   └── 001.json
└── notifications/  # OpenCode → Hermes 通知
    └── 微信推送.json
```

- Hermes 写入请求到 `requests/`，OpenCode 轮询处理
- OpenCode 写入通知到 `notifications/`，Hermes gateway 读取并推送
- 适合不需要实时响应的后台任务

---

## 技术选型

| 组件 | 方案 |
|------|------|
| 进程间通信 | MCP 协议（stdio，`hermes mcp serve`） |
| 消息发送 | Hermes MCP `messages_send` 工具 |
| 会话存储 | Hermes 原生会话管理 |
| 消息路由 | Hermes Gateway 技能机制 |
| 通知推送 | Hermes gateway 微信通道 |

## 实现状态

| 阶段 | 状态 | 说明 |
|------|------|------|
| 一：Hermes → OpenCode | ✅ 内置 | Hermes 已有 `opencode` built-in skill，通过 `opencode run` 调用 |
| 二：OpenCode → Hermes (MCP) | ✅ 已实现 | `.opencode/opencode.json` 配置 MCP，暴露 `messages_send` 工具 |
| 三：双向委托协作 | ⏳ 待测试 | 微信发指令 → Hermes → OpenCode 执行 → MCP 通知结果 |
| 四：共享工作区 | 📋 可选 | 按需推进 |

## 下一步

1. 测试完整闭环：微信发 `/opencode <任务>` → Hermes → OpenCode 执行 → MCP 微信通知结果
2. 清理冗余脚本（`tools/send-wechat` 等已由 MCP 替代）
3. 编写集成测试用例
