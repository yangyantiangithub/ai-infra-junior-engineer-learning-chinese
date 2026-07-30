# 练习 03：使用 ONNX 进行模型格式转换

## 练习概述

**目标**：将 PyTorch 和 TensorFlow 模型转换为 ONNX 格式，使用 ONNX Runtime 运行推理，并比较原生框架与 ONNX 之间的性能。

**难度**：初级-中级
**预计时间**：3-4 小时
**前置要求**：讲座 02（PyTorch 基础）和 03（TensorFlow 基础），练习 01

**你将学到**：
- 将 PyTorch 模型转换为 ONNX 格式
- 将 TensorFlow 模型转换为 ONNX 格式
- 使用 ONNX Runtime 运行推理
- 验证模型转换的准确性
- 跨框架比较性能
- 理解何时以及为何使用 ONNX

---

## 学习目标

到本练习结束时，你将能够：

✅ 使用 `torch.onnx.export()` 将 PyTorch 模型导出为 ONNX
✅ 使用 `tf2onnx` 将 TensorFlow 模型转换为 ONNX
✅ 使用 ONNX Runtime 运行推理
✅ 验证转换后的模型的数值精度
✅ 跨格式测量和比较性能
✅ 理解 ONNX 在 AI 基础设施中的角色
✅ 对何时使用 ONNX 做出明智的决策

---

## 背景：为什么基础设施工程师关心 ONNX

### 什么是 ONNX？

**ONNX（Open Neural Network Exchange）** 是一种开放的机器学习模型表示标准格式。可以将其视为 ML 模型的"通用翻译器"。

### 为什么基础设施工程师关心 ONNX

1. **框架无关性**：部署模型无需框架依赖
2. **优化**：ONNX Runtime 提供硬件特定的优化
3. **互操作性**：在 PyTorch 中训练，用 TensorFlow Serving 部署，反之亦然
4. **性能**：通常比原生框架更快，尤其是推理
5. **部署灵活性**：一种模型格式用于多个部署目标
6. **减少依赖**：部署占用更小，无需完整的训练框架

### 实际用例

- **边缘部署**：为 IoT 设备减少模型大小和依赖占用
- **多框架团队**：标准化部署，无论训练框架如何
- **性能优化**：利用 ONNX Runtime 的优化执行
- **云成本降低**：更快的推理 = 更低的计算成本
- **跨平台**：同一模型可在 Windows, Linux, ARM, 移动端, Web 上运行

---

## 第 1 部分：环境设置

### 步骤 1.1：安装依赖

```bash
# 创建虚拟环境
python -m venv onnx_env
source onnx_env/bin/activate  # Windows: onnx_env\Scripts\activate

# 安装 PyTorch 和 TensorFlow
pip install torch torchvision
pip install tensorflow

# 安装 ONNX 工具
pip install onnx onnxruntime
pip install tf2onnx
pip install onnxconverter-common

# 安装工具
pip install pillow numpy matplotlib
```

### 步骤 1.2：验证安装

```python
# verify_installation.py
import torch
import tensorflow as tf
import onnx
import onnxruntime as ort
import tf2onnx

print("=== Framework Versions ===")
print(f"PyTorch: {torch.__version__}")
print(f"TensorFlow: {tf.__version__}")
print(f"ONNX: {onnx.__version__}")
print(f"ONNX Runtime: {ort.__version__}")
print(f"tf2onnx: {tf2onnx.__version__}")

print("\n=== Device Information ===")
print(f"PyTorch CUDA available: {torch.cuda.is_available()}")
print(f"TensorFlow GPUs: {len(tf.config.list_physical_devices('GPU'))}")
print(f"ONNX Runtime providers: {ort.get_available_providers()}")
```

**预期输出**：
```
=== Framework Versions ===
PyTorch: 2.1.0
TensorFlow: 2.14.0
ONNX: 1.15.0
ONNX Runtime: 1.16.3
tf2onnx: 1.15.1

=== Device Information ===
PyTorch CUDA available: True
TensorFlow GPUs: 1
ONNX Runtime providers: ['CUDAExecutionProvider', 'CPUExecutionProvider']
```

### 步骤 1.3：下载测试资源

```python
# download_resources.py
import urllib.request
import os

# 创建目录
os.makedirs("models", exist_ok=True)
os.makedirs("test_images", exist_ok=True)

# 下载测试图像
image_url = "https://github.com/pytorch/hub/raw/master/images/dog.jpg"
urllib.request.urlretrieve(image_url, "test_images/dog.jpg")

# 下载 ImageNet 标签
labels_url = "https://raw.githubusercontent.com/pytorch/hub/master/imagenet_classes.txt"
urllib.request.urlretrieve(labels_url, "imagenet_classes.txt")

print("✅ Resources downloaded!")
print("  - test_images/dog.jpg")
print("  - imagenet_classes.txt")
print("  - models/ directory created")
```

✅ **检查点**：环境已设置，所有必要工具已安装。

---

## 第 2 部分：将 PyTorch 模型转换为 ONNX

### 步骤 2.1：导出简单 PyTorch 模型

```python
# pytorch_to_onnx_basic.py
import torch
import torch.nn as nn
import onnx

# 定义一个简单模型
class SimpleNet(nn.Module):
    def __init__(self):
        super(SimpleNet, self).__init__()
        self.fc1 = nn.Linear(10, 20)
        self.relu = nn.ReLU()
        self.fc2 = nn.Linear(20, 5)

    def forward(self, x):
        x = self.fc1(x)
        x = self.relu(x)
        x = self.fc2(x)
        return x

# 创建模型实例
model = SimpleNet()
model.eval()

# 创建与预期输入形状匹配的虚拟输入
dummy_input = torch.randn(1, 10)

# 导出为 ONNX
onnx_path = "models/simple_net.onnx"
torch.onnx.export(
    model,                          # 正在导出的模型
    dummy_input,                    # 用于追踪的示例输入
    onnx_path,                      # 保存 ONNX 文件的路径
    export_params=True,             # 存储训练后的参数
    opset_version=14,               # ONNX 版本
    do_constant_folding=True,       # 优化常量
    input_names=['input'],          # 输入 tensor 名称
    output_names=['output'],        # 输出 tensor 名称
    dynamic_axes={                  # 可变维度
        'input': {0: 'batch_size'},
        'output': {0: 'batch_size'}
    }
)

print(f"✅ Model exported to {onnx_path}")

# 验证 ONNX 模型
onnx_model = onnx.load(onnx_path)
onnx.checker.check_model(onnx_model)
print("✅ ONNX model is valid!")

# 打印模型信息
print(f"\nModel info:")
print(f"  Inputs: {[inp.name for inp in onnx_model.graph.input]}")
print(f"  Outputs: {[out.name for out in onnx_model.graph.output]}")
print(f"  Operators: {len(onnx_model.graph.node)}")
```

