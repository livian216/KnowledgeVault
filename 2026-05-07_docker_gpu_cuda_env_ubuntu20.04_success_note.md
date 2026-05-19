# Ubuntu 20.04 服务器 Docker + NVIDIA GPU + CUDA 训练环境配置备忘

## 1. 文档目的

本文记录在一台 x86_64 Ubuntu 20.04 服务器上，配置 Docker + NVIDIA GPU 容器运行环境的成功步骤。

配置完成后，可以在 Docker 容器内调用宿主机 NVIDIA 显卡，用于后续 PyTorch、YOLO、目标检测、语义分割等模型训练任务。

---

## 2. 服务器环境假设

本文适用于以下环境：

- 操作系统：Ubuntu 20.04
- CPU 架构：x86_64
- 显卡：NVIDIA GPU，例如 RTX 2080 Ti
- Docker：已在宿主机安装完成
- 目标：Docker 容器内可通过 `--gpus all` 调用宿主机 NVIDIA GPU
- NVIDIA Container Toolkit 安装源：USTC 国内镜像源
- Docker 镜像拉取源：DaoCloud 国内公共镜像源

---

## 3. 相关网络地址（备份）

### 3.1 USTC NVIDIA Container Toolkit 镜像源

USTC libnvidia-container 镜像说明页面：

```text
https://mirrors.ustc.edu.cn/help/libnvidia-container.html
```

USTC libnvidia-container 镜像目录：

```text
https://mirrors.ustc.edu.cn/libnvidia-container/
```

USTC GPG key 地址：

```text
https://mirrors.ustc.edu.cn/libnvidia-container/gpgkey
```

USTC deb 源目录：

```text
https://mirrors.ustc.edu.cn/libnvidia-container/stable/deb/amd64/
```

### 3.2 DaoCloud Docker Hub 公共镜像源

DaoCloud 公共镜像源项目地址：

```text
https://github.com/DaoCloud/public-image-mirror
```

DaoCloud Docker Hub 镜像前缀：

```text
docker.m.daocloud.io
```

完整加速地址：

```text
https://docker.m.daocloud.io
```

---

## 4. 配置前检查

### 4.1 检查宿主机 NVIDIA 驱动

在宿主机执行：

```bash
nvidia-smi
```

如果能看到 NVIDIA 显卡信息，例如 RTX 2080 Ti，说明宿主机显卡驱动正常。

示例关键信息：

```text
NVIDIA-SMI xxx.xx.xx
Driver Version: xxx.xx.xx
CUDA Version: xx.x
GPU  Name
0    NVIDIA GeForce RTX 2080 Ti
```

> 注意：容器内通常不安装 NVIDIA 显卡驱动。显卡驱动安装在宿主机，容器通过 NVIDIA Container Toolkit 使用宿主机驱动能力。

### 4.2 检查 Docker

执行：

```bash
docker --version
```

查看 Docker 服务状态：

```bash
systemctl status docker
```

如果 Docker 未启动，可执行：

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

---

## 5. 安装 NVIDIA Container Toolkit

NVIDIA Container Toolkit 的作用是让 Docker 容器能够访问宿主机 NVIDIA GPU。

没有该工具时，即使宿主机 `nvidia-smi` 正常，容器也无法通过 `docker run --gpus all` 调用显卡。

### 5.1 安装基础工具

```bash
sudo apt-get update
sudo apt-get install -y curl ca-certificates gnupg
```

### 5.2 添加 USTC GPG key

```bash
curl -fsSL https://mirrors.ustc.edu.cn/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
```

如果服务器 IPv6 网络不稳定，可以使用 IPv4：

```bash
curl -4 -fsSL https://mirrors.ustc.edu.cn/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
```

检查 key 文件是否生成：

```bash
ls -lh /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
```

### 5.3 添加 USTC apt 源

```bash
cat <<'EOF' | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://mirrors.ustc.edu.cn/libnvidia-container/stable/deb/amd64 /
EOF
```

### 5.4 可选：强制 apt 使用 IPv4

