# 08　learning-coach 如何生成课件

## 1. 它是一个 Tool Skill

learning-coach 目录中最重要的三类文件：

```text
manifest.json                 Octos 可见的工具合同
src/main.ts                  Skill 实现
oll-contract.json            锁定的 OLL 版本和 Schema 指纹
references/                  vendored OLL Schema
scripts/oll-contract.mjs     同步和检查依赖合同
test/                        mock Provider 与生成管线测试
eval/ + scripts/eval-*.mjs   真实模型评测
```

构建后生成一个 `main` 可执行文件。Octos 每次工具调用都会启动它。

## 2. manifest 是外部合同

`manifest.json` 声明：

- Skill 名称和版本；
- 总超时；
- 工具名称和说明；
- 输入 JSON Schema；
- 允许注入的环境变量；
- 并发等级。

`concurrency_class: exclusive` 表示同一实例中的重型生成不应并发互相争抢资源。

工具说明不是普通 README。它会影响主 Agent 何时选择工具，所以必须说清：什么时候调用、什么时候不要调用、输入来源如何确定。

## 3. `oll_generate_lesson` 的输入边界

核心输入：

```ts
{
  turn_id,
  learner_request,
  request_source,
  language?,
  tutor_context?,
  learner_context?,
  session_context?,
  board_summary?,
  last_applied_action?,
  base_revision?,
  board_context?,
  source_observation?
}
```

三个来源的安全边界：

- `self_contained`：只把当前完整请求当作题目；
- `current_image`：必须有上游识别出的 `source_observation`；
- `explicit_board_follow_up`：必须是用户明确延续白板，且只使用显式 `board_context` 引用。

这能避免用户说“再讲讲这个”时，系统在没有明确指向的情况下凭空猜测。

## 4. 为什么先做要求整理

直接让模型一次输出完整 OLL，常见失败是：

- 忘了用户要求的函数图；
- 有两张图却没把它们连起来；
- 动画写了两个互不相干的变量；
- 学生任务没有可操作控件；
- 结构很长，Provider 的受控 JSON Schema 被拒绝。

因此 learning-coach 先生成一份内部“本次课程要求清单”。代码中可称 Lesson Brief，但对产品和读者最清楚的中文就是“课程要求清单”。

它包含：

- 用户请求中可逐项核对的要求；
- 教学目标；
- 视觉对象及用途；
- 对象之间的关系；
- 共享变量；
- 学生任务和 3D 任务；
- 每条要求是支持、仅文字解释还是暂不支持。

它不是 OLL，也不写入最终课件。

## 5. 双向完整性检查

只检查“清单中的条目有用户依据”不够，因为模型可能漏项。完整检查必须双向进行：

1. 清单每项都有请求依据，不能自行添加无关要求；
2. 用户每项明确要求都被清单覆盖，或明确标成不支持。

不能强求 evidence 必须机械复制某段用户原话。自然语言要求经常是隐含和组合的。更可靠的方法是保存稳定的请求条目 ID、来源位置和语义关系，再由独立 verifier 判断覆盖。

## 6. 能力计划

程序把课程要求确定性映射到 OLL 能力：

```text
visual geometry → write:geometry
visual plot     → write:plot
relationship   → connect
shared value   → lesson.variables + bindings
motion         → animate
student input  → slider / geometry control
task           → lesson.tasks
```

同时自动加入 OLL 的依赖闭包，例如正常 Lesson 骨架和必要的 `focus`。

模型选择教学意图，程序负责能力依赖。不要针对“单位圆”写一个特殊 if；新增抛物线、物理运动或统计图时应复用同一能力映射。

## 7. Provider Schema 与完整 OLL Schema

完整 OLL Schema 是本地权威合同，但不应原样作为每次模型请求的巨型受控输出 Schema。

