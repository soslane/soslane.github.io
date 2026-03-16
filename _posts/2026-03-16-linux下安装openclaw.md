---
layout:       post
title:        "Linux下安装openclaw"
author:       "Gweek"
catalog:      true
tags:
    - 人工智能
---


一、先准备系统环境

更新系统：

sudo apt update
sudo apt upgrade -y

安装常用工具：

sudo apt install -y curl git build-essential


⸻

二、安装 Docker（强烈建议）

OpenClaw 通常 Docker 部署最稳。

安装 Docker：

curl -fsSL https://get.docker.com | sh

把当前用户加入 docker 组：

sudo usermod -aG docker $USER

重新登录一次 SSH。

测试：

docker --version

如果看到类似：

Docker version 26.x

说明成功。

⸻

三、安装 Docker Compose

Ubuntu 22.04 推荐：

sudo apt install docker-compose-plugin -y

测试：

docker compose version


⸻

四、安装 Ollama（本地模型）

安装 Ollama

curl -fsSL https://ollama.com/install.sh | sh

安装完成后：

启动服务：

ollama serve

测试：

ollama run qwen2:7b

或：

ollama run deepseek-r1:8b

如果正常，你会看到模型开始下载。

常见模型：

模型	命令
Qwen	ollama run qwen2:7b
DeepSeek	ollama run deepseek-r1:8b
Llama3	ollama run llama3:8b


⸻

五、让 Ollama 允许外部访问（OpenClaw 需要）

默认 Ollama 只监听：

127.0.0.1:11434

编辑：

sudo nano /etc/systemd/system/ollama.service

找到：

ExecStart=/usr/local/bin/ollama serve

改成：

Environment="OLLAMA_HOST=0.0.0.0:11434"
ExecStart=/usr/local/bin/ollama serve

然后：

sudo systemctl daemon-reload
sudo systemctl restart ollama

测试：

curl http://127.0.0.1:11434


⸻

六、部署 OpenClaw

获取源码：

git clone https://github.com/OpenClawAI/OpenClaw.git

进入目录：

cd OpenClaw

查看 docker compose：

docker-compose.yml

启动：

docker compose up -d

等待容器启动。

查看状态：

docker ps


⸻

七、连接 Ollama

OpenClaw Web UI 打开：

http://服务器IP:3000

设置模型：

provider: ollama
endpoint: http://host.docker.internal:11434
model: qwen2:7b

如果在 Linux Docker：

改成：

http://172.17.0.1:11434

或者：

http://你的服务器IP:11434


⸻

八、GPU 加速（如果有 NVIDIA）

如果你有显卡（比如你之前说的 4060 / RX580）：

NVIDIA：

安装：

sudo apt install nvidia-driver-535

然后：

sudo apt install nvidia-container-toolkit

重启 Docker：

sudo systemctl restart docker

Ollama 会自动检测 GPU。

⸻

九、推荐的生产架构

很多人实际部署是：

Nginx
   │
OpenClaw
   │
Ollama
   │
本地模型

结构：

Internet
   │
Nginx
   │
OpenClaw
   │
Ollama API
   │
DeepSeek / Qwen


⸻

十、一个非常重要的优化

如果你打算长期运行 Agent：

建议安装：
	•	Redis
	•	PostgreSQL

因为：

OpenClaw 会：
	•	存储任务
	•	存储记忆
	•	存储工具调用

