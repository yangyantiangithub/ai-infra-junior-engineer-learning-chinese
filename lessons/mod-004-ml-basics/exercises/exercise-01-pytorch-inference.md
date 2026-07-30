# 练习 01：PyTorch 模型推理

## 练习概述

**目标**：加载预训练的 PyTorch 图像分类模型，对真实图像运行推理，并测量性能指标。

**难度**：初级
**预计时间**：2-3 小时
**前置要求**：讲座 02（PyTorch 基础）

**你将学到**：
- 从 PyTorch Hub 加载预训练模型
- 预处理图像以供模型输入
- 在 eval 模式下运行推理
- 测量推理延迟和吞吐量
- 处理 CPU vs GPU 执行

---

## 学习目标

到本练习结束时，你将能够：

✅ 从多种来源加载预训练 PyTorch 模型
✅ 预处理图像以匹配模型要求
✅ 使用 `model.eval()` 和 `torch.no_grad()` 正确运行推理
✅ 测量和优化推理性能
✅ 处理设备管理（CPU/GPU）
✅ 调试常见的推理错误

---

## 第 1 部分：环境设置

### 步骤 1.1：安装依赖

```bash
# 创建虚拟环境
python -m venv pytorch_inference_env
source pytorch_inference_env/bin/activate  # Windows: pytorch_inference_env\Scripts\activate

# 安装 PyTorch 和依赖
pip install torch torchvision pillow requests
```

### 步骤 1.2：验证安装

```python
# verify_installation.py
import torch
import torchvision

print(f"PyTorch version: {torch.__version__}")
print(f"Torchvision version: {torchvision.__version__}")
print(f"CUDA available: {torch.cuda.is_available()}")

if torch.cuda.is_available():
    print(f"CUDA version: {torch.version.cuda}")
    print(f"GPU: {torch.cuda.get_device_name(0)}")
else:
    print("Running on CPU")
```

**预期输出**：
```
PyTorch version: 2.1.0+cu118
Torchvision version: 0.16.0+cu118
CUDA available: True
CUDA version: 11.8
GPU: NVIDIA Tesla T4
```

### 步骤 1.3：下载测试图像

```python
# download_images.py
import urllib.request

# 下载样本图像
images = {
    "dog.jpg": "https://github.com/pytorch/hub/raw/master/images/dog.jpg",
    "cat.jpg": "https://upload.wikimedia.org/wikipedia/commons/4/4d/Cat_November_2010-1a.jpg",
    "car.jpg": "https://upload.wikimedia.org/wikipedia/commons/9/9a/2014_Mazda_3_%28BM%29_2.0_sedan_%282015-08-07%29_01.jpg"
}

for filename, url in images.items():
    print(f"Downloading {filename}...")
    urllib.request.urlretrieve(url, filename)

print("All images downloaded!")
```

✅ **检查点**：你应该已经设置好 Python 环境并下载了 3 张测试图像。

---

## 第 2 部分：加载预训练模型

### 步骤 2.1：从 PyTorch Hub 加载 ResNet50

```python
# load_model.py
import torch

# 加载预训练 ResNet50
print("Loading ResNet50...")
model = torch.hub.load('pytorch/vision:v0.10.0', 'resnet50', pretrained=True)

# 关键：设置为评估模式
model.eval()

print(f"Model loaded: {model.__class__.__name__}")
print(f"Number of parameters: {sum(p.numel() for p in model.parameters()):,}")
```

**预期输出**：
```
Loading ResNet50...
Downloading: 100%|██████████| 97.8M/97.8M
Model loaded: ResNet
Number of parameters: 25,557,032
```

### 步骤 2.2：理解模型架构

**任务**：检查模型结构以理解其层。

```python
# examine_model.py
import torch

model = torch.hub.load('pytorch/vision', 'resnet50', pretrained=True)

# 打印模型摘要
print("Model Architecture:")
print(model)

# 按类型统计层数
layer_counts = {}
for name, module in model.named_modules():
    layer_type = module.__class__.__name__
    layer_counts[layer_type] = layer_counts.get(layer_type, 0) + 1

print("\nLayer Counts:")
for layer_type, count in sorted(layer_counts.items()):
    print(f"  {layer_type}: {count}")

# 模型输入要求
print("\nModel Expectations:")
print("  Input shape: [batch_size, 3, 224, 224]")
print("  Input type: float32 tensor")
print("  Output shape: [batch_size, 1000] (ImageNet classes)")
```

