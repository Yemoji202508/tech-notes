# WSL2 安装 LLaMA-Factory 完整操作手册

## 环境说明

- **宿主系统**: Windows 11
- **WSL 版本**: WSL2
- **Linux 发行版**: Ubuntu 22.04 / 24.04
- **GPU**: NVIDIA GeForce RTX 3060 (6GB)
- **Python 版本**: 3.10+ (推荐 3.10 或 3.11)
- **CUDA 版本**: 12.4 或 12.6

**注意**：Python 3.12 可能存在兼容性问题，建议使用 3.10 或 3.11。

## 一、前置条件检查

### 1.1 确认 WSL2 版本

```bash
wsl --version
```

### 1.2 检查 GPU 驱动

在 WSL2 Ubuntu 中运行：

```bash
nvidia-smi
```

**预期输出**：显示 GPU 信息、驱动版本和 CUDA 版本

**重要说明**：
- ✅ GPU 驱动只需在 Windows 宿主机安装
- ❌ WSL2 Ubuntu 中不需要安装 NVIDIA 驱动
- ✅ WSL2 通过虚拟化层直接使用 Windows 的 GPU 驱动

## 二、安装系统依赖

### 2.1 更新系统包

```bash
sudo apt update && sudo apt upgrade -y
```

### 2.2 安装基础工具

```bash
sudo apt install -y gcc g++ make build-essential git wget curl
```

验证安装：

```bash
gcc --version
git --version
```

### 2.3 安装 Python 开发环境

**Ubuntu 22.04（推荐 Python 3.10）**：

```bash
sudo apt install -y python3 python3-pip python3-dev python3-venv
```

**Ubuntu 24.04（如需使用 Python 3.11）**：

```bash
# 安装 Python 3.11
sudo apt install -y python3.11 python3.11-venv python3.11-dev python3-pip

# 设置 Python 3.11 为默认版本
sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.11 1
```

验证安装：

```bash
python3 --version  # 应显示 3.10.x 或 3.11.x
pip3 --version
```

## 三、安装 CUDA Toolkit

### 3.1 选择 CUDA 版本

**推荐版本**：
- CUDA 12.4（稳定版，推荐）
- CUDA 12.6（最新版）

**检查兼容性**：
- PyTorch 2.0+：支持 CUDA 11.8、12.1、12.4
- 确认你的 Windows GPU 驱动版本支持对应的 CUDA 版本

### 3.2 安装 CUDA 12.4（推荐）

#### 下载并安装 CUDA Keyring

```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
```

**注意**：Ubuntu 24.04 用户将 `ubuntu2204` 改为 `ubuntu2404`

#### 更新软件源并安装 CUDA Toolkit

```bash
sudo apt update
sudo apt install -y cuda-toolkit-12-4
```

**安装内容**：
- CUDA 编译器 (nvcc)
- CUDA 库 (cuBLAS, cuDNN, cuFFT, cuRAND, cuSolver, cuSPARSE, NPP, nvJPEG)
- CUDA 开发工具 (cuda-gdb, nsight-compute, nsight-systems)
- CUDA 运行时和驱动 API

**安装大小**：约 3.5 GB 下载，8 GB 磁盘空间

### 3.3 配置环境变量

```bash
echo 'export PATH=/usr/local/cuda-12.4/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda-12.4/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
```

### 3.4 验证 CUDA 安装

```bash
nvcc --version
```

**预期输出**：

```
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2024 NVIDIA Corporation
Built on Thu_Mar_28_02:18:24_PDT_2024
Cuda compilation tools, release 12.4, V12.4.131
Build cuda_12.4.r12.4/compiler.33961263_0
```

### 3.5 清理安装文件

```bash
rm -f cuda-keyring_1.1-1_all.deb
```

## 四、下载 LLaMA-Factory 源码

### 4.1 克隆仓库

```bash
# 从 GitHub 克隆
git clone https://github.com/hiyouga/LLaMA-Factory.git

# 或从 Gitee 克隆（国内推荐）
git clone https://gitee.com/hiyouga/LLaMA-Factory.git
```

### 4.2 进入项目目录

```bash
cd LLaMA-Factory
```

## 五、安装 LLaMA-Factory

### 5.1 创建 Python 虚拟环境

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

**永久配置清华镜像源**：

```bash
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```

**或手动创建配置文件**：

```bash
mkdir -p ~/.pip
cat > ~/.pip/pip.conf << EOF
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
[install]
trusted-host = pypi.tuna.tsinghua.edu.cn
EOF
```

**其他镜像源**：
- 阿里云：`https://mirrors.aliyun.com/pypi/simple/`
- 腾讯云：`https://mirrors.cloud.tencent.com/pypi/simple/`
- 豆瓣：`https://pypi.douban.com/simple/`