**预期输出**：
```
✅ Model exported to models/simple_net.onnx
✅ ONNX model is valid!

Model info:
  Inputs: ['input']
  Outputs: ['output']
  Operators: 3
```

### 步骤 2.2：理解导出参数

让我们理解 `torch.onnx.export()` 中的每个参数：

```python
# export_parameters_explained.py
import torch
import torch.nn as nn

model = nn.Sequential(
    nn.Linear(10, 20),
    nn.ReLU(),
    nn.Linear(20, 5)
)
model.eval()

dummy_input = torch.randn(1, 10)

torch.onnx.export(
    model,
    dummy_input,
    "models/explained.onnx",

    # export_params=True: 在 ONNX 文件中包含训练后的权重
    # 如果只需要模型结构，设为 False
    export_params=True,

    # opset_version: ONNX 算子集版本
    # 更高的版本支持更多操作，但可能支持面更窄
    # 推荐对现代模型使用版本 14+
    opset_version=14,

    # do_constant_folding=True: 在导出时优化常量表达式
    # 示例：2*3 直接在图中变为 6
    do_constant_folding=True,

    # input_names/output_names: tensor 的人类可读名称
    # 在加载模型时识别输入/输出至关重要
    input_names=['features'],
    output_names=['predictions'],

    # dynamic_axes: 允许可变维度
    # 维度 0（batch_size）可以在运行时变化
    dynamic_axes={
        'features': {0: 'batch_size'},
        'predictions': {0: 'batch_size'}
    },

    # verbose=True: 打印详细导出信息（用于调试）
    verbose=False
)

print("✅ Model exported with all parameters explained!")
```

### 步骤 2.3：将 ResNet50 导出为 ONNX

现在让我们导出一个真实的预训练模型：

```python
# pytorch_resnet_to_onnx.py
import torch
import torchvision.models as models
import onnx
import os

print("Loading ResNet50...")
model = models.resnet50(pretrained=True)
model.eval()

# 创建示例输入（ImageNet 尺寸）
batch_size = 1
dummy_input = torch.randn(batch_size, 3, 224, 224)

# 导出为 ONNX
onnx_path = "models/resnet50.onnx"
print(f"Exporting to {onnx_path}...")

torch.onnx.export(
    model,
    dummy_input,
    onnx_path,
    export_params=True,
    opset_version=14,
    do_constant_folding=True,
    input_names=['input'],
    output_names=['output'],
    dynamic_axes={
        'input': {0: 'batch_size'},
        'output': {0: 'batch_size'}
    }
)

print("✅ Export complete!")

# 验证 ONNX 模型
print("\nVerifying ONNX model...")
onnx_model = onnx.load(onnx_path)
onnx.checker.check_model(onnx_model)
print("✅ ONNX model is valid!")

# 模型统计
file_size_mb = os.path.getsize(onnx_path) / (1024 * 1024)
print(f"\nModel Statistics:")
print(f"  File size: {file_size_mb:.2f} MB")
print(f"  Graph nodes: {len(onnx_model.graph.node)}")
print(f"  Inputs: {[i.name for i in onnx_model.graph.input]}")
print(f"  Outputs: {[o.name for o in onnx_model.graph.output]}")
```

**预期输出**：
```
Loading ResNet50...
Exporting to models/resnet50.onnx...
✅ Export complete!

Verifying ONNX model...
✅ ONNX model is valid!

Model Statistics:
  File size: 97.53 MB
  Graph nodes: 176
  Inputs: ['input']
  Outputs: ['output']
```

✅ **检查点**：你能将 PyTorch 模型导出为 ONNX 格式了。

---

## 第 3 部分：将 TensorFlow 模型转换为 ONNX

### 步骤 3.1：将 Keras 模型导出为 ONNX

```python
# tensorflow_to_onnx_basic.py
import tensorflow as tf
import tf2onnx
import onnx

# 创建一个简单的 Keras 模型
model = tf.keras.Sequential([
    tf.keras.layers.Input(shape=(10,)),
    tf.keras.layers.Dense(20, activation='relu'),
    tf.keras.layers.Dense(5, activation='softmax')
])

print("Model created:")
model.summary()

# 转换为 ONNX
# 方法 1：直接从 Keras 模型转换
onnx_path = "models/keras_model.onnx"

spec = (tf.TensorSpec((None, 10), tf.float32, name="input"),)
model_proto, _ = tf2onnx.convert.from_keras(
    model,
    input_signature=spec,
    opset=14,
    output_path=onnx_path
)

print(f"\n✅ Model exported to {onnx_path}")

# 验证 ONNX 模型
onnx_model = onnx.load(onnx_path)
onnx.checker.check_model(onnx_model)
print("✅ ONNX model is valid!")
```

### 步骤 3.2：将 TensorFlow SavedModel 导出为 ONNX

```python
# tensorflow_savedmodel_to_onnx.py
import tensorflow as tf
import tf2onnx
import onnx
import tempfile
import os

print("Loading MobileNetV2...")
model = tf.keras.applications.MobileNetV2(
    weights='imagenet',
    input_shape=(224, 224, 3)
)

# 首先保存为 TensorFlow SavedModel
saved_model_dir = "models/mobilenet_saved"
print(f"Saving to {saved_model_dir}...")
model.save(saved_model_dir)

# 将 SavedModel 转换为 ONNX
onnx_path = "models/mobilenet_v2.onnx"
print(f"Converting to ONNX: {onnx_path}...")

os.system(f"python -m tf2onnx.convert \
    --saved-model {saved_model_dir} \
    --output {onnx_path} \
    --opset 14")

print("\n✅ Conversion complete!")

# 验证
onnx_model = onnx.load(onnx_path)
onnx.checker.check_model(onnx_model)
print("✅ ONNX model is valid!")

# 模型信息
file_size_mb = os.path.getsize(onnx_path) / (1024 * 1024)
print(f"\nModel Statistics:")
print(f"  File size: {file_size_mb:.2f} MB")
print(f"  Graph nodes: {len(onnx_model.graph.node)}")
```

### 步骤 3.3：替代转换方法

```python
# tensorflow_to_onnx_alternative.py
import tensorflow as tf
import tf2onnx
import numpy as np

# 加载模型
model = tf.keras.applications.MobileNetV2(
    weights='imagenet',
    input_shape=(224, 224, 3)
)

# 方法 2：使用 concrete function 转换
onnx_path = "models/mobilenet_v2_alt.onnx"

# 获取 concrete function
concrete_func = tf.function(lambda x: model(x))
concrete_func = concrete_func.get_concrete_function(
    tf.TensorSpec(model.inputs[0].shape, model.inputs[0].dtype)
)

# 转换为 ONNX
model_proto, _ = tf2onnx.convert.from_function(
    concrete_func,
    input_signature=[tf.TensorSpec([None, 224, 224, 3], tf.float32, name="input")],
    opset=14,
    output_path=onnx_path
)

print(f"✅ Model exported to {onnx_path}")
```

