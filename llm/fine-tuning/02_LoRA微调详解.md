# LoRA 微调详解

## LoRA 是什么

**LoRA** = **Lo**w-**R**ank **A**daptation（低秩适应）
- **读音**：洛拉（luo-la）
- **发音**：类似英文 "Laura"

### 通俗理解

想象你要改造一栋大楼（原始模型）：

**传统微调**：
- 把整栋楼推倒重建 → 成本高、耗时长
- 需要大量材料和工人 → 显存占用大

**LoRA 微调**：
- 只在关键位置加装"适配器"（小模块）→ 成本低、速度快
- 原建筑保持不变，随时可拆卸 → 灵活可控

## LoRA 的工作原理

### 简化版解释

```
原始模型权重（冻结，不改动）
    ↓
    + LoRA 适配器（只训练这部分，很小）
    ↓
微调后的效果
```

**数学上**：
- 原始权重矩阵：`W`（比如 4096×4096，很大）
- LoRA 分解：`W + A×B`
  - `A`：4096×8（很小）
  - `B`：8×4096（很小）
  - 只训练 A 和 B，参数量减少 99%+

### 为什么叫"低秩"

- **秩（Rank）**：矩阵的"信息量"
- **低秩**：用少量信息近似表达大量变化
- 就像用几个关键点就能画出一条曲线

### LoRA 的关键技术细节

#### 1. 缩放因子 Alpha

LoRA 的实际更新量 = `(alpha / rank) × A × B`

**为什么需要 alpha**：
- 控制 LoRA 的影响强度
- 通常设为 `2 × rank`（如 rank=8, alpha=16）
- 可以在不改变参数量的情况下调整学习强度

#### 2. 目标模块选择

**常见选择**：
```python
# 只微调 attention 的 Q 和 V（最常用）
lora_target: "q_proj,v_proj"

# 微调所有 attention 层
lora_target: "q_proj,k_proj,v_proj,o_proj"

# 微调所有线性层（效果最好，参数最多）
lora_target: "all"
```

**选择建议**：
- 显存充足：all
- 平衡选择：q_proj,v_proj
- 显存紧张：q_proj

#### 3. 初始化策略

- **A 矩阵**：随机高斯初始化
- **B 矩阵**：零初始化
- **目的**：训练开始时 LoRA 不影响原模型（A×B=0）

#### 4. Dropout 的作用

```python
lora_dropout: 0.05  # 5% 的神经元随机失活
```

**作用**：
- 防止过拟合
- 提高泛化能力
- 通常设为 0.05-0.1

## LoRA 的三个阶段

### 1. LoRA 微调（训练）

**目标**：训练出适配器权重

```bash
# 使用 LLaMA Factory 训练
llamafactory-cli train \
  --model_name_or_path meta-llama/Llama-3-8B \
  --dataset your_data \
  --finetuning_type lora \
  --lora_rank 8 \
  --output_dir ./output/lora
```

**关键参数**：
- `lora_rank`：适配器的"秩"，通常 8-64
  - 越大效果越好，但参数越多
  - 一般用 8 或 16 就够了
- `lora_alpha`：缩放因子，通常是 rank 的 2 倍
- `lora_dropout`：防止过拟合，通常 0.05-0.1

**输出**：
- `adapter_model.bin`：LoRA 权重文件（几十 MB）
- `adapter_config.json`：配置文件

### 2. LoRA 推理（使用）

**方式一：动态加载**（推荐）

```python
from transformers import AutoModelForCausalLM
from peft import PeftModel

# 加载基座模型
base_model = AutoModelForCausalLM.from_pretrained("meta-llama/Llama-3-8B")

# 加载 LoRA 适配器
model = PeftModel.from_pretrained(base_model, "./output/lora")

# 推理
output = model.generate(...)
```

**优势**：
- 可以快速切换不同的 LoRA 适配器
- 一个基座模型 + 多个适配器 = 多个专业模型
- 节省显存和磁盘空间

**方式二：使用 LLaMA Factory**

```bash
llamafactory-cli chat \
  --model_name_or_path meta-llama/Llama-3-8B \
  --adapter_name_or_path ./output/lora \
  --template llama3
```

### 3. LoRA 合并（可选）

**目标**：把适配器权重合并到基座模型，生成完整模型

```bash
llamafactory-cli export \
  --model_name_or_path meta-llama/Llama-3-8B \
  --adapter_name_or_path ./output/lora \
  --export_dir ./output/merged \
  --export_size 2 \
  --export_device cpu
```

**什么时候需要合并**：
- ✅ 需要部署到不支持 LoRA 的推理框架
- ✅ 只有一个固定的微调模型，不需要切换
- ✅ 需要进一步量化或优化

**什么时候不合并**：
- ❌ 需要频繁切换不同的专业模型
- ❌ 显存或磁盘空间有限
- ❌ 还在实验阶段，模型可能继续调整

## LoRA 的优势

### 1. 极低的训练成本

| 对比项 | 全量微调 | LoRA 微调 |
|--------|---------|-----------|
| 显存占用 | 80GB+ | 24GB |
| 训练时间 | 10 小时 | 2 小时 |
| 可训练参数 | 70 亿 | 400 万（0.06%） |
| 输出大小 | 14GB | 30MB |

### 2. 灵活的模型管理