### 5.5 安装 LLaMA-Factory

**推荐安装方式**（已配置镜像源）：

```bash
pip install -e ".[torch,metrics]"
```

**临时使用镜像源**（未配置镜像源时）：

```bash
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple -e ".[torch,metrics]"
```

**安装选项说明**：
- `torch`: 安装 PyTorch 和相关依赖（必需）
- `metrics`: 安装评估指标相关依赖（NLTK, ROUGE 等）
- `-e`: 以可编辑模式安装，方便开发和调试

**其他可选依赖**：
```bash
# 量化训练支持（推荐）
pip install -e ".[torch,metrics,bitsandbytes]"

# vLLM 推理加速
pip install -e ".[torch,metrics,vllm]"

# 完整安装（包含所有功能）
pip install -e ".[torch,metrics,bitsandbytes,vllm,deepspeed]"
```

**可选依赖说明**：
- `bitsandbytes`: 4-bit/8-bit 量化训练（QLoRA）
- `vllm`: vLLM 推理加速引擎
- `deepspeed`: DeepSpeed 分布式训练
- `galore`: GaLore 优化器
- `badam`: BAdam 优化器
- `qwen`: Qwen 模型专用支持
- `modelscope`: ModelScope 平台支持

**安装时间**：约 5-15 分钟（取决于网络速度）
- `vllm`: vLLM 推理加速引擎
- `deepspeed`: DeepSpeed 分布式训练
- `galore`: GaLore 优化器
- `badam`: BAdam 优化器
- `qwen`: Qwen 模型专用支持
- `modelscope`: ModelScope 平台支持

**安装时间**：约 5-15 分钟（取决于网络速度）

### 5.6 验证安装

```bash
# 检查 LLaMA-Factory 命令
llamafactory-cli version

# 或使用 Python 导入测试
python -c "import llamafactory; print(llamafactory.__version__)"
```

### 5.7 验证 GPU 可用性

```bash
python -c "import torch; print(f'CUDA available: {torch.cuda.is_available()}'); print(f'CUDA version: {torch.version.cuda}'); print(f'GPU count: {torch.cuda.device_count()}'); print(f'GPU name: {torch.cuda.get_device_name(0) if torch.cuda.is_available() else None}')"
```

**预期输出**：

```
CUDA available: True
CUDA version: 12.4
GPU count: 1
GPU name: NVIDIA GeForce RTX 3060 Laptop GPU
```

**如果 CUDA 不可用**，参考第 7.2 节排查问题。

## 六、启动 LLaMA-Factory

### 6.1 启动 Web UI

```bash
llamafactory-cli webui
```

或

```bash
python src/webui.py
```

**默认访问地址**：`http://127.0.0.1:7860`

### 6.2 命令行训练示例

```bash
llamafactory-cli train examples/train_lora/llama3_lora_sft.yaml
```

### 6.3 API 服务启动

```bash
llamafactory-cli api examples/inference/llama3_vllm.yaml
```

## 七、常见问题与解决方案

### 7.1 pip 安装超时

**问题**：`ReadTimeoutError: HTTPSConnectionPool: Read timed out`

**解决方案**：
1. 使用国内镜像源（见 5.4 节）
2. 增加超时时间：`pip install --timeout=1000 -e ".[torch,metrics]"`
3. 分步安装依赖

### 7.2 CUDA 不可用

**问题**：`torch.cuda.is_available()` 返回 `False`

**解决方案**：
1. 检查 Windows 驱动是否为 WSL 专用版本
2. 重启 WSL：`wsl --shutdown`（在 Windows PowerShell 中执行）
3. 确认 WSL2 内核版本：`uname -r`（需要 5.10.43.3 或更高）

### 7.3 虚拟环境激活失败

**问题**：`python3-venv` 未安装或版本不匹配

**解决方案**：

```bash
# Ubuntu 22.04
sudo apt install -y python3-venv

# Ubuntu 24.04 (Python 3.11)
sudo apt install -y python3.11-venv
```

### 7.4 PyTorch 与 CUDA 版本不匹配

**问题**：安装的 PyTorch 不支持你的 CUDA 版本

**解决方案**：

```bash
# 卸载现有 PyTorch
pip uninstall torch torchvision torchaudio

# 安装指定 CUDA 版本的 PyTorch（以 CUDA 12.4 为例）
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu124
```

**PyTorch 与 CUDA 对应关系**：
- CUDA 11.8: `cu118`
- CUDA 12.1: `cu121`
- CUDA 12.4: `cu124`

### 7.5 内存不足

**问题**：训练时 OOM (Out of Memory)

**解决方案**：
1. 使用量化训练：安装 `[bitsandbytes]`
2. 减小 batch size
3. 使用 LoRA/QLoRA 微调
4. 启用梯度检查点

