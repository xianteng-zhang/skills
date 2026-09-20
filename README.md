# skills

个人可复用技能库（Claude Code / DSH 风格）。每个技能是一个自包含目录，放入对应工具的 skills 路径即可生效。

技能不只是「提示词模板」——这里的每个技能都带**判据、证据纪律和输出契约**，目标是让判断过程可复现、可对质，而不是让模型自由发挥。

## 目录结构

```
skills/
├── README.md
├── .gitattributes
└── <skill-name>/
    ├── SKILL.md          # 必需：YAML frontmatter（name / description）+ 主流程
    └── references/       # 可选：细则、模板、命令库，由 SKILL.md 按需引用
```

**约定**

| 项 | 规则 |
|---|---|
| 目录名 | 与 frontmatter 的 `name` 一致，小写连字符 |
| frontmatter | 必须有 `name`、`description`；`description` 要写清**触发条件**（用户会怎么说话） |
| 拆分标准 | 主流程留在 `SKILL.md`，细则/模板/命令库拆进 `references/`，按需读取而非一次性注入 |
| 换行符 | 仓库内统一 LF（见 `.gitattributes`），技能含大量命令示例，CRLF 会污染 diff |

## 技能清单

| 技能 | 一句话 | 入口 |
|---|---|---|
| [`agent-project-review`](skills/agent-project-review/SKILL.md) | 判定一个 Agent 项目是简历级系统还是 Demo | `/agent-project-review` |

---

## agent-project-review

### 解决什么问题

「会调用模型」和「会构建系统」是两件事，但简历项目往往只有前者。审查时的常见困境是：**手里只有七条模糊标准（"要有完整后端""要有可观测性"），既打不出分数，也说不清差在哪一条。** 更麻烦的是 README 的自我描述不能当证据——写了"支持多轮会话"不代表代码里有会话表。

### 三步审查链路

```text
        ┌─────────────────────────────────────────────┐
        │ ① 30 秒 Demo 过滤器（5 条硬信号）           │
        │ 命中任一 → 直接初判 Demo，不再往下打分       │
        └────────────────────┬────────────────────────┘
                             ↓  未命中
        ┌─────────────────────────────────────────────┐
        │ ② 三条否决项（资格线，缺一即 Demo）         │
        │   后端工程体系｜可观测+评测｜真实业务闭环    │
        └────────────────────┬────────────────────────┘
                             ↓  全部通过
        ┌─────────────────────────────────────────────┐
        │ ③ 7 支柱 + 11 条清单逐条实测                │
        │   每条判定必须跟 rg 命中计数或文件路径       │
        └────────────────────┬────────────────────────┘
                             ↓
        Demo / 合格 / 优秀  +  矛盾清单  +  分级行动项
```

### 三个模式

| 模式 | 触发 | 输出 |
|---|---|---|
| **Audit**（默认） | `/agent-project-review` | 全量：过滤器 + 否决项 + 7 支柱 + 11 条清单 + 行动清单 |
| **Quick** | `/agent-project-review quick` | 只跑过滤器与否决项，给"能不能写简历"的初判 |
| **Compare** | `/agent-project-review compare` | 多项目并排，找互补与真空区 |

### 档位判定

| 档位 | 条件 |
|---|---|
| **Demo** | 命中任一过滤器；或三条否决项缺任一；或 11 条满足 ≤ 3 条 |
| **合格** | 三条否决项全在；11 条满足 4–7 条；Agent 能力不低于"部分" |
| **优秀** | 三条否决项全在；11 条满足 ≥ 8 条；且评测体系达标 |

### 四条纪律

1. **证据优先于描述** —— README 声称但代码里找不到对应机制，一律按缺失计，并进「矛盾清单」。
2. **实测与定性分列** —— `rg` 计数、文件路径属实测；"反思偏弱"这类属定性，显式标注，不混写。
3. **偏严** —— 没证据即缺失；拿不到的写「未验证」，绝不推测数值。
4. **不用精确总分** —— 只有三档 + 满足条数。"综合 78 分"这类数字没有校准依据。

### 文件

| 文件 | 内容 |
|---|---|
| [SKILL.md](skills/agent-project-review/SKILL.md) | 主流程：模式、过滤器、否决项、7 支柱、11 条清单、档位 |
| [references/rubric.md](skills/agent-project-review/references/rubric.md) | 标准原文口径 + 11 条判据判定细则 + 第 5/8 条分级 + workflow 项目专门口径 |
| [references/evidence-commands.md](skills/agent-project-review/references/evidence-commands.md) | 按判据组织的 `rg` / `Test-Path` / `git` 证据采集命令库 |
| [references/report-template.md](skills/agent-project-review/references/report-template.md) | 审查报告模板与写作纪律 |

## 安装

把技能目录复制到目标环境的 skills 路径：

```powershell
# 项目级（DSH / Claude Code 风格）
Copy-Item -Recurse skills/agent-project-review <workspace>/.dsh/skills/

# 用户级
Copy-Item -Recurse skills/agent-project-review $env:USERPROFILE/.dsh/skills/
```

## 新增技能

1. 建目录 `skills/<skill-name>/`，写 `SKILL.md`（frontmatter 的 `description` 必须包含触发条件）；
2. 超过一屏的细则、模板、命令库拆到 `references/`，在 `SKILL.md` 里按需引用；
3. 在本文件「技能清单」表里加一行；
4. 开分支提交，走 PR 合并到 `main`。