# 讲座 02：面向基础设施工程师的 PyTorch 基础

## 学习目标

到本讲座结束时，你将能够：

- 理解 PyTorch 架构及其在 ML 基础设施中的角色
- 使用 PyTorch tensor 并管理 CPU/GPU 设备
- 从多种来源加载预训练 PyTorch 模型
- 在生产场景下使用 PyTorch 模型运行推理
- 理解 PyTorch 模型序列化格式
- 识别并解决常见的 PyTorch 部署问题
- 估算 PyTorch 模型的资源需求

**时长**: 8-10 小时
**难度**: 初级到中级
**前置要求**: Module 001（Python），讲座 01（ML 概述）

---

## 1. PyTorch 简介

### 什么是 PyTorch？

PyTorch 是由 Meta（Facebook）AI Research 开发的开源机器学习框架。对于基础设施工程师来说，PyTorch 很重要，因为：

**行业采用情况**：
- 被主要公司使用：Meta, Tesla, Microsoft, OpenAI
- 为 GPT 模型、Tesla Autopilot 和 Meta 的推荐系统等产品提供动力
- 在研究领域实力强劲，在生产领域也越来越普遍

**基础设施视角**：
- **动态计算图**：模型就是 Python 代码，更容易调试
- **Python 原生**：与 Python 基础设施自然集成
- **GPU 加速**：针对 NVIDIA CUDA 优化
- **生产就绪**：TorchServe 用于模型服务
- **庞大生态系统**：HuggingFace Transformers, PyTorch Lightning 等

### PyTorch 与其它框架对比

| 特性 | PyTorch | TensorFlow | ONNX |
|---------|---------|------------|------|
| **图类型** | 动态（即时执行） | 静态（图模式） | 静态 |
| **调试** | Python 调试器可用 | 较困难 | 不适用 |
| **生产工具** | TorchServe | TF Serving | ONNX Runtime |
| **移动支持** | PyTorch Mobile | TFLite | ONNX Runtime Mobile |
| **生态系统** | 研究导向 | 生产导向 | 互操作性 |
| **学习曲线** | Pythonic，较容易 | 较陡峭 | 仅为模型格式 |

**对于基础设施**：
- PyTorch：更容易调试，在研究组织中常见
- TensorFlow：服务工具更成熟，企业采用率高
- ONNX：与框架无关的部署

---

## 2. PyTorch 安装与环境

### 安装选项

```bash
# 仅 CPU（轻量级，处处可用）
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cpu

# 启用 GPU（需要 NVIDIA GPU 和 CUDA）
# 先检查 CUDA 版本
nvidia-smi  # 查找 "CUDA Version: XX.X"

# CUDA 11.8
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118

# CUDA 12.1
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121
```

### 验证安装

```python
import torch

# 检查 PyTorch 版本
print(f"PyTorch version: {torch.__version__}")

# 检查 CUDA 可用性
print(f"CUDA available: {torch.cuda.is_available()}")

# 如果 CUDA 可用
if torch.cuda.is_available():
    print(f"CUDA version: {torch.version.cuda}")
    print(f"Number of GPUs: {torch.cuda.device_count()}")
    print(f"GPU name: {torch.cuda.get_device_name(0)}")

# 检查 CPU 能力
print(f"Number of CPU threads: {torch.get_num_threads()}")
```

**预期输出（有 GPU）**：
```
PyTorch version: 2.1.0+cu118
CUDA available: True
CUDA version: 11.8
Number of GPUs: 1
GPU name: NVIDIA Tesla T4
Number of CPU threads: 8
```

### 基础设施注意事项

**Docker 镜像**：
```dockerfile
# 官方 PyTorch 镜像
FROM pytorch/pytorch:2.1.0-cuda11.8-cudnn8-runtime

# 或自行构建
FROM nvidia/cuda:11.8.0-cudnn8-runtime-ubuntu22.04
RUN pip install torch==2.1.0 torchvision==0.16.0 --index-url https://download.pytorch.org/whl/cu118
```

