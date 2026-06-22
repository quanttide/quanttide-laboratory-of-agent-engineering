# 实验：契约—注册模型

## 目标

验证跨 Agent 管理的正确模型：**维护者写契约，各 Agent 自行注册**。

## 模型

```
契约（Ponytail 仓库）
├── AGENTS.md       ← 行为规则，Markdown 格式
├── ponytaill.mjs    ← 行为规则，JS Module 格式
└── SKILL.md        ← 行为规则，YAML + Markdown 格式
        │
        │  「注册」—— 各 Agent 以自己的方式引用，非复制
        │
   ┌────┼────┐
   │    │    │
  Zed OpenCode Hermes
  (symlink) (plugin) (skills dir)
```

- **契约**：一份行为规则，以三种格式表达，方便不同 Agent 消费
- **注册**：每个 Agent 用自己的配置机制引用契约。注册后，契约更新自动生效（引用而非复制）

## 与「复制一份」的对比

| | 契约—注册（本实验） | 各自复制一份 |
|---|---|---|
| 修改 | 改契约，全部生效 | 逐个 Agent 改 |
| 新增 Agent | 加一份注册 | 加一份完整副本 |
| 不一致风险 | 零 | 高 |
| 格式差异 | 契约内多格式共存 | Agent 自行维护格式 |

## 目录

```
contract-registry/
├── README.md
├── contract/                    # 契约
│   ├── AGENTS.md                → Zed 注册入口
│   ├── ponytaill.mjs             → OpenCode 注册入口
│   └── SKILL.md                 → Hermes 注册入口
└── registrations/               # 各 Agent 的注册示例
    ├── zed/AGENTS.md            # symlink → contract/AGENTS.md
    ├── opencode/opencode.json   # { "plugin": ["contract/ponytail.mjs"] }
    └── hermes/skills/ponytail/  # SKILL.md 放到 skills 目录
```
