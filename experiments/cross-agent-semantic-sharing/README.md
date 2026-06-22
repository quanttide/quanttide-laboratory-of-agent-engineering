# 实验：跨智能体语义共享

## 假设

语义层管理（同一份概念在各工具中以原生方式表达）比结构层管理（统一配置格式）维护成本更低。

## 实验场景

修改 Ponytail 身份提示词——为 "lazy senior dev mode" 新增一条规则：

> "Do not repeat dependency names in error messages."

## 对照方案

| | 语义管理 | 结构统一 |
|---|---|---|
| 做法 | 在各工具的原生位置各自修改 | 先抽象统一格式 + 写转换器生成各工具配置 |
| 本实验执行 | 语义管理方案 | 不执行，仅估算对比 |

## 修改点（语义管理方案）

| 工具 | 修改位置 | 修改方式 |
|------|---------|---------|
| **Zed** | `~/.config/zed/AGENTS.md` Rules 列表 | 直接加一行 |
| **OpenCode** | `ponytail/hooks/ponytail-instructions.js` `getFallbackInstructions` 函数 | 加一条规则文本 |
| **Hermes** | 尚未集成 Ponytail，暂不修改 | — |

## 实验结果
