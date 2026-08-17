# 08　learning-coach 如何生成课件

## 1. 它是一个带工具和直接操作入口的 Skill

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

构建后生成一个 `main` 可执行文件。Octos 每次工具调用都会启动它。完整课程通常由主 Agent 调用 `oll_generate_lesson`；白板选区按钮通过 Skill Action `learning.selection.enhance` 直接绑定到 `oll_enhance_selection`，不需要主 Agent 再决定一次。

## 2. manifest 是外部合同

`manifest.json` 声明：

- Skill 名称和版本；
- 总超时；
- 工具名称和说明；
- 输入 JSON Schema；
- 允许注入的环境变量；
- 并发等级；
- 可由前端直接触发的 `actions`、输入 Schema 和它们绑定的内部工具。

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
- 明确无法处理或仍有歧义的请求条目。

它不是 OLL，也不写入最终课件。

## 5. 当前怎样检查遗漏，以及检查不到什么

只检查“清单中的条目有用户依据”不够，因为模型可能漏项。当前实现把权威请求切成带稳定 `source_ref` 的语句，再要求每个请求条目映射到具体教学设计或 `unhandled_request_items`：

1. 清单每项都有请求依据，不能自行添加无关要求；
2. 用户每项明确要求都被清单覆盖，或明确标成不支持。

程序能检查引用是否存在、映射是否悬空、明确不支持的内容是否被报告。独立复核模型还能给出“可能漏了什么”的建议，但它只写诊断日志：它无条件说“没有遗漏”、与规划模型意见不同、超时或格式错误，都不能控制后续业务。

这仍不能证明自然语言理解百分之百正确。`source_ref` 证明的是“模型声称这项设计服务于哪段请求”，不是“模型一定理解了这段话”。跨学科真实评测仍然必需。

## 6. 本次生成能力范围

程序把课程设计候选映射到本次生成可使用的 OLL 能力：

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

映射按 `geometry`、`plot`、`scene3d`、`animation` 等通用能力工作，不按“单位圆”“弹簧”写题目补丁。OLL 自己在 `packages/core/src/capabilities.ts` 提供当前 Runtime 真正实现的节点、动作、Binding、控件和任务表；learning-coach 导入这张代码表，避免自己维护另一份口头清单。

生成结束后，程序从实际 Lesson 重新读取 `executable_capabilities`：真正出现了什么节点、哪些字段绑定变量、有哪些学生控件和可判定任务。规划范围用于约束生成，实际能力用于报告和执行检查，二者不能混为一谈。

## 7. Provider Schema 与完整 OLL Schema

完整 OLL Schema 是本地权威合同，但不应原样作为每次模型请求的巨型受控输出 Schema。

learning-coach 会按本次生成能力范围投影出较小的 Provider Schema，并做 Provider 兼容清洗。例如 Gemini 不接受某些 JSON Schema 关键字，Octos 的通用工具适配和 Skill 的结构化生成适配都必须在边界上处理，而不能改坏原始 OLL 合同。

原则：

- Provider Schema 只约束本次生成；
- 完整 OLL Schema 最终验收；
- 未使用的新能力不应让旧题目的 Provider Schema 变大；
- Provider 兼容层不能篡改调用方原始 Schema 对象；
- 发送前记录 Schema 指纹和复杂度诊断，不记录秘密。

## 8. 并行和分段生成

当前默认 `OLL_AUTHORING_STRATEGY=parallel`，意思是“拆成可独立组装的部分”，不等于一定同时向 Provider 发很多请求：

1. 课程要求整理；
2. 派生若干教学段落；
3. 独立生成段落讲解；
4. 独立生成 Geometry、Plot 等视觉组件；
5. 按原计划顺序确定性组装；
6. 每形成一个合法前缀就发布部分 Artifact；
7. 最后加入需要全局信息的动画和任务；
8. 发布最终 Artifact。

真正同时执行多少请求由 `OLL_PARALLELISM` 控制，默认值是 `1`。这是为了兼容免费层或低并发配额；把它调到 `2` 或更高之前要先确认 Vertex 配额，并把 429、延迟和首次 Artifact 时间纳入实测。

拆分也不是让模型自由拼接多个互相矛盾的 Lesson。共享变量、别名、布局关系和最终任务仍由主计划与本地组装代码协调。学生任务细节延后生成，不阻塞首个可播放前缀。

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
→ 本次能力范围与实际可执行能力检查
→ narration/visual teaching checks
→ normalize
→ reduce
```

若一张 Plot 缺曲线，只修 Plot 组件；若某个 Beat 没有 focus，只修该 Beat。模型常见但含义唯一的写法，例如 Plot 中的 `y = ...` 或曲面中的 `z = ...`，由程序安全地取等号右侧，再交给同一个受限数学解析器验证。这是按语言结构处理，不是枚举某几道题。

单个视觉组件经过局部修复仍无效时，可以降级为带稳定视觉 ID 的可重试提示，让其余课程继续播放。鉴权、配额、Provider Schema、取消和总超时属于系统错误，必须整体报错，不能伪装成“只有一张图没画好”。

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

局部增强使用独立的小工具和响应 Schema，而不是完整课程生成管线。产品按钮先通过 `skill.actions.v1` 能力协商调用 manifest action，Octos 把选区图片放入会话工作区，再按绑定规则把相对路径传给工具。输入包括：

- 不可变选区来源；
- 选区内容提示；
- 用户选择的工具 ID；
- 与选区重叠或接近的稳定白板目标；
- 会话工作区内经过边界检查的 PNG/JPEG/WebP 选区图片；
- 可选的备用识别文字及置信度；
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