**要回答的问题**（写下你的答案）：
1. ResNet50 有多少个卷积层？
2. 预期的输入形状是什么？
3. 模型预测多少个输出类别？

✅ **检查点**：模型加载成功，你理解了其架构。

---

## 第 3 部分：图像预处理

### 步骤 3.1：实现预处理管道

```python
# preprocess.py
import torch
from torchvision import transforms
from PIL import Image

def create_preprocessing_pipeline():
    """为 ResNet 模型创建图像预处理管道"""
    return transforms.Compose([
        transforms.Resize(256),                # 将最短边调整为 256
        transforms.CenterCrop(224),            # 中心裁剪 224x224
        transforms.ToTensor(),                 # 转换为 tensor [0, 1]
        transforms.Normalize(                  # 使用 ImageNet 统计数据进行归一化
            mean=[0.485, 0.456, 0.406],
            std=[0.229, 0.224, 0.225]
        ),
    ])

def preprocess_image(image_path: str):
    """加载并预处理图像"""
    # 加载图像
    image = Image.open(image_path).convert('RGB')
    print(f"Original image size: {image.size}")

    # 应用预处理
    preprocess = create_preprocessing_pipeline()
    tensor = preprocess(image)

    print(f"Preprocessed tensor shape: {tensor.shape}")
    print(f"Tensor dtype: {tensor.dtype}")
    print(f"Tensor range: [{tensor.min():.3f}, {tensor.max():.3f}]")

    # 添加 batch 维度 [C, H, W] -> [1, C, H, W]
    batch = tensor.unsqueeze(0)
    print(f"Batch shape: {batch.shape}")

    return batch

# 测试预处理
image_tensor = preprocess_image("dog.jpg")
```

**预期输出**：
```
Original image size: (576, 768)
Preprocessed tensor shape: torch.Size([3, 224, 224])
Tensor dtype: torch.float32
Tensor range: [-2.118, 2.640]
Batch shape: torch.Size([1, 3, 224, 224])
```

### 步骤 3.2：理解预处理步骤

**任务**：回答这些关于预处理的问题：

1. **为什么先缩放到 256 再裁剪到 224？**
   - 提示：在保持纵横比的同时获得正确尺寸

2. **归一化值的含义是什么？**
   - mean=[0.485, 0.456, 0.406]：ImageNet 数据集的 R, G, B 均值
   - std=[0.229, 0.224, 0.225]：ImageNet 数据集的 R, G, B 标准差

3. **为什么 tensor 范围是负数？**
   - 提示：归一化公式是 (x - mean) / std

✅ **检查点**：你能正确预处理图像了。

---

## 第 4 部分：运行推理

### 步骤 4.1：基本推理

```python
# inference.py
import torch
from torchvision import transforms
from PIL import Image
import urllib.request

# 加载模型
model = torch.hub.load('pytorch/vision', 'resnet50', pretrained=True)
model.eval()  # 重要！

# 加载并预处理图像
preprocess = transforms.Compose([
    transforms.Resize(256),
    transforms.CenterCrop(224),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]),
])

image = Image.open("dog.jpg")
input_tensor = preprocess(image)
input_batch = input_tensor.unsqueeze(0)

# 运行推理
with torch.no_grad():  # 重要：禁用梯度计算
    output = model(input_batch)

print(f"Output shape: {output.shape}")  # [1, 1000]
print(f"Output type: {output.dtype}")
print(f"Output sample: {output[0, :5]}")  # 前 5 个值
```

**预期输出**：
```
Output shape: torch.Size([1, 1000])
Output type: torch.float32
Output sample: tensor([-1.2345,  0.5678, -0.9123,  1.4567, -0.3456])
```

### 步骤 4.2：获取 Top 预测

