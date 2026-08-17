# 13　如何修改或新增一种 OLL 能力

这一章用“给 Plot 增加一条可绑定的垂直游标线”说明完整贡献路径。具体字段以当时 Schema 为准，重点是步骤。

## 1. 先写用户价值，不先写字段

需求：

> 教师可以在函数图上显示一条垂直游标线，游标位置由共享变量控制；学生拖动变量时游标同步移动。

验收：

- Authoring 能声明 guide；
- guide 有稳定 fragment alias；
- Binding 可绑定 `guide.value`；
- Core 拒绝不存在的 guide；
- Normalizer 保留语义；
- Reducer 用初值求值；
- Web Runtime 渲染；
- Slider 和动画同步更新；
- learning-coach 能在合适课程中生成；
- `/learn` 能播放、刷新和恢复。

## 2. 判断是否真的需要改语言

先问：

- 现有 `plot.guides` 已能表达吗？
- 只是 Runtime 没渲染吗？
- Binding target 已经允许，只是生成器没用吗？
- 只是 CSS 表现问题吗？

如果现有合法 OLL 足够，只修 Runtime 或 Skill，不新增字段。语言变更成本跨越所有仓库。

## 3. 在 OLL 仓库写正向和负向样本

正向样本：一份最小合法 Lesson 和期望 Canonical/Board State。

负向样本至少覆盖：

- guide alias 重复；
- Binding 指向不存在的 guide；
- value expression 引用未声明变量；
- 初值越界；
- 不支持的目标属性。

先写样本能逼迫设计回答真实问题，而不是只把一个字段塞进类型。

## 4. 修改 Authoring Schema

在 `schema/authoring/` 修改唯一权威 Schema：

- 保持 `additionalProperties` 策略一致；
- 定义必填字段；
- 给范围和枚举；
- 不加入 Provider 专用妥协；
- 更新 Schema ID/版本策略时遵循 decision record。

OLL Schema 应保持 Provider-neutral。Gemini 不支持某个关键字，应在 Provider adapter 处理。

## 5. 修改类型与语义校验

更新 `packages/core/src/types.ts`，再在 Core 语义校验中回答：

- alias 如何登记；
- fragment 如何寻址；
- Binding target 如何验证；
- 表达式变量如何检查；
- 与 revise、focus、connect 如何交互。

不要只依赖 TypeScript，因为模型输出和 Artifact 是运行时 JSON。

## 6. 修改 Normalizer 和 Reducer

Normalizer 必须：

- 生成稳定 fragment ID；
- 保留 guide 的数据；
- 把 Binding target 规范化；
- 相同输入产生相同输出。

Reducer 必须：

- 初始化 guide；
- 在变量变化时重算 value；
- 幂等应用；
- 产生可与 golden 比较的状态。

## 7. 修改 Player（如果需要）

若只是静态内容和 Binding，Player 可能不需要新操作。

若新增一种时序动作，则要修改：

- Canonical Action；
- playback operation；
- advance/pause/resume；
- checkpoint；
- incremental append；
- conformance test。

不要把时序动作只实现成 React effect。

## 8. 修改 Web Runtime

实现：

- SVG/DOM 渲染；
- 布局尺寸；
- fragment target rect；
- focus/emphasis；
- Binding 更新；
- reduced motion；
- 真实浏览器测试。

应在独立 Harness 验证，不先依赖 `/learn`。

## 9. 更新 OLL 文档和示例

至少更新：

- AUTHORING；
- SPEC（若 Canonical 变化）；
- README 当前状态；
- 示例和 acceptance；
- traceability/decision record（若为协议扩展）。

文档要给一个真实课程中的使用理由，不能只列字段类型。

## 10. 升级 learning-coach

1. 更新 OLL git dependency 到合并后的提交；
2. `npm install` 更新锁文件；
3. `npm run oll:sync` 同步 vendored Schema；
4. 更新本次生成能力映射；
5. 更新 Provider Schema 投影；
6. 更新 Prompt 中的可用语义；
7. 更新生成后 coverage；
8. 添加 mock 输出和真实 eval case；
9. `npm run oll:check && npm test`。

还要更新 OLL 导出的 `packages/core/src/capabilities.ts`。这张表只登记当前 Core/Runtime 真正能执行的节点、动作、Binding、控件和任务；learning-coach 从包中导入它，不能手抄一份容易漂移的能力清单。

对模型常见的等价写法，优先判断能否做通用、无歧义的程序转换。例如字段语义本来就是 `y` 的右侧表达式时，可以接受 `y = x^2` 并取右侧；但不能靠枚举“单位圆、弹簧、抛物面”题目修输出。转换后仍必须走同一个完整校验器。

如果 learning-coach 手抄了内容结构，应优先从 OLL Schema 组合/投影，减少漂移。

## 11. 升级 octos-web

1. 更新 OLL dependency；
2. 更新 lockfile；
3. 确认所需 exports 已打包；
4. 若 Runtime 包已完整支持，新 UI 只做装配；
5. 更新固定 fixture；
6. 增加 controller 和产品 E2E；
7. 验证增量加载和刷新。

涉及学生操作时，再验证播放所有权：学生输入是否错误暂停 TTS，老师动画是否只临时占用同一个变量。涉及相机或布局时，必须用真实浏览器确认正在讲的主视觉处于未被浮层遮挡的画面内。

不要在 octos-web 复制一份 Plot 实现来绕过 OLL Runtime。

## 12. Octos 何时需要修改

多数 OLL 内容能力不需要改 Octos。只有以下情况可能需要：

- Skill manifest 出现新的通用 Schema 形状，Provider adapter 不支持；
- Artifact 进度或文件交付协议需要新能力；
- Skill 超时/取消/并发合同变化；
- `/learn` 需要新的通用 UI 协议事件。

如果只为理解 `plot.guide` 去改 Octos，职责很可能放错了。

## 13. 跨仓库合并顺序

推荐：

```text
OLL
→ learning-coach 更新依赖
→ octos-web 更新依赖
→ 如有必要，Octos Provider/transport
→ 本地集成分支
→ E2E
→ 各 PR 按依赖顺序合并
```

消费方 PR 必须明确写出锁定的 OLL commit。OLL PR 还没合并时可以临时锁 commit，但最终应更新到可追踪的合并提交或版本标签。

## 14. 完成定义

一种新能力只有同时满足以下条件才算完成：

- 真实教学需求明确；
- Schema 和类型；
- 语义校验；
- Normalizer；
- Reducer；
- Player（如需）；
- Web Runtime；
- 正负测试；
- Harness 视觉验收；
- learning-coach 可生成；
- octos-web 可消费；
- 固定和真实 E2E；
- 文档和依赖锁定更新。

“前端看起来画出来了”不是协议完成。
