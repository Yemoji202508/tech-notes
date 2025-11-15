# Ubuntu 系统安装 LLaMA-Factory 完整指南

## 环境说明

- **操作系统**: Ubuntu 20.04 / 22.04 / 24.04 LTS
- **GPU**: NVIDIA GPU（可选，支持 CPU 运行）
- **Python 版本**: 3.10+ (推荐 3.10 或 3.11)
- **CUDA 版本**: 12.1 / 12.4（有 GPU 时）

## 一、系统环境检查

### 1.1 检查 Ubuntu 版本

```bash
lsb_release -a
```

### 1.2 检查 GPU（有 GPU 时）

```bash
# 检查 GPU 信息
lspci | grep -i nvidia

# 检查 NVIDIA 驱动
nvidia-smi
```

**预期输出**：显示 GPU 型号、驱动版本和 CUDA 版本

**如果未安装驱动**，参考第二章安装 NVIDIA 驱动。

### 1.3 检查 Python 版本

```bash
python3 --version
```

**推荐版本**：Python 3.10 或 3.11

## 二、安装 NVIDIA 驱动和 CUDA（有 GPU 时）

### 2.1 安装 NVIDIA 驱动

#### 方式一：使用 Ubuntu 驱动管理器（推荐）

```bash
# 安装驱动管理器
sudo apt update
sudo apt install -y ubuntu-drivers-common

# 查看推荐驱动
ubuntu-drivers devices

# 自动安装推荐驱动
sudo ubuntu-drivers autoinstall

# 或手动安装指定版本（如 535）
sudo apt install -y nvidia-driver-535

# 重启系统
sudo reboot
```

#### 方式二：使用官方 PPA

```bash
sudo add-apt-repository ppa:graphics-drivers/ppa
sudo apt update
sudo apt install -y nvidia-driver-535
sudo reboot
```

**验证安装**：

```bash
nvidia-smi
```

### 2.2 安装 CUDA Toolkit

#### 选择 CUDA 版本

- **CUDA 12.1**：兼容性好，推荐
- **CUDA 12.4**：较新版本

#### 安装 CUDA 12.1

```bash
# 下载 CUDA Keyring
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb

# 更新并安装
sudo apt update
sudo apt install -y cuda-toolkit-12-1

# 清理安装包
rm -f cuda-keyring_1.1-1_all.deb
```

**Ubuntu 20.04 用户**：将 `ubuntu2204` 改为 `ubuntu2004`  
**Ubuntu 24.04 用户**：将 `ubuntu2204` 改为 `ubuntu2404`

#### 配置环境变量

```bash
echo 'export PATH=/usr/local/cuda-12.1/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda-12.1/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
```

#### 验证 CUDA 安装

```bash
nvcc --version
```

## 三、安装系统依赖

### 3.1 更新系统

```bash
sudo apt update && sudo apt upgrade -y
```

### 3.2 安装基础工具

```bash
sudo apt install -y build-essential git wget curl vim
```

### 3.3 安装 Python 和开发工具

#### Ubuntu 22.04（Python 3.10）

```bash
sudo apt install -y python3 python3-pip python3-dev python3-venv
```

#### Ubuntu 24.04（Python 3.11 或 3.12）

```bash
# 使用系统默认 Python
sudo apt install -y python3 python3-pip python3-dev python3-venv

# 或安装 Python 3.11
sudo apt install -y python3.11 python3.11-venv python3.11-dev
sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.11 1
```

#### Ubuntu 20.04（需要安装 Python 3.10）

```bash
# 添加 deadsnakes PPA
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt update

# 安装 Python 3.10
sudo apt install -y python3.10 python3.10-venv python3.10-dev
sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.10 1

# 安装 pip
curl -sS https://bootstrap.pypa.io/get-pip.py | python3.10
```

### 3.4 验证安装

```bash
python3 --version
pip3 --version
git --version
```

## 四、下载 LLaMA-Factory

### 4.1 克隆仓库