```python
# get_predictions.py
import torch
from torchvision import transforms
from PIL import Image
import urllib.request

# 加载 ImageNet 类别标签
labels_url = "https://raw.githubusercontent.com/pytorch/hub/master/imagenet_classes.txt"
urllib.request.urlretrieve(labels_url, "imagenet_classes.txt")

with open("imagenet_classes.txt") as f:
    labels = [line.strip() for line in f.readlines()]

# 加载模型和图像
model = torch.hub.load('pytorch/vision', 'resnet50', pretrained=True)
model.eval()

preprocess = transforms.Compose([
    transforms.Resize(256),
    transforms.CenterCrop(224),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]),
])

image = Image.open("dog.jpg")
input_tensor = preprocess(image)
input_batch = input_tensor.unsqueeze(0)

# 运行推理
with torch.no_grad():
    output = model(input_batch)

# 获取概率
probabilities = torch.nn.functional.softmax(output[0], dim=0)

# 获取 top 5 预测
top5_prob, top5_idx = torch.topk(probabilities, 5)

print("Top 5 Predictions:")
for i in range(5):
    class_idx = top5_idx[i].item()
    prob = top5_prob[i].item()
    print(f"  {i+1}. {labels[class_idx]:<30} {prob*100:>6.2f}%")
```

**预期输出**：
```
Top 5 Predictions:
  1. Samoyed                      89.32%
  2. Pomeranian                    4.02%
  3. white wolf                    3.11%
  4. keeshond                      1.89%
  5. Great Pyrenees                1.42%
```

✅ **检查点**：你能运行推理并得到有意义的预测结果了！

---

## 第 5 部分：性能测量

### 步骤 5.1：测量推理延迟

```python
# measure_latency.py
import torch
from torchvision import transforms
from PIL import Image
import time

# 加载模型
model = torch.hub.load('pytorch/vision', 'resnet50', pretrained=True)
model.eval()

# 准备输入
preprocess = transforms.Compose([
    transforms.Resize(256),
    transforms.CenterCrop(224),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]),
])

image = Image.open("dog.jpg")
input_tensor = preprocess(image).unsqueeze(0)

# 预热（首次运行较慢）
print("Warming up...")
with torch.no_grad():
    for _ in range(10):
        _ = model(input_tensor)

# 测量延迟
print("\nMeasuring latency...")
iterations = 100
start = time.time()

with torch.no_grad():
    for _ in range(iterations):
        _ = model(input_tensor)

elapsed = time.time() - start
avg_latency_ms = (elapsed / iterations) * 1000

print(f"Average latency: {avg_latency_ms:.2f} ms")
print(f"Throughput: {1000/avg_latency_ms:.2f} images/second")
```

### 步骤 5.2：批量推理性能

```python
# batch_inference.py
import torch
import time

model = torch.hub.load('pytorch/vision', 'resnet50', pretrained=True)
model.eval()

# 测试不同的批大小
batch_sizes = [1, 4, 8, 16, 32]
print(f"{'Batch Size':<12} {'Latency (ms)':<15} {'Throughput (img/s)':<20}")

for batch_size in batch_sizes:
    # 创建虚拟批次
    dummy_input = torch.rand(batch_size, 3, 224, 224)

    # 预热
    with torch.no_grad():
        for _ in range(10):
            _ = model(dummy_input)

    # 测量
    iterations = 50
    start = time.time()

    with torch.no_grad():
        for _ in range(iterations):
            _ = model(dummy_input)

    elapsed = time.time() - start
    latency_per_batch = (elapsed / iterations) * 1000
    throughput = (batch_size * iterations) / elapsed

    print(f"{batch_size:<12} {latency_per_batch:<15.2f} {throughput:<20.2f}")
```

**预期输出（CPU）**：
```
Batch Size   Latency (ms)    Throughput (img/s)
1            45.23           22.11
4            152.34          26.24
8            298.45          26.81
16           589.23          27.15
32           1156.78         27.67
```

✅ **检查点**：你能测量和理解推理性能了。

---

## 第 6 部分：GPU 加速

