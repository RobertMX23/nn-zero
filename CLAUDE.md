# CLAUDE.md

本文件为 Claude Code（claude.ai/code）在本仓库中工作时提供指导。

**重要：与用户对话时，请始终使用中文。**

## 本仓库是什么

这是 Andrej Karpathy 的 **Neural Networks: Zero to Hero** 课程的克隆——即 YouTube 系列讲座中现场编写的 Jupyter notebook（各讲的视频链接见 `README.md`）。这里没有库、没有包、也没有测试套件：交付物就是 notebook 本身。

uv 脚手架（`pyproject.toml`、`uv.lock`、`.python-version`）是**在上游内容之上的本地补充**，已暂存但尚未提交。`lectures/` 目录下的所有内容都是上游内容。

## 环境配置

使用 `uv` 管理（Python 3.14）。**PyTorch 被刻意排除在基础依赖之外**——它来自一个指向自定义 wheel 索引的、与硬件相关的可选 extra，并且在 `[tool.uv]` 中声明了两个 extra 相互冲突：

```bash
uv sync --extra cu132     # NVIDIA，CUDA 13.2 wheel
uv sync --extra rocm72    # AMD，ROCm 7.2 wheel（同时会拉取 pytorch-triton-rocm）
```

直接执行 `uv sync` 会生成一个**不含 torch** 的虚拟环境，这正是 `.venv` 当前的状态——除 micrograd 之外的所有 notebook 都会在 `import torch` 处失败。在排查其他问题之前，先用 `uv run python -c "import torch"` 检查。

## 运行 notebook

```bash
uv run jupyter lab              # 或者：uv run jupyter notebook
uv run voila lectures/...ipynb  # 将 notebook 渲染为独立应用
```

两个看起来像代码 bug 的环境陷阱：

- **`names.txt` 不在本仓库中。** 全部五个 `makemore` notebook 都会相对于内核的 CWD 执行 `open('names.txt')`。请从 [karpathy/makemore](https://github.com/karpathy/makemore) 获取该文件，并放到内核运行的目录下（即如果在该目录启动 Jupyter，就放到 `lectures/makemore/`）。
- **PyPI 上的 `graphviz` 包只是一个封装。** micrograd notebook 中的 `draw_dot` 辅助函数会调用 `dot` 二进制程序，它是一个系统软件包（`apt install graphviz`），并且**当前未安装**在本机器上。

## 架构

每个 notebook 都是**自包含的，并从头重新构建其前置内容**。代码在 notebook 之间刻意重复——目的是让读者看到每个抽象被重新推导的过程。不要把共享的类重构成模块；那会破坏教学结构，并与上游产生分歧。

课程的递进顺序，以及每个概念首次出现的位置：

- `micrograd/..._first_half` —— `Value`：标量自动微分引擎，手写的 `_backward` 闭包，基于拓扑排序的 `backward()`。
- `micrograd/..._second_half` —— 在 `Value` 之上构建 `Neuron` → `Layer` → `MLP`；第一个训练循环。
- `makemore_part1_bigrams` —— 将 bigram 计数表示为矩阵，然后把同一个模型改写为 one-hot + 矩阵乘法的神经网络，用 NLL 训练。引入了 `torch.Tensor` 广播的微妙之处。
- `makemore_part2_mlp` —— Bengio 风格的 MLP（嵌入表 + 隐藏层），train/dev/test 划分，学习率搜索。
- `makemore_part3_bn` —— 手工重新实现 PyTorch 的 API 表面，即 `Linear` / `BatchNorm1d` / `Tanh`，并采用 `parameters()` 约定；激活值/梯度统计诊断。
- `makemore_part4_backprop` —— 对第 3 讲的网络**手动**进行张量级的反向传播，`loss.backward()` 仅用作 `cmp()` 检查的参照。
- `makemore_part5_cnn1` —— 在第 3 讲的层集合中加入 `Embedding`、`FlattenConsecutive`、`Sequential`，构建 WaveNet 风格的层级模型。

第 7 讲（GPT）和第 8 讲（Tokenizer）在 `README.md` 中有描述，但**这里没有对应的 notebook**——那部分代码在 README 链接的外部 `nanoGPT`/`minbpe` 仓库中。

## 约定

- notebook 提交时**保留输出**，包括 base64 内嵌的 matplotlib 图（part1 和 part3 约 400–500 KB）。diff 天然会很嘈杂；不要为了变干净而剥离输出，grep notebook JSON 时也要预期会遇到大块的二进制内容。
- 没有 linter、格式化工具或测试配置。请遵循现有 notebook 的风格：简洁、单字母张量命名（`C`、`W1`、`emb`、`logits`）、用行内注释标注张量形状。
- **commit message 遵循 Conventional Commits 约定**：`<type>(<scope>): <description>`，常用 type 为 `feat`、`fix`、`docs`、`chore`、`refactor`；描述使用祈使语气的小写短句。例如 `chore: add uv scaffolding`、`docs(claude): document commit convention`。注意上游历史（如 `add lecture 8 tokenization`）并不遵循该约定——它只适用于本仓库的新提交。
