---
layout:       post
title:        "Docker All in one Ollama+Dify"
author:       "Gweek"
catalog:      true
tags:
    - 人工智能
---



> 配置：i7 12700kf 内存 32GB 显卡 4060 8GB ，ollama和dify均在docker中运行，并且能够显卡直通，以达到最佳的性能，同时迁移也更方便。

* **Docker Desktop** 负责运行所有服务
* **Ollama** 负责推理模型
* **Qwen3.5 9B** 作为大模型
* **Dify** 负责 AI 应用平台
* **NVIDIA CUDA** 提供 GPU 加速


### 第一步：准备 GPU 环境与 Docker 底层迁移 (关键！)

**1. 安装 NVIDIA 驱动**
前往 NVIDIA 官网安装最新 CUDA 驱动。打开 PowerShell 输入 `nvidia-smi`，确认能看到你的 RTX 4060。

**2. 安装 WSL2 与 Docker Desktop**

* 以管理员身份打开 PowerShell，执行 `wsl --install`，按提示重启。
* 安装 Docker Desktop，确保在设置中开启 **Use WSL 2 instead of Hyper-V**。
* 启动 Docker Desktop 完成初始化后，**彻底退出 Docker**（右下角系统托盘右键 -> Quit）。

**3. 将 WSL 虚拟磁盘迁移到 D 盘 (释放 C 盘)**
这是防止 C 盘爆满的核心操作。以管理员身份打开 PowerShell，依次执行：

```powershell
# 关闭所有 WSL 实例
wsl --shutdown

# 导出 Docker 默认数据盘到 D 盘（请提前在 D 盘创建 D:\Docker\wsl 目录）
wsl --export docker-desktop-data D:\Docker\wsl\docker-desktop-data.tar
wsl --export docker-desktop D:\Docker\wsl\docker-desktop.tar

# 注销 C 盘的原有数据
wsl --unregister docker-desktop-data
wsl --unregister docker-desktop

# 重新将数据导入为 D 盘的虚拟磁盘
wsl --import docker-desktop-data D:\Docker\wsl\data D:\Docker\wsl\docker-desktop-data.tar --version 2
wsl --import docker-desktop D:\Docker\wsl\distro D:\Docker\wsl\docker-desktop.tar --version 2

```

*导入完成后，你可以安全地删除那两个 `.tar` 压缩包，然后重新启动 Docker Desktop。*

---

### 第二步：在 D 盘构建统一工作区与部署 Ollama

我们在 D 盘建立一个统一的文件夹，方便日后一键打包迁移。

**1. 创建 D 盘工作区**
在 D 盘创建以下目录结构：

```text
D:\ai-platform
 ├─ ollama-data     (用于存放模型)
 └─ docker-compose.yml

```

**2. 编写优化版的 docker-compose.yml**
在 `D:\ai-platform` 下创建 `docker-compose.yml`，使用最新的 GPU 调用语法：

```yaml
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: always
    ports:
      - "11434:11434"
    volumes:
      # 直接将模型映射到 D 盘当前目录的 ollama-data 文件夹下
      - ./ollama-data:/root/.ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]

```

**3. 启动并下载模型**
在 PowerShell 中进入该目录并启动：

```powershell
cd D:\ai-platform
docker compose up -d

```

进入容器下载 Qwen2.5 7B 模型（你的 D 盘 `ollama-data` 文件夹会开始变大）：

```powershell
docker exec -it ollama ollama run qwen2.5:7b

```

---

### 第三步：在 D 盘部署 Dify

因为我们已经把 Docker 底层的 WSL 数据放到了 D 盘，所以 Dify 内部产生的数据库文件（PostgreSQL、Redis 等）也会默认安全地存储在 D 盘。

**1. 下载 Dify 源码到 D 盘**
保持在 `D:\ai-platform` 目录下：

```powershell
cd D:\ai-platform
git clone https://github.com/langgenius/dify.git

```

**2. 启动 Dify**

```powershell
cd dify/docker
copy .env.example .env
docker compose up -d

```

*首次启动会拉取大量镜像，由于底层已迁移，你的 C 盘容量不会有任何波动。*

---

### 第四步：Dify 连接 Ollama (保持原样，非常完美)

1. 浏览器访问 `http://localhost` 初始化 Dify 管理员账号。
2. 进入 **设置 → 模型供应商 (Model Provider)**。
3. 选择 **Ollama**，填写如下参数：
* **模型名称**: `qwen2.5:7b` *(必须与刚才下载的名字严格一致)*
* **基础 URL**: `http://host.docker.internal:11434`
* **模型类型**: `LLM`



---

### 第五步：性能与优化总结

对于 **i7-12700KF + 32G + 4060 8GB**，跑 7B-9B 级别的模型是甜点区间。

| 模式 | 预估速度 (Qwen2.5 7B) | 显存占用预估 |
| --- | --- | --- |
| **纯 GPU 推理 (当前架构)** | 35~50 tokens/s | 约 5GB - 6GB |
| 纯 CPU 推理 | 5~8 tokens/s | - |

**优化建议：**
Ollama 默认会自动检测并尽最大可能将模型层（Layers）加载到 VRAM 中。因为你的显存（8GB）完全装得下量化后的 7B 模型，**你无需手动强制设置 `OLLAMA_GPU_LAYERS**`，让 Ollama 的自动调度器接管，反而能获得最稳定的性能，避免显存溢出（OOM）导致的崩溃。