### 步骤 6.1：CPU vs GPU 对比

```python
# cpu_vs_gpu.py
import torch
import time

# 加载模型
model = torch.hub.load('pytorch/vision', 'resnet50', pretrained=True)
model.eval()

# 准备输入
dummy_input = torch.rand(1, 3, 224, 224)

def benchmark(model, input_tensor, device, iterations=100):
    """在特定设备上对推理进行基准测试"""
    model = model.to(device)
    input_tensor = input_tensor.to(device)

    # 预热
    with torch.no_grad():
        for _ in range(10):
            _ = model(input_tensor)

    if device.type == 'cuda':
        torch.cuda.synchronize()

    # 测量
    start = time.time()
    with torch.no_grad():
        for _ in range(iterations):
            _ = model(input_tensor)

    if device.type == 'cuda':
        torch.cuda.synchronize()

    elapsed = time.time() - start
    return (elapsed / iterations) * 1000  # ms

# CPU 基准测试
cpu_latency = benchmark(model, dummy_input, torch.device('cpu'))
print(f"CPU latency: {cpu_latency:.2f} ms")

# GPU 基准测试（如果可用）
if torch.cuda.is_available():
    gpu_latency = benchmark(model, dummy_input, torch.device('cuda'))
    print(f"GPU latency: {gpu_latency:.2f} ms")
    print(f"Speedup: {cpu_latency/gpu_latency:.2f}x")
else:
    print("GPU not available")
```

**预期输出（有 GPU）**：
```
CPU latency: 45.23 ms
GPU latency: 3.45 ms
Speedup: 13.11x
```

✅ **检查点**：你理解了 CPU vs GPU 的性能差异。

---

## 第 7 部分：完整应用

### 步骤 7.1：构建图像分类器 CLI

```python
# image_classifier.py
import torch
from torchvision import transforms
from PIL import Image
import argparse
import time

class ImageClassifier:
    def __init__(self, model_name='resnet50', device='auto'):
        """初始化图像分类器"""
        print(f"Loading {model_name}...")

        # 确定设备
        if device == 'auto':
            self.device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        else:
            self.device = torch.device(device)

        print(f"Using device: {self.device}")

        # 加载模型
        self.model = torch.hub.load('pytorch/vision', model_name, pretrained=True)
        self.model.to(self.device)
        self.model.eval()

        # 加载标签
        import urllib.request
        labels_url = "https://raw.githubusercontent.com/pytorch/hub/master/imagenet_classes.txt"
        urllib.request.urlretrieve(labels_url, "imagenet_classes.txt")

        with open("imagenet_classes.txt") as f:
            self.labels = [line.strip() for line in f.readlines()]

        # 预处理管道
        self.preprocess = transforms.Compose([
            transforms.Resize(256),
            transforms.CenterCrop(224),
            transforms.ToTensor(),
            transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]),
        ])

        print("Classifier ready!")

    def predict(self, image_path, top_k=5):
        """预测图像类别"""
        # 加载并预处理图像
        image = Image.open(image_path).convert('RGB')
        input_tensor = self.preprocess(image)
        input_batch = input_tensor.unsqueeze(0).to(self.device)

        # 运行推理
        start = time.time()
        with torch.no_grad():
            output = self.model(input_batch)
        inference_time = (time.time() - start) * 1000

        # 获取概率
        probabilities = torch.nn.functional.softmax(output[0], dim=0)

        # 获取 top k 预测
        topk_prob, topk_idx = torch.topk(probabilities, top_k)

        results = []
        for i in range(top_k):
            class_idx = topk_idx[i].item()
            prob = topk_prob[i].item()
            results.append({
                'class': self.labels[class_idx],
                'probability': prob,
                'class_id': class_idx
            })

        return results, inference_time

def main():
    parser = argparse.ArgumentParser(description='PyTorch Image Classifier')
    parser.add_argument('image', type=str, help='Path to image file')
    parser.add_argument('--model', type=str, default='resnet50', help='Model name')
    parser.add_argument('--top-k', type=int, default=5, help='Number of top predictions')
    parser.add_argument('--device', type=str, default='auto', help='Device (cpu/cuda/auto)')

    args = parser.parse_args()

    # 创建分类器
    classifier = ImageClassifier(model_name=args.model, device=args.device)

    # 运行预测
    print(f"\nClassifying: {args.image}")
    results, inference_time = classifier.predict(args.image, top_k=args.top_k)

    # 打印结果
    print(f"\nTop {args.top_k} Predictions (inference: {inference_time:.2f}ms):")
    for i, result in enumerate(results):
        print(f"  {i+1}. {result['class']:<30} {result['probability']*100:>6.2f}%")

if __name__ == "__main__":
    main()
```