### 7.5 WSL2 磁盘空间不足

**问题**：安装过程中磁盘空间不足

**解决方案**：
1. 清理 apt 缓存：`sudo apt clean`
2. 清理 pip 缓存：`pip cache purge`
3. 扩展 WSL2 虚拟磁盘大小

## 八、性能优化建议

### 8.1 WSL2 内存配置

创建或编辑 `C:\Users\<用户名>\.wslconfig`：

```ini
[wsl2]
memory=16GB
processors=8
swap=8GB
```

重启 WSL2 生效：

```powershell
wsl --shutdown
```

### 8.2 使用 Flash Attention

```bash
pip install flash-attn --no-build-isolation
```

### 8.3 启用混合精度训练

在训练配置中设置：

```yaml
fp16: true
# 或
bf16: true
```

## 九、卸载与清理

### 9.1 卸载 LLaMA-Factory

```bash
cd LLaMA-Factory
source venv/bin/activate
pip uninstall llamafactory
```

### 9.2 删除虚拟环境

```bash
deactivate
rm -rf venv
```

### 9.3 卸载 CUDA Toolkit

```bash
sudo apt remove --purge cuda-toolkit-12-8
sudo apt autoremove
```

## 十、参考资源

- **LLaMA-Factory 官方文档**: https://llamafactory.readthedocs.io/
- **LLaMA-Factory GitHub**: https://github.com/hiyouga/LLaMA-Factory
- **NVIDIA CUDA on WSL**: https://docs.nvidia.com/cuda/wsl-user-guide/
- **WSL2 官方文档**: https://learn.microsoft.com/zh-cn/windows/wsl/

## 附录：完整安装脚本

```bash
#!/bin/bash
# WSL2 安装 LLaMA-Factory 自动化脚本

set -e

echo "=== 1. 更新系统 ==="
sudo apt update && sudo apt upgrade -y

echo "=== 2. 安装系统依赖 ==="
sudo apt install -y gcc g++ make build-essential git wget curl python3 python3-pip python3-dev python3-venv

echo "=== 3. 安装 CUDA Toolkit ==="
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt update
sudo apt install -y cuda-toolkit-12-4
rm -f cuda-keyring_1.1-1_all.deb

echo "=== 4. 配置环境变量 ==="
if ! grep -q "cuda-12.4" ~/.bashrc; then
    echo 'export PATH=/usr/local/cuda-12.4/bin:$PATH' >> ~/.bashrc
    echo 'export LD_LIBRARY_PATH=/usr/local/cuda-12.4/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
fi

echo "=== 5. 克隆 LLaMA-Factory ==="
if [ ! -d "LLaMA-Factory" ]; then
    git clone --depth=1 https://github.com/hiyouga/LLaMA-Factory.git
fi
cd LLaMA-Factory

echo "=== 6. 创建虚拟环境 ==="
python3 -m venv venv
source venv/bin/activate

echo "=== 7. 配置 pip 镜像源 ==="
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple

echo "=== 8. 安装 LLaMA-Factory ==="
pip install --upgrade pip
pip install -e ".[torch,metrics,bitsandbytes]"

echo "=== 9. 验证安装 ==="
echo "LLaMA-Factory 版本："
llamafactory-cli version

echo "CUDA 可用性："
python -c "import torch; print(f'CUDA available: {torch.cuda.is_available()}'); print(f'GPU: {torch.cuda.get_device_name(0) if torch.cuda.is_available() else \"N/A\"}')"

echo ""
echo "=== 安装完成！==="
echo ""
echo "下一步操作："
echo "1. 重新加载环境变量: source ~/.bashrc"
echo "2. 激活虚拟环境: cd LLaMA-Factory && source venv/bin/activate"
echo "3. 启动 Web UI: llamafactory-cli webui"
echo ""
echo "注意：首次使用需要重启终端或执行 'source ~/.bashrc' 使 CUDA 环境变量生效"
```

保存为 `install_llamafactory.sh`，然后执行：

```bash
chmod +x install_llamafactory.sh
./install_llamafactory.sh
```

**脚本说明**：
- 自动检测并跳过已安装的组件
- 使用 `--depth=1` 加速 Git 克隆
- 自动配置 pip 镜像源
- 包含 bitsandbytes 支持（QLoRA）
- 提供详细的验证输出

**安装后操作**：

```bash
# 重新加载环境变量
source ~/.bashrc

# 进入项目目录并激活虚拟环境
cd LLaMA-Factory
source venv/bin/activate

# 启动 Web UI
llamafactory-cli webui
```

---

**文档版本**: v1.1  
**更新日期**: 2025-11-15  
**适用版本**: LLaMA-Factory 0.9.0+  
**测试环境**: Ubuntu 22.04 LTS, CUDA 12.4, Python 3.10
