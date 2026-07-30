# 讲座 04：模型格式和部署准备

## 学习目标

到本讲座结束时，你将能够：

- 理解 ONNX（Open Neural Network Exchange）格式及其优势
- 将 PyTorch 模型转换为 ONNX 格式
- 将 TensorFlow 模型转换为 ONNX 格式
- 使用 ONNX Runtime 运行推理
- 为不同部署场景选择合适的模型格式
- 应用模型优化技术（量化、剪枝）
- 打包模型用于生产部署
- 理解模型版本管理和策略

**时长**: 6-8 小时
**难度**: 中级
**前置要求**: 讲座 01-03（ML 概述、PyTorch、TensorFlow）

---

## 1. ONNX 简介

### 什么是 ONNX？

**ONNX**（Open Neural Network Exchange）是一种开放的机器学习模型表示格式。它实现了不同 ML 框架之间的互操作性。

**关键概念**：在任何框架中训练，在任何地方部署。

```
PyTorch 模型 ─┐
              ├──→ ONNX 格式 ──→ ONNX Runtime ──→ 生产环境
TensorFlow 模型 ─┘
```

### 为什么 ONNX 对基础设施很重要

**没有 ONNX 的问题**：
```
数据科学家使用 PyTorch
  ↓
基础设施团队只支持 TensorFlow Serving
  ↓
需要手动重写或复杂的变通方案
```

**有了 ONNX 的解决方案**：
```
数据科学家使用任意框架
  ↓
转换为 ONNX
  ↓
使用 ONNX Runtime 部署（与框架无关）
```

### ONNX 的优势

| 优势 | 描述 | 影响 |
|---------|-------------|--------|
| **互操作性** | 与框架无关 | 无供应商锁定 |
| **性能** | 优化运行时 | 更快的推理 |
| **可移植性** | 跨平台运行 | 从边缘到云端 |
| **标准化** | 通用格式 | 更容易的工具支持 |
| **优化** | 内置优化 | 更好的资源利用 |

### ONNX 生态系统

```
┌─────────────────────────────────────────────────┐
│                   ONNX 格式                      │
└─────────────────────────────────────────────────┘
         ↑                              ↓
    ┌────────┐                    ┌──────────┐
    │ 导出   │                    │  部署    │
    └────────┘                    └──────────┘
         ↑                              ↓
┌────────────────────┐          ┌──────────────────┐
│  PyTorch           │          │ ONNX Runtime     │
│  TensorFlow        │          │ - Python         │
│  scikit-learn      │          │ - C++            │
│  Keras             │          │ - JavaScript     │
│  MXNet             │          │ - Java           │
└────────────────────┘          │ - C#             │
                                └──────────────────┘
```

---

## 2. 将 PyTorch 模型转换为 ONNX

### 基本转换

```python
import torch
import torch.onnx

# 加载 PyTorch 模型
model = torch.hub.load('pytorch/vision', 'resnet18', pretrained=True)
model.eval()

# 创建虚拟输入（必须匹配模型的预期输入）
dummy_input = torch.randn(1, 3, 224, 224)

# 导出为 ONNX
torch.onnx.export(
    model,                      # 要导出的模型
    dummy_input,                # 示例输入
    "resnet18.onnx",           # 输出文件路径
    export_params=True,         # 存储训练后的参数权重
    opset_version=13,          # ONNX 版本
    do_constant_folding=True,   # 优化常量折叠
    input_names=['input'],      # 模型输入名称
    output_names=['output'],    # 模型输出名称
    dynamic_axes={              # 可变长度轴
        'input': {0: 'batch_size'},
        'output': {0: 'batch_size'}
    }
)

print("Model exported to resnet18.onnx")
```

### 理解导出参数

```python
torch.onnx.export(
    model,
    dummy_input,
    "model.onnx",

    # 导出参数
    export_params=True,         # 包含权重（推理时始终为 True）

    # ONNX 版本（opset）
    opset_version=13,          # 11-15 普遍支持，13 推荐

    # 优化
    do_constant_folding=True,   # 在导出时折叠常量

    # 输入/输出配置
    input_names=['images'],
    output_names=['predictions'],

    # 动态形状（生产环境中很重要！）
    dynamic_axes={
        'images': {0: 'batch_size'},      # batch 维度是动态的
        'predictions': {0: 'batch_size'}
    },

    # 详细输出
    verbose=False,              # 调试时设为 True

    # 训练模式
    training=torch.onnx.TrainingMode.EVAL  # 推理模型始终为 EVAL
)
```

