# 源代码实现指南

**项目：** 简单模型 API 部署
**目录：** `src/`

---

## 概述

此目录包含模型 API 的核心应用代码。代码组织为三个主要模块：

1. **`config.py`** - 配置管理和环境变量
2. **`model_loader.py`** - ML 模型加载、预处理和推理
3. **`app.py`** - 使用 Flask/FastAPI 实现的 REST API

---

## 实现顺序

按以下顺序构建应用：

### 1. 从配置开始（`config.py`）
- 最简单的模块，无依赖
- 定义所有配置值
- 设置环境变量加载
- 预计时间：1-2 小时

### 2. 实现模型加载器（`model_loader.py`）
- 核心 ML 功能
- 仅依赖 ML 框架（PyTorch/TensorFlow）
- 可独立测试
- 预计时间：4-6 小时

### 3. 构建 API（`app.py`）
- 将所有内容串联起来
- 依赖于 config 和 model_loader
- 实现 REST 端点
- 预计时间：6-8 小时

---

## 代码结构

```
src/
├── README.md              # 本文件
├── app.py                 # 主应用（存根）
├── model_loader.py        # 模型管理（存根）
└── config.py              # 配置（存根）
```

---

## 模块说明

### `config.py` - 配置管理

**目的：** 集中管理所有配置值和环境变量

**需要实现的内容：**
- 带默认值的环境变量加载
- 配置验证
- 类型转换（字符串转整数、布尔值等）
- 配置类或字典

**关键配置值：**
```python
# 模型配置
MODEL_NAME: str = "resnet50"  # 或 "mobilenet_v2"
MODEL_PATH: str = "~/.cache/torch/hub"
DEVICE: str = "cpu"  # 或 "cuda"

# API 配置
HOST: str = "0.0.0.0"
PORT: int = 5000
DEBUG: bool = False

# 限制
MAX_FILE_SIZE: int = 10 * 1024 * 1024  # 10MB
REQUEST_TIMEOUT: int = 30  # 秒

# 日志
LOG_LEVEL: str = "INFO"
LOG_FORMAT: str = "json"  # 或 "text"
```

**测试：**
```python
# 测试配置是否正确加载
from config import Config

config = Config()
assert config.MODEL_NAME in ["resnet50", "mobilenet_v2"]
assert config.PORT > 0
assert config.MAX_FILE_SIZE > 0
```

---

### `model_loader.py` - 模型管理

**目的：** 处理所有 ML 模型操作（加载、预处理、推理）

**需要实现的内容：**

#### 1. 模型加载
```python
def load_model(model_name: str, device: str) -> torch.nn.Module:
    """
    从 torchvision 加载预训练模型。

    TODO:
    - 从 torchvision.models 导入相应模型
    - 以 pretrained=True 加载
    - 将模型移动到指定设备
    - 将模型设置为 eval() 模式
    - 返回模型实例
    """
```

#### 2. 图片预处理
```python
def preprocess_image(image: PIL.Image) -> torch.Tensor:
    """
    预处理图片以供模型推理。

    TODO:
    - 将图片转换为 RGB（处理灰度图、RGBA）
    - 调整为 224x224
    - 转换为 tensor
    - 使用 ImageNet 统计数据进行归一化
    - 添加批次维度
    - 返回预处理后的 tensor
    """
```

#### 3. 推理
```python
def predict(model: torch.nn.Module,
           image_tensor: torch.Tensor,
           top_k: int = 5) -> List[Dict]:
    """
    从模型生成预测结果。

    TODO:
    - 禁用梯度计算（torch.no_grad()）
    - 执行前向传播
    - 应用 softmax 获取概率
    - 获取 top-K 预测结果
    - 将索引映射到类别标签
    - 格式化为字典列表
    - 返回预测结果
    """
```

#### 4. 类别标签加载
```python
def load_imagenet_labels() -> Dict[int, str]:
    """
    加载 ImageNet 类别标签。

    TODO:
    - 下载或加载 ImageNet 标签
    - 解析为字典 {index: label}
    - 处理标签文件缺失的情况
    - 返回标签映射
    """
```

**模型加载器类结构：**
```python
class ModelLoader:
    """管理 ML 模型生命周期和推理。"""

    def __init__(self, model_name: str = "resnet50", device: str = "cpu"):
        """使用配置初始化 ModelLoader。"""

    def load(self) -> None:
        """加载模型权重并准备推理。"""

    def preprocess(self, image: PIL.Image) -> torch.Tensor:
        """预处理图片以输入模型。"""

    def predict(self, image: PIL.Image, top_k: int = 5) -> List[Dict]:
        """为图片生成 top-K 预测结果。"""

    def get_model_info(self) -> Dict:
        """返回模型元数据。"""
```