✅ **检查点**：你能将 PyTorch 和 TensorFlow 模型都转换为 ONNX 了。

---

## 第 4 部分：使用 ONNX Runtime 运行推理

### 步骤 4.1：基本 ONNX Runtime 推理

```python
# onnx_runtime_basic.py
import onnxruntime as ort
import numpy as np

# 加载 ONNX 模型
print("Loading ONNX model...")
onnx_path = "models/simple_net.onnx"
session = ort.InferenceSession(onnx_path)

# 获取模型元数据
print("\n=== Model Information ===")
print(f"Inputs:")
for input_meta in session.get_inputs():
    print(f"  Name: {input_meta.name}")
    print(f"  Shape: {input_meta.shape}")
    print(f"  Type: {input_meta.type}")

print(f"\nOutputs:")
for output_meta in session.get_outputs():
    print(f"  Name: {output_meta.name}")
    print(f"  Shape: {output_meta.shape}")
    print(f"  Type: {output_meta.type}")

# 准备输入
input_name = session.get_inputs()[0].name
input_data = np.random.randn(1, 10).astype(np.float32)

# 运行推理
print("\n=== Running Inference ===")
outputs = session.run(None, {input_name: input_data})

print(f"Output shape: {outputs[0].shape}")
print(f"Output values: {outputs[0]}")
```

**预期输出**：
```
Loading ONNX model...

=== Model Information ===
Inputs:
  Name: input
  Shape: ['batch_size', 10]
  Type: tensor(float)

Outputs:
  Name: output
  Shape: ['batch_size', 5]
  Type: tensor(float)

=== Running Inference ===
Output shape: (1, 5)
Output values: [[-0.123  0.456 -0.789  0.234 -0.567]]
```

### 步骤 4.2：使用 ONNX Runtime 进行 ResNet50 推理

```python
# onnx_resnet_inference.py
import onnxruntime as ort
import numpy as np
from PIL import Image
import urllib.request

# 加载 ImageNet 标签
labels_url = "https://raw.githubusercontent.com/pytorch/hub/master/imagenet_classes.txt"
urllib.request.urlretrieve(labels_url, "imagenet_classes.txt")
with open("imagenet_classes.txt") as f:
    labels = [line.strip() for line in f.readlines()]

# 加载 ONNX 模型
print("Loading ONNX ResNet50...")
session = ort.InferenceSession("models/resnet50.onnx")

# 获取输入名称
input_name = session.get_inputs()[0].name
output_name = session.get_outputs()[0].name

print(f"Input name: {input_name}")
print(f"Output name: {output_name}")

# 加载并预处理图像
def preprocess_image(image_path):
    """为 ResNet50 预处理图像"""
    from PIL import Image
    import numpy as np

    # 加载图像
    image = Image.open(image_path).convert('RGB')

    # 调整大小并中心裁剪
    image = image.resize((256, 256))
    left = (256 - 224) / 2
    top = (256 - 224) / 2
    right = left + 224
    bottom = top + 224
    image = image.crop((left, top, right, bottom))

    # 转换为数组并归一化
    img_array = np.array(image).astype(np.float32) / 255.0

    # 使用 ImageNet 统计数据归一化
    mean = np.array([0.485, 0.456, 0.406])
    std = np.array([0.229, 0.224, 0.225])
    img_array = (img_array - mean) / std

    # 转置为 CHW 格式并添加 batch 维度
    img_array = img_array.transpose(2, 0, 1)
    img_array = np.expand_dims(img_array, axis=0)

    return img_array

# 预处理图像
image_path = "test_images/dog.jpg"
print(f"\nPreprocessing {image_path}...")
input_data = preprocess_image(image_path)
print(f"Input shape: {input_data.shape}")
print(f"Input dtype: {input_data.dtype}")

# 运行推理
print("\nRunning inference...")
outputs = session.run([output_name], {input_name: input_data})
predictions = outputs[0][0]

# 获取 top 5 预测
top5_indices = np.argsort(predictions)[-5:][::-1]

# 应用 softmax 获取概率
exp_predictions = np.exp(predictions - np.max(predictions))
probabilities = exp_predictions / exp_predictions.sum()

print("\nTop 5 Predictions:")
for i, idx in enumerate(top5_indices):
    print(f"  {i+1}. {labels[idx]:<30} {probabilities[idx]*100:>6.2f}%")
```

**预期输出**：
```
Loading ONNX ResNet50...
Input name: input
Output name: output

Preprocessing test_images/dog.jpg...
Input shape: (1, 3, 224, 224)
Input dtype: float32

Running inference...

Top 5 Predictions:
  1. Samoyed                      87.34%
  2. Pomeranian                    5.23%
  3. white wolf                    3.45%
  4. keeshond                      2.11%
  5. Great Pyrenees                1.34%
```

### 步骤 4.3：选择执行提供者

ONNX Runtime 支持多种执行提供者以实现硬件加速：

```python
# onnx_execution_providers.py
import onnxruntime as ort
import numpy as np

# 列出可用的提供者
print("Available execution providers:")
for provider in ort.get_available_providers():
    print(f"  - {provider}")

# 使用不同提供者创建会话
onnx_path = "models/resnet50.onnx"

# CPU Provider
print("\n=== CPU Execution Provider ===")
cpu_session = ort.InferenceSession(
    onnx_path,
    providers=['CPUExecutionProvider']
)
print(f"Using: {cpu_session.get_providers()}")

# CUDA Provider（如果可用）
if 'CUDAExecutionProvider' in ort.get_available_providers():
    print("\n=== CUDA Execution Provider ===")
    cuda_session = ort.InferenceSession(
        onnx_path,
        providers=['CUDAExecutionProvider', 'CPUExecutionProvider']
    )
    print(f"Using: {cuda_session.get_providers()}")
else:
    print("\n⚠ CUDA not available")

# 其他提供者（如果可用）
# - TensorrtExecutionProvider: NVIDIA TensorRT
# - OpenVINOExecutionProvider: Intel OpenVINO
# - CoreMLExecutionProvider: Apple CoreML
# - DmlExecutionProvider: DirectML (Windows)
```

✅ **检查点**：你能使用 ONNX Runtime 在转换后的模型上运行推理了。

---

## 第 5 部分：模型验证

### 步骤 5.1：验证 PyTorch 到 ONNX 的转换