**依赖项**：
- **torch**：核心库（CPU 版约 800MB，GPU 版约 2GB）
- **torchvision**：计算机视觉模型和工具
- **torchaudio**：音频处理（大多数基础设施工作可选）
- **CUDA toolkit**：需要 GPU 支持（已包含在 nvidia/cuda 镜像中）

---

## 3. PyTorch Tensors：基础

### 什么是 Tensors？

Tensors 是多维数组——PyTorch 中的基本数据结构。可以把它们看作带有 GPU 加速和自动微分的 NumPy 数组。

```python
import torch
import numpy as np

# 创建 tensors
t1 = torch.tensor([1, 2, 3, 4])                # 从 Python 列表创建
t2 = torch.from_numpy(np.array([1, 2, 3, 4]))  # 从 NumPy 创建
t3 = torch.zeros(2, 3)                          # 2x3 零矩阵
t4 = torch.ones(3, 4)                           # 3x4 单位矩阵
t5 = torch.rand(2, 3)                           # 随机值 [0, 1)
t6 = torch.randn(2, 3)                          # 随机正态分布

print(f"t1: {t1}")
print(f"t3 shape: {t3.shape}")
print(f"t5:\n{t5}")
```

**输出**：
```
t1: tensor([1, 2, 3, 4])
t3 shape: torch.Size([2, 3])
t5:
tensor([[0.3745, 0.9507, 0.7320],
        [0.5987, 0.1560, 0.1560]])
```

### Tensor 维度和形状

```python
# ML 中常见的 tensor 形状
scalar = torch.tensor(42)                      # 0D：单个值
vector = torch.tensor([1, 2, 3])               # 1D：[3]
matrix = torch.tensor([[1, 2], [3, 4]])        # 2D：[2, 2]
image = torch.rand(3, 224, 224)                # 3D：[C, H, W]
batch_images = torch.rand(32, 3, 224, 224)     # 4D：[N, C, H, W]

print(f"Image shape: {image.shape}")           # torch.Size([3, 224, 224])
print(f"Batch shape: {batch_images.shape}")    # torch.Size([32, 3, 224, 224])

# 形状约定：
# N = batch size
# C = channels（3 表示 RGB，1 表示灰度）
# H = height
# W = width
```

### 数据类型（dtype）

```python
# 常见数据类型
t_float32 = torch.tensor([1.0, 2.0], dtype=torch.float32)  # 模型默认
t_float16 = torch.tensor([1.0, 2.0], dtype=torch.float16)  # 半精度（更快）
t_int64 = torch.tensor([1, 2], dtype=torch.int64)          # 整数默认
t_bool = torch.tensor([True, False], dtype=torch.bool)     # 布尔

print(f"float32 dtype: {t_float32.dtype}")
print(f"Memory: float32={t_float32.element_size()} bytes, float16={t_float16.element_size()} bytes")
```

**基础设施影响**：
- **float32**：标准精度，每个值 4 字节
- **float16**：半精度，每个值 2 字节，节省 2 倍内存，在现代 GPU 上更快
- **int8**：量化模型，每个值 1 字节，比 float32 节省 4 倍

---

## 4. 设备管理：CPU vs GPU

### 理解设备

```python
import torch

# 检查可用设备
cpu_device = torch.device('cpu')
cuda_device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')

print(f"Using device: {cuda_device}")

# 在特定设备上创建 tensors
t_cpu = torch.rand(3, 3, device='cpu')
t_gpu = torch.rand(3, 3, device='cuda') if torch.cuda.is_available() else torch.rand(3, 3)

print(f"t_cpu device: {t_cpu.device}")
print(f"t_gpu device: {t_gpu.device}")

# 在设备间移动 tensors
t_moved = t_cpu.to('cuda' if torch.cuda.is_available() else 'cpu')
print(f"Moved tensor device: {t_moved.device}")
```

### 性能对比

