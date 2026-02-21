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
```

### 2. 修改 docker-compose.yml

打开你的 Immich docker-compose.yml 文件，将官方的机器学习镜像替换为刚刚构建的本地镜像，并务必添加针对小显存显卡的资源限制。

Docker-Compose机器学习部分
```
immich-machine-learning:
    container_name: immich_machine_learning
    image: immich-ml-legacy:latest  # 使用我们刚刚构建的镜像
    volumes:
      - model-cache:/cache
    env_file:
      - .env
    environment:
      # --- 2GB 显卡显存优化 ---
      - MACHINE_LEARNING_WORKERS=1             # 禁止多线程
      - MACHINE_LEARNING_MAX_JOBS_PER_WORKER=1 # 每次只处理1个任务
      - MACHINE_LEARNING_MODEL_TTL=10          # 激进的显存释放策略 (10秒不用卸载)
      - MACHINE_LEARNING_ANN_FP16_TURBO=false  # 老架构跑半精度容易出问题，建议关闭
      - DEVICE=cuda
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    restart: always
```

### 3. 在 Web UI 中更换兼容模型（至关重要！）

老显卡不仅驱动旧，通常显存也很小（2GB）。必须在 Immich 网页端 (Administration -> Settings -> Machine Learning Settings) 中更换为小巧且兼容的模型，否则会报 CUBLAS_STATUS_ALLOC_FAILED (显存溢出) 错误。

建议使用以下配置组合
Smart Search (CLIP) | ViT-B-32__openai | 兼容旧版 ONNX，且显存占用小 ~400MB
Facial Recognition  | buffalo_s        | Small 版本，比默认的 buffalo_l 节省显存
OCR                 | DuOCR            | 新版 OCR 不支持旧架构

提示： 如果你的 Web UI 无法直接修改模型，可以通过 psql 修改 immich 数据库中的 system_config 表，或使用immich提供的API接口进行修改

### 🤝 贡献与免责声明
本项目仅为社区解决老硬件兼容性提供的一个 Workaround。随 Immich 官方版本的迭代，代码可能会产生变化，如遇构建失败请提交 Issue 或使用ChatGPT等工具自行修改。
