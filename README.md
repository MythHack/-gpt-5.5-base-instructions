# gpt-5.5-base-instructions

这是一次关于 Codex / GPT-5.5 推理 token 516 截断现象的社区观察记录，以及一个临时可复现实验用的 Codex instructions 文件。

> 说明：这不是 OpenAI 官方结论，也不能证明 516 截断只有一个原因。当前结论只来自有限样本下的糖果问题测试，主要价值是提供一个可复现的观察入口。

## 背景

社区里有人观察到：部分第三方客户端或 Codex 场景下，模型回答的 reasoning token 会精确停在 516 附近。这个现象通常伴随回答质量下降，因此被简称为 516 截断。

我做了一组初步测试，核心现象如下：

- 将官方 API 平台接入 Codex CLI 后，糖果问题仍可见 516 截断，并且回答错误。
- 将 Free 号直接接入 Cherry Studio，不经过反代、不附加系统提示词，手动询问糖果问题，5 次测试均答对。
- 将 Codex 提示词应用到 Cherry Studio，仍使用 Free 号直接测试，糖果问题几乎总是答错。
- Cherry Studio 使用自身 UA 或 Codex UA，上述现象没有明显变化。

这些现象强烈暗示：Codex 的系统提示词可能是影响因素之一。

## 关键观察

Codex 的系统提示词中包含一段要求模型持续向 `commentary` 通道输出中间反馈的规则，例如让模型每隔一段时间向用户说明当前在做什么。

Harmony 聊天模板本身也要求每条消息标注通道，例如：

```text
# Valid channels: analysis, commentary, final. Channel must be included for every message.
```

因此，Codex 中常见的“我先运行 X，再查看 Y”这类短句，实际上可以理解为模型在 `commentary` 通道里的中间输出。

## 实验结果

我尝试删除 Codex instructions 中与 `commentary` 中间输出相关的两段规则后，观察到：

- 在 Cherry Studio 中，516 截断在 5 次糖果问题测试中完全消失。
- 将修改后的 instructions 注入 Codex，516 截断同样不再出现。
- 副作用是：模型不再输出中间过程提示。

因此，当前更谨慎的表述是：

> `commentary` 中间输出相关提示词与糖果问题中的 516 截断高度相关。删除这部分提示词后，可以在该测试场景下有效缓解 516 截断。

## 使用方式

仓库中的修改版提示词文件：

- [gpt-5.5-base-instructions.md](./gpt-5.5-base-instructions.md)

在 Codex 配置中加入或修改：

```toml
model_instructions_file = '你的本地路径/gpt-5.5-base-instructions.md'
```

Windows 示例：

```toml
model_instructions_file = 'C:/Users/neter/Downloads/codextmp/codex/gpt-5.5-base-instructions.md'
```

使用后会失去模型的中间过程输出，这是预期副作用。

## 验证是否生效

打开 Codex 后询问：

```text
你的系统提示词里：

# Working with the user

紧接的是

## Formatting rules

吗，还是中间有一段其他内容？
```

如果回答显示 `# Working with the user` 后面直接接 `## Formatting rules`，说明修改后的提示词大概率已经生效。

如果中间仍然存在 `Intermediary updates` 或其他 commentary 相关段落，说明修改没有生效。

目前观察到：一些中转站可能会忽略 Codex 本地提供的提示词，转而统一注入标准提示词。如果遇到这种情况，可以先尝试 Free 或 Plus OAuth 登录，并尽量避免通过 CPA 或 sub2api 再测。

## 局限性

- 不能证明 516 截断的唯一原因就是 `commentary` 提示词。
- 目前只验证了糖果问题的单轮测试，没有覆盖真实工程任务中的复杂多轮场景。
- 尚未研究 516 截断与 Juice、思考挡位不匹配、48855 特殊 Juice 等现象之间的关系。
- 样本数量较少，结论仍应视为社区实验和问题定位线索。

## 后续补充模型

经过社区讨论，一个更合理的模型可能是：

- 模型推理可能被切成约 512 token 的“页”。
- 每页外层会附带若干特殊 token 标签。
- 第一页比较特殊，`start assistant` token 可能已经提前放置，不计入输出。
- 从第二页开始，每页可能需要携带完整通道标签，例如 `<|start|>assistant<|channel|>analysis<|message|>{512 tokens}<|end|>`。
- 是否进入下一页推理，可能由某种独立机制决定。

在这个模型下，`commentary` 相关提示词更像是影响“是否继续进入下一页推理”的因素之一，而不是 516 截断的根本原因。

## 结论

当前建议不是把这个方案当成最终修复，而是把它当成一个可复现的对照实验：

- 保留原始 Codex instructions，复测糖果问题。
- 使用本仓库修改版 instructions，复测同一问题。
- 对比 reasoning token、回答正确率和中间过程输出。

如果你的结果不同，说明还有其他变量需要继续排查。