### 进阶：转换自定义模型

```python
import torch
import torch.nn as nn

# 定义自定义模型
class CustomModel(nn.Module):
    def __init__(self):
        super(CustomModel, self).__init__()
        self.conv1 = nn.Conv2d(3, 64, kernel_size=3, padding=1)
        self.relu = nn.ReLU()
        self.pool = nn.MaxPool2d(2, 2)
        self.fc = nn.Linear(64 * 112 * 112, 10)

    def forward(self, x):
        x = self.conv1(x)
        x = self.relu(x)
        x = self.pool(x)
        x = x.view(x.size(0), -1)
        x = self.fc(x)
        return x

# 创建并导出
model = CustomModel()
model.eval()

dummy_input = torch.randn(1, 3, 224, 224)

torch.onnx.export(
    model,
    dummy_input,
    "custom_model.onnx",
    export_params=True,
    opset_version=13,
    input_names=['input'],
    output_names=['output'],
    dynamic_axes={'input': {0: 'batch_size'}, 'output': {0: 'batch_size'}}
)

print("Custom model exported successfully")
```

### 验证 ONNX 导出

```python
import onnx

# 加载 ONNX 模型
onnx_model = onnx.load("resnet18.onnx")

# 检查模型格式是否正确
onnx.checker.check_model(onnx_model)

# 打印模型图信息
print("Model Inputs:")
for input in onnx_model.graph.input:
    print(f"  Name: {input.name}")
    print(f"  Shape: {[d.dim_value if d.dim_value > 0 else 'dynamic' for d in input.type.tensor_type.shape.dim]}")

print("\nModel Outputs:")
for output in onnx_model.graph.output:
    print(f"  Name: {output.name}")
    print(f"  Shape: {[d.dim_value if d.dim_value > 0 else 'dynamic' for d in output.type.tensor_type.shape.dim]}")

# 获取模型元数据
print(f"\nONNX Opset Version: {onnx_model.opset_import[0].version}")
print(f"Producer: {onnx_model.producer_name}")
```

---

## 3. 将 TensorFlow 模型转换为 ONNX

### 使用 tf2onnx

```bash
# 安装 tf2onnx
pip install tf2onnx
```

```python
import tensorflow as tf
import tf2onnx
import onnx

# 加载 TensorFlow 模型
model = tf.keras.applications.MobileNetV2(weights='imagenet')

# 获取输入签名
spec = (tf.TensorSpec((None, 224, 224, 3), tf.float32, name="input"),)

# 转换为 ONNX
model_proto, _ = tf2onnx.convert.from_keras(
    model,
    input_signature=spec,
    opset=13,
    output_path="mobilenet_v2.onnx"
)

print("Model converted to ONNX")
```

### 从 SavedModel 转换

```python
import tf2onnx

# 将 SavedModel 转换为 ONNX
# 这是 TensorFlow 最可靠的方法
!python -m tf2onnx.convert \
    --saved-model tensorflow_model/ \
    --output model.onnx \
    --opset 13

# 或在 Python 中
model_proto, external_tensor_storage = tf2onnx.convert.from_saved_model(
    'tensorflow_model/',
    output_path='model.onnx',
    opset=13
)
```

### 处理自定义 TensorFlow 操作

```python
import tensorflow as tf
import tf2onnx

# 某些 TensorFlow 操作可能没有 ONNX 等效操作
# 检查支持的操作：https://github.com/onnx/tensorflow-onnx/blob/master/support_status.md

# 如果转换失败，尝试：
model_proto, _ = tf2onnx.convert.from_keras(
    model,
    input_signature=spec,
    opset=13,
    output_path="model.onnx",
    # 如果需要，自定义操作处理器
    custom_ops={
        'CustomOp': custom_op_handler
    }
)
```

---

## 4. 使用 ONNX Runtime 运行推理

### 安装

```bash
# CPU 版本
pip install onnxruntime

# GPU 版本
pip install onnxruntime-gpu
```

