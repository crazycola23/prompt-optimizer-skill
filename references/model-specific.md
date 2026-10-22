# 模型专属提示词要点（Model-Specific）

来源：Anthropic 官方各模型 Prompting 专页（URL 见 `SKILL.md` 第 5 节）。快照日期：2026-10-22。
用法：先在第 1 节定位目标模型 → 读该模型章节；第 2 节是**跨模型共性陷阱**，任何模型改写前都先扫一遍。

## 1. 路由表

| 模型 | 最该记住的一条差异 | 章节 |
|---|---|---|
| Claude Fable 5.1 / Mythos 5.1 | 长工具链里默认**少发**用户可见进度更新，要显式要求；effort 档位语义再次重标定 | §3.1 |
| Claude Fable 5 / Mythos 5 | 回合可长达数分钟、自主运行数小时，harness 要改异步；指令遵循强到可用一句话替代逐条枚举 | §3.2 |
| Claude Sonnet 5.5 | thinking 计入 `max_tokens`（上限 128k）；中途改 effort 要用 per-message 而非顶层字段 | §3.3 |
| Claude Sonnet 5 | 默认开启 adaptive thinking；`temperature`/`top_p`/`top_k` 一律 400；指令遵循更**字面** | §3.4 |
| Claude Opus 5.5 | effort 默认 `medium`；thinking **恒开不可禁用**；纯文本 `end_turn` 不等于任务完成 | §3.5 |
| Claude Opus 5 | 可见回复长度只能用 prompt 控制（effort 控的是 thinking）；**删掉**验证/自检类指令 | §3.6 |
| Claude Opus 4.8 | thinking 默认**关闭**，要显式开 adaptive；工具调用倾向偏低 | §3.7 |
| Claude Haiku 5.5 | 先按症状改、不要整体重写；搜索要配 `{{current_date}}`；thinking 默认开启并计入 `max_tokens` | §3.8 |

## 2. 跨模型共性陷阱（改写任何提示词前先扫）

- **effort 档位跨模型不等价**：`low`/`medium`/`high`/`xhigh`/`max` 在 Fable 5.1、Sonnet 5.5、Opus 5.5 上都重新标定过。迁移或换模型时必须重跑自家 evals，别照搬旧值；默认值也各不相同（Opus 5.5 默认 `medium`，Opus 5 默认 `high`）。
- **thinking 计入 `max_tokens`**：Sonnet 5.5 / Sonnet 5 / Haiku 5.5 / Opus 5.5 / Opus 5 上沿用旧模型的 `max_tokens` 会截断回复。Sonnet 5.5 / Haiku 5.5 / Opus 5.5 的文档写明 128,000 是模型上限，长 agentic 任务可直接给到该值并开 stream；Sonnet 5 的文档只要求为 thinking 留出余量、未给数值上限，按其实际上限设置。`stop_reason: "max_tokens"` 一律当失败重试，即使内容看起来合法。
- **中途改 effort 不要改顶层字段**：改顶层 `effort` 会作废 prompt cache；用 per-message effort change（beta）。该用法需要 adaptive thinking，配 `between_tools` 或 thinking 关闭会 400。
- **删掉为旧模型写的脚手架**：`After every 3 tool calls, summarize progress`、`If in doubt, use [tool]`、`CRITICAL: You MUST…`、`double-check your answer` 在新模型上要么多余、要么造成过度触发/过度验证。
- **不要要求模型复述推理**：要求把 thinking / reasoning 写进正文的 prompt、skill 或 tool description 可能触发 `reasoning_extraction` 类别拒答（`stop_reason: "refusal"`），且服务端 `fallbacks` 不重试该类别。改要简短解释或动作摘要，或读 `display: "summarized"` 的 thinking block。
- **进度更新可能藏在 thinking block 里**：默认 `display: "omitted"` 时这些 block 文本为空，只渲染 `text` block 的客户端会误判为"模型变沉默了"。要 `display: "updates"`（beta header `thinking-display-updates-2026-08-18`）。
- **代码审查类提示词的过滤指令会被字面执行**：`only report high-severity issues` / `be conservative` / `don't nitpick` 会让模型调查深度不变、报告量下降（recall 掉、precision 涨），看起来像能力回归。正确写法：finding 阶段要求**全量上报**并附 confidence 与 estimated severity，过滤交给独立的 verification/ranking 阶段。
- **结构化输出 + thinking 关闭 = 漏工具调用**：JSON 格式约束下模型可能不调工具直接作答。解法：改用 adaptive thinking；或从必须调工具的请求里移除 `output_config.format`；或用 `tool_choice` 强制调用。
- **设计类需求不要用通用否定指令**：`别用那个颜色` / `make it clean and minimal` 只会把模型推到另一套固定默认风格。要么给具体规格（色调系统、字体、圆角、动效参数写死），要么先让它提 4 个方向再由用户选。
- **用户中途输入不要放进 `tool_result`**：模型被训练抵抗经 tool result 的间接注入，会把你真实的中途消息当不可信文本忽略。应作为最后一个 `tool_result` 之后的 text block 追加到同一条 user message；harness 通知另放 mid-conversation system message。

## 3. 各模型细则

### 3.1 Claude Fable 5.1 / Claude Mythos 5.1

