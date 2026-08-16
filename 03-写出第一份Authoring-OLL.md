# 03　写出第一份 Authoring OLL

## 1. Authoring Profile 是什么

Authoring Profile 是给课件作者使用的 OLL 形式。当前主要作者是 learning-coach 中的模型，也可以是开发者手写的示例。

它让作者表达“讲什么、写什么、怎么关联”，不要求作者生成全局 ID、事件序号和白板 revision。

浏览器不直接执行 Authoring。Authoring 必须先经过 OLL Core 的校验和 Normalizer。

## 2. 最小完整结构

```json
{
  "dsl": "octos.lesson",
  "version": "0.1",
  "profile": "authoring",
  "lesson": {
    "mode": "explain",
    "language": "zh-CN",
    "title": "认识正弦函数",
    "goals": ["理解正弦值来自单位圆上点的纵坐标"]
  },
  "steps": [
    {
      "key": "introduce",
      "purpose": "给出核心定义",
      "beats": [
        {
          "key": "definition",
          "say": "单位圆上点 P 的纵坐标，就是这个角的正弦值。",
          "delivery": "patient",
          "actions": [
            {
              "do": "write",
              "as": "definition-note",
              "kind": "note",
              "role": "definition",
              "content": {
                "title": "核心关系",
                "text": "P(cos θ, sin θ) 的纵坐标为 sin θ"
              },
              "place": {
                "relation": "new_region",
                "region_role": "lesson_origin"
              }
            },
            {
              "do": "focus",
              "targets": ["definition-note"],
              "intent": "read_definition"
            }
          ]
        }
      ]
    }
  ],
  "close": {
    "summary": "正弦值可以看作单位圆上点的纵坐标。",
    "focus": ["definition-note"]
  }
}
```

这份课件已经包含了最重要的层级：

```text
Lesson
└── Step
    └── Beat
        ├── narration（say）
        └── Actions
```

## 3. Lesson：整轮课件

`lesson` 描述这一轮课程的基本信息：

- `mode`：当前 `0.1` 固定为 `explain`；
- `language`：讲解语言；
- `title`：用户看得懂的标题；
- `goals`：本轮要达成的学习目标；
- 可选 `variables`：全课共享的数值变量；
- 可选 `tasks`：课后开放的学生操作任务；
- 可选 `adaptation`：有来源的教学适配策略。

`lesson` 不是课程管理系统中的长期课程实体。一次用户请求生成一份 Authoring Lesson，多个 Lesson 可以被产品组合进同一块持续白板。

## 4. Step：一个有目的的教学段落

Step 不是视觉卡片，也不是固定时长的幻灯片。它是一段有明确目的的教学推进。

```json
{
  "key": "map-circle-to-wave",
  "purpose": "建立圆周点纵坐标和正弦图之间的对应",
  "beats": []
}
```

`purpose` 用于审计这一步为什么存在。它不是隐藏思维链，也不直接显示给学生。

好的 Step 往往能用一个动词说明：观察、比较、推导、验证、总结。

## 5. Beat：一次可播放的最小讲解单元

Beat 把旁白和同一时刻发生的白板动作放在一起：

```json
{
  "key": "show-projection",
  "say": "我们把点 P 的纵坐标投影到右侧函数图。",
  "delivery": "careful",
  "actions": []
}
```

`say` 可直接交给 TTS。`delivery` 表达语气意图：

- `neutral`
- `patient`
- `encouraging`
- `careful`
- `emphatic`

Beat 中每个动作可用 `when` 指定阶段：

- `before_speech`
- `during_speech`
- `after_speech`

缺省为 `during_speech`。

不要在 Beat 中写毫秒时间线。TTS 到达时间、语速和设备性能不稳定，Runtime 才能决定实际调度。

## 6. Action：老师对白板做的语义动作

当前动作集合：

| 动作 | 作用 |
| --- | --- |
| `write` | 创建一个新白板节点 |
| `revise` | 显式修订已有节点 |
| `emphasize` | 强调节点、片段、连接或分组 |
| `connect` | 在两个对象之间建立语义连接 |
| `group` | 把节点组织成教学分组 |
| `focus` | 调整学生当前应该看的内容 |
| `point` | 教师指向某个对象 |
| `expression` | 设置有限的教师表情 token |
| `animate` | 把共享变量动画到目标值 |

Action 描述意图，不描述 DOM 操作。

## 7. `as`、`key` 和局部别名

模型不生成全局 ID，而是在一份 Lesson 内使用局部别名：

```text
unit-circle
sine-plot
point-p
summary-group
```

一般规则是小写字母开头，只包含小写字母、数字和连字符。

- Step 和 Beat 使用 `key`；
- 创建对象的动作使用 `as`；
- 后续动作使用别名引用对象。

例如：

```json
{"do":"focus","targets":["unit-circle"],"intent":"inspect_circle"}
```

别名必须先定义后使用，同一作用域不能重复创建。Normalizer 会把它们变成稳定 Canonical ID。

## 8. Node、Connection 和 Group

`write` 创建 Node。Node 是白板上的内容对象，例如一张笔记卡、一组公式、一张图。

`connect` 创建 Connection。Connection 不只是视觉箭头，它有 `relation` 和可选 `label`，表达两个对象为什么相连。

`group` 创建 Group。Group 把多个 Node 组织成一个可聚焦的教学区域。

三者引用能力不同：

| 用途 | Node | Connection | Group |
| --- | --- | --- | --- |
| 强调/指向 | 可以 | 可以 | 可以 |
| 聚焦 | 可以 | 可以 | 可以 |
| 成为布局 anchor | 可以 | 不可以 | 可以 |
| 成为 group member | 可以 | 不可以 | 可以 |

## 9. `close` 不是最后一个 Beat

`close` 表示这一轮 Lesson 的结束结果：

```json
{
  "summary": "旋转角通过纵坐标投影形成正弦波。",
  "focus": ["summary-group"]
}
```

`summary` 是文字摘要；`focus` 是结束时应该保留在视觉焦点中的对象别名。二者不能混淆。

## 10. 第一次本地验证

在 OLL 仓库中：

```bash
npm install
npm run build
npm run check:examples
```

实际开发时不要只用 TypeScript 类型断言说“它应该是 AuthoringLesson”。应调用：

```ts
const lesson = parseAuthoringLessonJson(jsonText);
validateAuthoringLesson(lesson, resourceContext);
const events = normalizeAuthoringLesson(lesson, host);
const state = reduceCanonicalEvents(events);
```

这四步分别捕获 JSON、Schema、语义、转换和执行问题。

## 11. 什么叫一份好的 Authoring OLL

合法不等于好课。至少还要满足：

- 用户要求的对象真的出现；
- 课程是逐步展开，不是第一 Beat 铺满整页；
- 每个 Step 有明确教学目的；
- narration 和动作相互支持；
- `focus` 确实把视线带到正在讲的内容；
- 连接有教学含义，不是装饰线；
- 动态关系由共享变量表达，不是做两套互不一致的动画；
- 学生任务有真实可用的控制入口；
- 最终白板保留清晰的知识结构。