```python
import time

size = 5000

# CPU 计算
t_cpu = torch.rand(size, size)
start = time.time()
result_cpu = torch.matmul(t_cpu, t_cpu)
cpu_time = time.time() - start
print(f"CPU time: {cpu_time:.4f} seconds")

# GPU 计算（如果可用）
if torch.cuda.is_available():
    t_gpu = torch.rand(size, size, device='cuda')

    # 预热（首次运行较慢）
    _ = torch.matmul(t_gpu, t_gpu)
    torch.cuda.synchronize()

    # 实际基准测试
    start = time.time()
    result_gpu = torch.matmul(t_gpu, t_gpu)
    torch.cuda.synchronize()  # 等待 GPU 完成
    gpu_time = time.time() - start

    print(f"GPU time: {gpu_time:.4f} seconds")
    print(f"Speedup: {cpu_time/gpu_time:.2f}x")
```

**典型结果**：
```
CPU time: 2.3456 seconds
GPU time: 0.0234 seconds
Speedup: 100.24x
```

### 基础设施最佳实践

```python
# 模式 1：检测设备一次，到处使用
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')

# 模式 2：将模型和数据移动到同一设备
model = MyModel().to(device)
data = data.to(device)
output = model(data)

# 模式 3：完成后清除 GPU 内存
if torch.cuda.is_available():
    torch.cuda.empty_cache()

# 模式 4：监控 GPU 内存
if torch.cuda.is_available():
    print(f"Memory allocated: {torch.cuda.memory_allocated() / 1024**3:.2f} GB")
    print(f"Memory reserved: {torch.cuda.memory_reserved() / 1024**3:.2f} GB")
```

**常见错误**：
```python
# 这将会失败！
model = MyModel().to('cuda')
data = torch.rand(1, 3, 224, 224)  # 默认在 CPU
output = model(data)  # RuntimeError: Expected all tensors to be on the same device
```

**修复**：
```python
model = MyModel().to('cuda')
data = torch.rand(1, 3, 224, 224).to('cuda')  # 移动到 GPU
output = model(data)  # 正常！
```

---

## 5. 加载预训练模型

### 从 PyTorch Hub 加载

```python
import torch

# 加载预训练的 ResNet50 模型
model = torch.hub.load('pytorch/vision:v0.10.0', 'resnet50', pretrained=True)
model.eval()  # 设置为评估模式（重要！）

print(f"Model type: {type(model)}")
print(f"Number of parameters: {sum(p.numel() for p in model.parameters()):,}")
```

**输出**：
```
Model type: <class 'torchvision.models.resnet.ResNet'>
Number of parameters: 25,557,032
```

### 从 HuggingFace 加载

```python
# 首先安装 transformers
# pip install transformers

from transformers import AutoModel, AutoTokenizer

# 加载 BERT 模型和 tokenizer
model_name = "bert-base-uncased"
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModel.from_pretrained(model_name)

print(f"Model: {model.__class__.__name__}")
print(f"Vocab size: {tokenizer.vocab_size}")
```

### 从本地文件加载

```python
import torch

# 保存模型（你会在训练后这样做）
model = torch.hub.load('pytorch/vision', 'resnet18', pretrained=True)
torch.save(model.state_dict(), 'resnet18_weights.pth')
torch.save(model, 'resnet18_full.pth')

# 从 state_dict 加载（推荐）
model_new = torch.hub.load('pytorch/vision', 'resnet18', pretrained=False)
model_new.load_state_dict(torch.load('resnet18_weights.pth'))
model_new.eval()

# 加载完整模型
model_full = torch.load('resnet18_full.pth')
model_full.eval()

print("Models loaded successfully!")
```

### 基础设施注意事项

**模型存储**：
```bash
# 默认缓存模型
ls ~/.cache/torch/hub/checkpoints/
# resnet50-0676ba61.pth (97.8 MB)
# resnet18-5c106cde.pth (44.7 MB)

# HuggingFace 缓存
ls ~/.cache/huggingface/hub/
# models--bert-base-uncased/ (440 MB)
```

**最佳实践**：
```python
import os
from pathlib import Path

# 设置自定义缓存目录（容器中有用）
os.environ['TORCH_HOME'] = '/data/models/torch'
os.environ['HF_HOME'] = '/data/models/huggingface'

# 在 Docker 构建期间预下载模型
# 这避免了推理期间的下载延迟
model = torch.hub.load('pytorch/vision', 'resnet50', pretrained=True)
```

