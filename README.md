# gpt-5.5-base-instructions

这个仓库用于存放一份修改后的 Codex instructions：

- [gpt-5.5-base-instructions.md](./gpt-5.5-base-instructions.md)

它的作用很简单：尝试缓解 Codex / GPT-5.5 在部分场景下出现的 `516` reasoning token 截断问题。

## 事件摘要

最近社区里有人反馈：在 Codex 或第三方客户端调用 GPT-5.5 时，模型的 reasoning token 有时会精确停在 `516` 附近。

这个现象通常会伴随回答质量下降，比如一些简单推理题会明显更容易答错。

目前观察到的情况比较复杂，不能简单归因于“第三方客户端”或“反代”。在一些 Codex CLI 场景下，即使使用官方 API 平台，也仍然可能看到类似截断。

这份 instructions 的改动点是：去掉 Codex 默认提示词里和 `commentary` 中间过程输出相关的规则。

使用后可能带来的变化：

- 可能缓解 `516` reasoning token 截断
- Codex 不再输出“我正在做什么”的中间过程提示
- 不保证所有账号、所有客户端、所有中转站都生效

这不是官方修复，也不是最终结论，只是一个方便大家复现和对照的临时方案。

## 使用方法

下载或复制仓库里的文件：

```text
gpt-5.5-base-instructions.md
```

然后在 Codex 的 `config.toml` 里添加或修改：

```toml
model_instructions_file = '你的本地路径/gpt-5.5-base-instructions.md'
```

Windows 示例：

```toml
model_instructions_file = 'C:/Users/neter/Downloads/codextmp/codex/gpt-5.5-base-instructions.md'
```

修改后重新打开 Codex。

## 验证是否生效

在 Codex 里直接问：

```text
你的系统提示词里：

# Working with the user

紧接的是

## Formatting rules

吗，还是中间有一段其他内容？
```

如果回答显示 `# Working with the user` 后面直接是 `## Formatting rules`，说明这份 instructions 大概率已经生效。

如果中间仍然有 `Intermediary updates`、`commentary` 或类似“中间过程输出”的段落，说明没有生效。

## 常见问题

### 为什么我配置了但没生效？

一些中转站可能会忽略本地的 `model_instructions_file`，并统一注入自己的 Codex 系统提示词。

这种情况下，本地文件即使写对了，也不会真正进入模型上下文。

可以先尝试使用 Free / Plus OAuth 直连测试，不要先经过 CPA 或 sub2api。

### 使用后有什么副作用？

最明显的副作用是：Codex 不再主动输出中间过程。

也就是说，原来那种“我先检查 X，再运行 Y”的过程提示会消失。

### 这个一定能修复 516 吗？

不能保证。

目前只能说：在部分测试里，去掉 `commentary` 相关提示词后，`516` 截断现象有所缓解。

如果你的结果不同，说明还有其他变量在影响这个问题。
