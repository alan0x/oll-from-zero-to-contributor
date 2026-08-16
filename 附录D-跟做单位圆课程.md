# 附录 D　跟做“单位圆到正弦波”课程

这是一条适合新贡献者的半天实验。不要先改代码，先把现有系统跑通。

## 1. 找到四份证据

在 OLL 仓库打开：

```text
examples/unit-circle-sine/lesson.authoring.json
examples/unit-circle-sine/lesson.canonical.jsonl
examples/unit-circle-sine/expected-state.json
examples/unit-circle-sine/ACCEPTANCE.md
```

它们分别回答：

- 作者写了什么；
- Normalizer 生成了什么；
- Reducer 最终得到什么；
- 产品行为怎样才算通过。

## 2. 读 Authoring 顶层

找到：

```json
"dsl": "octos.lesson",
"version": "0.1",
"profile": "authoring"
```

再找到 Lesson 标题和 goals、`theta` 变量、slider control 和 `reach-sine-maximum` 任务。

回答：为什么 `theta` 放在 Lesson，而不是 Beat？为什么任务允许 slider 和 geometry_point 两种 control？

## 3. 读 Geometry

找到 `unit-circle`：

- axes 使用 equal scale；
- `origin`、`point-p`、`foot` 三个点；
- circle、radius、projection 和 theta arc；
- `point-p` 的 angle control；
- 四条 Binding。

手工计算 `theta=π/2` 时：

```text
point-p.x = 0
point-p.y = 1
foot.x = 0
arc.end_angle = π/2
```

然后在 Runtime 中拖到 `π/2`，核对结果。

## 4. 读 Plot

找到 `sine-plot`：

- x 范围 `0..2π`；
- y 范围 `-1.2..1.2`；
- `sin(x)` 曲线；
- 五个关键点；
- `current-angle`；
- 两条 Binding。

确认 `current-angle.x` 和 `.y` 使用的是同一个 `theta`，没有第二份 Plot 私有状态。

## 5. 读跨视图关系

找到：

```json
{
  "do": "connect",
  "from": "unit-circle#point-p",
  "to": "sine-plot#sine-curve",
  "relation": "maps_to"
}
```

这条连接说明教学关系，不驱动数值。数值同步由 Binding 完成。Connection 和 Binding 解决的是两个不同问题。

## 6. 读动画

找到 `animate theta → 2π`。回答：

- 为什么模型不写 8000ms？
- Headless Reducer 得到什么？
- Web Runtime 如何显示中间帧？
- TTS 未到达时谁阻止动画提前播放？

答案分别在第 5、6、7、10 章。

## 7. 读 Canonical

对照第一条 `lesson.open`：

- 局部变量如何被带入；
- Lesson/Board ID 从哪里来；
- sequence 如何生成。

再对照 `lesson.step`：

- Authoring key 变成了什么稳定 ID；
- Action 分到了哪个 stage；
- `unit-circle#point-p` 如何变成 Canonical target。

## 8. 读 expected state

确认 Node 数量、Connection 数量、focus、applied actions、`theta` 最终值以及 Binding 求值后的 Geometry/Plot 点。

若 expected state 与浏览器最后一帧语义不同，说明 Runtime 和 Reducer 已经漂移。

## 9. 在 Harness 播放

```bash
cd /Users/alan0x/Documents/projects/octos-lesson-language
npm run harness:dev
```

依次测试：

1. 逐操作；
2. 逐 Beat；
3. 自动播放；
4. 暂停动画；
5. 恢复；
6. 重放；
7. slider；
8. 几何点拖动；
9. 完成任务；
10. 刷新恢复。

## 10. 做一个无协议变化的小修改

先只修改 narration 或 note 文案，不改结构：

1. 修改 Authoring example；
2. 运行 `npm run check:examples`；
3. 按仓库脚本和评审规则更新 golden；
4. 在 Harness 看渐进教学是否更清楚；
5. 检查最终状态变化是否符合预期。

这能让你第一次走通 Authoring → Canonical → State → Browser，而不会立即承担协议设计风险。

## 11. 再做一个负向实验

复制到临时文件，任选一个破坏：

- 把 Binding variable 改成未声明的 `angle`；
- 把 connect target 改成不存在的 fragment；
- 把任务 controls 改成课程没有的入口；
- 把 `theta.initial` 改到 max 之外。

运行校验，观察错误发生在 Schema 还是 semantic validation，并找到对应 Core 代码。

## 12. 最后走产品 E2E

在 `/learn` 使用自然问题：

> 请结合单位圆和 y=sin(x) 的函数图像，解释角度旋转如何变成周期波动。

不要告诉模型内部字段名。确认 learning-coach 自己规划出 Geometry、Plot、共享变量、动画和任务。

完成后，你已经走过：

```text
真实用户语言
→ Agent tool call
→ Skill 规划和生成
→ Authoring Artifact
→ Octos 文件交付
→ octos-web normalize
→ Player/Web Runtime
→ 学生操作和恢复
```

这就是开始贡献代码所需的第一条完整心智链。
