# 10　octos-web 如何组成 `/learn` 课堂

## 1. LearningWorkspace 是产品编排层

`src/learning/learning-workspace.tsx` 把以下能力组合起来：

- 会话消息和 UI 协议；
- 文本、语音和摄像头输入；
- OLL Artifact 发现和加载；
- Player；
- TTS；
- 无限白板；
- Ink；
- 学生任务；
- 选区增强；
- 教师形象和加载反馈；
- 课程目录与复习。

它不应重新实现 OLL Reducer 和布局算法，而是调用 OLL 包。

## 2. Artifact 如何被发现

前端从两个来源收集 `.octos-lesson.json`：

1. 当前实时 thread projection 中的文件；
2. 刷新后从 session files 恢复的持久文件。

部分文件形如：

```text
turn.part-001.octos-lesson.json
turn.part-002.octos-lesson.json
```

最终文件：

```text
turn.octos-lesson.json
```

前端按同一 turn 的 rank 选择最新 Artifact，避免把每个前缀当成独立课程重复播放。

## 3. 加载 Authoring Artifact

`loadOllLessonArtifact`：

1. 通过会话文件 API 下载 JSON；
2. 取出当前 `board_context`；
3. 调用 `normalizeAuthoringLesson`；
4. 对无外部引用的 Lesson 先做 `reduceCanonicalEvents` 自检；
5. 返回 Canonical Events。

有外部 Board Context 的 Lesson 必须与前序课堂事件一起 reduce，因为它引用的节点来自之前的 Artifact。

## 4. 多轮课程组合

`composeOllClassroomEvents` 把多个 Lesson 合成一个会话课堂：

- 只保留一个会话级 `lesson.open`；
- 把每份 Lesson 的 Step 按顺序追加；
- 重写 sequence；
- 维持当前 topic region；
- 新主题切换 region，追问继续旧 region。

课程目录由每份 Artifact 的标题和 Step ID 构建，因此用户可以回看以前的主题。

## 5. 增量追加

首次拿到第一部分时创建 BrowserLessonSession。后续 Canonical 事件通过 `appendEvents` 加入当前 Player。

前端要区分：

- 当前已收到的操作数；
- 已经 append 的操作数；
- 最终交付是否 settled；
- Player 是 waiting 还是 completed。

不能因当前前缀暂时没有任务，就宣布整堂课已经完成。只有最终 Artifact 才包含完整的 after-lesson 任务。

## 6. TTS

`use-oll-narration-tts.ts` 负责：

- 发现下一段 narration；
- 请求语音；
- 防止同一段重复生成或播放；
- 音频开始/结束时推进 Player；
- 静音和音频解锁；
- 失败时允许课程继续或给出可理解提示。

TTS 等待期间，教师形象显示温和的等待动画。它表达的是“系统仍在工作，请稍等”，不向用户暴露内部技术阶段。

## 7. 白板渲染

`InfiniteBoard` 提供世界坐标、平移和缩放。`OllLessonBoard` 使用 OLL Web Runtime 渲染 Semantic Board State。

关键不变量：

- OLL 内容和 Ink 使用同一世界坐标；
- 拖动画布时两者一起移动；
- UI 浮层使用屏幕坐标，课程对象使用世界坐标；
- 用户可以在旁白期间拖动和缩放白板；新的老师 `focus` 到达时可以把相关内容重新带回合适视野；
- 相机取景必须避开工具栏、任务卡、旁白、输入框和教师形象等实际遮挡区；
- 节点测量后布局可修正，但语义 ID 不变。

Runtime 会把相连的主视觉按实际尺寸放近，并在讲解辅助公式/文字时把最近的主视觉一起保留。不要用“让模型改几个 `near`”修复画面外问题；同类课程反复发生时，根因通常在布局和 focus 相机。

## 8. 学生变量操作

React controller 暴露：

```ts
handleStudentVariableInput(alias, value, event)
```

event 包含：

- phase：start/update/commit；
- control：slider 或 geometry_point；
- input：mouse/touch/pen/keyboard；
- operation_id。

