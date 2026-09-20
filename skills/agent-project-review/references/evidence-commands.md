# 证据采集命令库

原则：**能实测的一律实测，判定必须写计数或文件路径。** 命令是 PowerShell / ripgrep，按项目类型挑用。

`rg` 不可用时用 `Get-ChildItem -Recurse -Include *.py,*.ts | Select-String -Pattern ...`，或直接用 grep 工具。

---

## 0. 项目轮廓（先跑这组，30 秒定性）

```powershell
# 目录规模
Get-ChildItem -Recurse -File -Include *.py,*.ts,*.js,*.tsx,*.java,*.go |
  Measure-Object | Select-Object Count        # 代码文件总数

# 顶层结构（有没有分层，还是一堆平铺文件）
Get-ChildItem -Directory | Select-Object Name

# 有没有数据库层
Test-Path app/models, app/db, models.py, alembic.ini, prisma/schema.prisma, migrations

# 有没有容器与部署
Test-Path Dockerfile, docker-compose.yml, nginx.conf, .github/workflows
```

判定参考：**全仓库只剩几个文件、没有 models / 迁移 / workflows → 高度疑似 Demo。**

---

## 1. 后端工程体系（第 2 条）

```powershell
# 分层证据
Test-Path app/services, app/routers, app/repositories, app/schemas, app/core

# 异常与配置
rg -n "class .*Exception|HTTPException|handle_exception" --stats
rg -n "Settings|BaseSettings|pydantic_settings|dotenv|@Configuration" --stats

# 鉴权与并发
rg -n "Depends|jwt|JWT|oauth|bcrypt|passlib|permission|require_role" --stats
rg -n "asyncio|async def|ThreadPool|Semaphore|rate_limit|slowapi" --stats
```

## 2. 持久化（第 3 条）

```powershell
rg -n "CREATE TABLE|Table\(|Column\(|mapped_column|prisma\.|models\.Model" --stats
rg -n "session|conversation|message|history" --glob "*model*" --stats
```

要点：看的是**会话/消息/任务/工具调用记录**这四类有没有落地，不是"有没有装 ORM"。

## 3. 任务状态管理（第 4 条）

```powershell
rg -n "class .*State|TypedDict|Enum.*status|status.*=.*pending|progress" --stats
rg -n "celery|arq|rq|BackgroundTasks|APScheduler" --stats
```

## 4. Agent 能力（第 5 条）—— 最关键的实测

```powershell
rg -n "@tool|bind_tools|tool_choice|tools=|ToolNode|ToolRegistry|FunctionTool" --stats   # 工具
rg -n "planner|plan_|decompose|subtask|break.*down|todo_write|task_list" --stats          # 规划
rg -n "retry|tenacity|max_retries|fallback|circuit" --stats                              # 失败重试
rg -n "critic|reflexion|self_check|verify|validate|reviewer|judge" --stats                # 反思/校验
rg -n "memory|ConversationBuffer|summary|vector_store|semantic_memory" --stats            # 记忆
rg -n "intent|classify_|router|route_|dispatch" --stats                                   # 意图识别
```

**判读**：命中 0 的能力直接判"缺"。命中 >0 还要看是否接进主链路（只被 import 没被调用不算）。
特别注意：`@tool` 0 命中但声称"支持工具调用" → 写进矛盾清单。

## 5. 上下文工程（第 6 条）

```powershell
rg -n "trim|truncate|slice.*history|last_n|summary|summarize|compress|token_budget|max_tokens" --stats
rg -n "system_prompt|messages\.append|history\s*\+" --stats     # 反面信号：把全部历史直接拼进去
rg -n "user_profile|preference|long_term|profile" --stats       # 用户记忆
```

反面信号明确时判"上下文工程缺失"：**`messages = history + [user_input]` 全量拼接且无裁剪。**

## 6. 可观测性（第 7 条）

```powershell
rg -n "logging|logger|getLogger|loguru|structlog" --stats
rg -n "langfuse|Langfuse|trace|span|opentelemetry|phoenix|langsmith" --stats
rg -n "token|usage|prompt_tokens|completion_tokens|cost" --stats
rg -n "timeout|time\.time|latency|duration|elapsed" --stats
rg -n "class .*Log.*\(|log_table|request_log|audit" --stats
```

**判读**：`logging` / `logger` 命中 0 → 第 7 条直接 ❌。有 logger 但无 token / 无 latency / 无 trace id → 只能判 ⚠️。

## 7. 评测体系（第 8 条）

```powershell
Test-Path eval.py, evals, evaluation, tests/golden, benchmarks
rg -n "golden|gold_standard|dataset|testset|jsonl" --stats
rg -n "threshold|min_score|assert.*score|sys.exit|exit\(" --glob "*eval*" --stats
rg -n "judge|LLM.*as.*judge|score|rubric|faithfulness|recall|hallucinat" --stats
Get-ChildItem .github/workflows -ErrorAction SilentlyContinue
```

**判读**：有 eval 脚本但无文件门槛、不在 CI、case 少于 10 条 → ⚠️ 或 ❌，不能算"体系"。
额外看一眼 judge 设计：**一次调用同时评两份答案 = 位置偏见**；无 rubric 的 0–10 打分为弱。

## 8. 人机协同（第 9 条）

```powershell
rg -n "interrupt|Command\(resume|checkpoint|human_in_the_loop|hitl|approve|approval|confirm" --stats
rg -n "confirm|are you sure|review_required|manual_review|escalate|transfer" --stats
```

**判读**：0 命中 → ❌。只有前端弹窗提示、没有真正的暂停-确认-恢复机制 → 只能算半个。

## 9. 架构解释与收益（第 10、11 条）

```powershell
Test-Path DESIGN.md, ARCHITECTURE.md, docs/design.md
rg -n "为什么|选型|trade-?off|替代方案|instead of|rather than" README.md docs -i --stats
rg -n "baseline|对比|提升|降低|节省|成本|人力" README.md docs -i --stats
```

## 10. 仓库卫生（D 类硬伤，与标准无关但影响可查性）

```powershell
git status --short | Measure-Object -Line        # 未提交文件数
git log -1 --date=short --format="%ad %s"        # 最后提交时间
Test-Path LICENSE, .github/workflows, .gitignore
Get-ChildItem -Filter "*.png","*.jpg" -Recurse | Measure-Object   # 有没有 demo 截图
```

**判据 1–11 全部建立在"面试官能点开你的代码"上**：没 push、没 LICENSE、没 README 截图，等于前面做的一切不可验证。

---

## 命令结果的呈现纪律

| 情况 | 怎么写 |
|---|---|
| 命中计数 | 写进证据列：`` `@tool` 0 命中、`bind_tools` 0 命中 `` |
| 文件存在性 | 写路径：`app/modules/{auth,kb,chat,admin}` 各含 router+service+repositories+schemas` |
| 定性判断 | 显式标注：`（定性：反思只有 2 次重试，无 self-critique）` |
| 拿不到 | 写 `未验证`，并说明为什么（无运行时环境 / 无线上数据） |

**不要把定性判断写进"实测证据"列。** 这份报告的价值全在"哪句能拿去对质"。