**测试：**
```python
# 测试模型加载
loader = ModelLoader(model_name="resnet50")
loader.load()
assert loader.model is not None

# 测试预处理
from PIL import Image
test_image = Image.open("test.jpg")
tensor = loader.preprocess(test_image)
assert tensor.shape == (1, 3, 224, 224)

# 测试预测
predictions = loader.predict(test_image)
assert len(predictions) == 5
assert all(0 <= p['confidence'] <= 1 for p in predictions)
```

---

### `app.py` - REST API 实现

**目的：** 实现 HTTP API 端点和请求处理

**需要实现的内容：**

#### 1. 应用设置
```python
# Flask 示例
from flask import Flask, request, jsonify
import logging

app = Flask(__name__)
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# 初始化模型加载器
model_loader = ModelLoader()
model_loader.load()
```

#### 2. 健康检查端点
```python
@app.route('/health', methods=['GET'])
def health():
    """
    健康检查端点。

    TODO:
    - 检查模型是否已加载
    - 返回状态："healthy" 或 "unhealthy"
    - 包含模型名称
    - 健康时返回 200，否则返回 503
    """
```

#### 3. 信息端点
```python
@app.route('/info', methods=['GET'])
def info():
    """
    模型信息端点。

    TODO:
    - 从 model_loader 获取模型元数据
    - 包含 API 版本
    - 包含支持的端点
    - 包含限制（文件大小、超时）
    - 以 JSON 格式返回
    """
```

#### 4. 预测端点
```python
@app.route('/predict', methods=['POST'])
def predict():
    """
    预测端点。

    TODO:
    - 验证请求包含文件
    - 检查 Content-Type
    - 验证文件大小
    - 使用 PIL 加载图片
    - 处理图片加载错误
    - 调用 model_loader.predict()
    - 格式化包含预测结果的响应
    - 添加延迟测量
    - 使用关联 ID 记录请求
    - 优雅地处理所有错误
    - 返回 JSON 响应
    """
```

#### 5. 错误处理
```python
@app.errorhandler(400)
def bad_request(error):
    """处理 400 错误。"""

@app.errorhandler(413)
def payload_too_large(error):
    """处理文件过大错误。"""

@app.errorhandler(500)
def internal_error(error):
    """处理服务器内部错误。"""
```

#### 6. 请求验证
```python
def validate_image_file(file) -> Tuple[bool, Optional[str]]:
    """
    验证上传的图片文件。

    TODO:
    - 检查文件不为 None
    - 验证文件大小
    - 尝试使用 PIL 打开
    - 确认它确实是图片
    - 返回 (is_valid, error_message)
    """
```

#### 7. 响应格式化
```python
def format_success_response(predictions: List[Dict],
                          latency_ms: float) -> Dict:
    """格式化成功的预测响应。"""

def format_error_response(error_code: str,
                         message: str,
                         correlation_id: str) -> Dict:
    """格式化错误响应。"""
```

#### 8. 关联 ID 生成
```python
import uuid

def generate_correlation_id() -> str:
    """生成用于请求跟踪的唯一关联 ID。"""
    return f"req-{uuid.uuid4().hex[:8]}"
```

**Flask vs FastAPI：**

本指南展示 Flask 示例，但你也可以使用 FastAPI：

```python
# FastAPI 示例
from fastapi import FastAPI, File, UploadFile, HTTPException
from typing import List

app = FastAPI()

@app.get("/health")
async def health():
    return {"status": "healthy"}

@app.post("/predict")
async def predict(file: UploadFile = File(...), top_k: int = 5):
    # 在此处实现
    pass
```

**选择一个框架并在整个项目中坚持使用。**

---

## 实现指南

### 代码风格

遵循 PEP 8 风格指南：
- 缩进使用 4 个空格（不使用制表符）
- 最大行长度：88 个字符（Black 格式化工具）
- 顶层函数/类之间空两行
- 方法之间空一行
- 函数/变量使用小写加下划线
- 类名使用 CamelCase

### 类型提示

为所有函数添加类型提示：
```python
from typing import List, Dict, Tuple, Optional
from PIL import Image
import torch

def predict(image: Image.Image, top_k: int = 5) -> List[Dict[str, any]]:
    """以类型安全的方式生成预测结果。"""
    pass
```

### 文档字符串

使用 Google 风格的文档字符串：
```python
def preprocess_image(image: Image.Image) -> torch.Tensor:
    """
    预处理图片以供模型推理。

    将图片转换为 RGB，调整为 224x224，使用 ImageNet 统计数据进行
    归一化，并转换为 PyTorch tensor。

    Args:
        image: 要预处理的 PIL Image 对象

    Returns:
        预处理后的图片 tensor，形状为 (1, 3, 224, 224)

    Raises:
        ValueError: 如果图片无法转换为 RGB

    Example:
        >>> from PIL import Image
        >>> img = Image.open('dog.jpg')
        >>> tensor = preprocess_image(img)
        >>> tensor.shape
        torch.Size([1, 3, 224, 224])
    """
    pass
```