### 基本推理

```python
import onnxruntime as ort
import numpy as np

# 加载 ONNX 模型
session = ort.InferenceSession("resnet18.onnx")

# 获取模型元数据
input_name = session.get_inputs()[0].name
output_name = session.get_outputs()[0].name

print(f"Input name: {input_name}")
print(f"Input shape: {session.get_inputs()[0].shape}")
print(f"Input type: {session.get_inputs()[0].type}")

# 准备输入
input_data = np.random.randn(1, 3, 224, 224).astype(np.float32)

# 运行推理
outputs = session.run([output_name], {input_name: input_data})

# 获取结果
predictions = outputs[0]
print(f"Output shape: {predictions.shape}")
```

### 生产推理类

```python
import onnxruntime as ort
import numpy as np
from typing import Dict, List

class ONNXModel:
    def __init__(self, model_path: str, providers: List[str] = None):
        """
        初始化 ONNX 模型

        Args:
            model_path: ONNX 模型文件路径
            providers: 执行提供者（如 ['CUDAExecutionProvider', 'CPUExecutionProvider']）
        """
        if providers is None:
            # 如果可用则使用 GPU，否则使用 CPU
            providers = ['CUDAExecutionProvider', 'CPUExecutionProvider'] if ort.get_device() == 'GPU' else ['CPUExecutionProvider']

        self.session = ort.InferenceSession(model_path, providers=providers)

        # 存储输入/输出元数据
        self.input_name = self.session.get_inputs()[0].name
        self.output_name = self.session.get_outputs()[0].name
        self.input_shape = self.session.get_inputs()[0].shape

        print(f"Model loaded with {providers}")
        print(f"Input: {self.input_name} {self.input_shape}")
        print(f"Output: {self.output_name}")

    def predict(self, input_data: np.ndarray) -> np.ndarray:
        """对输入数据运行推理"""
        # 确保正确的 dtype
        if input_data.dtype != np.float32:
            input_data = input_data.astype(np.float32)

        # 运行推理
        outputs = self.session.run(
            [self.output_name],
            {self.input_name: input_data}
        )

        return outputs[0]

    def predict_batch(self, batch: List[np.ndarray]) -> np.ndarray:
        """对批量输入运行推理"""
        # 堆叠输入
        batch_array = np.stack(batch)
        return self.predict(batch_array)

# 使用
model = ONNXModel("resnet18.onnx")
input_data = np.random.randn(1, 3, 224, 224).astype(np.float32)
output = model.predict(input_data)
print(f"Prediction shape: {output.shape}")
```

### 性能对比

```python
import torch
import onnxruntime as ort
import numpy as np
import time

# 加载 PyTorch 模型
torch_model = torch.hub.load('pytorch/vision', 'resnet18', pretrained=True)
torch_model.eval()

# 加载 ONNX 模型
onnx_session = ort.InferenceSession("resnet18.onnx")

# 准备输入
input_data = np.random.randn(1, 3, 224, 224).astype(np.float32)
torch_input = torch.from_numpy(input_data)

# PyTorch 基准测试
iterations = 100
start = time.time()
with torch.no_grad():
    for _ in range(iterations):
        _ = torch_model(torch_input)
pytorch_time = (time.time() - start) / iterations

# ONNX Runtime 基准测试
start = time.time()
for _ in range(iterations):
    _ = onnx_session.run(None, {"input": input_data})
onnx_time = (time.time() - start) / iterations

print(f"PyTorch inference time: {pytorch_time*1000:.2f} ms")
print(f"ONNX Runtime inference time: {onnx_time*1000:.2f} ms")
print(f"Speedup: {pytorch_time/onnx_time:.2f}x")
```

**典型结果（CPU）**：
```
PyTorch inference time: 45.23 ms
ONNX Runtime inference time: 32.18 ms
Speedup: 1.41x
```

---

## 5. 模型优化技术

### 量化：减小模型大小

**什么是量化？**
将模型权重从 float32（32 位）转换为 int8（8 位），大小减少约 4 倍。