```bash
# 从 GitHub 克隆（推荐）
git clone --depth=1 https://github.com/hiyouga/LLaMA-Factory.git

# 或从 Gitee 克隆（国内镜像）
git clone --depth=1 https://gitee.com/hiyouga/LLaMA-Factory.git
```

### 4.2 进入项目目录

```bash
cd LLaMA-Factory
```

## 五、安装 LLaMA-Factory

### 5.1 创建虚拟环境

```bash
python3 -m venv venv
```

### 5.2 激活虚拟环境

```bash
source venv/bin/activate
```

激活后，命令行提示符前会显示 `(venv)`

### 5.3 升级 pip

```bash
pip install --upgrade pip
```

### 5.4 配置 pip 镜像源（可选但推荐）

```bash
# 使用清华镜像源
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple

# 或使用阿里云镜像源
pip config set global.index-url https://mirrors.aliyun.com/pypi/simple/
```

### 5.5 安装 LLaMA-Factory

#### 基础安装（CPU 或 GPU）

```bash
pip install -e ".[torch,metrics]"
```

#### 推荐安装（包含量化支持）

```bash
pip install -e ".[torch,metrics,bitsandbytes]"
```

#### 完整安装（所有功能）

```bash
pip install -e ".[torch,metrics,bitsandbytes,vllm,deepspeed]"
```

**安装选项说明**：
- `torch`: PyTorch 和相关依赖（必需）
- `metrics`: 评估指标（BLEU、ROUGE 等）
- `bitsandbytes`: 4-bit/8-bit 量化（QLoRA）
- `vllm`: vLLM 推理加速
- `deepspeed`: DeepSpeed 分布式训练
- `galore`: GaLore 优化器
- `badam`: BAdam 优化器

**安装时间**：5-20 分钟（取决于网络和配置）

### 5.6 验证安装

```bash
# 检查版本
llamafactory-cli version

# 或使用 Python
python -c "import llamafactory; print(llamafactory.__version__)"
```

### 5.7 验证 GPU 可用性（有 GPU 时）

```bash
python -c "import torch; print(f'CUDA available: {torch.cuda.is_available()}'); print(f'CUDA version: {torch.version.cuda}'); print(f'GPU count: {torch.cuda.device_count()}'); print(f'GPU name: {torch.cuda.get_device_name(0) if torch.cuda.is_available() else \"N/A\"}')"
```

**预期输出**：

```
CUDA available: True
CUDA version: 12.1
GPU count: 1
GPU name: NVIDIA GeForce RTX 3090
```

## 六、配置 Hugging Face 镜像（可选）

### 6.1 使用国内镜像加速模型下载

```bash
# 临时使用（当前终端）
export HF_ENDPOINT=https://hf-mirror.com

# 永久配置
echo 'export HF_ENDPOINT=https://hf-mirror.com' >> ~/.bashrc
source ~/.bashrc
```

### 6.2 配置模型缓存目录

```bash
# 设置缓存目录（可选）
export HF_HOME=/path/to/huggingface
export TRANSFORMERS_CACHE=/path/to/huggingface/transformers

# 永久配置
echo 'export HF_HOME=~/huggingface' >> ~/.bashrc
echo 'export TRANSFORMERS_CACHE=~/huggingface/transformers' >> ~/.bashrc
source ~/.bashrc
```

## 七、启动 LLaMA-Factory

### 7.1 启动 Web UI

```bash
llamafactory-cli webui
```

**默认访问地址**：`http://127.0.0.1:7860`

**自定义端口和主机**：

```bash
llamafactory-cli webui --host 0.0.0.0 --port 8080
```

### 7.2 命令行训练

```bash
# 使用示例配置
llamafactory-cli train examples/train_lora/llama3_lora_sft.yaml

# 使用自定义配置
llamafactory-cli train your_config.yaml
```

### 7.3 命令行推理

```bash
llamafactory-cli chat examples/inference/llama3.yaml
```

