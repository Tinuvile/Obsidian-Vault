### **自注意力机制公式分步示例**

---

#### **1. 输入表示**
假设输入序列包含两个词，每个词的嵌入向量维度为3：  
$$
X = \begin{bmatrix}
1 & 2 & 3 \\
4 & 5 & 6
\end{bmatrix} \quad (\text{形状：2×3})
$$

---

#### **2. 生成Query、Key、Value向量**
定义可学习的参数矩阵 $W_Q, W_K, W_V$（简化维度为3×2）：  
$$
W_Q = \begin{bmatrix}
1 & 0 \\
0 & 1 \\
1 & 1
\end{bmatrix}, \quad
W_K = \begin{bmatrix}
0 & 1 \\
1 & 0 \\
1 & 1
\end{bmatrix}, \quad
W_V = \begin{bmatrix}
1 & 1 \\
0 & 1 \\
1 & 0
\end{bmatrix}
$$

- **计算Query（Q）**：  
  $$
  Q = X \cdot W_Q = \begin{bmatrix}
  1\cdot1 + 2\cdot0 + 3\cdot1 & 1\cdot0 + 2\cdot1 + 3\cdot1 \\
  4\cdot1 + 5\cdot0 + 6\cdot1 & 4\cdot0 + 5\cdot1 + 6\cdot1
  \end{bmatrix} = \begin{bmatrix}
  4 & 5 \\
  10 & 11
  \end{bmatrix}
  $$

- **计算Key（K）**：  
  $$
  K = X \cdot W_K = \begin{bmatrix}
  1\cdot0 + 2\cdot1 + 3\cdot1 & 1\cdot1 + 2\cdot0 + 3\cdot1 \\
  4\cdot0 + 5\cdot1 + 6\cdot1 & 4\cdot1 + 5\cdot0 + 6\cdot1
  \end{bmatrix} = \begin{bmatrix}
  5 & 4 \\
  11 & 10
  \end{bmatrix}
  $$

- **计算Value（V）**：  
  $$
  V = X \cdot W_V = \begin{bmatrix}
  1\cdot1 + 2\cdot0 + 3\cdot1 & 1\cdot1 + 2\cdot1 + 3\cdot0 \\
  4\cdot1 + 5\cdot0 + 6\cdot1 & 4\cdot1 + 5\cdot1 + 6\cdot0
  \end{bmatrix} = \begin{bmatrix}
  4 & 1 \\
  10 & 9
  \end{bmatrix}
  $$

---

#### **3. 计算注意力分数**
- **点积缩放**（假设 $d_k = 2$）：  
  $$
  \text{Scores} = \frac{Q \cdot K^T}{\sqrt{d_k}} = \frac{1}{\sqrt{2}} \begin{bmatrix}
  4\cdot5 + 5\cdot11 & 4\cdot4 + 5\cdot10 \\
  10\cdot5 + 11\cdot11 & 10\cdot4 + 11\cdot10
  \end{bmatrix} = \frac{1}{\sqrt{2}} \begin{bmatrix}
  95 & 66 \\
  221 & 150
  \end{bmatrix}
  $$
  实际计算：  
  $$
  \text{Scores} = \begin{bmatrix}
  67.18 & 46.67 \\
  156.21 & 106.07
  \end{bmatrix}
  $$

---

#### **4. Softmax归一化**
对每行进行Softmax（指数归一化）：  
$$
\text{Softmax}(\text{Scores}) = \begin{bmatrix}
\frac{e^{67.18}}{e^{67.18} + e^{46.67}} & \frac{e^{46.67}}{e^{67.18} + e^{46.67}} \\
\frac{e^{156.21}}{e^{156.21} + e^{106.07}} & \frac{e^{106.07}}{e^{156.21} + e^{106.07}}
\end{bmatrix} \approx \begin{bmatrix}
1 & 0 \\
1 & 0
\end{bmatrix}
$$
（实际中数值极大，此处简化为极端值以直观展示）

---

#### **5. 加权求和Value**
$$
\text{Output} = \text{Softmax}(\text{Scores}) \cdot V = \begin{bmatrix}
1 & 0 \\
1 & 0
\end{bmatrix} \cdot \begin{bmatrix}
4 & 1 \\
10 & 9
\end{bmatrix} = \begin{bmatrix}
4 & 1 \\
4 & 1
\end{bmatrix}
$$

---

#### **6. 多头注意力（以2头为例）**
- **拆分Q、K、V**：  
  将每个矩阵按列分成两部分（假设每头维度为1）：  
  $$
  Q_1 = \begin{bmatrix} 4 \\ 10 \end{bmatrix}, \quad Q_2 = \begin{bmatrix} 5 \\ 11 \end{bmatrix}, \quad \text{同理拆分 } K, V
  $$

- **独立计算每个头**：  
  按上述步骤计算每个头的输出，最后拼接结果：  
  $$
  \text{MultiHead}(Q, K, V) = \text{Concat}(\text{head}_1, \text{head}_2) \cdot W^O
  $$

---

#### **7. 位置编码示例**
- **正弦编码**（位置1和位置2，维度2）：  
  $$
  PE_{1,0} = \sin(1/10000^{0/2}) = \sin(1) \approx 0.8415
  $$
  $$
  PE_{1,1} = \cos(1/10000^{0/2}) = \cos(1) \approx 0.5403
  $$
  最终位置编码矩阵：  
  $$
  PE = \begin{bmatrix}
  0.8415 & 0.5403 \\
  0.9093 & -0.4161
  \end{bmatrix}
  $$

---

### **总结**
通过具体数值示例，自注意力机制的计算流程清晰可见：  
1. **线性变换**生成Q、K、V。  
2. **点积缩放**计算相关性。  
3. **Softmax归一化**得到注意力权重。  
4. **加权求和**Value生成最终输出。  
5. **多头机制**扩展模型对不同相关性的捕捉能力。  
6. **位置编码**显式引入顺序信息。