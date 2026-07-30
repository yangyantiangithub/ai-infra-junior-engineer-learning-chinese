# Module 004: 面向基础设施的 ML 基础 - 综合测验

## 概述

本测验涵盖 Module 004 的全部四节讲座：
- 讲座 01：机器学习概述
- 讲座 02：PyTorch 基础
- 讲座 03：TensorFlow 基础
- 讲座 04：模型格式和 ONNX

**总题数**：28
**预计时间**：45-60 分钟
**及格分数**：70%（28 题中答对 20 题）

---

## 说明

1. 仔细阅读每个问题
2. 为选择题选择最佳答案
3. 对于代码分析题，思考代码的实际行为
4. 部分问题带有场景上下文——请仔细阅读
5. 每个问题后附有解释，供学习参考

---

## A 部分：ML 概述（6 题）

### 问题 1（简单）
**传统编程和机器学习之间的根本区别是什么？**

A) 传统编程比机器学习更快
B) 机器学习从数据中学习模式，而传统编程使用显式规则
C) 传统编程比机器学习需要更多内存
D) 机器学习不需要任何代码，只需要数据

**正确答案**：B

**解释**：核心区别在于传统编程使用显式的、手写的规则（if/else 语句），而机器学习从示例（数据）中学习模式。ML 并没有取代编程，而是改变了我们解决问题的方式——我们不再编写规则，而是提供示例让模型发现模式。

---

### 问题 2（中等）
**你的 ML 团队正在将一个新模型部署到生产环境。训练阶段在 8 个 GPU 上耗时 3 天，但该模型将在生产中每秒服务 10,000 次预测。哪个基础设施考虑因素最为关键？**

A) 定期重新训练的训练 GPU 容量
B) 推理延迟和吞吐量优化
C) 训练数据集的数据存储
D) 训练检查点的模型版本管理

**正确答案**：B

**解释**：虽然训练是偶尔进行（数天/数周），但推理在生产环境中是持续进行的（每秒数千次）。关键的基础设施需求是优化低延迟和高吞吐量，以处理 10,000 QPS。训练基础设施很重要，但不是生产服务的即时瓶颈。

---

### 问题 3（简单）
**判断对错：在机器学习中，"推理"指的是在数据上训练模型的过程。**

A) 正确
B) 错误

**正确答案**：B（错误）

**解释**：推理是指用已训练好的模型对新的、未见过的数据进行预测。训练是模型从数据中学习模式的过程。这是基础设施工程师需要理解的关键区别——训练和推理有着完全不同的资源需求和优化策略。

---

### 问题 4（中等）
**你需要部署一个处理图像进行目标检测的模型。根据讲座内容，预期的基础设施需求是什么？**

A) 训练：低计算量，推理：非常快，模型大小：小（MB）
B) 训练：高计算量（需要 GPU），推理：中等速度（实时需要 GPU），模型大小：大（数百 MB 到 GB）
C) 训练：中等计算量，推理：非常快（微秒级），模型大小：小（MB）
D) 训练：非常高计算量，推理：慢，模型大小：TB 级

**正确答案**：B

**解释**：目标检测是一项复杂的计算机视觉任务，需要：(1) 使用 GPU 的高计算量训练，(2) 实时推理需要 GPU，因为目标检测模型计算密集，(3) 大模型大小（数百 MB 到 GB），因为要在图像中检测和定位多个对象。

---

### 问题 5（困难）
**一位数据科学家报告说，他们的模型预测置信度在生产环境中四个月内从平均 0.92 下降到了 0.75。最可能的问题和适当的基础设施响应是什么？**

A) 硬件故障——更换 GPU
B) 数据漂移或概念漂移——触发模型重新训练
C) 内存泄漏——重启服务容器
D) 网络延迟——增加更多负载均衡器

**正确答案**：B

**解释**：预测置信度随时间逐渐下降是数据漂移（输入分布变化）或概念漂移（输入和输出之间的关系变化）的典型症状。适当的基础设施响应是监控这种模式，提醒团队，并使用更新的数据触发重新训练管道。硬件或网络问题会导致突然故障，而非逐渐退化。

---

### 问题 6（中等）
**哪种类型的机器学习需要带有正确答案的标记训练数据？**

A) 无监督学习
B) 强化学习
C) 监督学习
D) 自监督学习

**正确答案**：C