```python
# validate_pytorch_onnx.py
import torch
import torch.nn as nn
import onnxruntime as ort
import numpy as np

# 加载 PyTorch 模型
print("Loading PyTorch ResNet50...")
pytorch_model = torch.hub.load('pytorch/vision', 'resnet50', pretrained=True)
pytorch_model.eval()

# 加载 ONNX 模型
print("Loading ONNX ResNet50...")
onnx_session = ort.InferenceSession("models/resnet50.onnx")

# 创建测试输入
batch_size = 1
test_input = torch.randn(batch_size, 3, 224, 224)
print(f"\nTest input shape: {test_input.shape}")

# PyTorch 推理
print("\nRunning PyTorch inference...")
with torch.no_grad():
    pytorch_output = pytorch_model(test_input)
pytorch_result = pytorch_output.numpy()

# ONNX 推理
print("Running ONNX inference...")
input_name = onnx_session.get_inputs()[0].name
onnx_result = onnx_session.run(None, {input_name: test_input.numpy()})[0]

# 比较输出
print("\n=== Validation Results ===")
print(f"PyTorch output shape: {pytorch_result.shape}")
print(f"ONNX output shape: {onnx_result.shape}")

# 计算差异
absolute_diff = np.abs(pytorch_result - onnx_result)
max_diff = np.max(absolute_diff)
mean_diff = np.mean(absolute_diff)
relative_diff = np.mean(np.abs(pytorch_result - onnx_result) / (np.abs(pytorch_result) + 1e-8))

print(f"\nNumerical Comparison:")
print(f"  Max absolute difference: {max_diff:.6f}")
print(f"  Mean absolute difference: {mean_diff:.6f}")
print(f"  Mean relative difference: {relative_diff:.6f}")

# 检查输出是否接近
tolerance = 1e-5
if np.allclose(pytorch_result, onnx_result, rtol=tolerance, atol=tolerance):
    print(f"\n✅ PASS: Outputs match within tolerance ({tolerance})")
else:
    print(f"\n❌ FAIL: Outputs differ beyond tolerance ({tolerance})")

# 比较 top 预测
pytorch_top5 = np.argsort(pytorch_result[0])[-5:][::-1]
onnx_top5 = np.argsort(onnx_result[0])[-5:][::-1]

print(f"\nTop 5 Class Predictions:")
print(f"  PyTorch: {pytorch_top5}")
print(f"  ONNX:    {onnx_top5}")

if np.array_equal(pytorch_top5, onnx_top5):
    print("  ✅ Top 5 predictions match!")
else:
    print("  ⚠ Top 5 predictions differ")
```

**预期输出**：
```
Loading PyTorch ResNet50...
Loading ONNX ResNet50...

Test input shape: torch.Size([1, 3, 224, 224])

Running PyTorch inference...
Running ONNX inference...

=== Validation Results ===
PyTorch output shape: (1, 1000)
ONNX output shape: (1, 1000)

Numerical Comparison:
  Max absolute difference: 0.000012
  Mean absolute difference: 0.000003
  Mean relative difference: 0.000001

✅ PASS: Outputs match within tolerance (1e-05)

Top 5 Class Predictions:
  PyTorch: [259 261 260 262 263]
  ONNX:    [259 261 260 262 263]
  ✅ Top 5 predictions match!
```

### 步骤 5.2：综合验证套件

```python
# validation_suite.py
import torch
import onnxruntime as ort
import numpy as np
from typing import Dict, Tuple

class ModelValidator:
    """验证 ONNX 模型与 PyTorch 原始模型的一致性"""

    def __init__(self, pytorch_model: torch.nn.Module, onnx_path: str):
        self.pytorch_model = pytorch_model
        self.pytorch_model.eval()
        self.onnx_session = ort.InferenceSession(onnx_path)
        self.input_name = self.onnx_session.get_inputs()[0].name

    def validate(self, test_inputs: list, tolerance: float = 1e-5) -> Dict:
        """在多个测试输入上运行验证"""
        results = {
            'passed': 0,
            'failed': 0,
            'max_diff': 0.0,
            'mean_diff': 0.0,
            'details': []
        }

        print(f"Running validation on {len(test_inputs)} test cases...")

        for i, test_input in enumerate(test_inputs):
            # PyTorch 推理
            with torch.no_grad():
                pytorch_output = self.pytorch_model(test_input).numpy()

            # ONNX 推理
            onnx_output = self.onnx_session.run(
                None,
                {self.input_name: test_input.numpy()}
            )[0]

            # 计算差异
            abs_diff = np.abs(pytorch_output - onnx_output)
            max_diff = np.max(abs_diff)
            mean_diff = np.mean(abs_diff)

            # 更新结果
            results['max_diff'] = max(results['max_diff'], max_diff)
            results['mean_diff'] += mean_diff

            # 检查是否通过
            passed = np.allclose(pytorch_output, onnx_output, rtol=tolerance, atol=tolerance)
            if passed:
                results['passed'] += 1
                status = '✅'
            else:
                results['failed'] += 1
                status = '❌'

            results['details'].append({
                'test_case': i,
                'max_diff': max_diff,
                'mean_diff': mean_diff,
                'passed': passed
            })

            print(f"  Test {i+1}: {status} (max_diff: {max_diff:.6f})")

        results['mean_diff'] /= len(test_inputs)
        return results

# 示例使用
print("Loading models...")
pytorch_model = torch.hub.load('pytorch/vision', 'resnet18', pretrained=True)
pytorch_model.eval()

# 导出为 ONNX
torch.onnx.export(
    pytorch_model,
    torch.randn(1, 3, 224, 224),
    "models/resnet18.onnx",
    export_params=True,
    opset_version=14,
    input_names=['input'],
    output_names=['output']
)

# 创建验证器
validator = ModelValidator(pytorch_model, "models/resnet18.onnx")

# 生成具有不同特征的测试输入
test_inputs = [
    torch.randn(1, 3, 224, 224),           # 随机正态
    torch.zeros(1, 3, 224, 224),           # 全零
    torch.ones(1, 3, 224, 224),            # 全一
    torch.randn(1, 3, 224, 224) * 0.1,     # 小值
    torch.randn(1, 3, 224, 224) * 10,      # 大值
]

# 运行验证
results = validator.validate(test_inputs, tolerance=1e-5)

# 打印摘要
print("\n=== Validation Summary ===")
print(f"Total tests: {len(test_inputs)}")
print(f"Passed: {results['passed']}")
print(f"Failed: {results['failed']}")
print(f"Max difference: {results['max_diff']:.6f}")
print(f"Mean difference: {results['mean_diff']:.6f}")

if results['failed'] == 0:
    print("\n✅ All validation tests passed!")
else:
    print(f"\n⚠ {results['failed']} tests failed")
```