---

## 6. 使用 PyTorch 运行推理

### 图像分类示例

```python
import torch
from torchvision import transforms
from PIL import Image
import urllib.request

# 加载模型
model = torch.hub.load('pytorch/vision', 'resnet50', pretrained=True)
model.eval()

# 加载和预处理图像
url = "https://github.com/pytorch/hub/raw/master/images/dog.jpg"
urllib.request.urlretrieve(url, "dog.jpg")
image = Image.open("dog.jpg")

# 定义预处理管道
preprocess = transforms.Compose([
    transforms.Resize(256),
    transforms.CenterCrop(224),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]),
])

# 预处理并添加 batch 维度
input_tensor = preprocess(image)
input_batch = input_tensor.unsqueeze(0)  # 添加 batch 维度：[3, 224, 224] -> [1, 3, 224, 224]

# 运行推理
with torch.no_grad():  # 禁用梯度计算（更快，更省内存）
    output = model(input_batch)

# 获取预测结果
probabilities = torch.nn.functional.softmax(output[0], dim=0)
top5_prob, top5_catid = torch.topk(probabilities, 5)

# 加载 ImageNet 标签
labels_url = "https://raw.githubusercontent.com/pytorch/hub/master/imagenet_classes.txt"
urllib.request.urlretrieve(labels_url, "imagenet_classes.txt")
with open("imagenet_classes.txt") as f:
    labels = [line.strip() for line in f.readlines()]

# 打印结果
print("\nTop 5 predictions:")
for i in range(5):
    print(f"{labels[top5_catid[i]]}: {top5_prob[i].item():.4f}")
```

**输出**：
```
Top 5 predictions:
Samoyed: 0.8932
Arctic fox: 0.0402
white wolf: 0.0311
Pomeranian: 0.0189
Great Pyrenees: 0.0142
```

### 批量推理

```python
import torch
import time

model = torch.hub.load('pytorch/vision', 'resnet50', pretrained=True)
model.eval()

# 单样本推理
single_input = torch.rand(1, 3, 224, 224)
start = time.time()
with torch.no_grad():
    _ = model(single_input)
single_time = time.time() - start
print(f"Single sample: {single_time:.4f} seconds")

# 批量推理
batch_sizes = [1, 8, 16, 32, 64]
for batch_size in batch_sizes:
    batch_input = torch.rand(batch_size, 3, 224, 224)
    start = time.time()
    with torch.no_grad():
        _ = model(batch_input)
    batch_time = time.time() - start
    throughput = batch_size / batch_time
    print(f"Batch {batch_size:2d}: {batch_time:.4f}s ({throughput:.2f} samples/sec)")
```

**典型输出（GPU）**：
```
Single sample: 0.0123 seconds
Batch  1: 0.0125s (80.00 samples/sec)
Batch  8: 0.0234s (341.88 samples/sec)
Batch 16: 0.0398s (402.01 samples/sec)
Batch 32: 0.0756s (423.28 samples/sec)
Batch 64: 0.1489s (429.82 samples/sec)
```

**关键见解**：
- 批处理显著提高吞吐量
- GPU 利用率随批次增大而提高
- 超过一定批大小后收益递减（内存限制）

---

## 7. 模型序列化和加载

### 两种序列化方法

**方法 1：保存 State Dict（推荐）**：
```python
# 保存
model = torch.hub.load('pytorch/vision', 'resnet18', pretrained=True)
torch.save(model.state_dict(), 'model_weights.pth')

# 加载
model = torch.hub.load('pytorch/vision', 'resnet18', pretrained=False)
model.load_state_dict(torch.load('model_weights.pth'))
model.eval()
```

**方法 2：保存整个模型**：
```python
# 保存
torch.save(model, 'model_full.pth')

# 加载
model = torch.load('model_full.pth')
model.eval()
```

### 对比

