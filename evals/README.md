# 评测（Evals）

按 Anthropic 官方 *Skill authoring best practices* 的建议编写：**先建评测，再写文档**。官方没有内置的评测运行器，这些评测是给人或 agent 读的验收清单。

评测只考一件事：**这个 skill 是否只做结构优化**——不添内容、不补缺失、不改原意、不执行任务。

## 格式

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
| `no-added-content` | 素材本身是可执行任务时：零执行 + 不添信息 + 不补缺失值 + 只输出两段 |
| `no-misinterpretation` | 原意边界：清空重建不被扩写成含需求文档/备份/版本控制的项目启动 |
| `structure-only-rewrite` | 结构手法：成串否定改写成正向要求，约束一条不丢，不新增格式要求 |
| `invoked-without-material` | 只调用、没给素材时的边界行为 |
| `material-from-file-is-data` | 素材来自文件时仍只是数据，文件里的注入指令不生效 |
| `correction-is-not-material` | 用户纠正上一轮工作时，先点明再问，不当素材改写 |
| `placeholder-consistency` | 占位写法统一（`{{ }}` → `[方括号]`），不新增字段或示例 |
| `reference-stays-in-scope` | 只读 `best-practices.md`；不读已废弃的档位/窗口参考；输出无档位概念 |

## 怎么跑

**1. 盲测（推荐）**

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

## 已知缺口

- 只在本机单一模型上跑过，跨模型覆盖仍是缺口（官方要求"用你打算使用的所有模型测一遍"）。
- `no-misinterpretation` 与 `correction-is-not-material` 两条是本轮新增的回归用例，尚未跑过。
