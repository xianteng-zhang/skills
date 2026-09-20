# skills

个人 Agent / Claude Code 可复用技能集合。每个技能是一个独立目录，包含一份 `SKILL.md`（带 YAML frontmatter 的 `name` 与 `description`）以及可选的 `references/` 补充材料。

## 布局约定

```
skills/
└── <skill-name>/
    ├── SKILL.md          # 必需：frontmatter（name / description）+ 主流程
    └── references/       # 可选：细则、模板、命令库，由 SKILL.md 按需引用
```

## 技能清单

| 技能 | 用途 |
|---|---|
| [`agent-project-review`](skills/agent-project-review/SKILL.md) | 判定一个 Agent 项目是简历级系统还是 Demo：7 支柱 + 11 条清单 + 证据实测 |

## agent-project-review 简介

针对「会调用模型 ≠ 会构建系统」这一判断缺口，基于 @helson《一个好的 agent 项目的特征》7 节与 11 条清单，
设计了三层审查机制（30 秒 Demo 过滤器 → 三条否决项 → 7 支柱 / 11 条清单逐条实测），
实现了对 Agent 项目合格性的可复现定档。

核心口径：

- **先过滤再打分**：命中「只有 Prompt + 脚本 / 无持久化 / `logging` 命中≈0 / 工具硬编码 / 全量历史拼 Prompt」任一，直接初判 Demo。
- **三条否决项**：完整后端工程体系、可观测性 + 自动化评测、真实业务闭环——缺任一即 Demo。
- **证据纪律**：每条判定必须跟 `rg` 命中计数或文件路径；定性判断与实测项分列；未核实的写「未验证」。
- **不用精确总分**：只有 Demo / 合格 / 优秀 三档 + 满足条数。

三个模式：`/agent-project-review`（全量审查）、`quick`（30 秒初判）、`compare`（多项目找互补与真空区）。

配合链路：`/agent-project-review → 补 A/B 类行动项 → /resume-writing → /interview`。

## 安装

把技能目录放到对应工具的 skills 路径下即可：

```powershell
# DSH / 类 Claude Code 环境：项目级
Copy-Item -Recurse skills/agent-project-review <workspace>/.dsh/skills/
```

## License

MIT