#### 要点
- **重跑 effort 扫描** — `low`/`medium`/`high`/`xhigh`/`max` 的名称在不同模型间不等价，从默认 `high` 起步对自家 evals 重测；能力增益在高档位最大，`medium` 约等于 Fable 5 的更低成本，`low` 常可与 Opus/Sonnet 比 cost per task。
- **补进度更新** — 默认行为比 Fable 5 更少在长工具链中输出用户可见文本，用户会看到长时间静默；先确认客户端拿到了 progress-update `thinking` block，默认 `thinking.display` 为 `"omitted"` 时这些 block 是空的。
- **删掉抑制叙述的旧指令** — 为旧模型写的 “hold all findings for the final response” 之类系统提示行会压掉更新，先删再加新指令。
- **工具输出不可见要告知模型** — 若 UI 折叠/隐藏 tool output，需在 `tool_result` 后跟一条 turn-scoped system message 说明“只有你看得到输出”，否则模型会反复跑命令试图“展示”用户看不到的内容。
- **独立 tool call 要批量** — 编码/computer-use 循环里，隐含的下一步独立调用常被拆成一回合一个，质量不变但多付 token、往返和时延；在当次用户消息末尾加一句批量提示即可。
- **历史保持 append-only** — 2026-08-31 后新建账号的 thinking block 只在其产生的原对话中有效；改动前缀（system prompt、tools、更早消息）会返回 400，或配合 `thinking.block_binding.prefix_mismatch_behavior: "drop_block"` 丢弃这些 block。
- **别原地改写历史** — 注入再删除每轮提醒、就地总结旧回合、会话中改 `system`，都会重启 prompt cache；应改用 turn-scoped / mid-conversation system message，并让服务端 compaction 或 context editing 做裁剪。
- **压缩点可后移** — cache 读取变便宜后，为省钱提前 compaction 未必再是划算的取舍；客户端 compaction 最简单的形状是用一条 summary 加新用户回合整体替换历史，不重放任何东西。
- **控写作密度** — prose 比 Fable 5 更密、句更长、段落更少；给出定义 anti-pattern 的 “mannered prose” 指令（或短的 “Please remove all mannered prose.”）比泛泛要求简洁有效。
- **撤销反格式化规则** — 它比旧模型更少用 bold、列表和标题，而旧 prompt 里常带为旧模型写的 anti-formatting 规则；应改为说明何时该用格式化。
- **引用要显式标注** — 总结文档时更爱直接复述原文而不加引号；在 system prompt 放一个完整正确示例（请求 + 回复 + 为什么对）最有效。
- **要求做完整个任务** — 复杂异步负载下会以 “Next, I'll …” 描述下一步或问 “Shall I apply this?” 就结束回合；需同时加自主性指令与 scope 指令，前者保留大部分效果。
- **压缩摘要要指定保留项** — 客户端 compaction 的总结指令必须显式要求保留难点及处理、被搁置的方案、已决定/排除的约束、当前进度、待办、以及难重建的具体细节。
- **改测试只改任务要求的** — 会顺手修邻近 bug、扩展未提及行为、提交超出改动所需的测试文件；显式说明“什么不要做”可大幅减少附加物且任务成功率不降。
- **低 effort 要显式要求搜索** — `low` 下更少调用搜索/检索工具而凭记忆作答；可只对该回合提高 effort，或加“认出名字不等于知道其当前状态”的提示。
- **降 safeguard 误报** — 误报返回 `stop_reason: "refusal"`；三个高发场景是 compile-check 措辞、冷门编程语言、tool output 里的 base64，分别改为问 “Are there any bugs in this program?”、补语言文档上下文、移除 base64。
- **改小文件用定点编辑** — 比 Fable 5 更容易为小改动重写整个文件，输出 token 与时间都更贵；加一句“token 最小化、能定点编辑就不要整体重写”即可对齐旧行为。
- **`xhigh`/`max` 给长输出留空间** — 会先在 thinking 里起草长交付物、再在回复里重写一遍；优先用 `high`，必须用高档位时把 `max_tokens` 按 thinking + 回复一起留足，并附上禁止重复起草的说明。
- **别让 lead agent 干等** — 让启动 subagent 的工具立即返回、结果以之后的 `user` 消息回传、另给一个 wait 工具；省时来自 lead 继续干别的那些 run。
- **视觉任务配裁剪工具** — 密集图表上最好在容器里跑 agent（预装 PIL/OpenCV 等）以迭代分析、裁剪、验证；退一步只给图像裁剪工具也能拿到大部分收益。
- **别要求复述推理** — 让模型写出 thinking/reasoning 的 prompt、skill 或 tool description 可能触发 `reasoning_extraction` 拒答；改为要简短解释或动作摘要，或读 `display: "summarized"` 的 summarized thinking block。

#### 可复用提示词片段
```text
Before you start, say in a line what you're about to do; brief updates while you work help the user follow along. Close with a short recap that stands on its own — what you found, what you did, and what's next — so a reader who only sees the last message has the full picture.

First privately list what you need next; then request every item that doesn't depend on another's result in this one response.
```
```text
You are operating autonomously. The user is not watching in real time and cannot answer questions mid-task, so asking 'Want me to…?' or 'Shall I…?' will block the work. For reversible actions that follow from the original request, proceed without asking. Stop only for destructive actions or genuine scope changes the user must decide.
Before ending your turn, check your last paragraph. If it is a plan, an analysis, a question, a list of next steps, or a promise about work you have not done ('I'll…', 'let me know when…'), do that work now with tool calls.
```
```text
The number of tokens used to edit files is best minimized, all else being equal. Therefore, when it will not affect the end result, try to surgically edit a file rather than rewrite the entire thing.
```

#### 常见坑
- 把 `medium`/`low` 当成“能力不足”的档位而一律跑 `high`：常规工作在高档位会做超出需要的上下文收集与整理，质量未必更高但成本与等待明显上升。
- 直接沿用 Fable 5 的 effort 档位或旧 prompt/skill：档位语义已变，为旧模型写的过度规定式 skill 会拉低输出质量，应先评估默认行为再决定是否删。
- 用改写 system prompt、注入并删除每轮提醒、就地总结历史来“维护上下文”：既触发 thinking block 的 400/丢弃，又清空 prompt cache。
- 客户端只渲染 `text` block：模型其实在发进度更新，用户侧仍显示长时间静默，于是团队误判为模型行为退化。
- 用 base64 把二进制塞进上下文，或用 “Does this program compile without errors?” 这类措辞探测代码：都会提高 benign 请求被 `stop_reason: "refusal"` 挡掉的概率。

### 3.2 Claude Fable 5 / Claude Mythos 5