**使用方法**：
```bash
# 分类一张图像
python image_classifier.py dog.jpg

# 使用不同模型
python image_classifier.py dog.jpg --model resnet18

# 强制使用 CPU
python image_classifier.py dog.jpg --device cpu

# 获取 top 10 预测
python image_classifier.py dog.jpg --top-k 10
```

✅ **检查点**：你有了一个完整、可运行的图像分类器！

---

## 拓展目标（可选挑战）

### 挑战 1：多图像批处理

**任务**：修改分类器以在单批次中处理多张图像。

```python
def predict_batch(self, image_paths, top_k=5):
    """预测多张图像的类别"""
    # TODO: 实现批量预测
    # 提示：将预处理后的图像堆叠成一个批次
    pass
```

### 挑战 2：模型比较

**任务**：比较不同的模型架构（ResNet18, ResNet50, MobileNet）。

测量：
- 推理延迟
- 准确率（在同一图像上）
- 模型大小

### 挑战 3：错误处理

**任务**：添加适当的错误处理：
- 无效的图像文件
- 内存不足错误
- 缺少 GPU

---

## 验证和测试

### 测试你的实现

```python
# test_classifier.py
import os

def test_classifier():
    """测试图像分类器实现"""
    from image_classifier import ImageClassifier

    # 测试初始化
    classifier = ImageClassifier(model_name='resnet18', device='cpu')
    assert classifier.device.type == 'cpu', "Device should be CPU"

    # 测试预测
    results, latency = classifier.predict('dog.jpg', top_k=5)
    assert len(results) == 5, "Should return 5 predictions"
    assert results[0]['probability'] > results[1]['probability'], "Should be sorted by probability"
    assert latency > 0, "Latency should be positive"

    # 测试概率之和约为 1
    total_prob = sum(r['probability'] for r in results[:5])
    assert 0.95 <= total_prob <= 1.05, f"Top 5 probabilities should sum close to 1, got {total_prob}"

    print("✅ All tests passed!")

if __name__ == "__main__":
    test_classifier()
```

---

## 反思问题

回答这些问题以巩固你的学习：

1. **为什么 `model.eval()` 是必要的？**
   - 你的答案：

2. **如果忘记 `torch.no_grad()` 会怎样？**
   - 你的答案：

3. **为什么我们需要用 ImageNet 统计数据归一化图像？**
   - 你的答案：

4. **批大小如何影响吞吐量和延迟？**
   - 你的答案：

5. **什么时候你会选择 CPU 而非 GPU 进行推理？**
   - 你的答案：

---

## 总结

**你完成了什么**：
✅ 加载并使用预训练 PyTorch 模型
✅ 正确预处理图像
✅ 使用适当的 eval 模式和 no_grad 上下文运行推理
✅ 测量推理性能
✅ 比较 CPU 和 GPU 执行
✅ 构建了完整的图像分类应用

**获得的关键技能**：
- 从 PyTorch Hub 加载模型
- 使用 torchvision.transforms 进行图像预处理
- 正确的推理模式（eval, no_grad）
- 性能测量和优化
- 设备管理（CPU/GPU）

---

## 下一步

1. **完成练习 02**：TensorFlow 模型推理
2. **比较框架**：注意 PyTorch 和 TensorFlow 之间的差异
3. **实验**：尝试不同的模型和图像

---

**练习版本**: 1.0
**最后更新**: 2025 年 10 月
**预计时间**: 2-3 小时
**难度**: 初级
