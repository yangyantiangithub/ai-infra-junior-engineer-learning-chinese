# Module 005 — Docker & Containers Resources

## Official documentation

- **Docker documentation** — [docs.docker.com](https://docs.docker.com/). 权威参考。
- **Dockerfile best practices** — [docs.docker.com/develop/develop-images/dockerfile_best-practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/). 请务必阅读；你将在职业生涯中不断编写 Dockerfile。
- **OCI Image Format Specification** — [github.com/opencontainers/image-spec](https://github.com/opencontainers/image-spec). Docker image 底层的标准规范。

## Books

- **Docker: Up & Running (3rd ed.)** by Sean Kane and Karl Matthias. 标准实用参考书。
- **Container Security** by Liz Rice (O'Reilly). 简洁而现代，覆盖生产环境所需的安全领域。
- **Docker Deep Dive** by Nigel Poulton. 最新，覆盖现代 Docker 生态。

## Online courses

- **Docker Mastery (Bret Fisher)** on Udemy. 最受推荐的 Docker 课程。
- **Play with Docker** — [labs.play-with-docker.com](https://labs.play-with-docker.com/). 免费的基于浏览器的 Docker 沙箱。

## Key tools beyond Docker

- **BuildKit** — [docs.docker.com/build/buildkit](https://docs.docker.com/build/buildkit/). 现代 Docker 构建引擎。快速多阶段构建、构建时密钥、多架构支持。
- **buildx** — Docker CLI 的 BuildKit 插件；支持多平台构建。
- **Trivy** — [trivy.dev](https://trivy.dev/). Image 漏洞 + SBOM 扫描。应在每次构建时运行。
- **dive** — [github.com/wagoodman/dive](https://github.com/wagoodman/dive). 逐层检查 Docker image 内容。
- **hadolint** — [github.com/hadolint/hadolint](https://github.com/hadolint/hadolint). Dockerfile linter。

## ML-specific container references

- **NVIDIA NGC Catalog** — [catalog.ngc.nvidia.com](https://catalog.ngc.nvidia.com/). NVIDIA 官方整理的 GPU 就绪 container image，适用于 PyTorch、TensorFlow、CUDA。将这些作为 ML 工作负载的基础 image。
- **PyTorch official images** — [hub.docker.com/r/pytorch/pytorch](https://hub.docker.com/r/pytorch/pytorch).
- **Chainguard images** — [chainguard.dev/chainguard-images](https://www.chainguard.dev/chainguard-images). 无发行版、最小化、签名的基础 image。适用于生产加固。

## Container alternatives + adjacent

- **Podman** — [podman.io](https://podman.io/). 无守护进程的 Docker 替代品；默认以 rootless 运行。可替代大多数 Docker 命令。
- **containerd** — [containerd.io](https://containerd.io/). Docker 本身底层使用的 container runtime；也是 Kubernetes 所使用的。
- **Buildah / Skopeo** — Red Hat 的 container 工具，无需守护进程即可工作。

## Production patterns

- **12-Factor App** — [12factor.net](https://12factor.net/). 云原生应用的 12 条原则。大多直接适用于 containerized ML 工作负载。
- **Docker security cheat sheet** — [cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html).

## Cross-references in this curriculum

- Module 006 (Kubernetes) — 编排你的 container。
- Module 007 (APIs) — Containerize 你的 ML serving API。
- Engineer track 的 `mod-103-containerization` 用于生产级 Docker 模式。
- Security track 的 `mod-008-runtime-security` 用于 container 运行时安全。
- Security track 的 `mod-010-supply-chain-security` 用于 image 签名 + SBOM。
