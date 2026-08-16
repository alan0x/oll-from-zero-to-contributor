# 附录 B　仓库与文件地图

本附录用于“我该从哪个文件开始看”。源码会演进；目录职责比具体行号更可靠。

## 1. octos-lesson-language

```text
README.md
AUTHORING.md
SPEC.md
PLAYBACK.md
INTERACTIVE-WHITEBOARD-MVP.md
schema/
  authoring/
  canonical/
packages/
  core/
  player-core/
  web-runtime/
  ink-runtime/
  eval-runner/
  quality-runner/
apps/
  playback-harness/
examples/
fixtures/
evals/
```

类型：

```text
packages/core/src/types.ts
```

校验、Normalizer、Reducer：

```text
packages/core/src/index.ts
packages/core/src/math-expression.ts
```

关键导出：

```text
validateAuthoringSchema
validateAuthoringLesson
normalizeAuthoringLesson
reduceCanonicalEvents
setLessonVariable
evaluateContentBindings
```

Player：

```text
packages/player-core/src/index.ts
packages/player-core/src/types.ts
```

浏览器 Runtime：

```text
packages/web-runtime/src/
packages/web-runtime/styles.css
```

重点模块包括 `layout.ts`、`plot.ts`、学生操作/任务、Scene3D、BrowserLessonSession 和 `teaching-observer.ts`。

学生笔迹：

```text
packages/ink-runtime/src/runtime.ts
packages/ink-runtime/src/persistence.ts
packages/ink-runtime/src/selection.ts
packages/ink-runtime/src/selection-record.ts
packages/ink-runtime/src/world-layer.ts
```

最值得先看的完整例子：

```text
examples/unit-circle-sine/lesson.authoring.json
examples/unit-circle-sine/lesson.canonical.jsonl
examples/unit-circle-sine/expected-state.json
examples/unit-circle-sine/ACCEPTANCE.md
```

## 2. learning-coach

```text
manifest.json
package.json
oll-contract.json
src/main.ts
references/oll-authoring-v0.1.schema.json
scripts/oll-contract.mjs
scripts/check-vertex-schema-contract.mjs
scripts/eval-visual-generation.mjs
test/
eval/visual-generation-cases.json
```

`src/main.ts` 当前包含多个职责，阅读时按函数名搜索：

```text
planLesson
verifyLessonRequirements
deriveAuthoringCapabilityPlan
buildAuthoringResponseJsonSchema
generateLessonInParallel
generateParallelSection
generateVisualComponent
assembleParallelLesson
validateGeneratedLesson
generateSelectionEnhancement
main
```

入口在文件底部 `main()`。建议从入口向上追，不要从第一行顺读数千行。

## 3. Octos

先读：

```text
AGENTS.md
README.md
docs/user-guide.md
docs/app-skill-dev-guide.md
docs/OCTOS_HARNESS_SKILL_COMPAT.md
```

Skill 安装：

```text
crates/octos-cli/src/commands/skills.rs
```

Skill 发现和执行：

```text
crates/octos-agent/src/plugins/loader.rs
crates/octos-agent/src/tools/
```

ToolSpec 和模型 Provider：

```text
crates/octos-llm/src/types.rs
crates/octos-llm/src/providers/
```

查 Gemini 兼容问题时搜索 `Gemini`、`tool schema`、`exclusiveMinimum`、`sanitize` 和 `function_declarations`。

HTTP/UI 协议：

```text
crates/octos-cli/src/api/
```

重点搜索 `tool_progress`、`files_to_send`、`session files`、`UiNotification` 和 `tool_call_id`。

## 4. octos-web

学习产品入口：

```text
src/learning/learning-page.tsx
src/learning/learning-workspace.tsx
src/learning/learning-context.ts
```

OLL Artifact 和课堂组合：

```text
src/learning/oll/oll-artifacts.ts
src/learning/oll/oll-playback-storage.ts
src/learning/oll/oll-course-outline.tsx
```

Player controller 和渲染：

```text
src/learning/oll/use-oll-lesson-runtime.ts
src/learning/oll/oll-lesson-runtime.tsx
src/learning/oll/lesson-delivery.ts
```

TTS：

```text
src/learning/oll/use-oll-narration-tts.ts
```

白板：

```text
src/learning/board/
src/learning/board/infinite-board.tsx
```

选区和局部增强：

```text
src/learning/selection-tools.ts
src/learning/selection-enhancements.ts
src/learning/selection-enhancement-layer.tsx
src/learning/composer-board-references.ts
```

UI 协议与文件 API：

```text
src/runtime/
src/store/projection-render-adapter.ts
src/api/sessions.ts
src/api/files.ts
```

## 5. 从现象反查文件

| 现象 | 第一检查点 |
| --- | --- |
| 模型漏掉函数图 | learning-coach 要求清单、coverage、visual component |
| Authoring 引用非法 | OLL Core semantic validator |
| 同一变量两张图不同步 | OLL bindings / Web Runtime variable state |
| 动画早于 TTS | octos-web narration lifecycle + Player external timing |
| 课程只显示第一部分 | octos-web Artifact rank/append/delivery settled |
| Gemini 在 tool call 前 400 | Octos Gemini Tool Schema adapter |
| Skill 内 Vertex 400 | learning-coach response schema |
| 笔迹随页面不随白板移动 | Ink world layer / InfiniteBoard transform |
| 框选后原稿移动 | Ink selection lock |
| 刷新后变量丢失 | Web Runtime student operation persistence |
| 后续问题引用错节点 | composer reference + board_context revision |
