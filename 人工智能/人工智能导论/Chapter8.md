### **《Transformer进阶：架构、应用与训练策略》PPT内容详解**

---

#### **一、自注意力机制在图像中的应用**
##### **1. 图像作为向量集合**
- **核心思想**：将图像分割为多个小块（如16x16像素），每个块视为一个向量，整体作为向量序列处理。  
  - **示例**：224x224图像 → 拆分为196个16x16块 → 每个块展平为256维向量。  
  - **优势**：突破CNN的局部限制，实现全局信息交互。

##### **2. Vision Transformer（ViT）**
- **流程**：  
  1. **图像分块**：输入图像 → 网格化分块 → 线性投影为嵌入向量。  
  2. **添加位置编码**：保留空间位置信息。  
  3. **Transformer编码器**：多层自注意力 + 前馈网络。  
  4. **分类头**：全局平均池化 → 全连接层输出类别。  
- **性能**：在大规模数据集（如ImageNet-21K）上媲美CNN，依赖数据量。

##### **3. Swin Transformer**
- **改进点**：引入层级结构 + 局部窗口注意力。  
  - **窗口划分**：每层将图像分为不重叠窗口，窗口内计算自注意力。  
  - **窗口移位**：逐层移动窗口，实现跨窗口信息交互。  
  - **优势**：降低计算复杂度（从 $O(N^2)$ 到 $O(N)$），适配密集预测任务（如检测、分割）。

---

#### **二、自注意力与CNN/GNN的对比**
##### **1. 自注意力 vs CNN**
| **特性**               | **CNN**                          | **自注意力**                     |
|------------------------|----------------------------------|----------------------------------|
| **感受野**             | 局部（堆叠扩大）                  | 全局（单层覆盖全图）              |
| **参数效率**           | 高（权重共享）                    | 较低（需学习QKV矩阵）             |
| **数据依赖**           | 小数据表现好                      | 需大数据训练（ViT需JFT-300M）     |
| **位置感知**           | 隐式（通过卷积核位置）            | 显式（位置编码）                  |

- **本质联系**：CNN是自注意力的特例（固定局部感受野），自注意力是动态感受野的泛化。

##### **2. 自注意力 vs GNN**
- **图注意力网络（GAT）**：  
  - 自注意力可视为全连接图的GAT，节点间边权重由注意力分数动态计算。  
  - **差异**：GNN通常处理稀疏图，自注意力处理全连接图。  
- **应用场景**：  
  - **GNN**：社交网络、分子结构等图数据。  
  - **自注意力**：序列、图像等结构化数据。

---

#### **三、Transformer完整框架**
##### **1. 编码器（Encoder）**

>[!info] 结构
  输入 → 嵌入 + 位置编码 → [多头自注意力 → 残差 + 层归一化 → 前馈网络 → 残差 + 层归一化] × N
  
- **关键组件**：  
  - **残差连接**：缓解梯度消失，公式 $\text{Output} = x + \text{Sublayer}(x)$。  
  - **层归一化（LayerNorm）**：对每个样本独立归一化，稳定训练。  
    $$
    \text{LayerNorm}(x) = \gamma \cdot \frac{x - \mu}{\sqrt{\sigma^2 + \epsilon}} + \beta
    $$

##### **2. 解码器（Decoder）**
- **自回归解码（Autoregressive, AT）**：  
  - **流程**：逐步生成输出，每一步依赖前一步结果（如机器翻译）。  
  - **掩码自注意力**：防止未来信息泄漏，确保第 \( t \) 步仅关注前 \( t-1 \) 步。  
    ```python
    # 掩码矩阵（下三角为1）
    mask = torch.tril(torch.ones(seq_len, seq_len))
    ```
  - **交叉注意力（Cross-Attention）**：解码器查询（Query）来自自身，键值（Key/Value）来自编码器输出。

- **非自回归解码（Non-Autoregressive, NAT）**：  
  - **特点**：并行生成全部输出，需预测输出长度或预设最大长度。  
  - **应用**：语音合成（Tacotron 2）、实时翻译。  
  - **挑战**：输出质量通常低于AT，因缺失时序依赖。

---

#### **四、训练策略与技巧**
##### **1. Teacher Forcing**
- **原理**：训练时解码器输入使用真实标签（而非模型预测结果）。  
  - **示例**：翻译任务中，即使上一步预测错误，下一步仍输入正确词。  
  - **优势**：加速收敛，避免错误累积。  
  - **缺陷**：导致曝光偏差（Exposure Bias），测试时模型未见过自身错误。

##### **2. 混合训练（Scheduled Sampling）**
- **改进**：逐步引入模型预测作为输入，缓解曝光偏差。  
  - **概率调整**：随训练进行，增加使用模型预测的概率。

##### **3. 束搜索（Beam Search）**
- **推理优化**：保留Top-K候选序列，平衡生成质量和多样性。  
  - **示例**：机器翻译时保留5条最优路径，最终选择最高分序列。

---

#### **五、典型应用场景**
##### **1. 机器翻译**
- **流程**：  
  ```plaintext
  源语言 → 编码器 → 上下文向量 → 解码器 → 目标语言
  ```
- **模型**：Google的Transformer-base、Facebook的FairSeq。

##### **2. 文本生成（GPT系列）**
- **架构**：仅使用解码器堆叠，通过掩码自注意力实现单向上下文建模。  
- **训练**：自回归语言模型目标（预测下一个词）。

##### **3. 多模态任务（CLIP、DALL·E）**
- **交叉注意力应用**：对齐文本与图像特征，支持文生图、图生文。

---

#### **六、代码示例（PyTorch实现掩码自注意力）**
```python
import torch
import torch.nn as nn

class MaskedSelfAttention(nn.Module):
    def __init__(self, embed_size, heads):
        super().__init__()
        self.embed_size = embed_size
        self.heads = heads
        self.head_dim = embed_size // heads

        self.qkv = nn.Linear(embed_size, 3 * embed_size)
        self.fc_out = nn.Linear(embed_size, embed_size)

    def forward(self, x, mask=None):
        N, seq_len, _ = x.shape
        qkv = self.qkv(x).reshape(N, seq_len, 3, self.heads, self.head_dim)
        q, k, v = qkv.permute(2, 0, 3, 1, 4)  # [3, N, heads, seq_len, head_dim]

        scores = torch.matmul(q, k.permute(0, 1, 3, 2)) / (self.head_dim ** 0.5)
        if mask is not None:
            scores = scores.masked_fill(mask == 0, float('-inf'))

        attention = torch.softmax(scores, dim=-1)
        out = torch.matmul(attention, v).permute(0, 2, 1, 3).reshape(N, seq_len, -1)
        return self.fc_out(out)
```

---

#### **七、总结**
- **核心价值**：Transformer通过自注意力实现全局依赖建模，突破CNN/RNN的局限性。  
- **架构演进**：从ViT的图像应用到Swin Transformer的层级设计，持续推动多领域发展。  
- **训练挑战**：平衡自回归生成质量与速度，探索非自回归模型的潜力。  
- **未来方向**：轻量化设计（如MobileViT）、多模态统一架构（如Flamingo）。