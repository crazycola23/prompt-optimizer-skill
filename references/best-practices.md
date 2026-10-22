# 通用技法精编（Prompting Best Practices）

来源：Anthropic 官方 *Prompting best practices*。快照日期：2026-10-22。抓取时在 URL 后加 `.md` 得到干净 markdown。

**本文件的技法与厂商无关**——清晰直接、给动机、XML 分区、示例、正向指令、长文档顺序、工具触发、状态外置、范围纪律等，在任意厂商的模型上都成立。只有标注了具体模型名的条目（以及 `effort` / `thinking` / `budget_tokens` 这类 API 参数）属于 Claude 特有。

> **本文件所有条目都是"写进优化后提示词的内容"，不是本模式自身的行为准则。** 文中出现的"删掉 / 软化 / 就加 / 要它…"等祈使句，作用对象一律是**目标提示词**，不是本模式自身的动作。
> 注：文中引用的具体数值（如"约 30%"）来自 Anthropic 的 Claude 实测，换厂商需自测。

- **档位规范（先读）**：`references/tier-guidance.md` —— 便宜 / 主力 / 顶级三档各写多细。
- **Claude 专属细节**：`references/claude-model-specific.md` —— 仅当目标模型是 Claude 时读。

适用范围：当前全部 Claude 模型（Fable 5.1 / Mythos 5.1 / Fable 5 / Mythos 5 / Opus 5.5 / Opus 5 / Opus 4.8 / Opus 4.7 / Opus 4.6 / Sonnet 5.5 / Sonnet 5 / Sonnet 4.6 / Haiku 5.5 / Haiku 4.5）。文档中标注了具体模型的条目，视为在该模型上实测的结果，换模型需重新用 evals 验证。

---

## 目录