```python
import torch

# 加载模型
model = torch.hub.load('pytorch/vision', 'resnet18', pretrained=True)
model.eval()

# 动态量化（最简单，仅 CPU）
quantized_model = torch.quantization.quantize_dynamic(
    model,
    {torch.nn.Linear, torch.nn.Conv2d},
    dtype=torch.qint8
)

# 保存量化模型
torch.save(quantized_model.state_dict(), "resnet18_quantized.pth")

# 比较大小
import os
original_size = os.path.getsize("resnet18.pth") / (1024 ** 2)
quantized_size = os.path.getsize("resnet18_quantized.pth") / (1024 ** 2)

print(f"Original size: {original_size:.2f} MB")
print(f"Quantized size: {quantized_size:.2f} MB")
print(f"Size reduction: {(1 - quantized_size/original_size)*100:.1f}%")
```

### ONNX 量化

```python
from onnxruntime.quantization import quantize_dynamic, QuantType

# 量化 ONNX 模型
quantize_dynamic(
    model_input="resnet18.onnx",
    model_output="resnet18_quantized.onnx",
    weight_type=QuantType.QUInt8  # 或 QInt8
)

print("Model quantized successfully")

# 比较大小
import os
original = os.path.getsize("resnet18.onnx") / (1024 ** 2)
quantized = os.path.getsize("resnet18_quantized.onnx") / (1024 ** 2)

print(f"Original: {original:.2f} MB")
print(f"Quantized: {quantized:.2f} MB")
print(f"Reduction: {(1 - quantized/original)*100:.1f}%")
```

### 模型剪枝（移除不必要的权重）

```python
import torch
import torch.nn.utils.prune as prune

# 加载模型
model = torch.hub.load('pytorch/vision', 'resnet18', pretrained=True)

# 在卷积层中剪枝 30% 的权重
for name, module in model.named_modules():
    if isinstance(module, torch.nn.Conv2d):
        prune.l1_unstructured(module, name='weight', amount=0.3)

# 使剪枝永久化
for name, module in model.named_modules():
    if isinstance(module, torch.nn.Conv2d):
        prune.remove(module, 'weight')

print("Model pruned (30% of conv weights removed)")

# 准确率会降低——可能需要微调
```

### 知识蒸馏（进阶）

**概念**：训练一个较小的"学生"模型来模仿较大的"教师"模型。

```python
# 伪代码——需要训练
# 1. 加载大型教师模型
teacher = load_large_model()

# 2. 创建较小的学生模型
student = create_small_model()

# 3. 训练学生匹配教师的输出
for data, labels in dataloader:
    teacher_predictions = teacher(data)
    student_predictions = student(data)

    # 损失包含匹配教师和真实标签
    loss = distillation_loss(student_predictions, teacher_predictions, labels)
    loss.backward()
    optimizer.step()

# 结果：更小、更快的模型，具有相似的准确率
```

---

## 6. 选择合适的模型格式

### 决策矩阵

```
                                         ┌─────────────┐
                                         │   部署目标  │
                                         └──────┬──────┘
                                                │
                     ┌──────────────────────────┼──────────────────────────┐
                     │                          │                          │
              ┌──────▼──────┐          ┌────────▼────────┐        ┌───────▼──────┐
              │    服务器    │          │   移动/边缘     │        │   浏览器     │
              └──────┬──────┘          └────────┬────────┘        └───────┬──────┘
                     │                          │                          │
         ┌───────────┼───────────┐              │                          │
         │           │           │              │                          │
    ┌────▼───┐  ┌───▼────┐  ┌───▼────┐   ┌─────▼─────┐            ┌──────▼──────┐
    │PyTorch │  │TensorFlow│ │  ONNX  │   │  TFLite   │            │TensorFlow.js│
    │TorchSrv│  │ Serving │  │ Runtime│   │  ONNX     │            │   ONNX.js   │
    └────────┘  └─────────┘  └────────┘   └───────────┘            └─────────────┘
```

### 格式选择指南

| 场景 | 推荐格式 | 原因 |
|----------|-------------------|---------|
| **PyTorch 原生服务** | PyTorch + TorchServe | 原生支持，无需转换 |
| **TensorFlow 原生服务** | SavedModel + TF Serving | 成熟的生产工具 |
| **与框架无关** | ONNX + ONNX Runtime | 最佳的互操作性 |
| **移动端（iOS/Android）** | TFLite 或 ONNX | 针对移动端优化 |
| **边缘设备** | TFLite 或 ONNX | 体积小，推理快 |
| **浏览器** | TensorFlow.js 或 ONNX.js | JavaScript 运行时 |
| **多框架支持** | ONNX | 单一服务基础设施 |
| **最大性能** | 原生格式 + 优化 | 框架特定的优化 |