learning-coach 会按本课能力计划投影出较小的 Provider Schema，并做 Provider 兼容清洗。例如 Gemini 不接受某些 JSON Schema 关键字，Octos 的通用工具适配和 Skill 的结构化生成适配都必须在边界上处理，而不能改坏原始 OLL 合同。

原则：

- Provider Schema 只约束本次生成；
- 完整 OLL Schema 最终验收；
- 未使用的新能力不应让旧题目的 Provider Schema 变大；
- Provider 兼容层不能篡改调用方原始 Schema 对象；
- 发送前记录 Schema 指纹和复杂度诊断，不记录秘密。

## 8. 并行和分段生成

目标集成版默认采用 parallel authoring：

1. 课程要求整理；
2. 派生若干教学段落；
3. 独立生成段落讲解；
4. 独立生成 Geometry、Plot 等视觉组件；
5. 按原计划顺序确定性组装；
6. 每形成一个合法前缀就发布部分 Artifact；
7. 最后加入需要全局信息的动画和任务；
8. 发布最终 Artifact。

并行不是让模型自由拼接多个互相矛盾的 Lesson。共享变量、别名、布局关系和最终任务仍由主计划与确定性组装器协调。

## 9. 为什么第一部分仍可能慢

主要耗时通常来自：

- 一次课程要求生成；
- 一次独立要求验证；
- 一次或多次教学段落生成；
- 视觉组件生成；
- Provider 排队和首 token 延迟；
- 输出 token 数量；
- 失败后的模型修复。

OLL Schema 复杂度会影响 Provider 接受率和受控解码速度，但不是全部原因。应按调用记录每阶段 latency、输入/输出 token、重试次数、Schema 大小和首个 Artifact 时间，而不是只测总时间。

## 10. 校验与局部修复

生成后调用完整链路：

```text
parse
→ Authoring Schema
→ OLL semantic validation
→ request coverage
→ capability allowlist
→ narration/visual teaching checks
→ normalize
→ reduce
```

若一张 Plot 缺曲线，只修 Plot 组件；若某个 Beat 没有 focus，只修该 Beat。只有结构无法局部恢复时才重新生成更大范围。

每次修复都要有明确 violation，不要给模型一句“再试一次”。

## 11. Artifact 输出

输出路径位于 `OCTOS_WORK_DIR/study/oll/`：

```text
<turn>.part-001.octos-lesson.json
<turn>.part-002.octos-lesson.json
<turn>.octos-lesson.json
```

Skill 通过 stderr 的结构化 Artifact 进度让宿主提前交付部分文件，通过 stdout 的 `files_to_send` 交付最终文件。

stdout 必须只包含最终协议对象。调试日志、模型原始输出和阶段日志写 stderr。

## 12. `oll_enhance_selection`

局部增强使用独立的小工具，而不是完整课程生成管线。输入包括：

- 不可变选区来源；
- 选区内容提示；
- 用户选择的工具 ID；
- 与选区重叠或接近的稳定白板目标；
- 视觉模型只对局部图片做出的识别及置信度；
- 可选的课程标题和白板摘要。

输出是独立 Enhancement Artifact，包含解释或安全单变量函数图。它不能输出“删除原稿”“替换笔迹”或“移动用户内容”。

局部增强使用小型响应 Schema 和较低 token 上限，正常情况下不应重复整堂课的几十秒管线。

## 13. OLL 依赖合同

`oll-contract.json` 记录：

- OLL 仓库；
- 锁定 commit；
- 包版本；
- Authoring Schema ID；
- vendored Schema SHA-256。

`npm run oll:check` 检查依赖和 vendored Schema 是否一致。

`npm run oll:sync` 在明确升级 OLL 后同步合同。不能只改 `package.json` 而留下旧 Schema。

## 14. 本仓库常用命令

```bash
npm install
npm run build
npm test
npm run oll:check
npm run contract:vertex
npm run eval:visual -- --case unit-circle-to-sine
```

真实评测会调用付费/联网模型；单元测试使用 mock Provider。贡献者应先离线测试，再明确运行真实评测。