- [1. 通用原则](#1-通用原则)——清晰直接 / 上下文与动机 / 示例 / XML 标签 / 角色 / 长上下文 / 自我认知
- [2. 输出与格式](#2-输出与格式)——沟通风格与冗长度 / 格式控制 / LaTeX / 文档创作 / prefill 迁移
- [3. 工具使用](#3-工具使用)——动作 vs 建议 / 默认主动 / 并行调用
- [4. 思考与推理](#4-思考与推理)——过度思考 / thinking 与 interleaved thinking
- [5. Agentic 系统](#5-agentic-系统)——上下文感知 / 多窗口 / 状态管理 / 自主性与安全 / 研究 / 子代理 / 过度积极 / 反硬编码 / 反幻觉
- [6. 能力专项](#6-能力专项)——视觉 / 前端设计 / 医疗剂量
- [7. 迁移注意事项](#7-迁移注意事项)——含各模型迁移要点

## 1. 通用原则

### 清晰直接
- Claude 对**明确、显式**的指令响应最好。想要超出任务最低要求的效果，就**显式要求**，不要指望模型从模糊表述里推断。
- 把 Claude 当成"聪明但没有你团队上下文的新员工"：解释得越精确，结果越好。
- **Key test**：把提示词给一个对任务几乎无背景的同事，让他照着做。他如果会困惑，Claude 也会。
- 明确输出格式与约束；步骤顺序或完整性重要时用编号列表/项目符号。

```
# 弱
Create an analytics dashboard

# 强
Create an analytics dashboard. Include as many relevant features and interactions as
possible. Go beyond the basics to create a fully-featured implementation.
```

### 给上下文与动机
- 说明"为什么这样做重要"，能给 Claude 更多目标信息，产出更贴合。Claude 能**从解释中泛化**。

```
# 弱
NEVER use ellipses

# 强
Your response will be read aloud by a text-to-speech engine, so never use ellipses since
the text-to-speech engine will not know how to pronounce them.
```

### 有效使用示例（few-shot / multishot）
- 示例是塑造输出格式、语气、结构最可靠的手段之一。
- 要求：**相关**（贴近真实用例）、**多样**（覆盖边界，避免模型学到意外模式）、**结构化**（用 `<example>` 包裹，多个放 `<examples>`）。
- 建议 **3–5 个**。也可以让 Claude 先评估示例的相关性与多样性，或基于已有示例生成更多。

### 用 XML 标签组织提示词
- 提示词里混有指令、上下文、示例、变量输入时，XML 标签能让解析无歧义：`<instructions>` `<context>` `<input>`。
- 标签名保持一致、描述性强；内容有天然层级时嵌套，如 `<documents>` 内每个 `<document index="n">`。

### 设定角色
- 在 system prompt 里设定角色即可聚焦行为与语气，**一句话就有效**：

```
You are a helpful coding assistant specializing in Python.
```

### 长上下文提示（20k+ tokens）
- **长资料放顶部**：把长文档与输入放在 query、指令、示例**之前**。
- 注意：query 放在**末尾**可提升响应质量，多文档复杂输入场景实测最高约 **30%**。
> 占位约定：本文件部分示例沿用 `{{变量}}` 写法。按 `references/tier-guidance.md` §5 的约定，改写时应统一换成 `[方括号]`，同一提示词内不要混用。（`claude-model-specific.md` 里的 `{{current_date}}` 是 Claude 搜索工具的真实模板变量，保持原样。）

- 多文档结构化：

```xml
<documents>
  <document index="1">
    <source>annual_report_2023.pdf</source>
    <document_content>
      {{ANNUAL_REPORT}}
    </document_content>
  </document>
  <document index="2">
    <source>competitor_analysis_q2.xlsx</source>
    <document_content>
      {{COMPETITOR_ANALYSIS}}
    </document_content>
  </document>
</documents>

Analyze the annual report and competitor analysis. Identify strategic advantages and recommend Q3 focus areas.
```

- **先引用再作答**（让模型聚焦相关内容、忽略其余）：

```xml
Find quotes from the patient records and appointment history that are relevant to
diagnosing the patient's reported symptoms. Place these in <quotes> tags. Then, based on
these quotes, list all information that would help the doctor diagnose the patient's
symptoms. Place your diagnostic information in <info> tags.
```

### 模型自我认知
- 需要模型正确自我介绍或使用特定 API 字符串时：

```
The assistant is Claude, created by Anthropic. The current model is Claude Opus 5.5.
```

```
When an LLM is needed, please default to Claude Opus 5.5 unless the user requests
otherwise. The exact model string for Claude Opus 5.5 is claude-opus-5-5.
```

---

## 2. 输出与格式

### 沟通风格与冗长度
- 新模型更简洁自然：更直接、基于事实的进度汇报（而非自我表扬式更新）、更口语、更少赘述；工具调用后可能**跳过口头总结**直接进入下一步。
- 需要更多可见性：

```
After completing a task that involves tool use, provide a quick summary of the work you've done.
```

- **Claude Opus 5 是例外**：默认面向用户的回答比前代更长，调高或调低 `effort` 不能可靠改变可见回答长度——必须**显式要求简洁**。
- **Claude Fable 5.1 在 agentic 工作中倾向相反**：工具调用之间写更少的用户可见更新。要**显式要求进度文本**，并删掉"保持简短"之类指令。

### 控制响应格式
1. **说"要什么"而不是"不要什么"**：不要 `Do not use markdown in your response`，改 `Your response should be composed of smoothly flowing prose paragraphs.`
2. **用 XML 格式指示器**：`Write the prose sections of your response in <smoothly_flowing_prose_paragraphs> tags.`
3. **提示词风格匹配输出风格**：想要少 markdown，提示词里就先少用 markdown。
4. **需要精细控制时给显式规范**：

```text
<avoid_excessive_markdown_and_bullet_points>
When writing reports, documents, technical explanations, analyses, or any long-form
content, write in clear, flowing prose using complete paragraphs and sentences. Use
standard paragraph breaks for organization and reserve markdown primarily for `inline
code`, code blocks (```...```), and simple headings (## and ###). Avoid using **bold**
and *italics*.

DO NOT use ordered lists (1. ...) or unordered lists (*) unless: a) you're presenting
truly discrete items where a list format is the best option, or b) the user explicitly
requests a list or ranking

Instead of listing items with bullets or numbers, incorporate them naturally into
sentences. This guidance applies especially to technical writing. Using prose instead of
excessive formatting will improve user satisfaction. NEVER output a series of overly
short bullet points.

Your goal is readable, flowing text that guides the reader naturally through ideas
rather than fragmenting information into isolated points.
</avoid_excessive_markdown_and_bullet_points>
```

注意：**Claude Fable 5.1 本身格式就比早期模型少**，在该模型上这段会压掉内容需要的结构——删掉，或换成更短的聊天格式规则。

### LaTeX
- 新模型默认对数学表达式用 LaTeX。要纯文本：

```
Format your response in plain text only. Do not use LaTeX, MathJax, or any markup
notation such as \( \), $, or \frac{}{}. Write all math expressions using standard text
characters (e.g., "/" for division, "*" for multiplication, and "^" for exponents).
```

### 文档创作
- 新模型做演示、动画、视觉文档的指令遵循强，通常一次成型：

```
Create a professional presentation on [topic]. Include thoughtful design elements,
visual hierarchy, and engaging animations where appropriate.
```

### 弃用 prefill（预填充响应）的迁移
Claude 4.6+ 与 Claude Mythos Preview 起，**最后一个 assistant turn 的 prefill 不再支持**，会返回 400。早期模型仍支持；对话中间插入 assistant 消息不受影响。

| 原用途 | 迁移做法 |
|---|---|
| 强制输出格式（JSON/YAML/分类） | 用 structured outputs 约束 schema；分类任务用带 enum 的工具或 structured outputs |
| 消除前言 | system prompt 直接写：`Respond directly without preamble. Do not start with phrases like "Here is...", "Based on...", etc.`；或输出到 XML 标签、用 structured outputs / tool calling |
| 规避不必要的拒答 | 现在在 user 消息里清晰提示即可 |
| 续写被中断的回复 | 移到 user 消息：`Your previous response was interrupted and ended with [previous_response]. Continue from where you left off.` |
| 周期性注入上下文 | 注入到 user turn；复杂 agent 系统改用工具或 context compaction |

---

## 3. 工具使用

- 新模型训练目标是精确遵循指令，**需要明确指向具体工具**。说 "can you suggest some changes" 往往只得到建议。

```
# 弱（只给建议）
Can you suggest some changes to improve this function?

# 强（直接动手）
Change this function to improve its performance.
Make these edits to the authentication flow.
```

- 默认要它**动手**：

```text
<default_to_action>
By default, implement changes rather than only suggesting them. If the user's intent is
unclear, infer the most useful likely action and proceed, using tools to discover any
missing details instead of guessing. Try to infer the user's intent about whether a tool
call (e.g., file edit or read) is intended or not, and act accordingly.
</default_to_action>
```

- 默认要它**先问再做**：

```text
<do_not_act_before_instructions>
Do not jump into implementation or change files unless clearly instructed to make
changes. When the user's intent is ambiguous, default to providing information, doing
research, and providing recommendations rather than taking action. Only proceed with
edits, modifications, or implementations when the user explicitly requests them.
</do_not_act_before_instructions>
```

- **软化措辞**：Claude Opus 4.5 / 4.6 对 system prompt 更敏感，为旧模型写的"减少欠触发"的激进措辞现在会**过度触发**。`CRITICAL: You MUST use this tool when...` → `Use this tool when...`

### 并行工具调用
- 新模型会自动并行独立调用（多路搜索、并行读多个文件、并行跑 bash，后者甚至可能压垮系统性能）。
- 拉到接近 100% 成功率 / 调节激进程度：

```text
<use_parallel_tool_calls>
If you intend to call multiple tools and there are no dependencies between the tool
calls, make all of the independent tool calls in parallel. Prioritize calling tools
simultaneously whenever the actions can be done in parallel rather than sequentially.
For example, when reading 3 files, run 3 tool calls in parallel to read all 3 files into
context at the same time. Maximize use of parallel tool calls where possible to increase
speed and efficiency. However, if some tool calls depend on previous calls to inform
dependent values like the parameters, do NOT call these tools in parallel and instead
call them sequentially. Never use placeholders or guess missing parameters in tool
calls.
</use_parallel_tool_calls>
```

```
Execute operations sequentially with brief pauses between each step to ensure stability.
```

- Claude Fable 5.1 的长 agent loop：并行指令要作为 **turn-scoped system message** 在每轮工具结果之后发送。

---

## 4. 思考与推理

### 过度思考与过度彻底
- Claude Opus 4.6 在更高 `effort` 下会做更多前置探索（常常有益，但可能收集过量上下文、跑偏多条线）。
- 修法：把一刀切默认改成**定向指令**（`Use [tool] when it would enhance your understanding of the problem.`）；删掉过度提示（`If in doubt, use [tool]` 会导致过度触发）；把 `effort` 当兜底。
- 约束思考的显式指令：

```
When you're deciding how to approach a problem, choose an approach and commit to it.
Avoid revisiting decisions unless you encounter new information that directly
contradicts your reasoning. If you're weighing two approaches, pick one and see it
through. You can always course-correct later if the chosen approach fails.
```

- 需要思考成本硬上限：Opus 4.6 / Sonnet 4.6 上 `budget_tokens` 仍可用但**已弃用**；**Claude 4.7+ 传 `budget_tokens` 直接 400**。改用降低 `effort`，或用 `max_tokens` + adaptive thinking 设硬限。

### 利用 thinking / interleaved thinking
- Claude 4.6+ 与 Mythos Preview 用 **adaptive thinking**（`thinking: {type: "adaptive"}`），模型自行决定何时思考、思考多少；校准依据是 `effort` 参数与 query 复杂度。简单 query 直接作答。
- **Fable 5.1 / Mythos 5.1 / Fable 5 / Mythos 5 / Opus 5.5：thinking 始终开启，adaptive 是唯一模式。**
- 内部评测中 adaptive thinking 稳定优于 extended thinking。
- 引导反思：

```
After receiving tool results, carefully reflect on their quality and determine optimal
next steps before proceeding. Use your thinking to plan and iterate based on this new
information, and then take the best next action.
```

- 大型/复杂 system prompt 会让模型思考得比你想要的多，可反向引导：

```
Thinking adds latency and should only be used when it will meaningfully improve
answer quality - typically for problems that require multistep reasoning. When in
doubt, respond directly.
```

- 迁移示例（配置层）：`thinking: {type:"enabled", budget_tokens: 10000}` → `thinking: {type:"adaptive"}` + `output_config: {effort: "high"}`。
- **未显式设置 `thinking` 时的默认**：
  - Opus 4.6–4.8、Sonnet 4.6：thinking **关闭**。
  - Opus 5、Sonnet 5.5、Sonnet 5、Haiku 5.5：thinking **默认开启**，响应可能以 `thinking` block 开头——按 `type` 而非位置取 content block，并在 `max_tokens` 里给思考留空间。
  - 关闭条件：Opus 5 / Haiku 5.5 只能在 `effort` ≤ `high` 时关闭；Sonnet 5.5 最低思考设置是 `between_tools`（`effort` ≤ high 可用）；Fable 5.1 / Mythos 5.1 / Fable 5 / Mythos 5 / Opus 5.5 **始终开启**。
- 实践要点：
  - **偏好概括性指令而非规定步骤**：`think thoroughly` 常优于手写分步计划，模型的推理往往超过人类预设。
  - **Multishot 示例对 thinking 同样有效**：示例按"问题 + 应使用的方法 + 期望答案"呈现。
  - **手写 CoT 仅作兜底**（thinking 关闭时）：要求先思考再回答，把最终答案放进 `<answer>` 标签。但 **Fable 5.1 / Fable 5 / Opus 5.5 / Opus 5 / Sonnet 5.5** 上请改用 thinking + 更低 `effort`：要求模型把推理写进 `<thinking>` 标签这类提示**可能被拒**。
  - **自检**：追加 `Before you finish, verify your answer against [test criteria].` 对代码与数学尤其有效。**例外：Claude Opus 5** 无需指示就会自检，从旧模型沿用来的验证指令会造成 over-verification、增加 token 与延迟——在 Opus 5 上**直接删除**而不是改写。
- Claude Opus 4.5 在关闭 extended thinking 时对 "think" 及其变体**特别敏感**，改用 `consider` / `evaluate` / `reason through`。

---

## 5. Agentic 系统

### 长时程推理与状态跟踪
- 新模型状态跟踪强：聚焦增量进展，每次稳步推进少数几件事，而非一次做完所有事。

#### 上下文感知与多窗口工作流
- Claude Sonnet 5 / Sonnet 4.6 / Sonnet 4.5（已弃用）/ Haiku 4.5 具备 **context awareness**，能跟踪剩余 token 预算。
- 若 harness 会压缩上下文或允许把上下文存外部文件，把这件事写进提示词，否则模型会在接近上限时**自行收尾**：

```
Your context window will be automatically compacted as it approaches its limit, allowing
you to continue working indefinitely from where you left off. Therefore, do not stop
tasks early due to token budget concerns. As you approach your token budget limit, save
your current progress and state to memory before the context window refreshes. Always be
as persistent and autonomous as possible and complete tasks fully, even if the end of
your budget is approaching. Never artificially stop any task early regardless of the
context remaining.
```

- memory tool 与 context awareness 搭配良好。

#### 跨多个 context window 的工作流
1. **第一个窗口用不同提示词**：先搭框架（写测试、建 setup 脚本），后续窗口按 todo-list 迭代。
2. **让模型用结构化格式写测试**：动手前先写测试并记在结构化文件（如 `tests.json`）；强调 `It is unacceptable to remove or edit tests because this could lead to missing or buggy functionality.`
3. **准备便利工具**：鼓励创建 setup 脚本（如 `init.sh`）来起服务、跑测试与 linter。
4. **全新窗口 vs 压缩**：清空上下文时，新模型从本地文件系统发现状态的能力极强，有时优于压缩。要**明确规定启动动作**：
   - `Call pwd; you can only read and write files in this directory.`
   - `Review progress.txt, tests.json, and the git logs.`
   - `Manually run through a fundamental integration test before moving on to implementing new features.`
5. **提供验证工具**：computer use / browser use 工具或浏览器自动化 MCP，让模型在无持续人工反馈下验证正确性。
6. **鼓励用满上下文**：

```
This is a very long task, so it may be beneficial to plan out your work clearly. It's
encouraged to spend your entire output context working on the task - just make sure you
don't run out of context with significant uncommitted work. Continue working
systematically until you have completed this task.
```

#### 状态管理最佳实践
- 结构化状态（测试结果、任务状态）用 **JSON** 等结构化格式，让 schema 要求显式。
- 进度笔记用**自由文本**。
- 用 **git** 做状态跟踪（新模型在这方面表现尤其好）。
- 明确要求**增量推进**。

```json
// tests.json
{
  "tests": [
    { "id": 1, "name": "authentication_flow", "status": "passing" },
    { "id": 2, "name": "user_management", "status": "failing" },
    { "id": 3, "name": "api_endpoints", "status": "not_started" }
  ],
  "total": 200, "passing": 150, "failing": 25, "not_started": 25
}
```

```text
// progress.txt
Session 3 progress:
- Fixed authentication token validation
- Updated user model to handle edge cases
- Next: investigate user_management test failures (test #2)
- Note: Do not remove tests as this could lead to missing functionality
```

### 自主性与安全的平衡
- 无引导时 Claude Opus 4.6 可能做出难以撤销或影响共享系统的动作（删文件、force push、对外发帖）。要它先确认就加：

```
Consider the reversibility and potential impact of your actions. You are encouraged to
take local, reversible actions like editing files or running tests, but for actions that
are hard to reverse, affect shared systems, or could be destructive, ask the user before
proceeding.

Examples of actions that warrant confirmation:
- Destructive operations: deleting files or branches, dropping database tables, rm -rf
- Hard to reverse operations: git push --force, git reset --hard, amending published commits
- Operations visible to others: pushing code, commenting on PRs/issues, sending
messages, modifying shared infrastructure

When encountering obstacles, do not use destructive actions as a shortcut. For example,
don't bypass safety checks (e.g. --no-verify) or discard unfamiliar files that may be
in-progress work.
```

### 研究与信息收集
1. 给出**清晰的成功标准**。
2. 要求**跨多来源验证**信息。
3. 复杂调研用结构化方法：

```
Search for this information in a structured way. As you gather data, develop several
competing hypotheses. Track your confidence levels in your progress notes to improve
calibration. Regularly self-critique your approach and plan. Update a hypothesis tree or
research notes file to persist information and provide transparency. Break down this
complex research task systematically.
```

### 子代理编排
- 新模型原生编排子代理，会主动委派，无需显式指令。前提：子代理工具齐备且在工具定义里描述清楚。
- **注意过度使用**：Claude Opus 4.6 倾向强（会用子代理做 grep 就能搞定的事）；Claude Opus 5 比前代更爱委派。抑制措辞：

```
Use subagents when tasks can run in parallel, require isolated context, or involve
independent workstreams that don't need to share state. For simple tasks, sequential
operations, single-file edits, or tasks where you need to maintain context across steps,
work directly rather than delegating.
```

### 复杂提示词链式化
- adaptive thinking + 子代理编排已能内化多数多步推理；显式链式（拆成顺序 API 调用）仍适用于需要检查中间输出或强制固定流水线的场景。
- 最常见模式是**自纠**：生成草稿 → 按标准评审 → 据评审精修。每步独立 API 调用，便于记录、评估、分支。

### 减少 agentic coding 里的文件创建
- 模型可能新建文件做测试与迭代（尤其 Python 脚本当临时草稿纸），这对结果有好处。想少留残留：

```
If you create any temporary new files, scripts, or helper files for iteration, clean up
these files by removing them at the end of the task.
```

### 过度积极（overeagerness）
- Claude Opus 4.5 / 4.6 倾向过度工程：多建文件、加不必要的抽象、做没要求的灵活性。收敛措辞：

```
Avoid over-engineering. Only make changes that are directly requested or clearly
necessary. Keep solutions simple and focused:

- Scope: Don't add features, refactor code, or make "improvements" beyond what was
asked. A bug fix doesn't need surrounding code cleaned up. A simple feature doesn't need
extra configurability.

- Documentation: Don't add docstrings, comments, or type annotations to code you didn't
change. Only add comments where the logic isn't self-evident.

- Defensive coding: Don't add error handling, fallbacks, or validation for scenarios
that can't happen. Trust internal code and framework guarantees. Only validate at system
boundaries (user input, external APIs).

- Abstractions: Don't create helpers, utilities, or abstractions for one-time
operations. Don't design for hypothetical future requirements. The right amount of
complexity is the minimum needed for the current task.
```

### 避免"只为过测试"与硬编码
- 模型有时过度追求测试通过、牺牲通用性，或用辅助脚本绕开标准工具。要通用解就加：

```
Please write a high-quality, general-purpose solution using the standard tools
available. Do not create helper scripts or workarounds to accomplish the task more
efficiently. Implement a solution that works correctly for all valid inputs, not just
the test cases. Do not hard-code values or create solutions that only work for specific
test inputs. Instead, implement the actual logic that solves the problem generally.

Focus on understanding the problem requirements and implementing the correct algorithm.
Tests are there to verify correctness, not to define the solution. Provide a principled
implementation that follows best practices and software design principles.

If the task is unreasonable or infeasible, or if any of the tests are incorrect, please
inform me rather than working around them. The solution should be robust, maintainable,
and extendable.
```

### 减少 agentic coding 中的幻觉

```
<investigate_before_answering>
Never speculate about code you have not opened. If the user references a specific file,
you MUST read the file before answering. Make sure to investigate and read relevant
files BEFORE answering questions about the codebase. Never make any claims about code
before investigating unless you are certain of the correct answer - give grounded and
hallucination-free answers.
</investigate_before_answering>
```

---

## 6. 能力专项

### 视觉能力
- Claude Opus 4.5 / 4.6 视觉更强，多图同处上下文的图像处理与数据抽取表现更好，并延续到 computer use（截图与 UI 元素理解）。
- 视频可按帧拆解分析。
- 提升手段：给 **crop 工具**或 agent skill，让模型能"放大"关键区域——图像评测上有一致提升。

### 前端设计
- 无引导时模型会落到通用模式，产出被称作 "AI slop" 的审美。要独特、有创意的前端：

```
<frontend_aesthetics>
You tend to converge toward generic, "on distribution" outputs. In frontend design, this
creates what users call the "AI slop" aesthetic. Avoid this: make creative, distinctive
frontends that surprise and delight.

Focus on:
- Typography: Choose fonts that are beautiful, unique, and interesting. Avoid generic
fonts like Arial and Inter; opt instead for distinctive choices that elevate the
frontend's aesthetics.
- Color & Theme: Commit to a cohesive aesthetic. Use CSS variables for consistency.
Dominant colors with sharp accents outperform timid, evenly-distributed palettes. Draw
from IDE themes and cultural aesthetics for inspiration.
- Motion: Use animations for effects and micro-interactions. Prioritize CSS-only
solutions for HTML. Use Motion library for React when available. Focus on high-impact
moments: one well-orchestrated page load with staggered reveals (animation-delay)
creates more delight than scattered micro-interactions.
- Backgrounds: Create atmosphere and depth rather than defaulting to solid colors. Layer
CSS gradients, use geometric patterns, or add contextual effects that match the overall
aesthetic.

Avoid generic AI-generated aesthetics:
- Overused font families (Inter, Roboto, Arial, system fonts)
- Clichéd color schemes (particularly purple gradients on white backgrounds)
- Predictable layouts and component patterns
- Cookie-cutter design that lacks context-specific character

Interpret creatively and make unexpected choices that feel genuinely designed for the
context. Vary between light and dark themes, different fonts, different aesthetics. You
still tend to converge on common choices (Space Grotesk, for example) across
generations. Avoid this: it is critical that you think outside the box!
</frontend_aesthetics>
```

### 医疗剂量（逐字使用）
- 若应用可能被问到用药剂量，把对应模型的 `<medical_guidance>` 块**原文照抄**进 system prompt。核心逻辑：仅当明确是给儿童的 OTC 用药时生效；剂量只从产品说明书（label）取，可联网时核对官方副本（如 DailyMed），取不到就不凭记忆给数字，改为指引用说明书剂量表、药剂师或医生；处方药一律以处方标签为准。不同模型版本措辞略有差异，务必用目标模型对应版本。

---

## 7. 迁移注意事项

1. **具体描述想要的行为**：写清输出里希望看到什么。
2. **用修饰语塑形**：`Include as many relevant features and interactions as possible. Go beyond the basics…`
3. **显式要求具体特性**：想要动画与交互元素就明说。
4. **更新 thinking 配置**：4.6+ 用 adaptive thinking；4.7+ 传 `budget_tokens` 会 400；用 `effort` 控制思考深度。
5. **弃用 prefill**（见第 2 节表格）。
6. **收敛防偷懒提示**：4.6 模型更主动，旧模型需要的加强语现在会过度触发。
7. **thinking block 原样回传、历史 append-only**：每轮 assistant 输出按 API 返回原样追加。在 **Fable 5.1 / Opus 5.5 / Sonnet 5.5 / Haiku 5.5** 上，**修改 thinking block 之前的对话**会导致报错，或（若选择容忍）该 block 被丢弃：编辑早前消息、重建 `system` 或 `tools`、就地摘要旧轮次都会让其后所有 thinking block 失效——这类变更要改用**对话中途的 system message** 与服务端上下文管理。
8. **迁移到 Claude Sonnet 5.5**：官方另有 *Claude Sonnet 5.5 migration guide*，覆盖从 Sonnet 5 迁移的 breaking changes 以及从 Sonnet 4.6 及更早版本的迁移步骤（含手动 extended thinking `budget_tokens` 的移除）。目标模型是 Sonnet 5.5 时先看该指南。