#### 要点
- **接受更长的回合** — 高 effort 下单个请求可跑数分钟，自主运行可延续数小时；迁移前调整客户端 timeout、streaming 与进度指示，并把 harness 改为异步查看（如 scheduled jobs）而非阻塞等待。
- **压制过度规划** — 信息足够时就行动，不要重复推导已确立事实、重开用户已定的决策、或在用户可见消息里铺陈不会采用的方案；判断权衡时给推荐而非穷举。
- **从 `high` 起调 effort** — `high` 为多数任务默认，能力敏感用 `xhigh`，常规工作用 `medium`/`low`；低档位常已超过旧模型 `xhigh`，任务能完成但耗时过长就降档。
- **禁止顺手重构** — 高档位下会做超出任务需要的上下文收集与整理；显式禁止额外 feature、抽象、未来需求的预留、不可能场景的错误处理与向后兼容 shim。
- **用一句话而非穷举行为来引导** — 指令遵循强到可用简短指令替代逐条枚举；过度展开（铺陈不采用的选项、长篇根因、重结构化的 PR 描述、逐行注释）用 brevity 指令即可抑制。
- **定义何时暂停** — 只需在破坏性/不可逆动作、真实 scope 变化、或只有用户能提供的输入时暂停并结束回合，不要以承诺结尾，无需枚举每种情况。
- **长跑中核对进度声明** — 报告进度前逐条对照本会话的 tool result，只报告有证据的工作，未验证的明说；测试失败就带输出说明，跳过某步就说明。
- **划清边界** — 偶尔会做未被请求的动作（擅自起草邮件、创建防御性 git 分支备份）；区分“描述问题/提问/自言自语”场景交付评估而非直接动手，并在改变系统状态的命令前确认证据真的支持该动作。
- **并行 subagent 与异步通信** — 派发并行 subagent 比旧模型积极，应经常使用、给出何时该委派的指引，orchestrator 与 subagent 之间走异步而非阻塞等待；长生命周期 subagent 靠 cache 读取省时省钱。
- **给模型建记忆系统** — 它在能记录并复用历史经验时表现突出；提供写笔记的位置（一个 Markdown 文件即可），一个文件一条教训、顶部一行摘要，同时记录纠正与已验证做法，避免重复或保存仓库/聊天已记录的内容；可用 subagent 复盘既有会话自举出初始教训库。
- **兜底早停** — 长会话深处偶尔只输出 “I'll now run X” 而不发对应 tool call，或在已有足够信息时停下问许可；加自主运行 system reminder 可覆盖，配合上面的暂停规则。
- **别暴露上下文预算计数** — 极长会话中它可能建议新开会话、提出总结交接或自行裁剪工作，常由 harness 显示剩余 token 倒计时触发；必须显示时加一句“上下文充足，不要因此停下或建议新会话”。
- **给理由而不只给请求** — 说明意图与上下文（为谁做、产出要支撑什么）能让它把任务接到相关信息上而不是自行猜测；对跨工作流的长跑 agent 尤其重要，事后纠正比写进 prompt 更费回合。
- **最终消息要重新落地** — 长 agentic 会话里它可能用密集箭头链、深层实现细节或用户没见过的 thinking 引用来写总结；要求最终总结首句给结果、用完整句子、展开术语、给每个文件/提交/flag 单独一句平实说明。
- **加 send-to-user 工具** — 长异步 agent 需要一个能把消息逐字呈现且不结束回合的工具（交付物、带具体数字的进度、回答用户中途提问）；只定义工具不够，必须在 system prompt 配一句 elicitation 指令，并禁止用它传叙述或推理。
- **scaffolding 调整** — 从难度区间上沿挑任务；长跑 prompt 里让验证显式化（独立、全新上下文的 verifier subagent 优于自我批判，按固定间隔对照规格验证）；重构为旧模型写的 prompt 与 skill。
- **别要求模型复述推理** — 要求 echo/transcribe/解释内部推理的 prompt、skill 或 harness 指令可能触发 `reasoning_extraction` 拒答，且 `fallbacks: "default"` 不会重试该类别；改用 adaptive thinking 的结构化 `thinking` block，或要简短解释/动作摘要。

#### 可复用提示词片段
```text
When you have enough information to act, act. Do not re-derive facts already established in the conversation, re-litigate a decision the user has already made, or narrate options you will not pursue in user-facing messages. If you are weighing a choice, give a recommendation, not an exhaustive survey. This does not apply to thinking blocks.
```
```text
Lead with the outcome. Your first sentence after finishing should answer "what happened" or "what did you find": the thing the user would ask for if they said "just give me the TLDR." Supporting detail and reasoning come after. Being readable and being concise are different things, and readability matters more.
```
```text
Before reporting progress, audit each claim against a tool result from this session. Only report work you can point to evidence for; if something is not yet verified, say so explicitly. Report outcomes faithfully: if tests fail, say so with the output; if a step was skipped, say that; when something is done and verified, state it plainly without hedging.
```
```text
Store one lesson per file with a one-line summary at the top. Record corrections and confirmed approaches alike, including why they mattered. Don't save what the repo or chat history already records; update an existing note rather than creating a duplicate; delete notes that turn out to be wrong.
```

#### 常见坑
- 直接套用旧模型的重脚手架：为旧模型写的 skill 与逐条行为枚举往往过于规定式，会压低 Fable 5 的输出质量，应重评哪些 guardrail 与指令还需要。
- 按旧模型预期设计同步阻塞的 harness：单请求数分钟的等待会拖垮体验与超时设置，需改异步查看。
- 在长跑里相信模型自述的进度：不加“对照 tool result 审计”的指令会出现编造的 status report。
- 让模型在回复文本里复述推理：触发 `reasoning_extraction` 拒答，且 server-side `fallbacks: "default"` 不重试，请求会直接返给你。

### 3.3 Claude Sonnet 5.5