| 方面 | State Dict | 完整模型 |
|--------|-----------|------------|
| **文件大小** | 较小（仅权重） | 较大（权重 + 架构） |
| **可移植性** | 需要模型类定义 | 自包含 |
| **灵活性** | 可以修改架构 | 架构固定 |
| **最适合** | 生产部署 | 快速实验 |
| **建议** | ✅ 使用这个 | ❌ 避免在生产中使用 |

### 生产序列化模式

```python
import torch
from pathlib import Path

class ModelSerializer:
    @staticmethod
    def save_model(model, path: str, metadata: dict = None):
        """保存带有元数据的模型，用于生产"""
        save_dict = {
            'model_state_dict': model.state_dict(),
            'model_class': model.__class__.__name__,
            'pytorch_version': torch.__version__,
        }
        if metadata:
            save_dict['metadata'] = metadata

        torch.save(save_dict, path)
        print(f"Model saved to {path}")

    @staticmethod
    def load_model(model, path: str):
        """加载模型并验证元数据"""
        checkpoint = torch.load(path)

        # 验证 PyTorch 版本
        saved_version = checkpoint.get('pytorch_version', 'unknown')
        current_version = torch.__version__
        if saved_version != current_version:
            print(f"Warning: Model saved with PyTorch {saved_version}, loading with {current_version}")

        # 加载权重
        model.load_state_dict(checkpoint['model_state_dict'])
        model.eval()

        # 如果存在则返回元数据
        return checkpoint.get('metadata', {})

# 使用
model = torch.hub.load('pytorch/vision', 'resnet18', pretrained=True)

# 保存带有元数据
metadata = {
    'training_date': '2025-10-18',
    'accuracy': 0.95,
    'dataset': 'ImageNet',
}
ModelSerializer.save_model(model, 'production_model.pth', metadata)

# 加载
model_new = torch.hub.load('pytorch/vision', 'resnet18', pretrained=False)
metadata = ModelSerializer.load_model(model_new, 'production_model.pth')
print(f"Loaded model with metadata: {metadata}")
```

---

## 8. TorchServe：生产模型服务

### 什么是 TorchServe？

TorchServe 是 PyTorch 官方的生产部署模型服务框架。

**特性**：
- RESTful 和 gRPC API
- 多模型服务
- 模型版本管理
- 自动扩展
- 指标和日志
- A/B 测试支持

### 安装 TorchServe

```bash
pip install torchserve torch-model-archiver torch-workflow-archiver
```

### 创建模型归档

```python
# handler.py - 自定义推理逻辑
import torch
from torchvision import transforms
from ts.torch_handler.base_handler import BaseHandler
from PIL import Image
import io

class ImageClassifierHandler(BaseHandler):
    def __init__(self):
        super(ImageClassifierHandler, self).__init__()
        self.transform = transforms.Compose([
            transforms.Resize(256),
            transforms.CenterCrop(224),
            transforms.ToTensor(),
            transforms.Normalize(mean=[0.485, 0.456, 0.406],
                              std=[0.229, 0.224, 0.225]),
        ])

    def preprocess(self, data):
        images = []
        for row in data:
            image = Image.open(io.BytesIO(row.get("data") or row.get("body")))
            image = self.transform(image)
            images.append(image)
        return torch.stack(images)

    def postprocess(self, data):
        return data.argmax(dim=1).tolist()
```

```bash
# 创建模型归档
torch-model-archiver \
    --model-name resnet50 \
    --version 1.0 \
    --model-file model.py \
    --serialized-file resnet50_weights.pth \
    --handler handler.py \
    --export-path model-store

# 启动 TorchServe
torchserve --start --model-store model-store --models resnet50=resnet50.mar

# 测试推理
curl -X POST http://localhost:8080/predictions/resnet50 -T image.jpg
```

### TorchServe 在 Docker 中

```dockerfile
FROM pytorch/torchserve:latest

# 复制模型归档
COPY resnet50.mar /home/model-server/model-store/

# 暴露端口
EXPOSE 8080 8081 8082

# 启动 TorchServe 并加载模型
CMD ["torchserve", \
     "--start", \
     "--model-store", "/home/model-server/model-store", \
     "--models", "resnet50=resnet50.mar"]
```

