# 附录 A　术语表

## Action

老师在一个 Beat 中执行的语义动作，如 write、focus、connect、animate。

## Agent

反复调用大模型、执行工具、把结果送回模型，直到完成用户请求的运行程序。Octos 提供 Agent Runtime。

## Alias

Authoring Lesson 内的局部别名。模型使用它引用 Step、Beat、Node、Fragment、Connection、Group 或 Variable；Normalizer 将它转换成稳定 ID。

## Artifact

Skill 生成并通过 `files_to_send` 或进度事件交付的文件。整课是 `.octos-lesson.json`，局部增强是 `.octos-selection-enhancement.json`。

## Authoring Profile

面向模型和人类作者的 OLL 创作形式。包含教学语义和局部别名，不包含大量机械执行字段。

## Beat

最小可播放讲解单元，把一段 narration 和若干白板动作放在一起。

## Binding

把 Geometry 或 Plot 内某个数值字段绑定到共享变量表达式的合同，例如 `point-p.y = sin(theta)`。

## Board Context

用户明确附加到后续问题中的只读白板引用，包含 board ID、revision 和稳定 target。不是整块白板的自由文本摘要。

## Canonical Profile

面向 Runtime 的 OLL 事件形式，包含稳定 ID、sequence、revision 和规范化引用。

## Capability Plan / 本次生成能力范围

learning-coach 根据课程设计候选派生的本次生成能力范围。它用于缩小 Provider Schema，但不是 OLL 标准，也不能证明最终课件实际拥有这些能力。

## Executable Capabilities

从最终 Lesson 实际读取出的节点、动作、Binding、学生控件和任务。OLL Core 的 `capabilities.ts` 定义当前版本真正实现的能力表，learning-coach 用它检查和报告结果。

## Checkpoint

Player 保存的可恢复播放位置和投影状态，带程序指纹，不能套到不匹配课程。

## Composer

`/learn` 屏幕底部的主要输入区。适合发起完整教学请求，也可附带明确白板选区引用。

## Connection

由 `connect` 创建的语义关系对象，有 from、to、relation 和可选 label。

## DSL

Domain-Specific Language，领域专用语言。OLL 只解决 AI 白板课程表达，不是通用编程语言。

## Fragment

Node 内部可被寻址的部分，例如公式项、Plot 曲线、Geometry 点、Diagram element。引用形式为 `node#fragment`。

## Group

若干 Node/Group 的教学分组，可被聚焦并作为布局 anchor。

## Harness

OLL 仓库内的独立浏览器实验室，不依赖 Octos 和 `/learn`，用于播放和验证 OLL Runtime。

## Ink Runtime

学生自由书写层，负责 js-draw 接入、SVG 持久化、世界坐标、框选和选区快照。

## Lesson

一次用户教学请求对应的完整 Authoring 课件。多个 Lesson 可以组成同一会话课堂。

## Lesson Brief / 课程要求清单

learning-coach 内部在正式生成课件前由模型提出、再由程序检查结构和映射的课程设计候选。它不是 OLL 标准，也不是绝对正确的用户意图。

## Node

由 `write` 创建的白板内容对象，如 Note、Math、Geometry、Plot、Scene3D。

## Normalizer

把合法 Authoring Lesson 和宿主上下文确定性转换成 Canonical Events 的 OLL Core 程序。

## Narration

Beat 的旁白文本，通常来自 `say`，可交给 TTS。

## OLL

Octos Lesson Language。描述 AI 教师按步骤说、写、画、强调、关联和开放交互的语言。

## Player

把 Canonical Events 编译成播放操作流并维护 cursor、pause、waiting、append 和 checkpoint 的状态机。

## Profile

同一 OLL 的用途形态。当前主要是 Authoring 和 Canonical。不要与 Octos 用户 Profile 混淆。

## Provider

提供大模型 API 的服务，例如 Google Vertex/Gemini。不同 Provider 支持的 JSON Schema 子集不同。

## Reducer

将 Canonical Event/Action 作用到 Semantic Board State 的确定性纯状态转换器。

## Region

无限白板中的主题区域。新主题可创建新 region，追问可继续已有 region。

## Resource Context

宿主授权给 OLL 使用的图片 asset 和 region 清单。模型不能自行发明资源。

## Revision

Semantic Board State 的版本号。后续引用用它检测白板是否已经变化。

## Runtime

执行协议的程序。OLL Web Runtime 负责浏览器白板，Octos Runtime 负责 Agent/Skill；二者不是同一个 Runtime。

## Scene3D

OLL 的三维内容类型，表达对象、相机、轴对齐截面和真实交集语义，由 Web Runtime 渲染并接收视角操作。

## Schema

描述 JSON 结构合法形状的合同。Schema 不能替代引用、变量和任务等语义校验。

## Selection Enhancement

根据不可变学生选区，在旁边生成解释、建议或 Plot 的局部辅助内容。它不是完整 Lesson。

## Semantic Board State

Reducer 得到的白板语义状态：节点、连接、分组、焦点、变量和已应用动作。不包含 DOM 像素和 Ink SVG。

## Skill

可由 Octos 安装和调用的独立工具包，通常包含 manifest、说明和可执行 `main`。

## Skill Action

Skill 在 manifest 中声明的、可以由产品 UI 直接触发的有限操作。前端和 Octos 先协商 `skill.actions.v1`，再用 `skill/action/invoke` 调用；它不需要主模型再次选择工具。

## Step

一段有明确教学目的的课程推进，包含一个或多个 Beat。

## Student Operation

学生对变量、几何点、3D 视角或 Ink 选区的稳定业务操作记录。

## Student Task

由 OLL 声明、Runtime 确定性评估的操作任务，包含 prompt、允许操作、完成条件、提示和成功反馈。

## TTS

Text-to-Speech，将 narration 文本转换为语音。实际音频生命周期由 octos-web 与 Player 协调。

## ToolSpec

Octos 暴露给主模型的工具名称、说明和参数 Schema。

## Variable

Lesson 顶层声明的共享数值状态。允许多个，全课可见；当前没有 Beat 局部变量。

## Web Runtime

OLL 包中的浏览器渲染和交互实现，负责布局、节点、动画、变量控制、任务和 3D。

## World Coordinates

无限白板内容使用的稳定坐标系。OLL 节点和 Ink 都应使用它；工具栏等 UI 浮层使用屏幕坐标。