#### 要点
- **重扫 effort** — 档位重新标定，与 Sonnet 5 同名档位思考量不同，须对自家 evals 重新扫描；默认 `high` 起，agentic coding 的明确任务从 `medium` 起、更难更长再上 `high`，聊天等时延敏感场景从 `medium`/`low` 起。
- **给 `max_tokens` 留出 thinking** — thinking 计入 `max_tokens` 即使内容不回传；按无 thinking 的请求设限会截断回复，agentic coding 建议设到模型上限 128,000 并 stream。
- **高档位要付代价** — `xhigh`/`max` 思考与回复都显著变长，且这两个档位不接受 `between_tools`，无法关掉 up-front thinking；想少思考就降 effort，在 system prompt 里写“少想”不可靠。
- **中途改 effort 用 per-message 而非顶层字段** — 改顶层 `effort` 会作废 prompt cache；用 per-message effort change（beta）可保 cache，改动持续生效到下一次替换，且需要 adaptive thinking，配 `between_tools` 会 400。
- **低 effort 会把活干完再走** — `low`/`medium` 下 agentic coding 可能中途停下确认计划、自问自答或只做多步任务的一部分；先试提高 effort，或加“干完再停”的系统提示（代价是会话更长更贵）。
- **压掉未被请求的附加物** — 各档位都会补测试、文档和小文件，高档位更多；只保留提示词第二段即可把改动收窄，`xhigh`/`max` 下还会减少自发的多轮 review 与 reviewer subagent（测试中省约三分之一成本、质量不变）。
- **开放式请求先要方案** — 容易被 “show me what you can do” 触发直接开始做演示/报告/视频；明确“给想法、选项或计划后停下，得到许可再动手”。
- **不用 up-front thinking 时改用 `between_tools`** — 它是最低思考设置，仅在 `high` 及以下被接受；此时要删掉所有“不要思考”的指令（否则模型会把内部 XML 标签写进可见输出），按 block type 读响应，并把 `thinking` block 原样回传（回传能拿到完整笔记而非摘要）。
- **无工具的推理任务不要用 `between_tools`** — 无 tools 时它等于不思考直接作答，需要几步推算的任务应改用 adaptive thinking。
- **JSON 输出要显式要求先思考** — structured outputs 下响应文本只有 JSON，思考只能在 thinking 里做，跳过思考会掉准确率；在 system prompt 末尾加 “Think the problem through before you answer.”，`high` 下接近 `xhigh` 的准确率。
- **`max_tokens` 截断一律视为失败** — `low`/`medium` 配 structured outputs 偶尔会一路思考到 `max_tokens`；只要 `stop_reason` 是 `"max_tokens"`，即使文本是合法 JSON 也重试。
- **非 structured outputs 时取最后一个 JSON 值** — 只读 `text` block，从每个 `{`/`[` 起尝试解析，成功则从该值末尾继续，保留最后一个值；不要取首个 `{` 到末个 `}`（可能把草稿和最终 JSON 一起框进去）。
- **进度更新要渲染 thinking block** — 超过一两句的注释以 progress-update `thinking` block 返回，默认 `display` 下文本为空，只渲染 `text` 的客户端会显得全程静默；设 `display: "updates"`（beta header `thinking-display-updates-2026-08-18`），`between_tools` 则自带摘要文本且不接受 `display`/`budget_tokens`/`block_binding`。
- **沉默太久由 harness 催更** — 统计连续未给用户任何文本的 tool-calling 步数，达到若干步后把一句提醒作为 turn-scoped system message 追加在最新 tool result 之后；同一提醒不要发超过两三次，且不要删除已发的副本。
- **聊天/知识工作要显式要求搜索** — 会凭训练知识回答本可搜索确认的“是否允许/必须/收费”类细节；先删掉 “only use tools when strictly necessary”“minimize tool calls” 这类措辞，再加搜索指令。
- **中途用户消息按用户回合投递** — 训练上抗间接注入，可能把真实用户消息误判为注入；绝不把用户文本放进 `tool_result`，应作为 `tool_result` 之后该用户消息里的 text block，harness 通知另放独立的 mid-conversation system message。
- **不要在 tool result 后加 token 倒计时** — 每个 tool result 后都追加文本（倒计时或逐步指令）会让模型把用户消息当注入并忽略或要求确认；task budgets 的倒计时未见此问题。
- **低 effort 要显式要求真实验证** — `low` 下可能没跑能覆盖改动的检查就报告完成（例如因依赖未安装而跳过项目测试）；加一段“跑项目测试/类型检查/构建或改动后的命令，纯语法检查或启动失败的检查不算”的提示，漏检几乎消失、成本略增。
- **工具调用名/参数容错** — 偶尔用大小写不同的名字（`bash` 对 `Bash`）或近似参数名；harness 可在匹配唯一时接受，或返回 `is_error: true` 并写明期望的确切名称，模型通常下一轮自行纠正。
- **视觉任务给裁剪/缩放/跑代码工具** — 图表在每个 effort 档位都受益，技术图纸只在 `high` 及以上受益（`xhigh`/`max` 最明显）；图表场景加工具比提 effort 更划算（测试中 `high` + 工具优于 `max` 无工具）。
- **拒答按 category 处理** — 拒答是带 `stop_reason: "refusal"` 的正常响应，`stop_details.category` 取 `cyber`/`bio`/`frontier_llm`/`reasoning_extraction`/`general_harms`；源码中找漏洞是允许的，高风险 dual-use 网络安全工作不允许；server-side fallback 只重试 `cyber` 与 `frontier_llm`。

#### 可复用提示词片段
```text
Keep working until everything the user asked for is done, and only stop to ask when you can't go on without the user or before a risky step.

When the work the user asked for is done and checked, stop and report. Don't add features, tests, files, docs or refactors that weren't asked for. If you think one would help, mention it at the end instead of doing it.
```
```text
Think the problem through before you answer.
```
```text
Use the search tool to check specifics that may have changed since your training, such as what is allowed, required or charged, even when you feel confident. For researched work such as a report or a comparison, gather current sources rather than writing from your training knowledge.
```

