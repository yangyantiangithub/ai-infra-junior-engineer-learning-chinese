# 模块 007 — API 和 Web 服务资源

## 官方文档

- **FastAPI 文档** — [fastapi.tiangolo.com](https://fastapi.tiangolo.com/)。增长最快的 Python Web 框架；原生异步；优秀的文档同时可作为教程。
- **Uvicorn 文档** — [uvicorn.org](https://www.uvicorn.org/)。运行 FastAPI 所需的 ASGI 服务器。
- **Pydantic 文档** — [docs.pydantic.dev](https://docs.pydantic.dev/)。Schema 验证。如果你是新手，请阅读迁移指南（v1 与 v2 的差异）。
- **HTTP/1.1 规范（RFC 7230-7235）** — 必读一次；你以后会经常查阅。

## Web 框架对比

- **FastAPI** — 现代、异步优先、对 ML 友好。新 ML API 的推荐默认选择。
- **Flask** — 较老、同步、更轻量。在遗留 ML 代码库中常见。
- **Django** — 功能齐全，通常对于 ML 服务来说过于庞大。适合通用 Web 应用。
- **Starlette** — FastAPI 构建于其之上的框架。如果你想要最底层的异步工具包，可直接使用。

## 书籍

- **《Designing Web APIs》** 作者 Brenda Jin 等（O'Reilly）。良好的通用设计原则。
- **《API Design Patterns》** 作者 JJ Geewax。全面的模式参考。
- **《High-Performance Python（第二版）》** 作者 Micha Gorelick 和 Ian Ozsvald。涵盖生产环境 Python 中的异步和并发。

## API 设计参考

- **REST API 设计规则手册** — [restful-api-design.readthedocs.io](https://restful-api-design.readthedocs.io/)。
- **JSON:API 规范** — [jsonapi.org](https://jsonapi.org/)。JSON API 设计规范。
- **Google API 设计指南** — [cloud.google.com/apis/design](https://cloud.google.com/apis/design)。实用。
- **GraphQL** — [graphql.org](https://graphql.org/)。REST 的替代方案。适合复杂的客户端需求；对于简单的 ML 服务来说过度了。
- **gRPC** — [grpc.io](https://grpc.io/)。二进制协议 RPC。用于高吞吐量的内部服务间通信。

## ML 特定 API 模式

- **TorchServe 文档** — [pytorch.org/serve](https://pytorch.org/serve/)。PyTorch 的服务框架。
- **TensorFlow Serving** — [tensorflow.org/tfx/serving](https://www.tensorflow.org/tfx/serving)。TensorFlow 的服务框架。
- **BentoML** — [bentoml.com](https://www.bentoml.com/)。更高级的 ML 服务框架。
- **vLLM** — [vllm.ai](https://vllm.ai/)。LLM 服务的标准方案（PagedAttention、continuous batching）。
- **Triton Inference Server** — [github.com/triton-inference-server/server](https://github.com/triton-inference-server/server)。NVIDIA 的高性能服务运行时。

## 异步和并发

- **asyncio 文档** — [docs.python.org/3/library/asyncio.html](https://docs.python.org/3/library/asyncio.html)。
- **httpx** — [www.python-httpx.org](https://www.python-httpx.org/)。异步 HTTP 客户端。在异步代码中替代 `requests`。
- **aiohttp** — 较老的异步 HTTP 库；仍然常见。

## 监控和可观测性

- **prometheus_client** — [github.com/prometheus/client_python](https://github.com/prometheus/client_python)。Prometheus 的 Python 仪表化。
- **OpenTelemetry Python** — [opentelemetry.io/docs/instrumentation/python](https://opentelemetry.io/docs/instrumentation/python/)。分布式追踪标准。
- **Sentry** — [sentry.io/for/python](https://sentry.io/for/python/)。错误追踪，与 FastAPI 集成良好。

## 安全

- **OWASP API 安全 Top 10** — [owasp.org/www-project-api-security](https://owasp.org/www-project-api-security/)。任何生产 API 的必读内容。
- **FastAPI 安全** — [fastapi.tiangolo.com/tutorial/security](https://fastapi.tiangolo.com/tutorial/security/)。涵盖的认证模式。

## 本课程中的交叉引用

- 模块 001（Python）和模块 005（Docker）是基础。
- 工程师轨道的 `mod-101 exercise-08` 涵盖生产级模型服务。
- 工程师轨道的 `mod-110-llm-infrastructure` 深入涵盖 LLM 服务。
- 安全轨道的 `mod-004-network-security` 涵盖网关加固。
