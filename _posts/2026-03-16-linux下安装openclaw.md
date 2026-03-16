---
layout:       post
title:        "Linux下安装openclaw"
author:       "Gweek"
catalog:      true
tags:
    - 人工智能
---



> - Ubuntu 22.04
> - 普通用户 + sudo
> - 有 GPU 更好（但不是必须）

---

# 一、先准备系统环境

更新系统：

```bash
sudo apt update
sudo apt upgrade -y
```

安装常用工具：

```bash
sudo apt install -y curl git build-essential
```

---

# 二、安装 Docker（强烈建议）

OpenClaw 通常 **Docker 部署最稳**。

安装 Docker：

```bash
curl -fsSL https://get.docker.com | sh
```

把当前用户加入 docker 组：

```bash
sudo usermod -aG docker $USER
```

重新登录一次 SSH。

测试：

```bash
docker --version
```

如果看到类似：

```
Docker version 26.x
```

说明成功。

---

# 三、安装 Docker Compose

Ubuntu 22.04 推荐：

```bash
sudo apt install docker-compose-plugin -y
```

测试：

```bash
docker compose version
```

---

# 四、安装 Ollama（本地模型）

安装 ollama

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

安装完成后：

启动服务：

```bash
ollama serve
```

测试：

```bash
ollama run qwen2:7b
```

或：

```bash
ollama run deepseek-r1:8b
```

如果正常，你会看到模型开始下载。

常见模型：

| 模型 | 命令 |
|---|---|
| Qwen | `ollama run qwen2:7b` |
| DeepSeek | `ollama run deepseek-r1:8b` |
| Llama3 | `ollama run llama3:8b` |

---

# 五、让 Ollama 允许外部访问（OpenClaw 需要）

默认 Ollama 只监听：

```
127.0.0.1:11434
```

编辑：

```bash
sudo nano /etc/systemd/system/ollama.service
```

找到：

```
ExecStart=/usr/local/bin/ollama serve
```

改成：

```
Environment="OLLAMA_HOST=0.0.0.0:11434"
ExecStart=/usr/local/bin/ollama serve
```

然后：

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

测试：

```bash
curl http://127.0.0.1:11434
```

---

# 六、部署 OpenClaw

获取源码：

```bash
git clone https://github.com/OpenClawAI/OpenClaw.git
```

进入目录：

```bash
cd OpenClaw
```

查看 docker compose：

```
docker-compose.yml
```

启动：

```bash
docker compose up -d
```

等待容器启动。

查看状态：

```bash
docker ps
```

---

# 七、连接 Ollama

OpenClaw Web UI 打开：

```
http://服务器IP:3000
```

设置模型：

```
provider: ollama
endpoint: http://host.docker.internal:11434
model: qwen2:7b
```

如果在 Linux Docker：

改成：

```
http://172.17.0.1:11434
```

或者：

```
http://你的服务器IP:11434
```

---

# 八、GPU 加速（如果有 NVIDIA）


NVIDIA：

安装：

```bash
sudo apt install nvidia-driver-535
```

然后：

```bash
sudo apt install nvidia-container-toolkit
```

重启 Docker：

```bash
sudo systemctl restart docker
```

Ollama 会自动检测 GPU。

---

# 九、推荐的生产架构

很多人实际部署是：

```
Nginx
   │
OpenClaw
   │
Ollama
   │
本地模型
```

结构：

```
Internet
   │
Nginx
   │
OpenClaw
   │
Ollama API
   │
DeepSeek / Qwen
```

---

# 十、一个非常重要的优化

如果你打算长期运行 Agent：

建议安装：

- [Redis](chatgpt://generic-entity?number=4)
- [PostgreSQL](chatgpt://generic-entity?number=5)

因为：

OpenClaw 会：

- 存储任务
- 存储记忆
- 存储工具调用

---