#### 常见坑
- 沿用 Sonnet 5 的 effort 档位与 `max_tokens`：思考量已变，且 thinking 计入上限，回复会被截断。
- 把“少思考”写进 system prompt 来替代降 effort：从 `medium` 起模型几乎每次回复前都会先思考，提示词不可靠地降低思考量，正确杠杆是 effort 档位。
- 用顶层 `effort` 字段中途改档：作废 prompt cache；配 `between_tools` 或 `xhigh`/`max` 还会直接 400。
- 把结构化输出的 `stop_reason: "max_tokens"` 响应当成功解析：低/中档位下这类响应实际是被截断的失败尝试。
- 在 tool result 之后追加 token 倒计时或每步指令，或把用户文本塞进 `tool_result`：触发模型把真实用户消息当注入而忽略或要求确认。

### 3.4 Claude Sonnet 5

#### 要点
- **长度按复杂度自适应** — 不再固定 verbosity：简单查询更短、开放分析更长；产品依赖固定风格时必须显式加简洁指令，而不是靠默认值。
- **effort 默认 `high`** — 最难编码/agentic 任务提到 `xhigh`；迁移时按实际 thinking 长度对齐而非按 effort 名称（Sonnet 5 `medium` ≈ Sonnet 4.6 `high`）。
- **低 effort 严格收窄范围** — `low`/`medium` 只做被要求的事，中等复杂任务有 under-thinking 风险；正确做法是提 effort，而非用 prompt 绕过。
- **adaptive thinking 默认开启** — 不带 `thinking` 字段即开启（与 Sonnet 4.6 相反）；关闭需显式传 `thinking: {type: "disabled"}`；thinking 触发频率可用 system prompt 引导。
- **手动 extended thinking 已移除** — `thinking: {type: "enabled", budget_tokens: N}` 返回 400；改用 adaptive thinking + effort。
- **重算 `max_tokens`** — thinking 与回复共享 `max_tokens`；新 tokenizer 对同文本约多 30% token，Sonnet 4.6 的限额会截断；紧预算会得到"几乎全是 thinking + 截断回复 + `stop_reason: "max_tokens"`"。
- **默认更 agentic** — 更主动调工具并跑自验证循环；thinking 关闭时反而少调工具，需在 system prompt 显式 nudge；`high`/`xhigh` 的工具使用显著更多。
- **删掉进度更新脚手架** — 模型自带高质量用户进度更新，旧的 "After every 3 tool calls, summarize progress" 应移除；形态不合预期时显式描述并给正例。
- **指令遵循更字面** — 不把一条指令静默泛化到其他条目，也不推断未提出的请求；要泛化必须显式写范围（如 "Apply this formatting to every section, not just the first one"）。
- **`temperature`/`top_p`/`top_k` 报 400** — Sonnet 类新约束；风格多样性改用 system prompt 引导，或让模型先提多个方向再由用户选。
- **前端/设计会固化默认风格** — 泛泛的"别用那个颜色""简洁一点"只是换到另一套固定调色板；要么给具体色彩/字号/圆角规格，要么先让模型提 4 个方向。
- **代码审查的召回陷阱** — "only report high-severity issues"/"be conservative" 会被字面执行：调查深度不变但报告变少；finding 阶段要求全量上报 + 置信度/严重度，过滤交给独立阶段。

#### 可复用提示词片段
```text
Provide concise, focused responses. Skip non-essential context, and keep examples minimal.
```
```text
Thinking adds latency and should only be used when it will meaningfully improve answer quality, typically for problems that require multistep reasoning. When in doubt, respond directly.
```
```text
Report every issue you find, including ones you are uncertain about or consider low-severity. Do not filter for importance or confidence at this stage - a separate verification step will do that. Your goal here is coverage: it is better to surface a finding that later gets filtered out than to silently drop a real bug. For each finding, include your confidence level and an estimated severity so a downstream filter can rank them.
```

#### 常见坑
- 直接沿用 Sonnet 4.6 的 `max_tokens` → 长任务被截断，误判为能力退化。
- 仍传 `temperature`/`top_p`/`top_k` 或手动 extended thinking → 请求直接 400。
- 在 `low` effort 下用 prompt 硬补推理深度 → 不如把 effort 提到 `high`/`xhigh` 有效。
- 保留"只报高危"类过滤语 → 模型照做，measured recall 下降，看起来像 bug-finding 能力退步。

### 3.5 Claude Opus 5.5

#### 要点
- **effort 起点改为 `medium`** — Opus 5.5 默认 `medium`（Opus 5 默认 `high`），必须显式设置并重跑 eval 扫描；`medium` 已匹配或超过 Opus 5 的 `high`，不要照搬旧值。
- **thinking 恒开，无法禁用** — Opus 5 允许 `thinking: {"type": "disabled"}`（effort ≤ `high`），Opus 5.5 不接受；要靠降 effort 控成本，并同步改请求体。
- **同 effort 下思考更多** — 尤其 `xhigh`/`max`；`max_tokens` 要给 thinking 留空间（Anthropic 测试中 128000 上限对长 agentic 任务表现良好），`xhigh`/`max` 只在实测有质量收益时使用。
- **中途改 effort 会废掉 prompt cache** — 对话中途换档用 per-message effort change（beta），不要改顶层 `effort`。
- **清理"代替 thinking"的提示词** — 让模型把推理写进正文的指令要删掉，改用 `display: "summarized"` 读 thinking 块；否则可能被 `reasoning_extraction` 类别拒绝。
- **按 block type 读响应** — 不要假设首个 content block 是 text：默认 `display: "omitted"` 下 thinking 块字段为空。
- **进度更新藏在 thinking 块里** — 默认 `display` 下这些 update 文本为空，只渲染 `text` 的客户端会显得"静默"；需设 `display: "updates"`（beta，header `thinking-display-updates-2026-08-18`）。
- **把纯文本 end_turn 当作报告而非完成** — 无人值守 loop 会在此停住；用清单（todo 工具/文件）跟踪子任务，未完成且无阻塞说明就发一条点名剩余项的短 user message，最多自动续跑 2–3 次。
- **后台任务/子 agent 未返回不算完成** — 等其结束并把输出作为下一条 user message 喂回模型。
- **系统提示里的"仔细思考"该删** — 聊天产品中模型自己决定思考量，effort 才是主控；删除后回复更快且质量无明显下降。
- **多轮聊天要显式"旧答案已结算"** — 否则模型会回头重审上一轮答案、增加 thinking 与延迟；该指令在需要反复修订的长分析/agentic 任务中不要加。
- **标记粘贴文本以抗注入** — 用同一随机 id 的开合标签包裹用户粘贴块并在 system prompt 说明，才能拿到对间接注入的鲁棒性；标签是纯文本、可被模仿，只算一道防线。
- **多 app 工作流要显式要求先探索** — 模型上手快，松散任务容易漏掉未点名的邮件/表格页/CRM 记录；加一句"先广泛探索再动手"能显著提高正确完成率（代价是略多工具调用），同时要把不可信内容排除在检索范围外。
- **给多 agent harness 加时间信号** — 每条回传消息附 `elapsed 340s / 1200s` 式预算行，或加"时间宝贵"一句；预算会让更多 agent 并行而非减少工作量，是 advisory，硬超时仍需自己兜。
- **前端默认风格需点名具体模式** — "avoid a generic AI look" 只会换个默认；直接列出要避免的具体样式并迭代检查首个产出。