### 生产建议

**场景 1：单一框架组织**
```
如果数据科学家只使用 PyTorch 或只使用 TensorFlow：
→ 使用原生格式和服务工具
→ 无需 ONNX 转换
→ 更简单的基础设施
```

**场景 2：多框架组织**
```
如果数据科学家同时使用 PyTorch 和 TensorFlow：
→ 标准化使用 ONNX 进行部署
→ 将所有模型转换为 ONNX
→ 使用 ONNX Runtime 进行服务
→ 单一服务基础设施
```

**场景 3：混合方法**
```
为了最大灵活性：
→ 同时支持原生和 ONNX 格式
→ 尽可能使用原生（更好的性能）
→ 对于与框架无关的部署使用 ONNX
→ 构建能处理两者的基础设施
```

---

## 7. 模型打包用于生产

### 创建模型包

```python
# model_package.py
import json
from pathlib import Path
import shutil
from datetime import datetime

class ModelPackage:
    """将模型与元数据打包用于生产部署"""

    def __init__(self, name: str, version: str):
        self.name = name
        self.version = version
        self.package_dir = Path(f"{name}_v{version}")

    def create(self, model_path: str, metadata: dict):
        """创建模型包"""
        # 创建目录结构
        self.package_dir.mkdir(exist_ok=True)
        (self.package_dir / "model").mkdir(exist_ok=True)
        (self.package_dir / "metadata").mkdir(exist_ok=True)

        # 复制模型文件
        if Path(model_path).is_file():
            shutil.copy(model_path, self.package_dir / "model" / Path(model_path).name)
        else:
            shutil.copytree(model_path, self.package_dir / "model" / Path(model_path).name)

        # 创建元数据
        full_metadata = {
            'name': self.name,
            'version': self.version,
            'created_at': datetime.now().isoformat(),
            'model_file': Path(model_path).name,
            **metadata
        }

        # 保存元数据
        with open(self.package_dir / "metadata" / "model.json", 'w') as f:
            json.dump(full_metadata, f, indent=2)

        # 创建 requirements.txt
        self._create_requirements(metadata.get('framework', 'pytorch'))

        # 创建 README
        self._create_readme(full_metadata)

        print(f"Model package created: {self.package_dir}")

    def _create_requirements(self, framework: str):
        """创建 requirements.txt"""
        requirements = {
            'pytorch': ['torch>=2.0.0', 'torchvision>=0.15.0'],
            'tensorflow': ['tensorflow>=2.13.0'],
            'onnx': ['onnxruntime>=1.15.0', 'onnx>=1.14.0']
        }

        with open(self.package_dir / "requirements.txt", 'w') as f:
            f.write('\n'.join(requirements.get(framework, [])))

    def _create_readme(self, metadata: dict):
        """创建 README.md"""
        readme = f"""# {metadata['name']} v{metadata['version']}

## 模型信息

- **模型**: {metadata['name']}
- **版本**: {metadata['version']}
- **创建时间**: {metadata['created_at']}
- **框架**: {metadata.get('framework', 'N/A')}
- **任务**: {metadata.get('task', 'N/A')}

## 使用方式

```python
# 加载模型
from model_loader import load_model
model = load_model('model/{metadata['model_file']}')

# 运行推理
output = model.predict(input_data)
```

## 元数据

```json
{json.dumps(metadata, indent=2)}
```
"""
        with open(self.package_dir / "README.md", 'w') as f:
            f.write(readme)

# 使用
packager = ModelPackage("resnet18", "1.0.0")
packager.create(
    "resnet18.onnx",
    metadata={
        'framework': 'onnx',
        'task': 'image_classification',
        'input_shape': [1, 3, 224, 224],
        'output_shape': [1, 1000],
        'accuracy': 0.697,
        'training_dataset': 'ImageNet',
        'author': 'Infrastructure Team'
    }
)
```

### 用于模型服务的 Docker 镜像

