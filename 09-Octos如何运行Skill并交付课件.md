# 09　Octos 如何运行 Skill 并交付课件

## 1. Octos 在这套系统中的位置

Octos 不是 OLL Runtime。它是承载学习产品的通用 Agent 和服务端基础设施。

一次简化 Agent 循环：

```text
收集消息
→ 调用主模型
→ 模型返回文字或 tool calls
→ 执行工具
→ 把工具结果加入消息
→ 继续调用模型
→ 得到最终回复
```

learning-coach 是其中一个可调用 Tool Skill。

## 2. ToolSpec

Octos 会把 Skill manifest 中的工具转换成通用 `ToolSpec`，包含：

- 工具名；
- 工具说明；
- 参数 JSON Schema。

不同模型 Provider 对工具 Schema 支持不同。Octos LLM 层必须做 Provider 专用适配。例如 Gemini 不支持 `exclusiveMinimum` 等字段时，适配器在发送前创建清洗副本；不能要求每个 Skill 都污染自己的标准 Schema。

## 3. Skill 安装位置

开发者可从本地目录安装：

```bash
octos skills --profile <profile-id> install /absolute/path/to/learning-coach --force
```

Profile 安装进入该 Profile 的数据目录，通常位于：

```text
~/.octos/profiles/<profile>/data/skills/
```

没有 `--profile` 的本地工作区模式会使用当前目录下的 `.octos/skills/`。产品开发应明确自己启动的 Octos 使用哪个 Profile，避免“改了源仓库但后端仍运行旧安装副本”。

查看和管理：

```bash
octos skills --profile <profile-id> list
octos skills --profile <profile-id> info learning-coach
octos skills --profile <profile-id> remove learning-coach
```

## 4. 安装时发生什么

本地安装会：

1. 检查 `SKILL.md` 和 `manifest.json`；
2. 用严格规则验证 manifest；
3. 复制到 Profile Skill 目录；
4. 按 manifest 或仓库类型准备 `main` 可执行文件；
5. 保存来源信息；
6. 在后续运行时发现工具。

如果只在源目录执行 `npm run build`，已安装目录不会自动变化。需要重新安装或使用明确的本地开发链接方式。

## 5. 进程协议

Octos 执行：

```text
./main <tool_name>
```

并把参数 JSON 写入 stdin。

Skill stdout：

```json
{
  "output": "给 Agent 看的简短结果",
  "success": true,
  "files_to_send": ["/absolute/path/to/artifact"]
}
```

stderr 用于日志和进度。进程退出码、stdout 解析和超时共同决定 ToolResult。

## 6. 环境变量和秘密

Skill 只能接收 manifest `tools[].env` 中允许的变量。learning-coach 需要 Vertex 凭据、Project、Location、模型名、超时和 OLL 工作目录。

Profile 可以把秘密保存为 Keychain 引用，Octos 运行时解析后只注入允许的 Skill 进程。

Skill 必须遵守：

- 不在 stdout 打印秘密；
- 不在 stderr 打印秘密；
- 不把秘密写入 Artifact；
- 错误信息只保留安全摘要。

## 7. Provider 工具 Schema 适配

主 Agent 自身调用 Skill 前，Octos 会把所有 ToolSpec 发送给 Gemini。若某个参数 Schema 包含 Provider 不认识的字段，整个模型请求可能在 Agent 还没选择工具前返回 HTTP 400。

这与 learning-coach 内部“让 Vertex 生成 OLL JSON”是两层不同的受控 Schema：

| 层 | 用途 | 负责人 |
| --- | --- | --- |
| Agent tool parameters | 让主模型知道如何调用 Skill | Octos Provider adapter |
| OLL authoring response schema | 让 Skill 内模型生成结构化课件 | learning-coach Provider client |

两层都需要 Provider 兼容，但不能混在一起修。

## 8. 超时、重试和取消

Skill manifest 给出总超时，learning-coach 内部又给每次模型请求设置超时和总预算。

重试应区分：

- 429/5xx 等临时 Provider 错误；
- 400 Schema 拒绝，通常不应原样重试；
- JSON 或 OLL 校验失败，可带 violation 做有限修复；
- 用户取消，应终止待执行的生成任务。

无条件重试会把 40 秒错误变成 120 秒错误。

## 9. 文件交付

Skill 把绝对路径列在 `files_to_send`。Octos 校验并把文件登记到会话工作区，然后向前端发送文件相关事件。

Artifact 内容不应被塞进主模型文字回复：

- 文件可单独下载和恢复；
- 大 JSON 不占聊天上下文；
- 前端可按文件后缀路由；
- 部分和最终 Artifact 可被正确替换。

## 10. UI 协议中的工具生命周期

前端关心：

- `tool_start`；
- `tool_progress`；
- Artifact/file 事件；
- `tool_end`；
- turn completed。

每条工具事件必须带稳定 `tool_call_id`，否则前端无法把等待状态归到正确的一轮。

learning-coach 生成较慢时，Octos 应把 stderr 的结构化阶段进度桥接到 UI，而不是让界面长时间无反馈。

## 11. 启动开发后端

Octos 是 Rust workspace。常用检查：

```bash
cargo fmt --all -- --check
cargo test -p octos-agent
cargo test -p octos-llm
cargo test -p octos-cli --features api --lib
```

要使用 `octos serve`，构建时必须启用 `api` feature。完整本地安装可按仓库 README/AGENTS 中的 feature 集执行：

```bash
cargo install --path crates/octos-cli --features "api,telegram,discord,whatsapp,feishu,twilio,wecom,wecom-bot,audio_mp3"
```

只测试学习链路时可以构建更小的 feature 集，但 `api` 不能省略。

## 12. 后端日志怎样读

按顺序寻找：

1. 用户 turn 是否进入；
2. 主模型是否成功返回 tool call；
3. 工具名和输入是否正确；
4. Skill 进程是否启动；
5. Skill stderr 到达哪个 stage；
6. stdout 是否解析成成功 ToolResult；
7. `files_to_send` 是否被接受；
8. 文件事件是否发送给会话；
9. turn 是否正常结束。

如果主模型请求在 tool call 前就 400，应查 Octos Tool Schema adapter；如果 Skill 已启动后其 Vertex 请求 400，应查 learning-coach。

## 13. Octos 不应承担的 OLL 逻辑

不要在 Octos 后端：

- 解析 `write geometry`；
- 补 OLL 节点；
- 判断 `sin(theta)=1`；
- 决定 Plot 如何采样；
- 修改学生 Ink SVG。

这些属于 OLL Core/Runtime、learning-coach 或 octos-web。Octos 保持通用，学习能力才能独立演进。
