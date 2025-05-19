### **《深度学习基础》课程第四章PPT内容详解**

---

#### **一、核心框架回顾：机器学习三要素**
1. **模型定义**：构建含未知参数的函数  
2. **损失函数**：量化预测误差  
3. **优化方法**：梯度下降寻找最优参数  
   - **关键公式**：  
     $$\theta_{new} = \theta_{old} - \eta \cdot \nabla L(\theta_{old})$$

---

#### **二、从线性到深度学习的演进**
##### **1. 线性模型局限性**
- **问题**：单层线性模型无法表达复杂非线性关系（如曲线、多模态分布）。  
- **解决方案**：引入非线性激活函数（如Sigmoid、ReLU）构建多层网络。 ^5a7ba8

##### **2. 非线性扩展：Sigmoid叠加**
- **数学形式**：  
  $$y = b + \sum_{i} c_i \cdot \text{sigmoid}(b_i + \sum_{j} w_{ij} x_j)$$
  - **参数规模**：若输入特征数为 $M$，使用$N$个Sigmoid，总参数量为$(M + 1) \times N + 1$。  
  - **示例**：$M=3$, $N=3$ → 参数总量$3 \times 3 + 3 + 1 = 13$。

##### **3. [[矩阵化表示]]**
- **前向传播公式**：  
  $$\mathbf{r} = \mathbf{W} \mathbf{x} + \mathbf{b}, \quad \mathbf{a} = \sigma(\mathbf{r}), \quad y = \mathbf{c}^T \mathbf{a} + b$$
  - **矩阵维度**：  
    - $\mathbf{W} \in \mathbb{R}^{N \times M}$（权重矩阵）  
    - $\mathbf{b} \in \mathbb{R}^{N}$（偏置向量）  
    - $\mathbf{c} \in \mathbb{R}^{N}$（输出层权重）

---

#### **三、深度学习核心：全连接前馈网络**
##### **1. 网络结构**
- **层级划分**：  
  - **输入层**：原始特征（如28×28图像展平为784维向量）  
  - **隐藏层**：多个全连接层 + 激活函数（如Sigmoid）  
  - **输出层**：Softmax归一化（分类任务）或线性输出（回归任务）  

- **示例结构**：  
  ```plaintext
  输入层 (784) → 隐藏层1 (256) → 隐藏层2 (128) → 输出层 (10)
  ```

##### **2. 计算复杂度（FLOPs）**
- **单层计算量**：  
  $$\text{FLOPs} = 2 \times I \times O \quad (I: \text{输入维度}, O: \text{输出维度})$$
  - **示例**：输入层784 → 隐藏层256，计算量$2 \times 784 \times 256 = 401,408$ FLOPs。

##### **3. 并行加速**
- **矩阵运算**：利用GPU并行计算加速全连接层的矩阵乘法。  
- **效率对比**：GPU比CPU快10-100倍，支撑深度学习大规模计算需求。

---

#### **四、损失函数与优化**
##### **1. 交叉熵损失（分类任务）**
- **公式**：  
  $$L = -\sum_{i=1}^C y_i \log(\hat{y}_i)$$
  - **Softmax输出**：
    $$\hat{y}_i = \frac{e^{z_i}}{\sum_{j=1}^C e^{z_j}}$$
  - **特性**：对错误预测惩罚更强（相比MSE），适合分类。

##### **2. 梯度下降优化**
- **批量梯度下降 (BGD)**：使用全量数据计算梯度，稳定但计算量大。  
- **小批量梯度下降 (MBGD)**：折中方案（如batch_size=64），兼顾速度与稳定性。  
- **反向传播**：链式法则计算梯度，逐层更新权重。

---

#### **五、深度学习优势与设计**
##### **1. 深度 vs 宽度**
- **实验结论**（ILSVRC竞赛）：  
  - **深层网络**（如ResNet-50）错误率17.1%，**浅层宽网络**错误率22.5%。  
  - **原因**：深层网络通过层级抽象实现模块化特征学习。

##### **2. 模块化设计**
- **低级特征**：边缘、纹理（浅层学习）  
- **高级特征**：物体部件、整体结构（深层学习）  
- **类比编程**：函数封装复用，提升代码可维护性。

##### **3. 网络结构设计**
- **经验法则**：  
  - **输入层**：与数据维度一致（如图像展平）  
  - **隐藏层**：逐层减少维度（如784→256→128→64）  
  - **输出层**：与任务匹配（分类：Softmax；回归：线性）  
- **自动搜索**：神经架构搜索（NAS）自动化设计网络。

---

#### **六、代码示例与实战**
##### **1. PyTorch实现框架**
```python
import torch
import torch.nn as nn

class NeuralNetwork(nn.Module):
    def __init__(self):
        super().__init__()
        self.flatten = nn.Flatten()
        self.linear_relu_stack = nn.Sequential(
            nn.Linear(784, 256),  # 输入层→隐藏层1
            nn.ReLU(),
            nn.Linear(256, 128),  # 隐藏层1→隐藏层2
            nn.ReLU(),
            nn.Linear(128, 10)    # 隐藏层2→输出层
        )

    def forward(self, x):
        x = self.flatten(x)
        logits = self.linear_relu_stack(x)
        return logits
```

##### **2. 训练流程**
```python
model = NeuralNetwork()
loss_fn = nn.CrossEntropyLoss()
optimizer = torch.optim.SGD(model.parameters(), lr=0.01)

for epoch in range(10):
    for X, y in dataloader:
        pred = model(X)
        loss = loss_fn(pred, y)
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
```

---

#### **七、总结**
- **核心思想**：通过多层非线性变换逐级提取特征，解决复杂任务。  
- **关键优势**：自动特征学习、模块化表达、并行计算高效性。  
- **实践建议**：优先尝试经典网络结构（如全连接网络、CNN），结合任务调整深度与宽度。