**解释**：监督学习需要标记的示例（输入 + 正确输出）进行训练。例如，标记为"猫"或"狗"的图像。这是生产系统中最常见的 ML 类型。无监督学习在没有标签的情况下发现模式，强化学习从奖励中学习。创建标签可能成本高昂，这是一个重要的基础设施成本考虑因素。

---

## B 部分：PyTorch 基础（8 题）

### 问题 7（简单）
**在推理前对 PyTorch 模型调用 `.eval()` 方法的作用是什么？**

A) 评估模型在测试数据集上的准确率
B) 将模型设置为评估模式，禁用 dropout 和 batch normalization 的训练行为
C) 从内存中删除模型
D) 将模型导出为 ONNX 格式

**正确答案**：B

**解释**：`.eval()` 方法将模型设置为评估模式，这会改变 dropout（推理时禁用）和 batch normalization（使用 running statistics 而非 batch statistics）等层的行为。忘记调用 `.eval()` 是一个常见错误，会导致不一致的预测结果。这对生产推理至关重要。

---

### 问题 8（中等）
**你收到此错误："RuntimeError: Expected all tensors to be on the same device, but found at least two devices, cuda:0 and cpu!" 问题出在哪里？**

A) PyTorch 没有正确安装
B) 模型和输入 tensor 位于不同的设备上（一个在 GPU 上，一个在 CPU 上）
C) GPU 内存不足
D) 模型架构与输入不兼容

**正确答案**：B

**解释**：当模型在一个设备上（如 GPU）而输入 tensor 在另一个设备上（如 CPU）时，会出现此错误。PyTorch 要求计算中的所有 tensor 都在同一个设备上。修复方法是使用 `.to(device)` 确保模型和输入都移动到同一设备。

---

### 问题 9（简单）
**在生产环境中运行 PyTorch 推理时，为了节省内存和计算，应该始终使用哪个上下文管理器？**

A) `with torch.cuda.empty_cache():`
B) `with torch.no_grad():`
C) `with torch.autograd.profiler():`
D) `with torch.jit.optimize():`

**正确答案**：B

**解释**：`torch.no_grad()` 禁用梯度计算，这仅在训练时需要。在推理期间使用它可以节省大量内存（无需存储梯度）和计算时间。在生产推理代码中应始终使用它。这是 PyTorch 推理最重要的优化之一。

---

### 问题 10（困难）
**分析以下代码。主要的性能问题是什么？**

```python
model = torch.hub.load('pytorch/vision', 'resnet50', pretrained=True)
model = model.to('cuda')

for image in image_list:  # 1000 images
    input_tensor = preprocess(image)  # Returns shape [3, 224, 224]
    input_tensor = input_tensor.to('cuda')
    with torch.no_grad():
        output = model(input_tensor)
    results.append(output)
```

A) 模型未设置为 eval 模式
B) 逐张处理图像而非批处理
C) 使用 CUDA 而非 CPU
D) 输入 tensor 没有 batch 维度

**正确答案**：B（虽然 D 在技术上也存在问题）

**解释**：主要的性能问题是逐张处理图像。批处理可以显著提高 GPU 利用率和吞吐量。虽然缺少 batch 维度（D）会导致立即报错，但性能问题在于缺少批处理。在生产环境中，你应该批量处理请求以最大限度地提高 GPU 效率——同时处理 32 张图像可能比逐张处理快 10-20 倍。

---

### 问题 11（中等）
**使用 `torch.save(model.state_dict(), path)` 与 `torch.save(model, path)` 保存 PyTorch 模型有什么区别？**

A) state_dict 只保存权重，完整模型保存权重 + 架构
B) state_dict 更大，完整模型更小
C) state_dict 需要 CUDA，完整模型可在 CPU 上运行
D) 没有区别

**正确答案**：A

**解释**：`state_dict()` 只保存模型权重（参数），你需要单独有模型架构代码。`torch.save(model, path)` 保存整个模型包括架构。生产环境的最佳实践是使用 state_dict，因为它更具可移植性和灵活性——你可以修改架构代码而不受保存结构的限制。

---

### 问题 12（中等）
**你的 GPU 在推理期间显示"CUDA out of memory"错误。哪个不是有效的解决方案？**

A) 减少批大小
B) 调用 torch.cuda.empty_cache() 释放未使用的内存
C) 使用混合精度（float16 替代 float32）
D) 增加学习率

**正确答案**：D