---

## 9. 常见问题和故障排查

### 问题 1：CUDA 内存不足

**错误**：
```
RuntimeError: CUDA out of memory. Tried to allocate 1024.00 MiB
```

**解决方案**：
```python
# 1. 减少批大小
batch_size = 8  # 替代 32

# 2. 使用梯度检查点（训练期间）
# 对于推理，确保 torch.no_grad()
with torch.no_grad():
    output = model(input)

# 3. 在批次之间清除缓存
torch.cuda.empty_cache()

# 4. 使用混合精度（float16 替代 float32）
with torch.cuda.amp.autocast():
    output = model(input)

# 5. 监控内存使用
print(f"Allocated: {torch.cuda.memory_allocated() / 1024**3:.2f} GB")
print(f"Reserved: {torch.cuda.memory_reserved() / 1024**3:.2f} GB")
```

### 问题 2：模型在错误的设备上

**错误**：
```
RuntimeError: Expected all tensors to be on the same device, but found at least two devices, cuda:0 and cpu!
```

**解决方案**：
```python
# 确保模型和输入在同一设备上
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
model = model.to(device)
input_tensor = input_tensor.to(device)
output = model(input_tensor)
```

### 问题 3：模型处于训练模式

**问题**：模型每次给出不同输出（由于 dropout/batchnorm）

**解决方案**：
```python
# 推理时始终将模型设置为 eval 模式
model.eval()

# 使用 no_grad 上下文
with torch.no_grad():
    output = model(input)
```

### 问题 4：输入形状不匹配

**错误**：
```
RuntimeError: shape '[32, 3, 224, 224]' is invalid for input of size 150528
```

**解决方案**：
```python
# 检查预期的输入形状
print(f"Input shape: {input_tensor.shape}")  # torch.Size([224, 224, 3])

# 重排维度：HWC -> CHW
input_tensor = input_tensor.permute(2, 0, 1)  # 现在 [3, 224, 224]

# 添加 batch 维度
input_tensor = input_tensor.unsqueeze(0)  # 现在 [1, 3, 224, 224]
```

### 问题 5：版本不匹配

**错误**：
```
RuntimeError: version_ <= kMaxSupportedFileFormatVersion INTERNAL ASSERT FAILED
```

**解决方案**：
```python
# 使用特定协议版本保存以确保兼容性
torch.save(model.state_dict(), 'model.pth', _use_new_zipfile_serialization=False)

# 或升级 PyTorch 版本以匹配保存模型的版本
```

---

## 10. 资源估算和优化

### 模型大小和内存

```python
import torch

def estimate_model_size(model):
    """估算模型在内存中的大小"""
    param_size = 0
    buffer_size = 0

    for param in model.parameters():
        param_size += param.nelement() * param.element_size()

    for buffer in model.buffers():
        buffer_size += buffer.nelement() * buffer.element_size()

    size_mb = (param_size + buffer_size) / 1024**2
    return size_mb

model = torch.hub.load('pytorch/vision', 'resnet50', pretrained=True)
print(f"Model size: {estimate_model_size(model):.2f} MB")
```

### 推理性能基准测试

```python
import torch
import time

def benchmark_inference(model, input_shape, batch_sizes=[1, 8, 16, 32], warmup=10, iterations=100):
    """对模型推理性能进行基准测试"""
    device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
    model = model.to(device).eval()

    results = []
    for batch_size in batch_sizes:
        # 创建虚拟输入
        dummy_input = torch.rand(batch_size, *input_shape).to(device)

        # 预热
        with torch.no_grad():
            for _ in range(warmup):
                _ = model(dummy_input)

        # 基准测试
        start = time.time()
        with torch.no_grad():
            for _ in range(iterations):
                _ = model(dummy_input)

        if torch.cuda.is_available():
            torch.cuda.synchronize()

        elapsed = time.time() - start
        throughput = (batch_size * iterations) / elapsed
        latency = (elapsed / iterations) * 1000  # ms

        results.append({
            'batch_size': batch_size,
            'throughput': throughput,
            'latency_per_batch': latency,
            'latency_per_sample': latency / batch_size,
        })

    return results

# 使用
model = torch.hub.load('pytorch/vision', 'resnet50', pretrained=True)
results = benchmark_inference(model, input_shape=(3, 224, 224))

print("\nBenchmark Results:")
print(f"{'Batch Size':<12} {'Throughput':<15} {'Latency/Batch':<15} {'Latency/Sample':<15}")
for r in results:
    print(f"{r['batch_size']:<12} {r['throughput']:<15.2f} {r['latency_per_batch']:<15.2f} {r['latency_per_sample']:<15.2f}")
```