✅ **检查点**：你能验证 ONNX 模型的数值精度了。

---

## 第 6 部分：性能对比

### 步骤 6.1：延迟对比

```python
# performance_comparison.py
import torch
import onnxruntime as ort
import numpy as np
import time
from typing import Dict

def benchmark_pytorch(model, input_tensor, iterations=100, warmup=10):
    """对 PyTorch 模型进行基准测试"""
    model.eval()

    # 预热
    with torch.no_grad():
        for _ in range(warmup):
            _ = model(input_tensor)

    # 测量
    start = time.time()
    with torch.no_grad():
        for _ in range(iterations):
            _ = model(input_tensor)
    elapsed = time.time() - start

    return (elapsed / iterations) * 1000  # ms

def benchmark_onnx(session, input_name, input_data, iterations=100, warmup=10):
    """对 ONNX Runtime 进行基准测试"""

    # 预热
    for _ in range(warmup):
        _ = session.run(None, {input_name: input_data})

    # 测量
    start = time.time()
    for _ in range(iterations):
        _ = session.run(None, {input_name: input_data})
    elapsed = time.time() - start

    return (elapsed / iterations) * 1000  # ms

# 加载模型
print("Loading models...")
pytorch_model = torch.hub.load('pytorch/vision', 'resnet50', pretrained=True)
pytorch_model.eval()

onnx_session = ort.InferenceSession("models/resnet50.onnx")
input_name = onnx_session.get_inputs()[0].name

# 测试不同的批大小
batch_sizes = [1, 4, 8, 16]
print("\n=== Performance Comparison (CPU) ===")
print(f"{'Batch Size':<12} {'PyTorch (ms)':<15} {'ONNX (ms)':<15} {'Speedup':<10}")
print("-" * 55)

for batch_size in batch_sizes:
    # 准备输入
    pytorch_input = torch.randn(batch_size, 3, 224, 224)
    onnx_input = pytorch_input.numpy()

    # 基准测试
    pytorch_time = benchmark_pytorch(pytorch_model, pytorch_input, iterations=50)
    onnx_time = benchmark_onnx(onnx_session, input_name, onnx_input, iterations=50)

    speedup = pytorch_time / onnx_time

    print(f"{batch_size:<12} {pytorch_time:<15.2f} {onnx_time:<15.2f} {speedup:<10.2f}x")
```

**预期输出（CPU）**：
```
Loading models...

=== Performance Comparison (CPU) ===
Batch Size   PyTorch (ms)    ONNX (ms)       Speedup
-------------------------------------------------------
1            45.23           32.18           1.41x
4            167.89          118.45          1.42x
8            329.34          228.91          1.44x
16           645.78          447.23          1.44x
```

### 步骤 6.2：综合性能套件

```python
# performance_suite.py
import torch
import onnxruntime as ort
import numpy as np
import time
import psutil
import os

class PerformanceBenchmark:
    """综合性能基准测试"""

    def __init__(self):
        self.results = {}

    def measure_latency(self, func, iterations=100, warmup=10):
        """测量平均延迟"""
        # 预热
        for _ in range(warmup):
            func()

        # 测量
        latencies = []
        for _ in range(iterations):
            start = time.perf_counter()
            func()
            elapsed = time.perf_counter() - start
            latencies.append(elapsed * 1000)  # ms

        return {
            'mean': np.mean(latencies),
            'std': np.std(latencies),
            'min': np.min(latencies),
            'max': np.max(latencies),
            'p50': np.percentile(latencies, 50),
            'p95': np.percentile(latencies, 95),
            'p99': np.percentile(latencies, 99)
        }

    def measure_throughput(self, func, duration_sec=5):
        """测量吞吐量（每秒迭代次数）"""
        count = 0
        start = time.time()

        while time.time() - start < duration_sec:
            func()
            count += 1

        elapsed = time.time() - start
        return count / elapsed

    def measure_memory(self, func):
        """测量内存使用"""
        process = psutil.Process(os.getpid())

        # 初始内存
        mem_before = process.memory_info().rss / 1024 / 1024  # MB

        # 运行函数
        func()

        # 最终内存
        mem_after = process.memory_info().rss / 1024 / 1024  # MB

        return {
            'before_mb': mem_before,
            'after_mb': mem_after,
            'delta_mb': mem_after - mem_before
        }

    def benchmark_model(self, name, inference_func):
        """运行完整基准测试套件"""
        print(f"\n{'='*50}")
        print(f"Benchmarking: {name}")
        print(f"{'='*50}")

        # 延迟
        print("\n📊 Latency Statistics:")
        latency_stats = self.measure_latency(inference_func, iterations=100)
        for key, value in latency_stats.items():
            print(f"  {key:>6}: {value:>8.3f} ms")

        # 吞吐量
        print("\n🚀 Throughput:")
        throughput = self.measure_throughput(inference_func, duration_sec=3)
        print(f"  {throughput:.2f} inferences/second")

        # 内存
        print("\n💾 Memory Usage:")
        memory_stats = self.measure_memory(inference_func)
        print(f"  Before: {memory_stats['before_mb']:.2f} MB")
        print(f"  After:  {memory_stats['after_mb']:.2f} MB")
        print(f"  Delta:  {memory_stats['delta_mb']:.2f} MB")

        self.results[name] = {
            'latency': latency_stats,
            'throughput': throughput,
            'memory': memory_stats
        }

    def print_comparison(self):
        """打印所有基准测试模型之间的比较"""
        print(f"\n\n{'='*70}")
        print("PERFORMANCE COMPARISON SUMMARY")
        print(f"{'='*70}")

        if len(self.results) < 2:
            print("Need at least 2 models to compare")
            return

        models = list(self.results.keys())
        baseline = models[0]
        comparison = models[1]

        baseline_latency = self.results[baseline]['latency']['mean']
        comparison_latency = self.results[comparison]['latency']['mean']
        latency_speedup = baseline_latency / comparison_latency

        baseline_throughput = self.results[baseline]['throughput']
        comparison_throughput = self.results[comparison]['throughput']
        throughput_improvement = comparison_throughput / baseline_throughput

        print(f"\n📊 {comparison} vs {baseline}:")
        print(f"  Latency:    {latency_speedup:>6.2f}x {'faster' if latency_speedup > 1 else 'slower'}")
        print(f"  Throughput: {throughput_improvement:>6.2f}x {'higher' if throughput_improvement > 1 else 'lower'}")

# 示例使用
print("Setting up benchmark...")

# 加载 PyTorch 模型
pytorch_model = torch.hub.load('pytorch/vision', 'mobilenet_v2', pretrained=True)
pytorch_model.eval()
pytorch_input = torch.randn(1, 3, 224, 224)

# 加载 ONNX 模型
onnx_session = ort.InferenceSession("models/mobilenet_v2.onnx")
input_name = onnx_session.get_inputs()[0].name
onnx_input = pytorch_input.numpy()

# 创建基准测试
benchmark = PerformanceBenchmark()

# PyTorch 基准测试
pytorch_func = lambda: pytorch_model(pytorch_input)
benchmark.benchmark_model("PyTorch MobileNetV2", pytorch_func)

# ONNX 基准测试
onnx_func = lambda: onnx_session.run(None, {input_name: onnx_input})
benchmark.benchmark_model("ONNX MobileNetV2", onnx_func)

# 打印比较
benchmark.print_comparison()
```