### 7.4 启动 API 服务

```bash
llamafactory-cli api examples/inference/llama3_vllm.yaml
```

## 八、常见问题与解决方案

### 8.1 CUDA 不可用

**问题**：`torch.cuda.is_available()` 返回 `False`

**解决方案**：

1. 检查驱动安装：`nvidia-smi`
2. 检查 CUDA 路径：`echo $PATH | grep cuda`
3. 重新安装 PyTorch：
   ```bash
   pip uninstall torch torchvision torchaudio
   pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121
   ```

### 8.2 pip 安装超时

**问题**：`ReadTimeoutError` 或下载缓慢

**解决方案**：

```bash
# 使用镜像源
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple

# 增加超时时间
pip install --timeout=1000 -e ".[torch,metrics]"
```

### 8.3 权限问题

**问题**：`Permission denied` 或需要 sudo

**解决方案**：

```bash
# 不要使用 sudo pip install
# 使用虚拟环境或用户安装
pip install --user -e ".[torch,metrics]"
```

### 8.4 显存不足（OOM）

**问题**：训练时显存溢出

**解决方案**：

1. 使用量化：`pip install bitsandbytes`
2. 减小 batch size：`per_device_train_batch_size: 1`
3. 使用梯度累积：`gradient_accumulation_steps: 8`
4. 启用梯度检查点：`gradient_checkpointing: true`
5. 使用 LoRA/QLoRA 微调

### 8.5 模型下载失败

**问题**：无法从 Hugging Face 下载模型

**解决方案**：

```bash
# 使用镜像站
export HF_ENDPOINT=https://hf-mirror.com

# 或手动下载后指定本地路径
llamafactory-cli train --model_name_or_path /path/to/local/model
```

### 8.6 Flash Attention 安装失败

**问题**：编译 Flash Attention 失败

**解决方案**：

```bash
# 确保安装了 CUDA 和 ninja
sudo apt install -y ninja-build

# 使用预编译版本
pip install flash-attn --no-build-isolation

# 或跳过 Flash Attention
# 在配置中设置 use_flash_attention: false
```

## 九、性能优化建议

### 9.1 使用 Flash Attention 2

```bash
pip install flash-attn --no-build-isolation
```

在训练配置中启用：

```yaml
use_flash_attention: true
```

### 9.2 使用混合精度训练

```yaml
# FP16（推荐用于 Ampere 之前的 GPU）
fp16: true

# BF16（推荐用于 Ampere 及以后的 GPU）
bf16: true
```

### 9.3 优化数据加载

```yaml
dataloader_num_workers: 4
dataloader_pin_memory: true
```

### 9.4 使用梯度检查点

```yaml
gradient_checkpointing: true
```

**作用**：减少显存占用，但会略微降低训练速度

### 9.5 使用 DeepSpeed（多 GPU）

```bash
# 安装 DeepSpeed
pip install deepspeed

# 使用 DeepSpeed 训练
deepspeed --num_gpus=2 src/train.py --deepspeed examples/deepspeed/ds_z3_config.json
```

## 十、卸载与清理

### 10.1 卸载 LLaMA-Factory

```bash
cd LLaMA-Factory
source venv/bin/activate
pip uninstall llamafactory
```

### 10.2 删除虚拟环境

```bash
deactivate
cd ..
rm -rf LLaMA-Factory
```

### 10.3 卸载 CUDA Toolkit

```bash
sudo apt remove --purge cuda-toolkit-12-1
sudo apt autoremove
```

### 10.4 卸载 NVIDIA 驱动

```bash
sudo apt remove --purge nvidia-*
sudo apt autoremove
sudo reboot
```

## 十一、Docker 安装方式（推荐）

### 11.1 安装 Docker

```bash
# 安装 Docker
curl -fsSL https://get.docker.com | bash -s docker

# 添加当前用户到 docker 组
sudo usermod -aG docker $USER

# 重新登录或执行
newgrp docker
```

