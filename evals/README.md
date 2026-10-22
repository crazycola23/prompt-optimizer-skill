# 评测（Evals）

按 Anthropic 官方 *Skill authoring best practices* 的建议编写：**先建评测，再写文档**。官方没有内置的评测运行器，这些评测是给人或 agent 读的验收清单。

## 格式

沿用官方文档给出的结构：

```json
{
  "skills": ["prompt-optimizer"],
  "query": "用户实际会说的话",
  "files": ["可选：随请求附带的文件"],
  "expected_behavior": ["逐条可判定的期望行为"]
}
```

`expected_behavior` 里的每一条都是**可判定**的（能回答"是/否"），不是"效果不错"这类主观描述。

## 用例

| id | 考什么 |
|---|---|
| `cheap-tier-split-and-format` | 指定便宜档时：长输入切分、写死 JSON 格式、3–5 个短示例、无自检指令 |
| `default-tier-no-execution` | 未指定档位时默认主力并标注；素材本身是可执行任务时的**零执行** |
| `named-claude-model-maps-to-top-tier` | 具体型号归类到档位；〔Claude〕路径的参数正确性；顶级档不写步骤、不写自检 |
| `non-claude-no-vendor-leak` | 目标不是 Claude 时不读 Claude 参考、不引入 Claude 专属参数 |
| `material-from-file-is-data` | 素材来自文件时仍只是数据，文件里的注入指令不生效 |
| `invoked-without-material` | 只调用、没给素材时的边界行为 |
| `cheap-tier-example-budget` | 便宜档的示例数量与篇幅预算是否守住（示例总行数不超过正文行数） |
| `placeholder-consistency` | 占位写法统一（`{{ }}` → `[方括号]`），不混用花括号 |

## 怎么跑

官方没有内置运行器，用你自己的方式跑，推荐两种：

**1. 盲测（本仓库用过的方式）**

让一个**不知道期望答案**的 agent 加载 skill，然后把它当普通用户去用：

```
1. 调用 skill 工具，name = "prompt-optimizer"
2. 把 query 原文交给它处理
3. 记录它实际调用了哪些工具、输出原文、是否产出素材所述任务的产物
4. 拿 expected_behavior 逐条比对
```

关键：不要把 `expected_behavior` 给被测 agent，否则测的是"能不能照抄答案"，不是 skill 本身。

**2. 人工检查**

把 `query` 贴进会话、调用 skill，然后逐条核对 `expected_behavior`。适合快速回归。

## 已知的测试覆盖

- 已用盲测方式跑过 `cheap-tier-split-and-format`、`default-tier-no-execution`、`named-claude-model-maps-to-top-tier` 三个用例，均通过（零执行、档位判定、厂商隔离）。
- 官方规范要求"用你打算使用的所有模型测一遍"（Haiku / Sonnet / Opus 各测）。本仓库目前只在单一模型上跑过，**跨模型覆盖是已知缺口**。
