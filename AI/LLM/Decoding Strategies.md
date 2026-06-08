主要的推理逻辑：
* **Greedy Search**：最大可能性的下一个token（很多语言模型推理默认的设置）
* **Beam Search**：批量采样多个token推理结果，权重加和后返回最大值
* **Top-K Sampling**：随机选择K个最高概率的token返回
* **Top-P Sampling**：
* **Temperature**：温度值取0.1~1来控制softmax函数的比例范围