UI 不直接改 Plot 点的位置，而是提交变量操作；Runtime 更新变量后所有 Binding 同步变化。

这些操作默认不会暂停 Player，也不会取消正在播放的 TTS。老师动画正在写同一变量时，前端只暂时禁用这一个变量的学生输入；画布浏览、3D 观察和其他控件仍可工作。

## 9. 3D 操作

3D 控件把 orbit、zoom、preset、reset 统一为 Scene3D view operation。Runtime 保存当前视角，评估 3D Student Task，刷新后恢复。

不要只保存 Three.js 或 `<model-viewer>` 的临时组件状态；业务恢复应依赖稳定、可版本化的视角记录。

## 10. Ink 挂载

Ink Runtime 被挂在无限白板世界层中，而不是盖在页面 viewport 上。工具栏可以浮在上层，但笔迹坐标属于白板。

用户不需要“进入一个会让白板消失的书写模式”。画笔、橡皮、选择和导航是同一白板的工具状态。切回导航后笔迹仍然存在。

## 11. 选区工具栏

选择工具默认画矩形；需要沿不规则轮廓选择时，用户可以单独切换自由套索。完成选区后，前端先列出“我的笔迹”和选区命中的稳定白板对象，用户选择这次到底在问哪一个来源。切换来源不会修改 Plot 或笔迹，只会改变提交给辅助工具的上下文。

前端维护有限工具注册表：

| 工具 | 行为 |
| --- | --- |
| 解释这部分 | 生成旁边的解释 |
| 检查并建议 | 保留原稿，在旁边指出问题和建议 |
| 生成函数图像 | 对明确单变量公式生成安全 Plot |
| 围绕这部分讲一课 | 把选区作为 composer 的显式引用 |

模型不决定工具栏里出现哪些工具。前端根据内容类型和语义目标过滤注册表，用户做最终选择。

“解释这部分”“检查并建议”“生成函数图像”通过 `learning.selection.enhance` Skill Action 直接执行。前端要先确认后端协商了 `skill.actions.v1`，再调用 `skill/action/invoke`；它们不是一条普通聊天消息。

## 12. 局部增强层

Enhancement Artifact 不进入完整 OLL Lesson 播放流程。它作为独立辅助层渲染在选区旁边，并保存：

- 来源 `source_id`；
- 生成 turn；
- 响应类型；
- 解释或 Plot；
- 与来源的关系；
- 隐藏状态。

用户可以删除或最小化辅助内容，但原稿始终不变。最小化后只保留选区附近的 `?` 按钮；按钮平时尺寸较小，鼠标经过时放大，点击恢复原卡片。卡片本身在正常状态保持可读尺寸，不随 rollover 反复缩放。

课程重播时，前端为这次播放使用独立的 Ink 运行记录：旧笔迹暂时隐藏，播放结束后恢复，并与重播期间新增的笔迹合并。重播不是清空学生原稿。

## 13. Composer 白板引用

用户选择“围绕这部分讲一课”时，前端把选区暂存为 composer reference。发送新问题时上传局部图片，并加入：

- reference ID；
- source/checksum；
- 内容提示；
- board ID/revision；
- 稳定 board targets。

Agent 明确知道这是 `explicit_board_follow_up`。发送完成后 UI 可清理临时 composer 附件，但会话 Artifact 和白板记录仍可恢复。

## 14. 前端常用命令

```bash
pnpm install
pnpm dev
pnpm run build
pnpm run lint
pnpm run test:unit
pnpm test
```

单元测试覆盖 Artifact 排序、课程组合、Player controller、selection 工具和持久化；Playwright 用于真实浏览器 E2E。

## 15. 前端问题的定位顺序

1. 会话中有没有 Artifact？
2. 文件下载是否成功？
3. Authoring normalize 是否失败？
4. compose 后 Canonical sequence 是否正确？
5. Player cursor/status 是否推进？
6. Semantic Board State 是否有对象？
7. Web Runtime 是否布局和渲染？
8. UI 遮挡、相机或 CSS 是否让它不可见？

不要一看到“白板没出现”就直接修改模型 Prompt。