**预期输出**：
```
==================================================
Benchmarking: PyTorch MobileNetV2
==================================================

📊 Latency Statistics:
    mean:   18.234 ms
     std:    1.456 ms
     min:   16.123 ms
     max:   23.456 ms
     p50:   18.012 ms
     p95:   20.567 ms
     p99:   22.345 ms

🚀 Throughput:
  54.32 inferences/second

💾 Memory Usage:
  Before: 245.67 MB
  After:  245.89 MB
  Delta:  0.22 MB

==================================================
Benchmarking: ONNX MobileNetV2
==================================================

📊 Latency Statistics:
    mean:   12.456 ms
     std:    0.987 ms
     min:   11.234 ms
     max:   15.678 ms
     p50:   12.345 ms
     p95:   13.456 ms
     p99:   14.234 ms

🚀 Throughput:
  79.87 inferences/second

💾 Memory Usage:
  Before: 245.89 MB
  After:  246.01 MB
  Delta:  0.12 MB

======================================================================
PERFORMANCE COMPARISON SUMMARY
======================================================================

📊 ONNX MobileNetV2 vs PyTorch MobileNetV2:
  Latency:      1.46x faster
  Throughput:   1.47x higher
```

✅ **检查点**：你能全面测量和比较性能了。

---

## 第 7 部分：生产就绪的转换脚本

### 步骤 7.1：完整转换工具

```python
# model_converter.py
"""
生产就绪的模型转换工具
将 PyTorch 和 TensorFlow 模型转换为 ONNX 并验证
"""

import torch
import onnx
import onnxruntime as ort
import numpy as np
import argparse
import os
from typing import Optional, Tuple
import json

class ModelConverter:
    """将模型转换并验证为 ONNX 格式"""

    def __init__(self, validate: bool = True, verbose: bool = True):
        self.validate = validate
        self.verbose = verbose
        self.conversion_report = {}

    def log(self, message: str):
        """如果 verbose，打印消息"""
        if self.verbose:
            print(message)

    def convert_pytorch(
        self,
        model: torch.nn.Module,
        dummy_input: torch.Tensor,
        output_path: str,
        input_names: list = ['input'],
        output_names: list = ['output'],
        dynamic_axes: Optional[dict] = None,
        opset_version: int = 14
    ) -> bool:
        """将 PyTorch 模型转换为 ONNX"""

        self.log(f"\n{'='*60}")
        self.log("Converting PyTorch Model to ONNX")
        self.log(f"{'='*60}")

        try:
            # 确保模型处于 eval 模式
            model.eval()

            # batch size 的默认动态轴
            if dynamic_axes is None:
                dynamic_axes = {
                    input_names[0]: {0: 'batch_size'},
                    output_names[0]: {0: 'batch_size'}
                }

            # 导出为 ONNX
            self.log(f"\n📤 Exporting to {output_path}...")
            torch.onnx.export(
                model,
                dummy_input,
                output_path,
                export_params=True,
                opset_version=opset_version,
                do_constant_folding=True,
                input_names=input_names,
                output_names=output_names,
                dynamic_axes=dynamic_axes,
                verbose=False
            )

            # 验证 ONNX 模型
            self.log("✅ Checking ONNX model validity...")
            onnx_model = onnx.load(output_path)
            onnx.checker.check_model(onnx_model)
            self.log("✅ ONNX model is valid!")

            # 存储模型信息
            file_size = os.path.getsize(output_path) / (1024 * 1024)
            self.conversion_report['output_path'] = output_path
            self.conversion_report['file_size_mb'] = round(file_size, 2)
            self.conversion_report['opset_version'] = opset_version

            self.log(f"\n📊 Model Statistics:")
            self.log(f"  File size: {file_size:.2f} MB")
            self.log(f"  Opset version: {opset_version}")
            self.log(f"  Graph nodes: {len(onnx_model.graph.node)}")

            # 如果请求，进行验证
            if self.validate:
                self.log("\n🔍 Validating conversion accuracy...")
                validation_passed = self._validate_pytorch_conversion(
                    model, output_path, dummy_input
                )
                self.conversion_report['validation_passed'] = validation_passed

                if validation_passed:
                    self.log("✅ Validation passed!")
                else:
                    self.log("⚠ Validation failed - outputs differ significantly")
                    return False

            self.log("\n✅ Conversion successful!")
            return True

        except Exception as e:
            self.log(f"\n❌ Conversion failed: {str(e)}")
            self.conversion_report['error'] = str(e)
            return False

    def _validate_pytorch_conversion(
        self,
        pytorch_model: torch.nn.Module,
        onnx_path: str,
        test_input: torch.Tensor,
        tolerance: float = 1e-5
    ) -> bool:
        """验证 PyTorch 到 ONNX 的转换"""

        # PyTorch 推理
        with torch.no_grad():
            pytorch_output = pytorch_model(test_input).numpy()

        # ONNX 推理
        onnx_session = ort.InferenceSession(onnx_path)
        input_name = onnx_session.get_inputs()[0].name
        onnx_output = onnx_session.run(None, {input_name: test_input.numpy()})[0]

        # 比较
        abs_diff = np.abs(pytorch_output - onnx_output)
        max_diff = np.max(abs_diff)
        mean_diff = np.mean(abs_diff)

        self.log(f"  Max difference: {max_diff:.6f}")
        self.log(f"  Mean difference: {mean_diff:.6f}")

        self.conversion_report['max_difference'] = float(max_diff)
        self.conversion_report['mean_difference'] = float(mean_diff)

        return np.allclose(pytorch_output, onnx_output, rtol=tolerance, atol=tolerance)

    def save_report(self, output_path: str = "conversion_report.json"):
        """将转换报告保存为 JSON"""
        with open(output_path, 'w') as f:
            json.dump(self.conversion_report, f, indent=2)
        self.log(f"\n📄 Report saved to {output_path}")

def main():
    parser = argparse.ArgumentParser(description='Convert models to ONNX format')
    parser.add_argument('--model', type=str, default='resnet18',
                       help='PyTorch model name')
    parser.add_argument('--output', type=str, default='models/converted_model.onnx',
                       help='Output ONNX path')
    parser.add_argument('--no-validate', action='store_true',
                       help='Skip validation')
    parser.add_argument('--opset', type=int, default=14,
                       help='ONNX opset version')

    args = parser.parse_args()

    # 加载 PyTorch 模型
    print(f"Loading PyTorch model: {args.model}")
    model = torch.hub.load('pytorch/vision', args.model, pretrained=True)
    model.eval()

    # 创建虚拟输入
    dummy_input = torch.randn(1, 3, 224, 224)

    # 创建转换器
    converter = ModelConverter(validate=not args.no_validate)

    # 转换模型
    success = converter.convert_pytorch(
        model=model,
        dummy_input=dummy_input,
        output_path=args.output,
        opset_version=args.opset
    )

    # 保存报告
    if success:
        converter.save_report()

    return 0 if success else 1

if __name__ == "__main__":
    exit(main())
```

