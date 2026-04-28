# Hunyuan3D 2.1 DGX Spark Agent Runbook

本文档给后续 AI agent 快速接手使用。当前部署目标是 NVIDIA DGX Spark / GB10 / aarch64 / Ubuntu 24.04，通过 Docker 运行 Hunyuan3D 2.1 Web UI。

## 当前结论

- 服务已经部署成功，访问地址：`http://127.0.0.1:7860`
- 部署目录：`/home/dgx/github/hunyuan3d-2.1-dgx-spark`
- 容器名：`hy3d`
- 镜像名：`hunyuan3d-21-dgx-spark-hy3d:latest`
- 镜像大小约 `23.4GB`
- Hugging Face 模型缓存目录：`/home/dgx/github/hunyuan3d-2.1-dgx-spark/hf_cache`
- RealESRGAN 权重目录：`/home/dgx/github/hunyuan3d-2.1-dgx-spark/hy3dpaint/ckpt`

不要优先使用官方仓库自带 Dockerfile。官方 Dockerfile 偏向 x86_64 + CUDA 12.4，本机是 aarch64 + GB10 `sm_121` + CUDA 13。当前可用方案来自 DGX Spark 专用 fork。

## 来源与版本

外层 Docker 项目：

```bash
cd /home/dgx/github/hunyuan3d-2.1-dgx-spark
git log -1 --oneline --decorate
# 6ee8108 (HEAD -> main, origin/main, origin/HEAD) Documentation updated
```

Hunyuan3D 源码 fork：

```bash
cd /home/dgx/github/hunyuan3d-2.1-dgx-spark/Hunyuan3D-2.1-DGX
git log -1 --oneline --decorate
# 53afbce (HEAD -> DGX-Spark, origin/DGX-Spark) gitignore fixed
```

参考来源：

- `https://forums.developer.nvidia.com/t/dgx-spark-hunyuan3d-2-1/353211`
- `https://github.com/dr-vij/Hunyuan3D-2.1-DGX-Spark-Docker`
- `https://github.com/dr-vij/Hunyuan3D-2.1-DGX`

本机关键硬件：

```bash
nvidia-smi --query-gpu=name,driver_version,compute_cap --format=csv,noheader
# NVIDIA GB10, 580.126.09, 12.1
```

## 关键本地修改

`docker-compose.yml` 已从原始端口映射改为 `network_mode: host`，并透传本机 HTTP/HTTPS 代理：

```yaml
network_mode: host
environment:
  HF_HOME: /workspace/cache/huggingface
  HTTP_PROXY: http://127.0.0.1:10808/
  HTTPS_PROXY: http://127.0.0.1:10808/
  http_proxy: http://127.0.0.1:10808/
  https_proxy: http://127.0.0.1:10808/
```

原因：容器直连 `huggingface.co` 会超时；宿主机通过 `127.0.0.1:10808` 代理访问正常。使用 host 网络后，容器里的 `127.0.0.1:10808` 才能指向宿主机代理。

不要加入 `ALL_PROXY=socks5://127.0.0.1:10808` 或 `all_proxy=socks5://127.0.0.1:10808`，除非同时安装 `socksio`。否则 Gradio/httpx 会报错：

```text
ImportError: Using SOCKS proxy, but the 'socksio' package is not installed.
```

## 安装或重建

首次部署或需要重建镜像时执行：

```bash
cd /home/dgx/github/hunyuan3d-2.1-dgx-spark
docker compose build
```

构建会做这些事：

- 使用 `nvcr.io/nvidia/cuda:13.0.2-devel-ubuntu24.04`
- 从源码编译 Python `3.10.19`
- 从源码编译 Blender/bpy
- 安装 PyTorch `2.11.0+cu130`
- 安装 Hunyuan3D 依赖
- 编译 `custom_rasterizer`，目标包含 GB10 `compute_121/sm_121`
- 编译 `mesh_inpaint_processor`

构建很慢是正常的，尤其是 Python PGO 编译和 Blender/bpy 编译。成功后再次启动不需要重复构建。

## 启动、停止、查看日志

启动：

```bash
cd /home/dgx/github/hunyuan3d-2.1-dgx-spark
docker compose up -d
```

停止：

```bash
cd /home/dgx/github/hunyuan3d-2.1-dgx-spark
docker compose down
```

查看状态：

```bash
cd /home/dgx/github/hunyuan3d-2.1-dgx-spark
docker compose ps
```

查看日志：

```bash
cd /home/dgx/github/hunyuan3d-2.1-dgx-spark
docker compose logs -f hy3d
```

访问：

```text
http://127.0.0.1:7860
```

`.env` 当前内容：

```dotenv
UID=1000
GID=1000
GRADIO_SERVER_HOST=0.0.0.0
GRADIO_SERVER_PORT=7860
```

如果 7860 被占用，修改 `.env` 的 `GRADIO_SERVER_PORT`，然后 `docker compose down && docker compose up -d`。