#### 可复用提示词片段
```text
Before taking any action, explore broadly with tool calls: list and open the emails, documents, spreadsheet tabs and records across the available apps that could be relevant to this task, including ones the task does not explicitly mention, and use what you find.
```
```text
The user hasn't heard from you in a while — say in a few words what you're doing, then continue.
```
```text
Once you have answered something, treat that answer as done. On later turns, focus your thinking on what the user is asking now, and don't go back over an earlier answer unless the user asks about it or points out a problem with it.
```

#### 常见坑
- 照搬 Opus 5 的 effort 值 → 轮次更长、输出 token 更多、成本上升。
- 仍发 `thinking: {"type": "disabled"}` → 请求被拒，需按 migration guide 改请求体。
- 无人值守 agent 把带文本的 `end_turn` 当收尾 → 任务半途停住；无限自动续跑同样有害（卡死无法审阅）。
- 让模型在正文里复述推理 → 触发 `reasoning_extraction` 拒绝（`stop_reason: "refusal"` + `stop_details`），且该类拒绝不会被服务端 fallback 重试。

### 3.6 Claude Opus 5

#### 要点
- **verbosity 必须用 prompt 控** — 默认回复比前代 Opus 更长；effort 控的是 thinking 而非可见长度，降 effort 不会可靠缩短回复，要显式写简洁指令。
- **长 system prompt 里要就近复述** — 简洁要求配合 prompt 末尾的短提醒（如 `<tone_preference>` 块）才稳。
- **写盘文档另算长度** — 报告/Markdown/摘要普遍偏长，需显式校准："不要用填充章节、重复摘要或模板化内容"。
- **删掉验证类指令** — 模型会自行验证；"include a final verification step"/"use a subagent to verify" 及遗留 harness 脚手架会导致 over-verification，白烧 token 且无质量收益。
- **窄任务要显式限界** — 模型会自行扩范围或改判任务目标；用 scope 指令要求"交付被要求的、按原意范围"，异常发现只用一句话说明后继续。
- **限制 subagent 委派** — 委派倾向高于前代；只对真正独立且可并行的大块工作委派，禁止用 subagent 自我验证，能一个就一个。
- **子 agent 需要确定性上限** — `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH`、`CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS`、SDK 的 `max_budget_usd`（TS：`maxBudgetUsd`）；旧版 Claude Code 没有这些开关。
- **删掉自检指令并收敛纠错叙述** — "double-check your answer"/"re-verify before responding" 会叠加成本；同时模型会过度播报对早前说法的更正，需限定为"只有会改变用户代码/结论/决策的错误才更正"。
- **进度叙述需显式指定节奏** — 模型爱先宣告再动手，per-message 输出偏长；给出"首次工具调用前一句话、重要发现或转向时才更新、结束时先给结论"的形态。
- **低 effort 值得当主力** — `low`/`medium` 以少量 token 与延迟维持不错质量，`xhigh` 留给高难编码与 agentic；换了模型就重跑 effort 扫描。
- **代码审查要求"全报 + 另设过滤"** — "only report high-severity issues"/"be conservative" 会被字面执行，报告量下降；改为全量上报、由独立 pass 排序过滤。
- **thinking 关闭的代价与替代** — 仅在 effort ≤ `high` 可禁用，`xhigh`/`max` + `disabled` 返回 400；首选保持 thinking 开启并用低 effort 控成本（`low` + thinking 通常优于同成本 disabled）。
- **禁用 thinking 的两个伪影** — 工具调用被写进可见文本（turn 正常结束但调用从不执行，且泄漏文本留在历史里污染后续轮次，搜索类工具密集负载最易发生）；内部 XML 标签泄漏进可见回复。
- **不要把推理写进正文** — 要求逐字或固定格式输出 thinking 可能被 `reasoning_extraction` 类别拒绝；改要简短解释/动作摘要，或开 thinking 读 `display: "summarized"` 块。

#### 可复用提示词片段
```text
Keep responses focused, brief, and concise. Keep disclaimers and caveats short, and spend most of the response on the main answer. When asked to explain something, give a high-level summary unless an in-depth explanation is specifically requested.
```
```text
Deliver what was asked, at the scope intended. Make routine judgment calls yourself, and check in only when different readings of the request would lead to materially different work. If the request seems mistaken or a better approach exists, say so in a sentence and continue with the task as asked rather than quietly narrowing, widening, or transforming it. Finish the whole task, and stop short of actions that are clearly beyond what was asked.
```
```text
When you use a tool, you may say a brief sentence first. If no tool can express what the user asked for, say so instead of guessing. Do not include internal or system XML tags in your response.
```

#### 常见坑
- 靠降 effort 压回复长度 → 思考变少但可见回复仍长，问题没解决。
- 保留 "re-verify"/"double-check" 类指令 → 与模型自带行为叠加，成本上升、结果不变。
- 系统提示里写"不要思考/不要推理" → 反而加重 `<thinking>` 标签泄漏。
- 用指名 thinking 标签的规则去堵标签泄漏 → 效果不如上面那种通用表述。

