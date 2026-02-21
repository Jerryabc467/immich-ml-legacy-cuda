# Immich Machine Learning - Legacy GPU Fix (ONNX Downgrade)

本项目提供了一个定制的 Dockerfile，用于为 [Immich](https://immich.app/) 的机器学习容器（Machine Learning）降级 ONNX Runtime。

专为优化 **NVIDIA GTX 750 Ti、GTX 900 系列、早期 Quadro** 等老旧架构/小显存显卡运行而制作。

## ⚠️ 为什么需要这个项目？

如果你在老旧显卡上部署 Immich 时遇到了以下报错：
1. `cudaErrorNoKernelImageForDevice` (在 Resize 节点崩溃)
2. `Could not find an implementation for Reshape(19) node`
3. 显卡驱动正常，但容器无限重启 (`fetch failed`)

**原因在于：** Immich 官方镜像使用的 `onnxruntime-gpu >= 1.15` 正式移除了对 Compute Capability 5.0 / 5.2 (Maxwell 架构) 的完整支持。此外，新版模型（如 Opset 19 导出的 CLIP 模型）也无法在老显卡上运行。

**本项目的解决方案：**
- 强制将环境降级到 **Python 3.10**。
- 强制安装 **ONNX Runtime 1.14.1**（支持老架构）。
- 通过修改源码配置，绕过 Immich 对 Python 3.11+ 的硬性依赖拦截。

## 🛠️ 适用硬件

- Compute Capability 5.0 以下的老旧 NVIDIA 显卡。
- 典型的“亮机卡”：GTX 750 Ti, GTX 960 等 2GB 显存设备。

## 🚀 如何使用

### 1. 构建镜像

克隆本仓库，并在包含 `Dockerfile` 的目录下执行构建命令。由于网络原因，构建可能需要几分钟。

```bash
docker build --no-cache -t immich-ml-legacy:latest .

### 2. 修改 docker-compose.yml

打开你的 Immich docker-compose.yml 文件，将官方的机器学习镜像替换为刚刚构建的本地镜像，并务必添加针对小显卡的资源限制。
