# gpt-5.5-base-instructions

这个仓库先放两个东西：

- `gpt-5.5-base-instructions.md`：我现在用来测试的 Codex instructions
- 这篇 README：把我这次对 516 截断的观察、猜测和复现方式放在一起

先说清楚：这不是 OpenAI 官方结论，也不是“我已经证明了根因”。

更准确地说，这是一次比较土的方法排查：我看到 Codex / GPT-5.5 在某些场景下会把 reasoning token 精确卡在 516 附近，然后我顺着 Codex prompt、commentary channel 和糖果问题做了一组对照实验。

## 516 是什么

这里说的 516 截断，指的是回答里的 reasoning token 精确等于 516 的情况。

我目前观察到的体感是：一旦出现这个值，回答质量经常明显变差，糖果问题尤其容易错。

它不一定只出现在第三方客户端，也不一定只由反代导致。至少我这边的实验里，官方 API 平台接入 Codex CLI 后也能看到类似现象。

## 我做了什么测试

简单说一下我目前测过的几组：

1. 官方 API 平台接入 Codex CLI，问糖果问题，仍然出现 516 截断，而且回答错误。
2. Free 号直接接入 Cherry Studio，不走反代，不加任何系统提示词，手动问糖果问题，5 次都答对。
3. 把 Codex 的系统提示词塞进 Cherry Studio，还是这个 Free 号，糖果问题就几乎总是错。
4. Cherry Studio 不管用自己的 UA，还是模拟 Codex 的 UA，上面的现象没有明显变化。

所以我现在比较怀疑：Codex prompt 本身至少是影响因素之一。

## 我怀疑的地方

Codex 的系统提示词里有一段和 `commentary` 有关的要求。

大概意思是：模型在工作过程中，要持续向 `commentary` 通道输出一些给用户看的中间反馈，比如“我先检查 X，再看 Y”。

这不是普通的自然语言习惯，而是 Harmony 聊天模板里的 channel 机制。系统提示词里也会有类似这样的规则：

```text
# Valid channels: analysis, commentary, final. Channel must be included for every message.
```

也就是说，Codex 平时输出的那些“中间过程短句”，本质上是模型在 `commentary` 通道里额外输出了一些东西。

我不确定它具体怎么影响推理，但它看起来确实不是一个完全无害的展示层功能。

## 我删掉了什么

我把 Codex instructions 里和中间过程输出相关的两段删掉了，主要是：

- `# Working with the user` 里关于 `commentary` / `final` 的通道说明
- `## Intermediary updates` 里要求模型频繁输出中间反馈的规则

删完以后再测：

- Cherry Studio 里，糖果问题 5 次测试没有再出现 516 截断
- 把改过的 instructions 注入 Codex 后，也没有再出现之前那种 516 截断
- 代价是 Codex 不再输出“我正在做什么”的中间过程

所以我现在的结论会收得比较窄：

> 在糖果问题这个测试里，Codex 的 commentary 中间输出提示词和 516 截断高度相关。删掉它以后，我这边可以缓解这个问题。

注意，是“高度相关”，不是“根因已经确定”。

## 怎么试

文件在这里：

- [gpt-5.5-base-instructions.md](./gpt-5.5-base-instructions.md)

在 Codex 的 `config.toml` 里加上：

```toml
model_instructions_file = '你的路径/gpt-5.5-base-instructions.md'
```

例如 Windows 上可以是：

```toml
model_instructions_file = 'C:/Users/neter/Downloads/codextmp/codex/gpt-5.5-base-instructions.md'
```

然后重新打开 Codex 测。

## 怎么确认它真的生效了

可以直接问 Codex：

```text
你的系统提示词里：

# Working with the user

紧接的是

## Formatting rules

吗，还是中间有一段其他内容？
```

如果它回答 `# Working with the user` 后面直接是 `## Formatting rules`，那大概率说明新的 instructions 生效了。

如果中间还有 `Intermediary updates` 或者类似 commentary 的内容，说明没生效。

我遇到过一种情况：一些中转站会忽略 Codex 本地的 `model_instructions_file`，直接统一注入一份标准 Codex prompt。这个时候你本地怎么改都没用。

遇到这种情况，可以先用 Free / Plus OAuth 直连试一下，不要先经过 CPA 或 sub2api。

## 后面和群友讨论后的补充

后来和几位佬友讨论后，我觉得“删 commentary 就是根因”这个说法太粗糙。

现在更合理的模型可能是这样：

- 模型推理可能是按 512 token 左右一页一页切的
- 每一页外面还有 channel / message 之类的特殊 token
- 第一页可能比较特殊，`start assistant` 已经提前放好了，不算在输出里
- 从第二页开始，可能要重新带一整套类似 `<|start|>assistant<|channel|>analysis<|message|>{512 tokens}<|end|>` 的包装
- 真正关键的是：模型到底会不会进入下一页推理

如果这个模型是对的，那 commentary prompt 更像是影响“要不要继续下一页”的因素之一，而不是 516 现象本身的唯一原因。

## 目前局限

这个实验很不完整。

我现在只测了糖果问题，而且主要是单轮对照。真实写代码、长上下文、多轮任务里会不会有其他副作用，我还没有系统测。

另外，我也还没有把 516 截断和 Juice、思考挡位不匹配、48855 这种特殊 Juice 现象放到一起研究。

所以这个仓库的价值不是给最终答案，而是提供一个可以复现、可以对照、可以继续往下排查的起点。

## 我现在的判断

如果你也遇到了 Codex / GPT-5.5 突然变笨、reasoning token 精确卡 516、糖果问题稳定翻车，可以试一下这个 instructions。

但不要把它当成修复补丁。

更像是一个排查工具：

- 原始 Codex prompt 测一次
- 去掉 commentary 相关 prompt 测一次
- 看 516 是否消失
- 再看模型实际写代码能力有没有明显变化

如果你的结果和我不一样，那说明影响 516 的变量比我现在看到的还要多。