### 11.2 安装 NVIDIA Container Toolkit

```bash
# 添加 NVIDIA 仓库
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list

# 安装
sudo apt update
sudo apt install -y nvidia-container-toolkit

# 重启 Docker
sudo systemctl restart docker
```

### 11.3 使用 Docker 运行 LLaMA-Factory

```bash
# 拉取镜像
docker pull hiyouga/llama-factory:latest

# 运行容器（CPU）
docker run -it --rm \
  -v ./data:/app/data \
  -p 7860:7860 \
  hiyouga/llama-factory:latest \
  llamafactory-cli webui

# 运行容器（GPU）
docker run -it --rm --gpus all \
  -v ./data:/app/data \
  -v ./models:/app/models \
  -p 7860:7860 \
  hiyouga/llama-factory:latest \
  llamafactory-cli webui
```

### 11.4 使用 Docker Compose

创建 `docker-compose.yml`：

```yaml
version: '3.8'

services:
  llama-factory:
    image: hiyouga/llama-factory:latest
    container_name: llama-factory
    ports:
      - "7860:7860"
    volumes:
      - ./data:/app/data
      - ./models:/app/models
      - ./output:/app/output
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    command: llamafactory-cli webui --host 0.0.0.0
```

启动：

```bash
docker-compose up -d
```

## 十二、参考资源

- **LLaMA-Factory 官方文档**: https://llamafactory.readthedocs.io/
- **LLaMA-Factory GitHub**: https://github.com/hiyouga/LLaMA-Factory
- **Hugging Face 镜像站**: https://hf-mirror.com
- **PyTorch 官方文档**: https://pytorch.org/get-started/locally/
- **NVIDIA CUDA 文档**: https://docs.nvidia.com/cuda/

## 附录：自动化安装脚本

### A.1 完整安装脚本（GPU）

```bash
#!/bin/bash
# Ubuntu 安装 LLaMA-Factory 自动化脚本（GPU 版本）

set -e

echo "=== LLaMA-Factory 自动安装脚本 ==="
echo ""

# 检查是否有 NVIDIA GPU
if ! command -v nvidia-smi &> /dev/null; then
    echo "警告：未检测到 NVIDIA GPU 或驱动未安装"
    read -p "是否继续安装（CPU 模式）？(y/n) " -n 1 -r
    echo
    if [[ ! $REPLY =~ ^[Yy]$ ]]; then
        exit 1
    fi
    GPU_MODE=false
else
    echo "检测到 NVIDIA GPU"
    nvidia-smi
    GPU_MODE=true
fi

echo ""
echo "=== 1. 更新系统 ==="
sudo apt update && sudo apt upgrade -y

echo ""
echo "=== 2. 安装系统依赖 ==="
sudo apt install -y build-essential git wget curl vim python3 python3-pip python3-dev python3-venv

# 安装 CUDA（如果有 GPU）
if [ "$GPU_MODE" = true ]; then
    echo ""
    echo "=== 3. 安装 CUDA Toolkit ==="
    
    # 检查 CUDA 是否已安装
    if command -v nvcc &> /dev/null; then
        echo "CUDA 已安装："
        nvcc --version
    else
        echo "开始安装 CUDA 12.1..."
        wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb
        sudo dpkg -i cuda-keyring_1.1-1_all.deb
        sudo apt update
        sudo apt install -y cuda-toolkit-12-1
        rm -f cuda-keyring_1.1-1_all.deb
        
        # 配置环境变量
        if ! grep -q "cuda-12.1" ~/.bashrc; then
            echo 'export PATH=/usr/local/cuda-12.1/bin:$PATH' >> ~/.bashrc
            echo 'export LD_LIBRARY_PATH=/usr/local/cuda-12.1/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
        fi
    fi
fi

echo ""
echo "=== 4. 克隆 LLaMA-Factory ==="
if [ ! -d "LLaMA-Factory" ]; then
    git clone --depth=1 https://github.com/hiyouga/LLaMA-Factory.git
else
    echo "LLaMA-Factory 目录已存在，跳过克隆"
fi

cd LLaMA-Factory

echo ""
echo "=== 5. 创建虚拟环境 ==="
python3 -m venv venv
source venv/bin/activate

echo ""
echo "=== 6. 配置 pip 镜像源 ==="
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple

echo ""
echo "=== 7. 安装 LLaMA-Factory ==="
pip install --upgrade pip

if [ "$GPU_MODE" = true ]; then
    echo "安装 GPU 版本（包含量化支持）..."
    pip install -e ".[torch,metrics,bitsandbytes]"
else
    echo "安装 CPU 版本..."
    pip install -e ".[torch,metrics]"
fi

echo ""
echo "=== 8. 配置 Hugging Face 镜像 ==="
if ! grep -q "HF_ENDPOINT" ~/.bashrc; then
    echo 'export HF_ENDPOINT=https://hf-mirror.com' >> ~/.bashrc
fi

echo ""
echo "=== 9. 验证安装 ==="
echo "LLaMA-Factory 版本："
llamafactory-cli version

if [ "$GPU_MODE" = true ]; then
    echo ""
    echo "CUDA 可用性："
    python -c "import torch; print(f'CUDA available: {torch.cuda.is_available()}'); print(f'GPU: {torch.cuda.get_device_name(0) if torch.cuda.is_available() else \"N/A\"}')"
fi

echo ""
echo "=== 安装完成！==="
echo ""
echo "下一步操作："
echo "1. 重新加载环境变量: source ~/.bashrc"
echo "2. 激活虚拟环境: cd LLaMA-Factory && source venv/bin/activate"
echo "3. 启动 Web UI: llamafactory-cli webui"
echo ""
if [ "$GPU_MODE" = true ]; then
    echo "注意：首次使用需要重启终端或执行 'source ~/.bashrc' 使 CUDA 环境变量生效"
fi
```