```dockerfile
# Dockerfile
FROM python:3.11-slim

# 安装依赖
RUN pip install onnxruntime fastapi uvicorn numpy pillow

# 创建应用目录
WORKDIR /app

# 复制模型
COPY resnet18.onnx /app/model.onnx

# 复制服务代码
COPY serve.py /app/

# 暴露端口
EXPOSE 8000

# 运行服务器
CMD ["python", "serve.py"]
```

```python
# serve.py
from fastapi import FastAPI, File, UploadFile
from fastapi.responses import JSONResponse
import onnxruntime as ort
import numpy as np
from PIL import Image
import io

app = FastAPI()

# 启动时加载模型
session = ort.InferenceSession("model.onnx")
input_name = session.get_inputs()[0].name

def preprocess_image(image_bytes):
    """预处理图像以供模型使用"""
    image = Image.open(io.BytesIO(image_bytes))
    image = image.resize((224, 224))
    image_array = np.array(image).astype(np.float32) / 255.0
    image_array = np.transpose(image_array, (2, 0, 1))  # HWC -> CHW
    image_array = np.expand_dims(image_array, axis=0)   # 添加 batch 维度
    return image_array

@app.get("/health")
def health():
    return {"status": "healthy"}

@app.post("/predict")
async def predict(file: UploadFile = File(...)):
    # 读取图像
    image_bytes = await file.read()

    # 预处理
    input_data = preprocess_image(image_bytes)

    # 运行推理
    outputs = session.run(None, {input_name: input_data})

    # 获取 top 预测
    predictions = outputs[0][0]
    top_idx = int(np.argmax(predictions))
    confidence = float(predictions[top_idx])

    return JSONResponse({
        "prediction": top_idx,
        "confidence": confidence
    })

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## 8. 模型版本管理

### 版本管理策略

```python
# model_version.py
from typing import Dict
from pathlib import Path
import json

class ModelVersionManager:
    """管理模型版本"""

    def __init__(self, base_path: str = "./models"):
        self.base_path = Path(base_path)
        self.base_path.mkdir(exist_ok=True)

    def register_model(self, name: str, version: str, model_path: str, metadata: Dict):
        """注册新模型版本"""
        # 创建版本目录
        version_dir = self.base_path / name / version
        version_dir.mkdir(parents=True, exist_ok=True)

        # 复制模型
        import shutil
        shutil.copy(model_path, version_dir / "model.onnx")

        # 保存元数据
        with open(version_dir / "metadata.json", 'w') as f:
            json.dump(metadata, f, indent=2)

        print(f"Registered {name} v{version}")

    def list_versions(self, name: str):
        """列出模型的所有版本"""
        model_dir = self.base_path / name
        if not model_dir.exists():
            return []

        versions = []
        for version_dir in sorted(model_dir.iterdir()):
            if version_dir.is_dir():
                metadata_file = version_dir / "metadata.json"
                if metadata_file.exists():
                    with open(metadata_file) as f:
                        metadata = json.load(f)
                    versions.append({
                        'version': version_dir.name,
                        'metadata': metadata
                    })

        return versions

    def get_latest_version(self, name: str):
        """获取模型的最新版本"""
        versions = self.list_versions(name)
        if not versions:
            return None

        # 假设语义化版本（如 1.0.0, 1.1.0, 2.0.0）
        latest = sorted(versions, key=lambda x: x['version'], reverse=True)[0]
        return latest

# 使用
manager = ModelVersionManager()

# 注册模型 v1
manager.register_model(
    "resnet18",
    "1.0.0",
    "resnet18_v1.onnx",
    {'accuracy': 0.695, 'size_mb': 44.7}
)

# 注册模型 v2（改进版）
manager.register_model(
    "resnet18",
    "1.1.0",
    "resnet18_v2.onnx",
    {'accuracy': 0.705, 'size_mb': 42.3}
)

# 列出版本
versions = manager.list_versions("resnet18")
for v in versions:
    print(f"Version {v['version']}: accuracy={v['metadata']['accuracy']}")

# 获取最新版本
latest = manager.get_latest_version("resnet18")
print(f"Latest version: {latest['version']}")
```

### A/B 测试基础设施

```python
import random
from typing import Dict

