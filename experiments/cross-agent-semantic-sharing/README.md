# 实验：Ponytail 一次性配置，多智能体复用

## 目标

验证 Ponytail 作为单一实体管理后，一次配置变更能否自然覆盖所有智能体。

## 架构

```
ponytail 仓库（单一源头）
├── AGENTS.md                  ──→  Zed（符号链接 ~/.config/zed/AGENTS.md）
├── hooks/ponytail-instructions.js  ──→  OpenCode（ponytail.mjs 插件注入）
└── skills/ponytail/           ──→  Hermes（skills 机制加载，待集成）
```

## 实验步骤

### 1. 建立模拟环境

创建三个模拟的 "agent 配置目录"，分别模拟 Zed、OpenCode、Hermes 的配置加载方式：

```
experiments/cross-agent-semantic-sharing/
├── README.md          ← 本文件
├── simulated/
│   ├── zed/
│   │   └── AGENTS.md  ← 模拟符号链接：复制 ponytail/AGENTS.md 到此
│   ├── opencode/
│   │   └── prompt.txt ← 模拟插件注入结果：复制 getPonytailInstructions() 输出到此
│   └── hermes/
│       └── SKILL.md   ← 模拟 skills 加载：复制 ponytail/skills/ponytail/SKILL.md 到此
└── ponytail-source/    ← 模拟 ponytail 仓库的源文件
    ├── AGENTS.md
    ├── instructions.txt
    └── SKILL.md
```

### 2. 执行变更

修改 `ponytail-source/AGENTS.md`（单一修改点），然后查看三个模拟 agent 是否都反映了变更。

### 3. 验证

对比变更前后，三个模拟 agent 的配置内容是否一致同步。

## 预期结论

- 如果 Ponytail 的 AGENTS.md 修改后，三个模拟目录都自动包含新内容 → 架构有效
- 如果有某个模拟目录没有同步 → 说明该 agent 的引用链路需要补齐

## 与 "各自维护一份副本" 的对比

| | 单一源头 + 引用 | 各自维护副本 |
|---|---|---|
| 修改一次 | ✅ 所有 agent 生效 | ❌ 需要逐个改 |
| 新增 agent | ✅ 只需增加一个引用 | ❌ 需要复制一份 |
| 版本管理 | ✅ git revert 源头即可回滚 | ❌ 需要 revert N 个位置 |

## 模拟程序

`./simulate` — 一键运行，自动搭建模拟环境并展示各 agent 从 ponytail 加载的实时内容。

```bash
# 查看当前状态
./simulate

# 新增规则并验证传播
./simulate "- Do not repeat dependency names in error messages."
```

模拟方式：
- **Zed**：符号链接指向 `ponytail/AGENTS.md`（真实 symlink）
- **OpenCode**：调用 `getPonytailInstructions()` 输出到文件（真实函数调用）
- **Hermes**：复制 `ponytail/skills/ponytail/SKILL.md`（真实技能文件）