### 错误处理

始终优雅地处理错误：
```python
try:
    image = Image.open(file_bytes)
    image = image.convert('RGB')
except Exception as e:
    logger.error(f"Failed to load image: {e}")
    return {"error": "Invalid image file"}, 400
```

### 日志记录

在适当的级别记录日志：
```python
logger.debug("Preprocessing image")       # 详细调试信息
logger.info("Model loaded successfully")  # 正常操作
logger.warning("File size near limit")    # 异常但可恢复
logger.error("Invalid image format")      # 请求失败
logger.critical("Model failed to load")   # 系统故障
```

---

## 测试你的实现

### 单元测试

独立测试每个模块：

```python
# tests/test_config.py
def test_config_defaults():
    config = Config()
    assert config.MODEL_NAME == "resnet50"
    assert config.PORT == 5000

# tests/test_model_loader.py
def test_model_loads():
    loader = ModelLoader()
    loader.load()
    assert loader.model is not None

def test_prediction():
    loader = ModelLoader()
    loader.load()
    image = Image.open('test_image.jpg')
    predictions = loader.predict(image)
    assert len(predictions) == 5

# tests/test_app.py
def test_health_endpoint(client):
    response = client.get('/health')
    assert response.status_code == 200

def test_predict_endpoint(client):
    with open('test_image.jpg', 'rb') as f:
        response = client.post('/predict',
                              data={'file': f})
    assert response.status_code == 200
```

### 手动测试

使用 cURL 测试：
```bash
# 健康检查
curl http://localhost:5000/health

# 模型信息
curl http://localhost:5000/info

# 预测
curl -X POST \
  -F "file=@dog.jpg" \
  http://localhost:5000/predict
```

---

## 常见陷阱

### 1. 模型未处于评估模式
```python
# 错误
model = models.resnet50(pretrained=True)

# 正确
model = models.resnet50(pretrained=True)
model.eval()  # 禁用 dropout 和 batch normalization
```

### 2. 错误的 Tensor 维度
```python
# 错误 - 缺少批次维度
tensor = transform(image)  # 形状: (3, 224, 224)
output = model(tensor)     # 错误！

# 正确 - 添加批次维度
tensor = transform(image).unsqueeze(0)  # 形状: (1, 3, 224, 224)
output = model(tensor)  # 正常工作！
```

### 3. 忘记使用 torch.no_grad()
```python
# 错误 - 浪费内存跟踪梯度
output = model(tensor)

# 正确 - 禁用梯度计算
with torch.no_grad():
    output = model(tensor)
```

### 4. 未将图片转换为 RGB
```python
# 错误 - 可能是灰度图或 RGBA
image = Image.open(file)

# 正确 - 确保为 RGB
image = Image.open(file).convert('RGB')
```

### 5. 硬编码值
```python
# 错误
PORT = 5000
MODEL_NAME = "resnet50"

# 正确
from config import Config
config = Config()
PORT = config.PORT
MODEL_NAME = config.MODEL_NAME
```

---

## 性能提示

### 1. 仅加载一次模型
```python
# 错误 - 每次请求都加载模型（非常慢！）
@app.route('/predict', methods=['POST'])
def predict():
    model = load_model()  # 不要这样做

# 正确 - 启动时仅加载一次
model_loader = ModelLoader()
model_loader.load()  # 初始化期间仅一次

@app.route('/predict', methods=['POST'])
def predict():
    predictions = model_loader.predict(image)  # 快速
```

### 2. 测量延迟
```python
import time

start_time = time.time()
predictions = model_loader.predict(image)
latency_ms = (time.time() - start_time) * 1000
logger.info(f"Prediction latency: {latency_ms:.2f}ms")
```

### 3. 高效的图片加载
```python
# 直接从字节加载（无磁盘 I/O）
import io
from PIL import Image

file_bytes = request.files['file'].read()
image = Image.open(io.BytesIO(file_bytes))
```

---

## 后续步骤

1. **实现 config.py** - 从配置开始
2. **实现 model_loader.py** - 构建核心 ML 功能
3. **独立测试 model_loader** - 确保在 API 之前正常工作
4. **实现 app.py** - 构建 REST API
5. **本地测试** - 运行服务器并使用 cURL/Postman 测试
6. **编写单元测试** - 确保代码质量
7. **容器化** - 进入 Docker 阶段

---

## 获取帮助

- 仔细阅读每个存根文件中的 TODO 注释
- 参考 architecture.md 了解系统设计
- 查看官方文档（Flask、PyTorch 等）
- 在进入下一步之前测试每个组件
- 开发时大量使用 print 语句
- 遇到困难时在答疑时间寻求帮助

---

**祝你实现顺利！**

**文档版本：** 1.0
**最后更新：** 2025 年 10 月
