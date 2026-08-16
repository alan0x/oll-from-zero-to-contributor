# 附录 C　开发者检查清单

## 1. 开始前

- [ ] 我能用一句话说明用户价值。
- [ ] 我知道问题首次出现在哪一层。
- [ ] 我检查过现有 OLL 能否表达，不会为了前端方便随意扩语言。
- [ ] 我读过目标仓库的 AGENTS/README。
- [ ] 我确认工作树中的既有修改属于谁。
- [ ] 我记录了四仓库当前分支、提交和依赖。

## 2. OLL 改动

- [ ] Authoring Schema 已更新。
- [ ] TypeScript 类型已更新。
- [ ] 运行时 JSON 校验已更新。
- [ ] 语义校验已更新。
- [ ] Normalizer 已更新。
- [ ] Canonical Schema/类型在需要时已更新。
- [ ] Reducer 已更新。
- [ ] 正向 fixture 已增加。
- [ ] 负向 fixture 已增加。
- [ ] expected Semantic Board State 已更新。
- [ ] Player/checkpoint/append 在需要时已更新。
- [ ] Web Runtime 已实现。
- [ ] Ink Runtime 在需要时已实现。
- [ ] Harness 真实浏览器通过。
- [ ] 文档和 decision record 已更新。

## 3. learning-coach 改动

- [ ] OLL dependency 锁定到正确提交。
- [ ] lockfile 已更新。
- [ ] `oll-contract.json` 已更新。
- [ ] vendored Schema 已同步。
- [ ] 课程要求能表达新需求。
- [ ] 双向 coverage 不会静默漏项。
- [ ] 能力计划和依赖闭包已更新。
- [ ] Provider Schema 投影已更新。
- [ ] 完整 OLL 校验仍为最终门禁。
- [ ] 失败有明确 violation。
- [ ] 局部修复范围合理。
- [ ] mock Provider 测试通过。
- [ ] 真实 eval 覆盖自然用户表达。
- [ ] 性能和重试次数有记录。

## 4. Octos 改动

- [ ] 这项变化真的属于通用 Agent/Skill/Provider/transport。
- [ ] Provider Schema 适配只修改副本。
- [ ] 其他 Provider 不受影响。
- [ ] ToolSpec 的标准语义没有被削弱。
- [ ] Skill stdout/stderr 协议保持兼容。
- [ ] `tool_call_id` 能关联完整生命周期。
- [ ] 文件交付可恢复。
- [ ] 超时、取消、重试分类正确。
- [ ] 未泄露环境变量或凭据。
- [ ] Rust fmt/check/test 通过。

## 5. octos-web 改动

- [ ] 使用 OLL 包而不是复制 Runtime。
- [ ] 部分/最终 Artifact 正确去重。
- [ ] 多 Lesson 组合 sequence 正确。
- [ ] Player waiting/completed 正确。
- [ ] TTS 开始/完成与 Beat 生命周期一致。
- [ ] 加载期间有可理解反馈。
- [ ] OLL 与 Ink 使用同一世界坐标。
- [ ] mouse/touch/pen/keyboard 都可用。
- [ ] reduced motion 可用。
- [ ] 原始学生笔迹没有被修改。
- [ ] 刷新恢复完整。
- [ ] Unit、Playwright 和真实设备测试通过。

## 6. 生成质量

- [ ] 测试问题像真实用户说的话。
- [ ] 没有把测试要求中的内部字段名教给模型。
- [ ] 用户明确要求的每项表现都出现。
- [ ] 视觉对象有教学用途。
- [ ] 多视图共享同一变量。
- [ ] narration 与动作相互支持。
- [ ] 课程逐步展开。
- [ ] Student Task 有真实控制入口。
- [ ] 任务判定由 Runtime 确定性完成。
- [ ] 最终白板结构清晰。

## 7. 性能

- [ ] 记录 Agent 工具选择耗时。
- [ ] 记录要求整理与验证耗时。
- [ ] 记录第一段生成耗时。
- [ ] 记录视觉组件耗时。
- [ ] 记录首 Artifact 到达耗时。
- [ ] 记录首个可见动作耗时。
- [ ] 记录首段 TTS 到达耗时。
- [ ] 记录最终课件耗时。
- [ ] 记录输入/输出 token。
- [ ] 记录重试和修复率。
- [ ] 局部增强使用独立预算。

## 8. 安全与隐私

- [ ] 模型输出经过 Schema 和语义校验。
- [ ] 数学表达式使用安全解析器。
- [ ] 图片来自授权 Resource Context。
- [ ] Board Context 由用户明确附加。
- [ ] board ID/revision 匹配。
- [ ] 选区 checksum 验证。
- [ ] 低置信度识别不会当成事实。
- [ ] 日志不含凭据和完整敏感输入。
- [ ] Artifact 路径无法逃逸工作目录。

## 9. 提交与交付

- [ ] diff 只包含本任务相关文件。
- [ ] `git diff --check` 通过。
- [ ] 自动测试结果已记录。
- [ ] E2E 步骤可由他人重复。
- [ ] 上游 commit/PR 已写明。
- [ ] 下游依赖更新已写明。
- [ ] PR 是 draft 还是 ready 状态明确。
- [ ] 临时 worktree 不是唯一提交所在地。
- [ ] TODO 的分支—提交—PR—依赖—状态已同步。
- [ ] 没有声称未测试的能力已经完成。