## 首次启动下载

首次启动会下载：

- `RealESRGAN_x4plus.pth`，约 `64MB`，保存到 `hy3dpaint/ckpt`
- `rembg` 的 `u2net.onnx`，约 `176MB`，容器内缓存
- Hunyuan3D 贴图模型，约数 GB
- DINOv2 giant 图像编码器，约 `4.55GB`
- Hunyuan3D shape 主权重 `model.fp16.ckpt`，约 `7.37GB`

当前缓存已存在，`hf_cache` 约 `11GB`，后续重启通常不再重新下载。

Hugging Face/Xet 多文件下载时，日志有时长时间显示 `0%`，但实际缓存文件在增长。不要只看日志百分比，确认缓存增长：

```bash
cd /home/dgx/github/hunyuan3d-2.1-dgx-spark
du -sh hf_cache
find hf_cache -type f -printf '%s %p\n' | sort -nr | head
```

## 验证命令

页面健康检查：

```bash
curl -L -o /tmp/hy3d-index.html -w 'HTTP %{http_code} size %{size_download}\n' http://127.0.0.1:7860/
# 期望：HTTP 200，下载约 145KB HTML
```

容器内依赖和 CUDA 检查：

```bash
docker exec hy3d bash -lc '
cd /workspace/Hunyuan3D-2.1-DGX
export PYTHONPATH="hy3dpaint/DifferentiableRenderer:hy3dpaint/custom_rasterizer:$PYTHONPATH"
python - << "PY"
import torch, bpy, xatlas, pymeshlab, custom_rasterizer_kernel, mesh_inpaint_processor
print("torch", torch.__version__)
print("cuda", torch.cuda.is_available(), torch.cuda.get_device_name(0), torch.cuda.get_device_capability(0))
print("imports ok")
PY'
```

当前成功输出关键项：

```text
torch 2.11.0+cu130
cuda True NVIDIA GB10 (12, 1)
imports ok
```

GPU 占用检查：

```bash
nvidia-smi --query-compute-apps=pid,process_name,used_memory --format=csv,noheader,nounits
```

服务加载完成后，`python` 主进程约占 `14GB` GPU。运行生成任务时显存会继续上升。

## 常见问题

### Hugging Face 连接超时

现象：

```text
Connection to huggingface.co timed out
```

处理：

1. 确认宿主机代理在监听：

```bash
ss -ltnp | rg ':10808\b'
```

2. 确认宿主机可访问 Hugging Face：

```bash
curl -I -L https://huggingface.co
```

3. 确认容器使用 host 网络且有 HTTP/HTTPS 代理：

```bash
docker exec hy3d env | grep -i proxy
docker exec hy3d curl -I -L https://huggingface.co
```

### Gradio 启动时报 socksio 缺失

现象：

```text
ImportError: Using SOCKS proxy, but the 'socksio' package is not installed.
```

处理：从 `docker-compose.yml` 删除 `ALL_PROXY` 和 `all_proxy`，只保留 `HTTP_PROXY` / `HTTPS_PROXY`。

### `mesh_inpaint_processor` 单独 import 失败

它不是安装在全局 site-packages，而是项目目录 `.so`。验证时必须追加：

```bash
export PYTHONPATH="hy3dpaint/DifferentiableRenderer:hy3dpaint/custom_rasterizer:$PYTHONPATH"
```

### `bpy` 单独 import 失败

不要覆盖 Dockerfile 设置的 `PYTHONPATH`。容器里原本应包含：

```text
/workspace/blender_dev/build_bpy/bin
```

如果要追加项目路径，用 `:$PYTHONPATH`，不要直接替换。

### LM Studio 或其他模型占用 GPU

部署前曾停止 LM Studio 的模型 worker。若再次显存不足，先看 GPU 进程：

```bash
nvidia-smi --query-compute-apps=pid,process_name,used_memory --format=csv,noheader,nounits
```

如果 LM Studio 的 `llmworker` 占用几十 GB，可先在 LM Studio 里卸载模型，或温和停止对应 worker。不要误杀当前 `hy3d` 容器的 `python` 进程。

## 不要遗忘的约束

- 当前方案依赖本机代理端口 `127.0.0.1:10808`。
- 当前 compose 使用 `network_mode: host`，因此不会显示普通 `PORTS` 映射；这是预期。
- 不要用官方 x86_64 Dockerfile 替换当前 Dockerfile。
- 不要随意升级 PyTorch 或 CUDA；当前已验证的是 `torch 2.11.0+cu130` + GB10 `sm_121`。
- 不要删除 `hf_cache`，否则下次启动要重新下载大模型。
- 不要删除 `hy3dpaint/ckpt/RealESRGAN_x4plus.pth`，否则下次启动会重新从 GitHub release 下载。
- 腾讯 Hunyuan3D 模型许可证包含非商业/社区许可限制，使用前需要遵守对应条款。