**解释**：学习率是一个训练超参数，与推理期间的内存使用无关。解决 OOM 错误的有效方法包括：减少批大小（减少内存中的数据）、清除缓存（释放未使用的分配）、使用混合精度（float16 使用 float32 一半的内存），或确保使用了 torch.no_grad()。

---

### 问题 13（简单）
**什么是 TorchServe？**

A) 一个 PyTorch 训练优化库
B) PyTorch 官方的生产部署模型服务框架
C) 将 PyTorch 模型转换为 TensorFlow 的工具
D) 用于 PyTorch 模型的云存储服务

**正确答案**：B

**解释**：TorchServe 是 PyTorch 官方为生产 ML 模型部署设计的服务框架。它提供 REST/gRPC API、模型版本管理、自动扩展、指标和其他生产特性。你应该使用 TorchServe，而不是从头构建自己的服务基础设施。

---

### 问题 14（困难）
**你正在对推理速度进行基准测试。批大小为 1 时，你得到 80 samples/sec。批大小为 32 时，你得到 430 samples/sec。批大小为 32 时每个样本的近似延迟是多少？**

A) 2.3 ms
B) 12.5 ms
C) 74.4 ms
D) 430 ms

**正确答案**：A

**解释**：每个样本的延迟 = 1 / (吞吐量 / 批大小) = 1 / (430/32) = 1 / 13.44 = 0.074 秒 = 74.4 ms 总批处理延迟。但每个样本：74.4 ms / 32 = 2.3 ms。这展示了权衡：批处理提高了吞吐量，但也增加了每个请求的延迟，因为请求需要等待一个完整的批次。

---

## C 部分：TensorFlow 基础（8 题）

### 问题 15（简单）
**将 TensorFlow 模型部署到 TensorFlow Serving 推荐使用哪种模型格式？**

A) HDF5 (.h5)
B) SavedModel
C) TFLite
D) Checkpoint 文件

**正确答案**：B

**解释**：SavedModel 是用于 TensorFlow Serving 生产部署的推荐格式。它是一种基于目录的格式，包含计算图和权重，跨语言兼容，且与 TF Serving 完全兼容。H5 比较方便但不兼容 TF Serving。

---

### 问题 16（中等）
**TensorFlow 和 PyTorch 在图像的数据格式上有什么区别？**

A) 没有区别，两者使用相同格式
B) TensorFlow 使用 channels-first [N, C, H, W]，PyTorch 使用 channels-last [N, H, W, C]
C) TensorFlow 使用 channels-last [N, H, W, C]，PyTorch 使用 channels-first [N, C, H, W]
D) TensorFlow 需要 RGB，PyTorch 需要 BGR

**正确答案**：C

**解释**：TensorFlow 使用 channels-last 格式 [batch, height, width, channels]，而 PyTorch 使用 channels-first [batch, channels, height, width]。这是在框架之间转换模型或数据时常见的错误来源。TensorFlow 的 (1, 224, 224, 3) tensor 需要转换为 PyTorch 的 (1, 3, 224, 224)。

---

### 问题 17（简单）
**你在运行 TensorFlow 时遇到 OOM 错误。哪个命令可以启用内存增长，防止 TensorFlow 一次分配所有 GPU 内存？**

A) `tf.config.set_memory_limit(gpu, 4096)`
B) `tf.config.experimental.set_memory_growth(gpu, True)`
C) `tf.cuda.empty_cache()`
D) `tf.config.enable_memory_saving()`

**正确答案**：B

**解释**：`tf.config.experimental.set_memory_growth(gpu, True)` 告诉 TensorFlow 按需分配 GPU 内存，而不是在启动时占用所有可用内存。当在同一 GPU 上运行多个模型或进程时，这尤其重要。应在代码早期、创建任何 TensorFlow 操作之前设置此项。

---

### 问题 18（中等）
**对于高吞吐量场景，哪个 TensorFlow Serving API 更快？**

A) REST API
B) gRPC API
C) HTTP/2 API
D) WebSocket API

**正确答案**：B

**解释**：对于高吞吐量场景，gRPC 明显快于 REST，因为它使用二进制 protocol buffers 而不是 JSON，开销更低，并支持流式处理等功能。REST 更方便测试和人类可读的调试，但处理数百万请求的生产系统应使用 gRPC 以获得更好的性能。

---

### 问题 19（困难）
**分析以下 TensorFlow Serving 配置。它启用了什么？**