### 优化技术

**1. 混合精度推理**：
```python
# 使用 float16 替代 float32
model = model.half()  # 将模型转换为 float16
input = input.half()  # 将输入转换为 float16

# 或使用自动混合精度
with torch.cuda.amp.autocast():
    output = model(input)
```

**2. TorchScript 编译**：
```python
# 跟踪模型以实现更快的推理
example_input = torch.rand(1, 3, 224, 224)
traced_model = torch.jit.trace(model, example_input)
traced_model.save('model_traced.pt')

# 加载并使用
model = torch.jit.load('model_traced.pt')
output = model(input)
```

**3. 量化**：
```python
# 动态量化（最简单，仅 CPU）
quantized_model = torch.quantization.quantize_dynamic(
    model, {torch.nn.Linear}, dtype=torch.qint8
)

# 模型大小减少约 4 倍，CPU 上更快
```

---

## 关键要点

### 对于基础设施工程师

1. **PyTorch 是 Python 原生**：易于调试，自然集成
2. **设备管理至关重要**：始终匹配模型和输入设备
3. **批量推理提高吞吐量**：平衡延迟和吞吐量
4. **推理时使用 torch.no_grad()**：节省内存和计算
5. **生产环境使用 TorchServe**：不要自己构建服务框架
6. **模型序列化很重要**：使用 state_dict 以获得灵活性
7. **监控 GPU 内存**：OOM 错误很常见，使用 empty_cache()
8. **版本兼容性**：PyTorch 版本必须匹配才能加载模型

### 生产清单

部署 PyTorch 模型前：

- [ ] 模型设置为 `.eval()` 模式
- [ ] 所有推理在 `torch.no_grad()` 上下文中
- [ ] 输入预处理已定义和测试
- [ ] 输出后处理已定义和测试
- [ ] 设备管理（CPU/GPU）正确处理
- [ ] OOM 和设备错误的处理
- [ ] 模型版本和元数据已保存
- [ ] 性能基准测试（延迟/吞吐量）
- [ ] 内存需求已估算
- [ ] TorchServe 或自定义服务已测试

---

## 快速参考

### 基本命令

```python
# 安装检查
import torch
print(torch.__version__)
print(torch.cuda.is_available())

# 设备管理
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
model = model.to(device)
tensor = tensor.to(device)

# 加载模型
model = torch.hub.load('repo', 'model_name', pretrained=True)
model.eval()

# 推理
with torch.no_grad():
    output = model(input)

# 内存管理
torch.cuda.empty_cache()
print(torch.cuda.memory_allocated())
```

### 常见模型来源

- **PyTorch Hub**: `torch.hub.load('pytorch/vision', 'resnet50')`
- **HuggingFace**: `AutoModel.from_pretrained('bert-base-uncased')`
- **Torchvision**: `torchvision.models.resnet50(pretrained=True)`
- **自定义**: `torch.load('model.pth')`

---

## 下一步

在下一讲中，我们将涵盖：
- **TensorFlow 基础**：与 PyTorch 对比
- **TensorFlow Serving**：生产服务框架
- **模型格式差异**：SavedModel 与 state_dict

继续学习 `lecture-notes/03-tensorflow-basics.md`

---

**讲座版本**: 1.0
**最后更新**: 2025 年 10 月
**预计完成时间**: 8-10 小时
**难度**: 初级到中级