如果服务器 IPv6 网络不可用，建议执行（正常来说，无需操作本步骤）：

```bash
echo 'Acquire::ForceIPv4 "true";' | sudo tee /etc/apt/apt.conf.d/99force-ipv4
```

### 5.5 安装 nvidia-container-toolkit

```bash
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
```

### 5.6 配置 Docker runtime

```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

### 5.7 检查安装结果

```bash
nvidia-ctk --version
nvidia-container-cli --version
```

能够正常输出版本号，说明 NVIDIA Container Toolkit 已安装。

---

## 6. 使用 DaoCloud 拉取 CUDA 测试镜像

由于默认 Docker Hub 在国内网络环境下可能较慢，建议使用 DaoCloud 镜像前缀拉取常用镜像。

### 6.1 拉取 CUDA 11.8 测试镜像（临时生效）

```bash
docker pull docker.m.daocloud.io/nvidia/cuda:11.8.0-base-ubuntu20.04
```

### 6.2 拉取 CUDA 12.1 测试镜像

如果需要 CUDA 12.1，也可以执行：

```bash
docker pull docker.m.daocloud.io/nvidia/cuda:12.1.1-base-ubuntu20.04
```

### 6.3 镜像说明

`nvidia/cuda:*base-ubuntu20.04` 属于 CUDA 基础运行镜像，适合用于 GPU 连通性测试。

正式训练时，通常建议使用 `devel` 或 `cudnn-devel` 镜像，例如：

```bash
docker pull docker.m.daocloud.io/nvidia/cuda:11.8.0-cudnn8-devel-ubuntu20.04
```

---

## 7. 验证 Docker 容器是否能调用 GPU （验证一个即可）

### 7.1 使用 CUDA 11.8 镜像验证

```bash
docker run --rm --gpus all \
  docker.m.daocloud.io/nvidia/cuda:11.8.0-base-ubuntu20.04 \
  nvidia-smi
```

### 7.2 使用 CUDA 12.1 镜像验证

```bash
docker run --rm --gpus all \
  docker.m.daocloud.io/nvidia/cuda:12.1.1-base-ubuntu20.04 \
  nvidia-smi
```

如果容器内部可以正常显示 NVIDIA 显卡信息，例如 RTX 2080 Ti，则说明：

- 宿主机 NVIDIA 驱动正常；
- Docker 正常；
- NVIDIA Container Toolkit 正常；
- Docker 容器可以访问宿主机 GPU；
- 后续可以在容器中配置 PyTorch、YOLO 等训练环境。

---

## 8. GPU 验证命令解释

示例命令：

```bash
docker run --rm --gpus all \
  docker.m.daocloud.io/nvidia/cuda:11.8.0-base-ubuntu20.04 \
  nvidia-smi
```

参数说明：

```text
docker run
```

启动一个新的 Docker 容器。

```text
--rm
```

容器运行结束后自动删除。该命令只是测试用途，不需要保留容器。

```text
--gpus all
```

将宿主机所有 NVIDIA GPU 暴露给容器。

```text
docker.m.daocloud.io/nvidia/cuda:11.8.0-base-ubuntu20.04
```

使用 DaoCloud 镜像源代理的 NVIDIA CUDA 镜像。

```text
nvidia-smi
```

容器启动后执行的命令，用于在容器内部查看 GPU 状态。

---

## 9. 可选：将 DaoCloud 配置为 Docker 默认镜像加速器

如果不想每次拉取镜像都手动添加 `docker.m.daocloud.io/` 前缀，可以配置 Docker 默认镜像加速器。

### 9.1 备份原 Docker 配置

```bash
sudo cp /etc/docker/daemon.json /etc/docker/daemon.json.bak.$(date +%F_%H%M%S) 2>/dev/null || true
```

### 9.2 写入 DaoCloud 镜像加速配置

```bash
sudo mkdir -p /etc/docker