### 3.7 Claude Opus 4.8

#### 要点
- **输出长度按任务自适应** — 不再默认固定 verbosity，简单查询更短、开放式分析更长；产品若依赖固定风格需显式加简洁指令，且给正面示例比给"不要做什么"更有效。
- **effort 是第一控制杆** — 编码/agentic 起手 `xhigh`，智能敏感场景最低 `high`；`max` 可能因 token 增多而收益递减并出现 overthinking。
- **low/medium 会严格收敛** — 模型只做被要求的事；中等复杂任务在 `low` 下存在 under-thinking 风险，应提 effort 而不是靠 prompt 绕。
- **thinking 默认关闭** — Opus 4.8 需显式 `thinking: {type: "adaptive"}`；触发频率可用 prompt 调节（大/复杂 system prompt 易过度思考，可加引导文本）。
- **大 token 预算** — 跑 `max`/`xhigh` 时把 max output tokens 放大，给 thinking + 工具 + subagent 留空间，起始 64k 再调。
- **工具调用倾向偏低** — 该模型偏好推理而非调工具；要更多工具使用就提 effort，或在 prompt 里明确"何时、如何"使用工具（例如说明为什么该用 web search）。
- **移除强制进度更新 scaffolding** — 它本身已给出更高频、更高质量的用户可见更新；若长度/内容不合预期，改为显式描述期望样式并给示例。
- **指令遵循更字面** — 不会把一条指令静默泛化到其它对象，也不会推断未提出的需求；需要广覆盖时显式声明作用域（如"应用到每一个 section，不只是第一个"）。
- **写作语气基线变了** — 偏直接、有观点、少 validation 前缀、少 emoji；依赖特定 voice 的产品要针对新基线重评 style prompt。
- **subagent 生成更保守** — 默认 spawn 更少，但可被 prompt 引导；显式写清何时该/不该开 subagent。
- **前端设计默认风格顽固** — 固定 house style：暖奶油/off-white 背景（~`#F4F1EA`）、衬线标题（Georgia/Fraunces/Playfair）、斜体强调词、terracotta/amber 强调色；适合 editorial/hospitality/portfolio，不适合 dashboard/dev tool/fintech/healthcare/enterprise。
- **改设计默认的两种可靠手法** — ①给具体替代规格（色调系统、圆角、字体、动效参数都写死）；②先让模型提 4 个方向让用户选，再只实现被选中的那个（比 `temperature` 更可靠）。
- **前端"AI slop"提示可精简** — 相比旧模型所需的长 snippet，Opus 4.8 用更少引导即可产出有辨识度的前端，`<frontend_aesthetics>` 这类短块仍可用。
- **交互式编码更耗 token** — 多轮交互场景它会在用户轮后继续推理，token 用量高于单轮自治；用 `xhigh`/`high` effort、加 auto 模式、减少人类交互轮次，并在第一轮把任务/意图/约束写全。
- **code review 要改报告策略** — 过滤型指令（"only report high-severity"/"be conservative"/"don't nitpick"）会被更忠实地执行，导致相同调查深度下报告的 finding 变少、measured recall 下降（precision 通常上升），这是 harness 效应而非能力回归。
- **code review 推荐写法** — finding 阶段以 coverage 为目标：报告所有发现（含不确定/低严重度），附带 confidence 与 estimated severity，把过滤交给下游 verification/ranking 阶段。
- **单遍自过滤要给硬阈值** — 若坚持一遍出结果，用具体条件（如"会导致错误行为、测试失败或误导性结果的 bug；只省略纯风格/命名 nit"），不要用"important"这类定性词。
- **迁移兼容性** — 现有 Claude Opus 4.7 的 prompt 开箱可用；自 4.7 起的 API 变化（1M context window 默认、mid-conversation system messages）在 Opus 4.8 上同样成立，调 prompt 时不必为此改动。
- **强项分布决定取舍** — 该模型在 long-horizon agentic 工作、knowledge work、vision 与 memory 任务上表现突出，这些场景更值得开高 effort 与更大 token 预算。
- **computer use 规格** — 支持 `computer_toolset_20260801`（Claude API 与 Google Cloud）与 `computer_20251124`；同平台还支持 browser use tool（`browser_toolset_20260801`）；最大分辨率 2576px / 3.75MP，1080p 是性能/成本平衡点，720p 或 1366×768 更省。

#### 可复用提示词片段
```text
Provide concise, focused responses. Skip non-essential context, and keep examples minimal.
```
```text
Report every issue you find, including ones you are uncertain about or consider low-severity. Do not filter for importance or confidence at this stage - a separate verification step will do that. Your goal here is coverage: it is better to surface a finding that later gets filtered out than to silently drop a real bug. For each finding, include your confidence level and an estimated severity so a downstream filter can rank them.
```
```text
<frontend_aesthetics>
NEVER use generic AI-generated aesthetics like overused font families (Inter, Roboto, Arial, system fonts), cliched color schemes (particularly purple gradients on white or dark backgrounds), predictable layouts and component patterns, and cookie-cutter design that lacks context-specific character. Use unique fonts, cohesive colors and themes, and animations for effects and micro-interactions.
</frontend_aesthetics>
```

#### 常见坑
- 用"只报高严重度/保守一点/别挑刺"收紧 review，会把 recall 一起打掉；先确认是 harness 过滤造成的假回归，再改 finding 阶段措辞。
- 在 `low` effort 上跑中等复杂任务出现浅层推理时，改 prompt 收效有限，正确动作是提到 `high`/`xhigh`。
- 用通用否定指令（"don't use cream"、"make it clean and minimal"）改设计默认，只会把模型推到另一个固定配色，不会产生多样性。
- 继续保留"每 3 次工具调用总结一次进度"这类强制 scaffolding，属于多余干预，应删除并改为描述期望的更新样式。

### 3.8 Claude Haiku 5.5