### A.2 CPU 版本安装脚本

```bash
#!/bin/bash
# Ubuntu 安装 LLaMA-Factory 自动化脚本（CPU 版本）

set -e

echo "=== LLaMA-Factory CPU 版本安装 ==="

echo "=== 1. 更新系统 ==="
sudo apt update && sudo apt upgrade -y

echo "=== 2. 安装系统依赖 ==="
sudo apt install -y build-essential git wget curl python3 python3-pip python3-dev python3-venv

echo "=== 3. 克隆 LLaMA-Factory ==="
if [ ! -d "LLaMA-Factory" ]; then
    git clone --depth=1 https://github.com/hiyouga/LLaMA-Factory.git
fi
cd LLaMA-Factory

echo "=== 4. 创建虚拟环境 ==="
python3 -m venv venv
source venv/bin/activate

echo "=== 5. 配置 pip 镜像源 ==="
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple

echo "=== 6. 安装 LLaMA-Factory ==="
pip install --upgrade pip
pip install -e ".[torch,metrics]"

echo "=== 7. 配置 Hugging Face 镜像 ==="
if ! grep -q "HF_ENDPOINT" ~/.bashrc; then
    echo 'export HF_ENDPOINT=https://hf-mirror.com' >> ~/.bashrc
fi

echo "=== 8. 验证安装 ==="
llamafactory-cli version

echo ""
echo "=== 安装完成！==="
echo "启动 Web UI: llamafactory-cli webui"
```

保存脚本并执行：

```bash
# GPU 版本
chmod +x install_llamafactory_gpu.sh
./install_llamafactory_gpu.sh

# CPU 版本
chmod +x install_llamafactory_cpu.sh
./install_llamafactory_cpu.sh
```

---

**文档版本**: v1.0  
**更新日期**: 2025-11-15  
**适用版本**: LLaMA-Factory 0.9.0+  
**测试环境**: Ubuntu 22.04 LTS, CUDA 12.1, Python 3.10