cat <<'EOF' | sudo tee /etc/docker/daemon.json
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io"
  ]
}
EOF
```

### 9.3 重启 Docker

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### 9.4 检查配置是否生效

```bash
docker info | grep -A 10 "Registry Mirrors"
```

如果看到：

```text
https://docker.m.daocloud.io/
```

说明配置已生效。

此后可以尝试直接执行：

```bash
docker pull nvidia/cuda:11.8.0-base-ubuntu20.04
```

不过为了确保拉取走 DaoCloud，也可以继续显式使用完整前缀：

```bash
docker pull docker.m.daocloud.io/nvidia/cuda:11.8.0-base-ubuntu20.04
```

---

## 10. 创建后续训练开发容器示例

GPU 工具链验证完成后，可以创建一个长期使用的训练容器。

假设宿主机训练工程目录为：

```text
/home/livian/workspaces/smt_ai_workspace
```

建议先拉取训练用 CUDA + cuDNN 开发镜像：

```bash
docker pull docker.m.daocloud.io/nvidia/cuda:12.1.1-cudnn8-devel-ubuntu20.04
```

启动训练开发容器：

```bash
docker run --name livian-gpu-train \
  --gpus all \
  -p 7777:22 \
  --workdir=/workspace \
  --privileged=true \
  -v /home/kolbey/program:/workspace/program \
  --shm-size=16g \
  --ulimit memlock=-1 \
  --ulimit stack=67108864 \
  -v /etc/localtime:/etc/localtime \
  -e LOCAL_USER_ID=1036 \
  -it docker.m.daocloud.io/nvidia/cuda:12.1.1-cudnn8-devel-ubuntu20.04 \
  bash
```

参数说明：

```text
--name livian-gpu-train
```

指定容器名称，方便后续启动、停止、进入和删除该容器。

```text
--gpus all
```

允许容器访问宿主机上的所有 NVIDIA GPU。容器内执行 nvidia-smi 后，应该可以看到宿主机的 RTX 2080 Ti 显卡信息。

```text
-p 7777:22
```

将宿主机的 7777 端口映射到容器内部的 22 端口。若容器内安装并启动了 SSH 服务，则可以通过宿主机 7777 端口远程连接容器。

```text
--workdir=/workspace
```

设置容器启动后的默认工作目录为 /workspace。进入容器后会默认位于该目录下，便于统一管理训练工程。

```text
--privileged=true
```

赋予容器较高权限，便于容器访问更多宿主机设备和系统能力。对于单纯 GPU 训练不是必须项，但在开发调试、设备访问、底层工具使用等场景下可以减少权限问题。

```text
-v /home/kolbey/program:/workspace/program
```

将宿主机目录 /home/kolbey/program 挂载到容器内 /workspace/program。代码、数据、模型文件保存在宿主机，不会因为删除容器而丢失。

```text
--shm-size=16g
```

增大容器内 /dev/shm 共享内存。训练目标检测、分割模型时，PyTorch DataLoader 多进程加载数据可能需要较大的共享内存，可减少 Bus error、数据加载异常或训练卡死等问题。

```text
--ulimit memlock=-1
```

取消容器内进程可锁定内存大小限制。对 CUDA pinned memory、PyTorch pin_memory=True、NCCL 通信以及高速数据加载等场景更友好，适合作为 GPU 训练容器的稳妥配置。

```text
--ulimit stack=67108864
```

将容器内进程栈空间上限设置为 67108864 字节，即 64MB。对 C/C++ 扩展编译、OpenCV、CUDA 扩展、多线程库或复杂数据预处理场景有一定稳定性帮助。

```text
-v /etc/localtime:/etc/localtime
```

将宿主机时间配置挂载到容器内，使容器时间与宿主机保持一致。这样训练日志、模型保存时间、文件时间戳更容易对应。

```text
-e LOCAL_USER_ID=1036
```

向容器内传入环境变量 LOCAL_USER_ID=1036。该变量常用于自定义启动脚本根据宿主机用户 UID 创建容器内用户，避免挂载目录文件权限混乱。是否生效取决于容器内是否有对应脚本读取该变量。

```text
-it
```

以交互方式启动容器，并分配终端。开发阶段通常保留该参数，方便进入容器后手动执行命令。

```text
docker.m.daocloud.io/nvidia/cuda:12.1.1-cudnn8-devel-ubuntu20.04
```

指定容器基础镜像。该镜像基于 Ubuntu 20.04，包含 CUDA 12.1.1、cuDNN 8 以及 CUDA 开发工具，适合后续安装 PyTorch、YOLO、OpenCV 等模型训练环境。前缀 docker.m.daocloud.io 表示通过 DaoCloud 国内镜像源拉取。

```text
bash
```

容器启动后执行 bash，进入命令行交互环境，便于后续安装依赖、验证 GPU、运行训练脚本。

---

## 11. 容器常用管理命令

### 11.1 查看正在运行的容器

```bash
docker ps
```

### 11.2 查看所有容器

```bash
docker ps -a
```

### 11.3 退出容器但不停止

在容器内按：

```text
Ctrl + P + Q
```

### 11.4 重新进入已存在容器

```bash
docker start smt_ai_train_env
docker exec -it smt_ai_train_env /bin/bash
```

### 11.5 停止容器

```bash
docker stop smt_ai_train_env
```

### 11.6 删除容器

```bash
docker rm -f smt_ai_train_env
```

### 11.7 查看本地镜像

```bash
docker images
```

---

## 12. 后续 PyTorch 环境建议

进入训练容器后，可以先安装基础工具：

```bash
apt update
apt install -y \
  python3 \
  python3-pip \
  python3-venv \
  git \
  vim \
  wget \
  curl \
  build-essential \
  cmake \
  libgl1 \
  libglib2.0-0
