# 06　从 Authoring 到 Canonical，再到白板状态

## 1. 为什么不能直接执行 Authoring

Authoring 对模型友好，但对 Runtime 来说还不够稳定：

- 使用局部别名；
- 没有全局 action ID；
- 没有严格事件 sequence；
- 没有 lesson/board 稳定 ID；
- 相同动作重放时缺少幂等依据。

Canonical Profile 解决这些执行问题。

## 2. Canonical 是什么

Canonical 是由程序确定性生成的事件流。事件类型为：

- `lesson.open`
- `lesson.step`
- `lesson.close`

一个简化的 `lesson.open`：

```json
{
  "dsl": "octos.lesson",
  "version": "0.1",
  "profile": "canonical",
  "event": "lesson.open",
  "lesson_id": "lesson-001",
  "sequence": 0,
  "board": {
    "board_id": "board-001",
    "base_revision": 0,
    "region_intent": "new_topic"
  },
  "lesson": {}
}
```

`lesson.step` 包含已分阶段的动作：

```text
stage.before_speech
stage.during_speech
stage.after_speech
```

## 3. Normalizer 的输入

```ts
normalizeAuthoringLesson(authoring, {
  lessonId: "lesson-001",
  boardId: "board-001",
  baseRevision: 0,
  regionIntent: "new_topic",
  regionId: "topic-trigonometry",
  resourceContext,
});
```

左边是模型生成的教学语义，右边是宿主提供的机械上下文。

给定相同两者，输出必须完全相同。这叫确定性。

## 4. Normalizer 做什么

- 为 Lesson、Step、Beat、Action、Node、Connection、Group 分配稳定 ID；
- 把局部别名解析为 Canonical Target；
- 把 Fragment 别名解析为稳定 fragment ID；
- 把动作归入三个阶段；
- 补齐缺省动画设置；
- 生成 sequence；
- 规范化受控资源引用；
- 把变量和任务带入 `lesson.open`；
- 生成 `lesson.close` 结果。

Normalizer 不做什么：

- 不补写漏掉的教学解释；
- 不纠正错误数学内容；
- 不重新规划课程；
- 不移动学生笔迹；
- 不调用模型。

## 5. Schema 校验和语义校验不是一回事

Schema 校验回答：结构像不像一份 OLL？

例如：

- `steps` 是否是数组；
- `do` 是否为允许的枚举；
- `initial` 是否是数字；
- 必填字段是否存在。

语义校验回答：这份结构在含义上能不能执行？

例如：

- 别名是否先定义后引用；
- Binding 是否只引用已声明变量；
- `min <= initial <= max`；
- 学生任务是否有真实控制入口；
- Geometry angle control 是否指向合法中心和变量；
- Scene3D 任务的 node 是否真是 3D 场景；
- Board Context 是否匹配宿主白板和 revision。

两层都必须存在。

## 6. Reducer 是什么

Reducer 是一个纯状态转换器：给定当前状态和一个 Canonical 事件或动作，得到新的 Semantic Board State。

```text
旧状态 + Canonical Action → 新状态
```

它不访问网络，不查询 DOM，不依赖动画时钟。

## 7. Semantic Board State 是什么

它是课堂内容的语义状态，不是屏幕截图：

```ts
interface SemanticBoardState {
  board_id: string;
  revision: number;
  nodes: Record<string, Node>;
  connections: Record<string, Connection>;
  groups: Record<string, Group>;
  focus: string[];
  applied_lessons: string[];
  applied_steps: string[];
  applied_actions: string[];
  variables?: Record<string, VariableState>;
}
```

它能回答：

- 白板上语义上存在哪些内容；
- 哪些连接和分组存在；
- 当前焦点是什么；
- 哪些动作已经执行；
- 变量当前是多少。

它不能回答某张卡片此刻位于浏览器第几个像素，也不包含学生 Ink SVG。

## 8. revision 和幂等

Canonical Action 有稳定 `action_id`。Reducer 保存 `applied_actions`，同一动作重复送达时不会重复创建内容。

每次实际修改白板状态会推进 revision。后续 Lesson 若引用 revision 12 的白板，却在 revision 15 才到达，必须重新确认或拒绝，不能盲目套用过期引用。

这对增量文件、网络重连和页面刷新非常重要。

## 9. 变量在 Reducer 中如何工作

`lesson.open` 初始化变量。创建带 Binding 的节点时，Reducer用初值计算字段。

`animate` 在 Headless Reducer 中直接设置终值；浏览器 Runtime 额外产生中间帧。

学生操作调用 `setLessonVariable`，再对所有相关内容重新求值。变量范围在 Core 中被限制，不能由 UI 绕过。

## 10. 多份 Lesson 如何组成同一课堂

用户第一次问题生成 Lesson A，追问生成 Lesson B。octos-web 会把它们组合为一个课堂事件流：

- 使用一个会话级 `lesson_id` 和 `board_id`；
- 保留 Lesson A 的 Step；
- 把 Lesson B 的 Step 追加到后面；
- 根据 `new_topic` 或 `continue_topic` 选择 region；
- 保证 sequence 连续；
- 让 B 的显式 Board Context 引用解析到 A 已有节点。

这就是“聊天轮次很多，但白板是一块持续课堂”的基础。

## 11. 为什么 Canonical 通常使用 JSONL

JSONL 是一行一个 JSON 事件：

```text
{lesson.open...}
{lesson.step...}
{lesson.step...}
{lesson.close...}
```

优点：

- 可以逐事件追加；
- 易于流式处理；
- 单行失败容易定位；
- 不必重写一个巨大的 JSON 数组。

learning-coach 交付的是 Authoring JSON；octos-web 当前在浏览器内 Normalizer 后得到 Canonical 事件。固定 fixture 和独立 Harness 可以直接使用 Canonical JSONL。

## 12. Core 的标准验收链

贡献者应习惯这条链：

```text
parse Authoring JSON
→ validateAuthoringSchema
→ validateAuthoringLesson
→ normalizeAuthoringLesson
→ reduceCanonicalEvents
→ compare expected Semantic Board State
```

如果新增协议字段只改了 TypeScript 类型，没有进入这条链，就还没有真正完成。