**使用方法**：
```bash
# 转换 ResNet18
python model_converter.py --model resnet18 --output models/resnet18.onnx

# 转换 MobileNetV2 不验证
python model_converter.py --model mobilenet_v2 --output models/mobilenet.onnx --no-validate

# 使用特定 opset 版本转换
python model_converter.py --model efficientnet_b0 --opset 15
```

✅ **检查点**：你有了一个生产就绪的转换工具！

---

## 第 8 部分：实际应用

### 步骤 8.1：完整推理对比工具

```python
# inference_comparison_app.py
"""
比较 PyTorch、TensorFlow 和 ONNX 之间的推理
"""

import torch
import onnxruntime as ort
import numpy as np
from PIL import Image
import time
import argparse
from typing import Dict, List

class MultiFrameworkInference:
    """跨多个框架运行推理"""

    def __init__(self, image_path: str):
        self.image_path = image_path
        self.image = None
        self.labels = None
        self._load_labels()

    def _load_labels(self):
        """加载 ImageNet 标签"""
        import urllib.request
        url = "https://raw.githubusercontent.com/pytorch/hub/master/imagenet_classes.txt"
        urllib.request.urlretrieve(url, "imagenet_classes.txt")
        with open("imagenet_classes.txt") as f:
            self.labels = [line.strip() for line in f.readlines()]

    def preprocess_pytorch(self) -> torch.Tensor:
        """为 PyTorch 预处理"""
        from torchvision import transforms

        transform = transforms.Compose([
            transforms.Resize(256),
            transforms.CenterCrop(224),
            transforms.ToTensor(),
            transforms.Normalize(
                mean=[0.485, 0.456, 0.406],
                std=[0.229, 0.224, 0.225]
            )
        ])

        image = Image.open(self.image_path).convert('RGB')
        return transform(image).unsqueeze(0)

    def preprocess_onnx(self) -> np.ndarray:
        """为 ONNX 预处理"""
        image = Image.open(self.image_path).convert('RGB')
        image = image.resize((256, 256))

        # 中心裁剪
        left = (256 - 224) / 2
        top = (256 - 224) / 2
        image = image.crop((left, top, left + 224, top + 224))

        # 归一化
        img_array = np.array(image).astype(np.float32) / 255.0
        mean = np.array([0.485, 0.456, 0.406])
        std = np.array([0.229, 0.224, 0.225])
        img_array = (img_array - mean) / std

        # CHW 格式
        img_array = img_array.transpose(2, 0, 1)
        return np.expand_dims(img_array, axis=0)

    def run_pytorch(self, model_name: str = 'resnet18') -> Dict:
        """运行 PyTorch 推理"""
        print(f"\n{'='*50}")
        print(f"PyTorch Inference ({model_name})")
        print(f"{'='*50}")

        # 加载模型
        print("Loading model...")
        model = torch.hub.load('pytorch/vision', model_name, pretrained=True)
        model.eval()

        # 预处理
        input_tensor = self.preprocess_pytorch()

        # 预热
        with torch.no_grad():
            for _ in range(10):
                _ = model(input_tensor)

        # 测量推理
        start = time.perf_counter()
        with torch.no_grad():
            output = model(input_tensor)
        latency_ms = (time.perf_counter() - start) * 1000

        # 获取预测
        probabilities = torch.nn.functional.softmax(output[0], dim=0)
        top5_prob, top5_idx = torch.topk(probabilities, 5)

        results = {
            'framework': 'PyTorch',
            'model': model_name,
            'latency_ms': latency_ms,
            'predictions': [
                {
                    'class': self.labels[idx.item()],
                    'probability': prob.item()
                }
                for prob, idx in zip(top5_prob, top5_idx)
            ]
        }

        self._print_results(results)
        return results

    def run_onnx(self, onnx_path: str) -> Dict:
        """运行 ONNX Runtime 推理"""
        print(f"\n{'='*50}")
        print(f"ONNX Runtime Inference")
        print(f"{'='*50}")

        # 加载模型
        print(f"Loading model from {onnx_path}...")
        session = ort.InferenceSession(onnx_path)
        input_name = session.get_inputs()[0].name

        # 预处理
        input_data = self.preprocess_onnx()

        # 预热
        for _ in range(10):
            _ = session.run(None, {input_name: input_data})

        # 测量推理
        start = time.perf_counter()
        output = session.run(None, {input_name: input_data})[0]
        latency_ms = (time.perf_counter() - start) * 1000

        # 获取预测
        exp_output = np.exp(output[0] - np.max(output[0]))
        probabilities = exp_output / exp_output.sum()
        top5_idx = np.argsort(probabilities)[-5:][::-1]

        results = {
            'framework': 'ONNX Runtime',
            'model': os.path.basename(onnx_path),
            'latency_ms': latency_ms,
            'predictions': [
                {
                    'class': self.labels[idx],
                    'probability': float(probabilities[idx])
                }
                for idx in top5_idx
            ]
        }

        self._print_results(results)
        return results

    def _print_results(self, results: Dict):
        """打印推理结果"""
        print(f"\n⏱️  Inference time: {results['latency_ms']:.2f} ms")
        print(f"\n🎯 Top 5 Predictions:")
        for i, pred in enumerate(results['predictions']):
            print(f"  {i+1}. {pred['class']:<30} {pred['probability']*100:>6.2f}%")

    def compare(self, results: List[Dict]):
        """跨框架比较结果"""
        print(f"\n\n{'='*70}")
        print("COMPARISON SUMMARY")
        print(f"{'='*70}")

        print(f"\n{'Framework':<20} {'Model':<25} {'Latency (ms)':<15}")
        print("-" * 60)

        for result in results:
            print(f"{result['framework']:<20} {result['model']:<25} {result['latency_ms']:<15.2f}")

        # 找到最快的
        fastest = min(results, key=lambda x: x['latency_ms'])
        print(f"\n🏆 Fastest: {fastest['framework']} ({fastest['latency_ms']:.2f} ms)")

        # 比较预测
        print(f"\n🎯 Top Prediction Comparison:")
        for result in results:
            top_pred = result['predictions'][0]
            print(f"  {result['framework']:<20} {top_pred['class']:<30} {top_pred['probability']*100:>6.2f}%")

def main():
    parser = argparse.ArgumentParser(description='Compare inference across frameworks')
    parser.add_argument('image', type=str, help='Image path')
    parser.add_argument('--pytorch-model', type=str, default='resnet18',
                       help='PyTorch model name')
    parser.add_argument('--onnx-model', type=str, default='models/resnet18.onnx',
                       help='ONNX model path')

    args = parser.parse_args()

    # 创建推理引擎
    engine = MultiFrameworkInference(args.image)

    # 在两个框架上运行推理
    results = []

    # PyTorch
    pytorch_result = engine.run_pytorch(args.pytorch_model)
    results.append(pytorch_result)

    # ONNX
    if os.path.exists(args.onnx_model):
        onnx_result = engine.run_onnx(args.onnx_model)
        results.append(onnx_result)
    else:
        print(f"\n⚠ ONNX model not found: {args.onnx_model}")
        print("Convert PyTorch model first using model_converter.py")

    # 比较
    if len(results) > 1:
        engine.compare(results)

if __name__ == "__main__":
    main()
```