class ABTestRouter:
    """将请求路由到不同模型版本进行 A/B 测试"""

    def __init__(self):
        self.models: Dict[str, Dict] = {}

    def register_variant(self, variant_name: str, model_path: str, traffic_percentage: float):
        """注册用于 A/B 测试的模型变体"""
        self.models[variant_name] = {
            'model_path': model_path,
            'traffic': traffic_percentage,
            'requests': 0,
            'total_latency': 0.0
        }

    def route_request(self):
        """根据流量拆分将请求路由到一个变体"""
        # 基于流量百分比的简单随机路由
        rand = random.random() * 100
        cumulative = 0

        for variant_name, config in self.models.items():
            cumulative += config['traffic']
            if rand <= cumulative:
                return variant_name

        # 回退到第一个变体
        return list(self.models.keys())[0]

    def log_metrics(self, variant_name: str, latency: float):
        """记录指标用于分析"""
        self.models[variant_name]['requests'] += 1
        self.models[variant_name]['total_latency'] += latency

    def get_stats(self):
        """获取性能统计"""
        stats = {}
        for variant, config in self.models.items():
            avg_latency = (config['total_latency'] / config['requests']
                          if config['requests'] > 0 else 0)
            stats[variant] = {
                'requests': config['requests'],
                'avg_latency_ms': avg_latency * 1000,
                'traffic_allocation': config['traffic']
            }
        return stats

# 使用
router = ABTestRouter()
router.register_variant("model_v1", "resnet18_v1.onnx", traffic_percentage=50)
router.register_variant("model_v2", "resnet18_v2.onnx", traffic_percentage=50)

# 模拟请求
for _ in range(1000):
    variant = router.route_request()
    # ... 运行推理 ...
    latency = 0.05  # 示例延迟
    router.log_metrics(variant, latency)

# 检查统计
stats = router.get_stats()
print("A/B Test Results:")
for variant, metrics in stats.items():
    print(f"{variant}: {metrics['requests']} requests, {metrics['avg_latency_ms']:.2f}ms avg")
```

---

## 关键要点

### 对于基础设施工程师

1. **ONNX 实现互操作性**：在任何地方训练，在任何地方部署
2. **存在多种格式**：根据部署目标选择
3. **优化很重要**：量化可以将大小减少 4 倍
4. **版本管理很重要**：始终为模型设置版本
5. **正确打包模型**：包含元数据和文档
6. **A/B 测试至关重要**：在生产环境中比较模型版本
7. **ONNX Runtime 很快**：通常优于原生框架

### 生产清单

- [ ] 模型导出为适当格式（ONNX, SavedModel 等）
- [ ] 模型已验证（准确率与原始匹配）
- [ ] 已应用优化（适当时进行量化）
- [ ] 模型已打包并附带元数据
- [ ] 已分配版本号
- [ ] Docker 镜像已构建和测试
- [ ] 服务端点已实现
- [ ] 健康检查已配置
- [ ] 监控和日志已添加
- [ ] 文档已完成

---

## 快速参考

### ONNX 转换

```python
# PyTorch → ONNX
torch.onnx.export(model, dummy_input, "model.onnx", opset_version=13)

# TensorFlow → ONNX
python -m tf2onnx.convert --saved-model model/ --output model.onnx

# ONNX 推理
import onnxruntime as ort
session = ort.InferenceSession("model.onnx")
outputs = session.run(None, {input_name: input_data})
```

### 格式对比

| 格式 | 最适合 | 大小 | 速度 |
|--------|----------|------|-------|
| PyTorch (pth) | PyTorch 开发 | 中等 | 快 |
| TensorFlow (SavedModel) | TensorFlow Serving | 大 | 快 |
| ONNX | 跨框架 | 中等 | 非常快 |
| TFLite | 移动/边缘 | 小 | 非常快 |

---

## 下一步

你已经完成了 Module 004 的所有讲座！现在该：

1. **完成练习**：练习转换和部署模型
2. **参加测验**：测试你的理解
3. **构建项目**：创建端到端模型服务系统

继续学习 `exercises/exercise-01-pytorch-inference.md`

---

**讲座版本**: 1.0
**最后更新**: 2025 年 10 月
**预计完成时间**: 6-8 小时
**难度**: 中级