```

升级 pip：

```bash
python3 -m pip install --upgrade pip -i https://pypi.tuna.tsinghua.edu.cn/simple
```

如果使用 CUDA 12.1 镜像，可以安装 PyTorch CUDA 12.1 版本：

```bash
pip3 install torch==2.4.1 torchvision==0.19.1 torchaudio==2.4.1 \
  --index-url https://download.pytorch.org/whl/cu121
```

安装完成后验证 PyTorch 是否可以调用 GPU：

```bash
python3 - <<'EOF'
import torch

print("torch version:", torch.__version__)
print("cuda available:", torch.cuda.is_available())
print("torch cuda version:", torch.version.cuda)
print("gpu count:", torch.cuda.device_count())

if torch.cuda.is_available():
    print("gpu name:", torch.cuda.get_device_name(0))
    x = torch.randn(1024, 1024).cuda()
    y = x @ x
    print("cuda tensor test ok:", y.shape)
else:
    print("CUDA is not available in PyTorch.")
EOF
```

正常情况下应看到：

```text
torch version: 2.4.1+cu121
cuda available: True
torch cuda version: 12.1
gpu name: NVIDIA GeForce RTX 2080 Ti
cuda tensor test ok: torch.Size([1024, 1024])
```

---

## 13. 最小成功流程汇总

如果在一台新的 Ubuntu 20.04 服务器上重复配置，核心命令流程如下：

```bash
# 1. 检查 NVIDIA 驱动和 Docker
nvidia-smi
docker --version

# 2. 安装基础工具
sudo apt-get update
sudo apt-get install -y curl ca-certificates gnupg

# 3. 添加 USTC GPG key
curl -fsSL https://mirrors.ustc.edu.cn/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

# 4. 添加 USTC NVIDIA Container Toolkit 源
cat <<'EOF' | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://mirrors.ustc.edu.cn/libnvidia-container/stable/deb/amd64 /
EOF

# 5. 安装 NVIDIA Container Toolkit
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit

# 6. 配置 Docker runtime 并重启 Docker
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# 7. 使用 DaoCloud 拉取 CUDA 测试镜像
docker pull docker.m.daocloud.io/nvidia/cuda:11.8.0-base-ubuntu20.04

# 8. 验证容器 GPU 调用
docker run --rm --gpus all \
  docker.m.daocloud.io/nvidia/cuda:11.8.0-base-ubuntu20.04 \
  nvidia-smi
```

如果最后一步能在容器内看到 NVIDIA 显卡信息，则 Docker + GPU 容器运行环境配置成功。