#### 要点
- **先按症状定位** — 现有 Claude Haiku 4.5 的 prompt 无需改动即可用；只有当出现具体症状（漏搜索、漏工具调用、提前收工、无验证、忽略中途输入、脱离 system prompt、推理外泄、`refusal`）时才逐项改，不要整体重写。
- **effort 取代 thinking budget** — `budget_tokens` 不再适用，无旧设置可迁移；`medium` 是默认，多数工作（含 agentic coding）从它起步。
- **按场景选 effort** — `low` 最便宜最快，适合 chat、短工具任务、高并发简单请求；`high` 适合知识工作、长 agent 任务与严格指令遵循；`xhigh`/`max` 仅在 evals 上证明有质量收益时使用，并应与 Claude Sonnet 5.5 对比性能/成本/速度。
- **thinking 默认开启且计入 max_tokens** — 上限 128,000；沿用 Haiku 4.5（无 thinking）时代的 `max_tokens` 会截断回复，必须留出 thinking 空间。
- **降低 thinking 只能靠 effort 或关闭开关** — 在 prompt 里要求"直接回答"经测试无效；`thinking: {"type": "disabled"}` 仅在 `low`/`medium`/`high` 可用，在 `xhigh`/`max` 下返回 400。
- **xhigh 多轮空回复** — 有时把整段答案写进 thinking 并以无可见文本结束回合，需要检查空回复。
- **改 effort 会失效 prompt cache** — 顶层 `effort` 变更使会话消息缓存失效；可用 per-message effort change（beta）在会话中途改且保留缓存，需 `mid-conversation-output-config-2026-07-01` beta header + adaptive thinking（默认），且改动持续到下次替换；thinking 关闭时该用法返回 400。
- **搜索要给日期** — 提供搜索工具时同时给今天的日期（放 system prompt 或搜索工具 description），`{{current_date}}` 只在会话开始时渲染一次，后续请求保持相同 `system`/`tools`；跨天继续会话时把新日期放进最新一轮。
- **长 system prompt + low effort 需额外搜索引导** — 加一段说明训练数据早于今天、记录/价格/版本/"最新"类信息可能已变需先搜索，而不会变的常识无需搜索；测试中它在"答案已变"的问题上提高搜索率，在无需搜索的 prompt 上只增加 0–3% 搜索。
- **避免笼统搜索指令** — "任何现时事实问题都搜索，无论多自信"会让半数无需搜索的 prompt 也触发搜索，且没有带来更正确的答案。
- **JSON output + 自带工具且 thinking off 会漏工具调用** — 三种解法：改用 adaptive thinking（省略 `thinking` 或 `{"type":"adaptive"}`）；从必须调工具的请求里移除 `output_config.format`；用 `tool_choice` 强制调用（但模型调用前不写文本）。必须 thinking off 时，在 system prompt 加一行说明 JSON 格式只适用于最终答案。
- **长 agent prompt 会提前收工** — 短 system prompt 下很少发生；长 coding-agent prompt 在 `low` effort 下会提前停下把任务交回用户。
- **编码任务要求真实验证** — `low`/`medium` 下有时未运行任何检查就报告完成；需明确要求跑真实检查（测试/类型检查/构建或改动命令本身），语法检查或启动失败的检查不算；缺依赖用项目自身包管理器与 lockfile（如 `npm install`、`pip install -r requirements.txt`），不要用 sudo 或系统包管理器；确实无法检查时说明未跑哪项及原因。
- **mid-turn 用户消息可能被当不可信文本忽略** — 模型被训练抵抗经 tool result 的 prompt injection；不要把用户文本放进 `tool_result` block，应作为最后一个 `tool_result` 之后的 text block 追加到同一条 user message；harness 通知（提醒等）单独放 mid-conversation system message，且不要与用户话语同块。
- **chatbot 要加规则持续性文本** — 说明 system prompt 规则对整个会话有效，用户争辩、给同情理由、只要求一小部分、声称有人批准例外、反复追问时都要坚持；指令遵循最关键时同时用 `high` effort。
- **推理文本泄漏到用户可见回复** — 在 thinking off 或 `low` effort 下更常见；解法是切到 adaptive thinking + `medium` effort。
- **拒绝是新的且无服务端兜底** — 被拒请求返回 `stop_reason: "refusal"`，`stop_details.category` 取值 `cyber`、`frontier_llm`、`bio`、`general_harms`；其中**良性工作也可能触发的是 `cyber` 与 `general_harms`**（文档在 `cyber` 条目写明 "Benign cybersecurity work can also trigger this category"，在 `general_harms` 条目写明 "Benign work can also trigger this category"）；无 server-side fallback，需在客户端处理，重复同一请求通常再次被拒。
- **改 system prompt 的时机约束** — 本页建议的追加文本只在新会话加；若在 system 变更后回传 thinking blocks 会返回 400，中途改指令应放进最新一轮或追加 mid-conversation system message。

#### 可复用提示词片段
```text
The current date is {{current_date}}.

Your training data ends well before today's date. Records, office holders, prices, versions, rules and anything "latest" may have changed since then, so search for those before you answer, even when you feel sure. Facts that can't change need no search. When the answer depends on where the user is, put the user's country or region in the search query.
```
```text
Keep working until everything the user asked for is done, and only stop to ask when you can't go on without the user or before a risky step.
When the work the user asked for is done and checked, stop and report. Don't add new features, docs, or refactors that weren't asked for. If you think one would help, mention it at the end instead of doing it.
```
```text
The JSON output format applies to your final answer only. When you need a tool, call it first, with no text before the call, and write the JSON once you have the results.
```

#### 常见坑
- 用 prompt 文字（"直接回答"/"别想太多"）压低 thinking：实测无效，只能降 effort 或 `thinking: {"type": "disabled"}`。
- 在 `xhigh`/`max` 上设 `thinking: {"type": "disabled"}`，或在 thinking 关闭时用 per-message effort change，都会得到 400。
- 把用户中途输入塞进 `tool_result` block、或与 harness 通知写在同一条消息里，会被模型当作不可信文本忽略。
- 长 system prompt 下沿用 Haiku 4.5 的 `max_tokens` 且不做搜索引导，会同时踩到回复截断与漏搜索两件事。