```yaml
model_config_list {
  config {
    name: 'my_model'
    base_path: '/models/my_model'
    model_version_policy {
      specific {
        versions: 1
        versions: 2
      }
    }
  }
}
```

A) 跨多个服务器的负载均衡
B) 同时提供两个特定版本（A/B 测试）
C) 自动模型重新训练
D) 模型压缩

**正确答案**：B

**解释**：此配置告诉 TensorFlow Serving 同时提供版本 1 和版本 2 的模型。这通常用于 A/B 测试——你可以将一定比例的流量路由到 v1，其余路由到 v2，以在完全推出新版本之前比较性能。`specific` 策略明确列出了要服务的版本。

---

### 问题 20（中等）
**TensorFlow Lite (TFLite) 针对什么进行了优化？**

A) 大规模分布式训练
B) 移动和边缘设备部署
C) 基于云的模型服务
D) 数据预处理管道

**正确答案**：B

**解释**：TensorFlow Lite 专门针对移动（Android, iOS）和边缘设备（Raspberry Pi, 微控制器）进行了优化。它生成更小的模型文件，推理速度更快，并包含针对资源受限设备的优化。然而，并非所有 TensorFlow 操作都受 TFLite 支持，因此转换可能需要修改模型。

---

### 问题 21（简单）
**判断对错：在 TensorFlow 中，你需要像在 PyTorch 中一样在推理前显式调用 model.eval()。**

A) 正确
B) 错误

**正确答案**：B（错误）

**解释**：TensorFlow/Keras 不需要显式的 eval() 调用。训练特定的行为（dropout, batch normalization）由 model.predict() 或 model() 调用中的 `training` 参数自动控制。使用 model.predict() 时，training 模式自动设为 False。这是与 PyTorch 显式模式切换的关键区别。

---

### 问题 22（中等）
**你想在 TensorFlow 中启用混合精度以获得 2 倍更快的推理。哪个命令可以实现？**

A) `tf.config.set_precision('float16')`
B) `tf.keras.mixed_precision.set_global_policy('mixed_float16')`
C) `model.compile(mixed_precision=True)`
D) `tf.experimental.enable_float16()`

**正确答案**：B

**解释**：`tf.keras.mixed_precision.set_global_policy('mixed_float16')` 启用混合精度训练/推理，使用 float16 进行计算（更快）和 float32 在需要时保持数值稳定性。在现代 GPU 上可提供约 2 倍的加速，且精度损失极小。这是最简单且有显著影响的优化技术之一。

---

## D 部分：模型格式和 ONNX（6 题）

### 问题 23（简单）
**ONNX 格式对基础设施团队的主要好处是什么？**

A) 它比 PyTorch 或 TensorFlow 训练模型更快
B) 它实现与框架无关的模型部署
C) 它自动优化模型准确率
D) 它提供更好的数据增强

**正确答案**：B

**解释**：ONNX（Open Neural Network Exchange）是一种与框架无关的格式，允许你在任何框架（PyTorch, TensorFlow 等）中训练并在任何地方部署。这解决了供应商锁定问题，并允许基础设施团队无论数据科学家使用哪种框架进行训练，都能标准化使用一种服务基础设施（ONNX Runtime）。

---

### 问题 24（中等）
**PyTorch 的 ONNX 导出中的哪个参数允许批大小在推理时可变？**

A) `variable_batch=True`
B) `dynamic_axes={'input': {0: 'batch_size'}}`
C) `batch_size=None`
D) `flexible_input=True`

**正确答案**：B

**解释**：`dynamic_axes` 参数指定哪些维度应该是动态的（可变的）而不是固定的。`{'input': {0: 'batch_size'}}` 表示输入 tensor 的第一个维度（索引 0）是动态的，代表批大小。这在生产中很重要，因为你可能希望高效地处理不同的批大小。

---

### 问题 25（困难）
**你的团队想要部署在 PyTorch 和 TensorFlow 中训练的模型。最好的基础设施方法是什么？**

A) 为 PyTorch（TorchServe）和 TensorFlow（TF Serving）构建独立的服务系统
B) 将所有模型转换为 ONNX 并使用 ONNX Runtime 进行统一服务
C) 强制数据科学家只使用 PyTorch
D) 手动将所有模型重写为单一框架

**正确答案**：B