**使用方法**：
```bash
# 比较 PyTorch 和 ONNX 推理
python inference_comparison_app.py test_images/dog.jpg \
    --pytorch-model resnet18 \
    --onnx-model models/resnet18.onnx
```

✅ **最终检查点**：你有了一个完整的跨框架推理系统！

---

## 拓展目标（可选挑战）

### 挑战 1：优化 ONNX 模型

**任务**：应用优化技术以减少模型大小并提高速度。

```python
# 提示：使用 onnxruntime.transformers.optimizer
from onnxruntime.transformers import optimizer

optimized_model = optimizer.optimize_model(
    "models/resnet50.onnx",
    model_type='bert',  # 或适当的类型
    num_heads=0,
    hidden_size=0
)
```

### 挑战 2：量化

**任务**：将模型转换为 INT8 量化以实现更快的推理。

```python
# 提示：使用动态量化
import onnxruntime as ort
from onnxruntime.quantization import quantize_dynamic, QuantType

quantize_dynamic(
    "models/resnet50.onnx",
    "models/resnet50_quant.onnx",
    weight_type=QuantType.QUInt8
)
```

### 挑战 3：自定义算子

**任务**：在 ONNX 转换中处理具有自定义 PyTorch 算子的模型。

研究：
- 如何注册自定义 ONNX 算子
- 符号函数定义
- 跨版本的算子兼容性

---

## 验证和测试

### 测试你的知识

```python
# test_conversions.py
"""测试你的转换实现"""

def test_conversion_validity():
    """测试转换后的模型是否为有效的 ONNX"""
    import onnx

    onnx_files = [
        "models/resnet50.onnx",
        "models/mobilenet_v2.onnx"
    ]

    for onnx_file in onnx_files:
        if os.path.exists(onnx_file):
            model = onnx.load(onnx_file)
            onnx.checker.check_model(model)
            print(f"✅ {onnx_file} is valid")
        else:
            print(f"⚠ {onnx_file} not found")

def test_numerical_accuracy():
    """测试 ONNX 输出是否与 PyTorch 匹配"""
    # 在此处实现
    pass

def test_performance_improvement():
    """测试 ONNX 是否比 PyTorch 更快"""
    # 在此处实现
    pass

if __name__ == "__main__":
    test_conversion_validity()
    test_numerical_accuracy()
    test_performance_improvement()
```

---

## 反思问题

回答这些问题以巩固你的学习：

1. **在生产中使用 ONNX 的主要好处是什么？**
   - 你的答案：

2. **什么时候你不会使用 ONNX（坚持使用原生框架）？**
   - 你的答案：

3. **为什么转换后的模型验证很重要？**
   - 你的答案：

4. **ONNX Runtime 如何实现比原生框架更好的性能？**
   - 你的答案：

5. **不同 opset 版本之间的权衡是什么？**
   - 你的答案：

6. **在什么场景下你会使用 dynamic_axes？**
   - 你的答案：

7. **如果模型转换成功但输出错误，你如何调试？**
   - 你的答案：

---

## 总结

**你完成了什么**：
✅ 将 PyTorch 模型转换为 ONNX 格式
✅ 将 TensorFlow 模型转换为 ONNX 格式
✅ 使用 ONNX Runtime 运行推理
✅ 验证转换后模型的精度
✅ 跨框架测量性能
✅ 构建生产就绪的转换工具
✅ 理解 ONNX 在 ML 基础设施中的角色

**获得的关键技能**：
- 模型格式转换（PyTorch → ONNX）
- 模型格式转换（TensorFlow → ONNX）
- ONNX Runtime 推理
- 数值验证技术
- 性能基准测试
- 多框架部署策略

**基础设施工程师要点**：
- **部署灵活性**：ONNX 实现与框架无关的部署
- **性能**：ONNX Runtime 通常优于原生框架的推理性能
- **优化**：标准格式支持硬件特定的优化
- **互操作性**：在一个框架中训练，在另一个框架中部署
- **成本节约**：更快的推理减少了云计算成本

---

## 下一步

1. **探索 ONNX Runtime 优化**：了解图优化、量化和剪枝
2. **尝试不同模型**：转换各种架构（transformers, GANs 等）
3. **生产部署**：在服务框架中使用 ONNX 模型（TorchServe, TensorFlow Serving, Triton）
4. **硬件加速**：尝试 TensorRT, OpenVINO, CoreML 执行提供者
5. **完成项目 02**：将 ONNX 模型部署为 REST API

---

## 额外资源

### 文档
- [ONNX 官方文档](https://onnx.ai/onnx/)
- [ONNX Runtime 文档](https://onnxruntime.ai/)
- [PyTorch ONNX 导出](https://pytorch.org/docs/stable/onnx.html)
- [TF2ONNX 文档](https://github.com/onnx/tensorflow-onnx)

### 教程
- ONNX Runtime 性能调优指南
- 模型优化最佳实践
- 量化和剪枝技术

### 工具
- Netron（可视化 ONNX 模型）
- ONNX Optimizer
- ONNX Runtime Tools

---

**练习版本**: 1.0
**最后更新**: 2025 年 10 月
**预计时间**: 3-4 小时
**难度**: 初级-中级