```
基座模型（14GB）
  ├── 客服 LoRA（30MB）
  ├── 代码 LoRA（30MB）
  ├── 医疗 LoRA（30MB）
  └── 法律 LoRA（30MB）
```

一个基座 + 多个适配器 = 多个专业模型

### 3. 快速实验迭代

- 训练快：2-4 小时就能完成
- 切换快：秒级切换不同模型
- 成本低：个人电脑也能训练

### 4. 易于分享和部署

- 只需分享 30MB 的适配器文件
- 不涉及完整模型权重，降低法律风险
- 方便版本管理和回滚

## LoRA 的变体

### QLoRA（Quantized LoRA）

- **读音**：Q-洛拉
- **特点**：在量化模型上做 LoRA 微调
- **优势**：显存占用再降 50%，消费级显卡也能训练
- **适用**：显存不足时的首选

### LoRA+

- 对 A 和 B 矩阵使用不同的学习率
- 效果略好于标准 LoRA

### AdaLoRA

- 动态调整不同层的 rank
- 重要层用高 rank，不重要层用低 rank
- 效果更好但实现复杂

## 实战建议

### 关键超参数详解

#### 1. 学习率（Learning Rate）

**推荐值**：
```python
# LoRA 微调
learning_rate: 1e-4 到 5e-4  # 比全量微调大 10 倍

# 全量微调
learning_rate: 1e-5 到 5e-5
```

**调整原则**：
- 学习率太大：训练不稳定，损失震荡
- 学习率太小：收敛慢，可能欠拟合
- 建议：从 1e-4 开始，观察损失曲线调整

#### 2. 批次大小（Batch Size）

**推荐值**：
```python
per_device_train_batch_size: 4-8  # 单卡批次
gradient_accumulation_steps: 4-8  # 梯度累积

# 实际批次 = per_device × accumulation × GPU数量
# 例如：4 × 4 × 1 = 16
```

**显存不足时**：
- 减小 batch_size
- 增大 gradient_accumulation_steps
- 两者乘积保持不变

#### 3. 训练轮数（Epochs）

**推荐值**：
```python
num_train_epochs: 3-5  # 一般 3 轮就够

# 数据量少时
epochs: 5-10

# 数据量大时
epochs: 1-3
```

**判断标准**：
- 观察验证集损失
- 不再下降时停止
- 使用 early stopping

#### 4. 权重衰减（Weight Decay）

```python
weight_decay: 0.01  # L2 正则化，防止过拟合
```

#### 5. 预热步数（Warmup Steps）

```python
warmup_steps: 100  # 或总步数的 10%
warmup_ratio: 0.1
```

**作用**：
- 训练初期使用较小学习率
- 避免梯度爆炸
- 提高训练稳定性

#### 6. 最大序列长度（Max Length）

```python
max_seq_length: 512-2048

# 根据任务选择
短文本（分类）: 512
中等文本（问答）: 1024
长文本（文档）: 2048-4096
```

**注意**：
- 越长显存占用越大（平方级增长）
- 超过训练长度的推理效果会下降

### 选择合适的 Rank

```python
# 任务复杂度 → Rank 选择
简单任务（情感分类）     → rank=4-8
中等任务（问答、摘要）   → rank=8-16
复杂任务（代码生成）     → rank=16-64
```

### 常见问题

**Q1：LoRA 效果不如全量微调怎么办？**
- 增大 rank（8 → 16 → 32）
- 增加训练数据
- 调整学习率

**Q2：训练后效果反而变差？**
- 可能过拟合，减少训练轮数
- 增加 dropout
- 检查数据质量

**Q3：如何选择在哪些层加 LoRA？**
- 默认：所有 attention 层（推荐）
- 显存不足：只在 query 和 value 层
- 效果优先：所有线性层

## 代码示例

### 训练配置（LLaMA Factory）

```yaml
# examples/train_lora/llama3_lora_sft.yaml
model_name_or_path: meta-llama/Llama-3-8B
dataset: your_dataset
finetuning_type: lora

# LoRA 配置
lora_rank: 8
lora_alpha: 16
lora_dropout: 0.05
lora_target: all  # 或 q_proj,v_proj

# 训练配置
num_train_epochs: 3
per_device_train_batch_size: 4
learning_rate: 5e-5
```

### 推理示例

```python
from llamafactory.chat import ChatModel

# 加载模型
chat_model = ChatModel(dict(
    model_name_or_path="meta-llama/Llama-3-8B",
    adapter_name_or_path="./output/lora",
    template="llama3"
))

# 对话
messages = [{"role": "user", "content": "你好"}]
response = chat_model.chat(messages)
print(response[0].response_text)
```

## 总结

LoRA 是目前最流行的微调方法，因为它：
- ✅ 成本低（显存、时间、金钱）
- ✅ 效果好（接近全量微调）
- ✅ 灵活（可切换、可合并）
- ✅ 易用（工具链成熟）

对于初学者，建议：
1. 从 rank=8 开始实验
2. 使用 QLoRA 节省显存
3. 先不合并，保持灵活性
4. 多尝试不同的超参数

## 相关阅读

- [模型量化详解](./03_模型量化详解.md)
- [模型评估方法](./04_模型评估方法.md)
- [LLaMA Factory 实战](./05_LLaMA_Factory实战.md)