**解释**：将所有模型转换为 ONNX 并使用 ONNX Runtime 提供了一个统一的服务基础设施，减少了复杂性和维护负担。虽然选项 A 可行，但需要维护两个独立的系统。ONNX Runtime 的性能通常与原生框架相当或更好，并且单一服务栈的运维简洁性对基础设施团队非常有价值。

---

### 问题 26（中等）
**量化可以将模型大小减少大约多少？**

A) 10%
B) 50%
C) 75%
D) 90%

**正确答案**：C（约 4 倍减少）

**解释**：量化通常将 float32（32 位）转换为 int8（8 位），实现约 4 倍的大小减少或约 75% 的缩小。例如，一个 100MB 的模型变成约 25MB。这还能加速推理，尤其是在 CPU 上，且精度损失极小（通常小于 1%）。这是模型部署中最有效的优化技术之一。

---

### 问题 27（简单）
**什么工具用于将 TensorFlow 模型转换为 ONNX 格式？**

A) torch.onnx
B) tf2onnx
C) onnx-tf
D) tensorflow-converter

**正确答案**：B

**解释**：`tf2onnx` 是将 TensorFlow 模型转换为 ONNX 格式的官方工具。你可以将其作为命令行工具（`python -m tf2onnx.convert`）或 Python 库使用。PyTorch 内置了 ONNX 导出（`torch.onnx.export`），但 TensorFlow 需要单独的 tf2onnx 包。

---

### 问题 28（困难）
**在生产模型服务系统中，你想逐步推出新的模型版本。你应该实现哪种基础设施模式？**

A) 蓝绿部署——瞬时切换所有流量
B) 带流量拆分的 A/B 测试（如 90% 旧版本，10% 新版本）
C) 金丝雀部署——先部署到一台服务器
D) 滚动重启——逐个更新服务器

**正确答案**：B

**解释**：带逐步流量拆分的 A/B 测试是 ML 模型部署最安全的方法。首先将 5-10% 的流量路由到新模型版本，同时监控指标（延迟、准确率、错误率）。如果指标良好，逐步增加新版本的流量。这使你能够在问题影响所有用户之前捕获问题，并使用真实生产数据比较模型版本。这比简单的蓝绿部署或金丝雀部署更为复杂。

---

## 评分指南

### 分数解读

- **24-28（86-100%）**：优秀！你对 ML 基础设施基础有很强的理解。准备好进行动手项目。
- **20-23（71-85%）**：良好！你理解了核心概念。复习做错的题目并尝试练习。
- **16-19（57-70%）**：一般。你掌握了基本概念但需要更多练习。重新阅读答错题目对应的讲座部分。
- **低于 16（<57%）**：需要改进。全面复习所有讲座并重新参加测验。

### 难度分布

你的测验包含：
- **简单题（40%）**：11 题 - 测试基本记忆和理解
- **中等题（40%）**：11 题 - 测试应用和分析
- **困难题（20%）**：6 题 - 测试综合和问题解决

### 主题分布

- **ML 概述（A 部分）**：6 题（21%）
- **PyTorch 基础（B 部分）**：8 题（29%）
- **TensorFlow 基础（C 部分）**：8 题（29%）
- **模型格式/ONNX（D 部分）**：6 题（21%）

---

## 下一步

根据你的分数：

### 如果得分 70% 或更高：
1. 进入动手练习：`exercises/exercise-01-pytorch-inference.md`
2. 开始构建模块项目
3. 考虑帮助其他在概念上有困难的人

### 如果得分低于 70%：
1. 复习做错题目对应讲座笔记
2. 重点关注出错最多的部分
3. 尝试实践练习以强化学习
4. 学习后重新参加测验

### 需要掌握的关键概念

在继续之前，确保你理解：
- [ ] 训练与推理的区别
- [ ] PyTorch 设备管理（CPU/GPU）
- [ ] TensorFlow 模型格式（SavedModel, H5, TFLite）
- [ ] ONNX 用于框架互操作性
- [ ] 模型优化技术（量化）
- [ ] 生产服务注意事项

---

## 额外资源

- PyTorch 文档：https://pytorch.org/docs/
- TensorFlow 指南：https://www.tensorflow.org/guide
- ONNX 格式规范：https://onnx.ai/
- TorchServe：https://pytorch.org/serve/
- TensorFlow Serving：https://www.tensorflow.org/tfx/guide/serving

---

**测验版本**：1.0
**最后更新**：2025 年 10 月
**预计完成时间**：45-60 分钟
