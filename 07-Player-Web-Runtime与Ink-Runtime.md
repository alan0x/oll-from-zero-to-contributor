# 07　Player、Web Runtime 与 Ink Runtime

## 1. 三者为何分开

OLL 仓库不是只有 Schema。它还提供三个不同层次的运行能力：

| 模块 | 是否依赖 DOM | 主要职责 |
| --- | --- | --- |
| Player Core | 否 | 把 Canonical 事件变成可暂停、恢复、追加的播放状态机 |
| Web Runtime | 是 | 布局并渲染语义白板，处理变量、动画、交互和 3D |
| Ink Runtime | 是 | 学生自由书写、擦除、框选、SVG 保存和选区快照 |

把它们分开后，Core 和 Player 可以在 Node.js 中快速测试，浏览器视觉问题则在 Web Runtime 中测试。

## 2. Player 如何编译课程

`compilePlaybackOperations(events)` 把事件编译为一串操作，例如：

```text
lesson.open
step.begin
beat.begin
action.before_speech
narration.begin
action.during_speech
narration.end
action.after_speech
beat.end
step.end
lesson.close
```

Player 维护：

- `cursor`：当前执行到哪；
- `status`：ready、playing、paused、waiting、completed；
- 当前 Step/Beat；
- 当前 narration；
- 当前变量动画；
- 投影出来的白板状态；
- checkpoint。

## 3. 增量追加和 waiting

动态分段生成时，第一部分课件可能先播放完，但最终部分尚未到达。Player 使用 `incremental` 模式：

- 播放到当前可用末尾时进入 `waiting`；
- `appendEvents` 验证并追加新事件；
- 新内容到达后继续播放；
- 最终 `lesson.close` 到达后才可进入 completed。

这比前端手工拼接 DOM 安全，因为追加仍受 sequence、Lesson 身份和已执行前缀约束。

## 4. Checkpoint 和恢复

Checkpoint 保存：

- 所属程序指纹；
- Lesson ID；
- cursor；
- Playback Projection。

恢复时 Player 会检查 checkpoint 是否真的属于同一事件程序。课程文件变了却硬套旧 cursor，应被拒绝。

## 5. 外部 narration 时序

在独立 Harness 中，可以估算旁白时间。在 `/learn` 中，TTS 是异步外部系统，所以使用 external narration timing：

1. Player 暴露下一段 narration；
2. octos-web 生成 TTS；
3. 音频开始时调用 `startNarration(beatId)`；
4. 音频结束时调用 `completeNarration(beatId)`；
5. Player 再推进依赖旁白边界的动作。

这防止动画和旁白完全分离。

## 6. Web Runtime 的布局

Web Runtime 读取 Semantic Board State，用相对关系计算布局。典型步骤：

1. 估算或测量节点尺寸；
2. 建立 anchor 依赖；
3. 按 `below/right_of/near` 等关系放置；
4. 处理新 region；
5. 计算连接端点；
6. 确保 focus 目标可见；
7. 根据真实 DOM 尺寸修正。

布局错误应在 Runtime 的布局测试和真实浏览器观察中修复，不应让模型改成绝对坐标。

## 7. 节点渲染

不同 kind 有各自渲染器：

- text/note/table：HTML；
- math：KaTeX；
- diagram/geometry/plot：DOM + SVG；
- scene3d：三维渲染层；
- image：经宿主 resolver 取得授权资源。

内容必须先经过协议校验。渲染器仍需防御异常值，不能信任任意 HTML。

## 8. 动画和降低动态效果

Runtime 接到变量动画后：

- 从当前值插值到目标值；
- 每帧重新求值 Binding；
- 支持 pause/resume/replay/reset；
- 遵守外部 narration 生命周期；
- 在用户开启 reduced motion 时降低或跳过中间动画；
- 最终值必须与 Headless Reducer 一致。

动画的视觉过程可以因设备不同，但最终语义状态不能不同。

## 9. 学生操作日志

变量滑杆、几何拖点和 3D 视角操作使用统一业务语义：

- 稳定操作 ID；
- 顺序；
- 输入方式：mouse、touch、pen、keyboard；
- start/update/commit 生命周期；
- 去重；
- 刷新恢复。

Web Runtime 根据操作更新变量或相机，再重新评估 Student Task。

## 10. Ink Runtime 是白板的一部分，但不是 OLL 节点

学生画笔必须与原白板使用同一个世界坐标系。拖动画布时，课程内容和笔迹一起移动；缩放时也一起缩放。

Ink Runtime 提供：

- navigate、draw、erase、select；
- 笔迹颜色；
- 橡皮；
- 撤销；
- SVG 序列化；
- 本地文档存储；
- lasso 选区；
- 不允许误拖动选区原稿；
- 选区快照和校验和。

底层使用 `js-draw`，OLL 仓库在外层统一世界坐标、持久化合同和只读选区行为。

## 11. 手写中文、英文和图形如何保存

保存层不先判断“这是中文还是图形”。它保存笔画形成的 SVG：路径、颜色、粗细和变换。

因此中文、英文、公式和手绘图形使用同一种原始笔迹格式。识别发生在需要理解选区时，由视觉模型读取选区图片；识别结果是带置信度的解释，不替代原始 SVG。

这样不会因为识别错误而破坏学生原稿。

## 12. 选区快照

一次选区生成不可变来源记录，包含：

- `source_id`；
- Ink 文档 ID 和版本；
- 世界坐标 bounds 或 lasso region；
- 选中内容 SVG；
- SHA-256 checksum；
- 创建时间。

后续“解释”“检查并建议”“生成函数图”都引用这个来源。若文档版本或 checksum 对不上，系统不能假装它还是同一份原稿。

## 13. 选区和语义白板对象的关系

用户圈选的区域可能同时覆盖：

- 学生笔迹；
- OLL 生成的公式节点；
- Plot 的某个点；
- Table 的一行。

前端会根据世界坐标查找附近或重叠的 Semantic Board Target，保存 overlap、distance、z-index 和稳定 ID。这样 Agent 不只收到一张截图，还能知道用户圈到了哪个结构化对象。

## 14. 独立 Harness 的作用

OLL Harness 不依赖 Octos 或 `/learn`，可以：

- 加载 Canonical fixture；
- 单步、逐 Beat 和连续播放；
- 暂停、恢复、变速；
- 缩放和拖动画布；
- 验证刷新恢复；
- 测试 Ink；
- 观察学生任务和 3D。

它是区分“OLL/Runtime 错误”和“产品集成错误”的关键。如果 Harness 也错，先修 OLL；Harness 正确但 `/learn` 错，再查消费方。
