# Hermes + OpenCode 智能体整合方案

## 概述

本实验项目旨在整合 **Hermes Agent**（消息网关型 AI 助手）与 **OpenCode**（软件工程 AI 助手），实现跨平台的智能体协作。

| 智能体 | 核心能力 | 入口 |
|--------|---------|------|
| **Hermes** | 消息网关（微信/Telegram/Discord 等）、75+ 技能、28+ 工具 | `hermes chat` / `hermes gateway` |
| **OpenCode** | 代码编辑、文件操作、Bash 执行、Git 操作、子代理委托 | `opencode` CLI |

## 设计原则

- **CLI 优先**：两个智能体都在 CLI 环境中运行，直接通过 CLI 相互调用，不引入额外协议层
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
           │     CLI 相互调用                │
           │  ┌─────────────────────────┐   │
           └──┤  hermes chat -q "..."   ├───┘
              │  opencode chat -q "..." │
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

## 阶段二：OpenCode → Hermes（消息发送能力）

### 目标

OpenCode 通过调用 Hermes CLI 向微信等平台发送消息。

### 流程

```
OpenCode 任务 → hermes chat -q "请给用户微信发消息: ..." → Hermes 通过 gateway 发送
```

### 实现

OpenCode 在需要推送通知时，通过 `Task` 或 `Bash` 工具调用 Hermes：

```bash
hermes chat -q "请给微信用户 {{user_id}} 发送消息：{{content}}" --quiet
```

### 示例

```
用户 (OpenCode CLI): 帮我重构这个文件，完成后微信通知我
    ↓
OpenCode: 执行重构
    ↓
OpenCode: → hermes chat -q "请给用户微信发送消息：重构已完成，改动如下：..."
    ↓
Hermes: 通过 gateway 发送微信消息
```

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

CLI 调用天然支持双向委托，不依赖额外协议：

- **Hermes → OpenCode**：Hermes skill 中配置 `opencode chat -q` 命令
- **OpenCode → Hermes**：OpenCode 的 Task/Bash 工具执行 `hermes chat -q` 命令

### 流程示例

```
用户 (微信): 帮我重构 src/utils.py，完成后通知我
    ↓
Hermes: 理解意图，调用 opencode skill
    ↓
Hermes → opencode chat -q "重构 src/utils.py，完成后回复'重构完成'"
    ↓
OpenCode: 执行重构任务
    ↓
OpenCode → hermes chat -q "请给用户微信发送消息：重构完成，改动如下：..."
    ↓
Hermes gateway: 发送微信消息给用户
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
| 进程间通信 | CLI 相互调用（`hermes chat -q` / `opencode chat -q`） |
| 异步通信 | 共享文件（`agent-inbox/` 目录协议） |
| 会话存储 | Hermes 原生会话管理 |
| 消息路由 | Hermes Gateway 技能机制 |
| 通知推送 | Hermes gateway 微信通道 |

## 下一步

1. 实现阶段一 PoC：创建 opencode skill 并验证微信 → OpenCode 通路
2. 验证阶段二：从 OpenCode 调用 hermes 发微信消息
3. 编写集成测试用例，覆盖双向调用场景
