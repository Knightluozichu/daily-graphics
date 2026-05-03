# 2026-05-03 · Neural Shader 第一印象

## 我现在理解的（用我自己的话）
- DX12 / Vulkan 扩展支持 Cooperative Vector —— 这是底层基建，让 GPU 着色器能直接调神经网络
- 复用 Tensor Core —— 这是性能关键，不是新硬件，是把 AI 加速单元拉进图形管线
- Neural Material 5x 性能 —— 当前最成熟的应用
- 能扩展到 Neural Shader / Neural Texture Compression —— 同一套机制的不同应用面
- 下一个 1000x 性能的着力点 

## 跟我工作的连接点
优化性能

## 我接下来想做的
下周用 unity跑下neural shader，对比下性能

## 我的初步理解（review 前）和 Review 后的修正
- 为什么不能在 shader 里调 import torch？
  - 之前理解：shader 是一套语言，借用 C，没有 Python 的库，也没有环境
  - 正确理解：Cooperative Vector 解决的不是"语言不通"，是"数据不能离开 GPU 那一帧的渲染上下文"。
- Neural Material 提升 5x，5x 的是什么？是渲染速度？是质量？是材质压缩比？还是其他？
  - 之前理解：5x 消耗
  - 正确理解：
    - 存储压缩：传统 PBR 材质要存 albedo / normal / roughness / metallic / AO 多张贴图（每张 4-16 MB），Neural Material 用一个小神经网络代替，显存占用大幅降低
    - 采样成本：传统着色每像素要做 5+ 次纹理采样 + 多次插值，Neural Material 一次推理出全部参数
    - 带宽节省：贴图小了，GPU 显存带宽压力小——这在车机这种带宽受限平台上是命根子
- 这跟你车机 HMI 工作有什么具体的连接点？
  - 之前理解：同等消耗下，提升的 hmi 视觉效果，当然首当其冲就是优化车机渲染性能
  - 正确理解：
    - 释放的 GPU 算力可以喂给车机 AI 任务（语音、DMS、360 环视、推荐算法）。
    - Neural Shader 把渲染从 GPU 算力的"吞噬者"变成"共享者"。
    - 这是智能座舱时代的关键解锁。

## 我还没搞懂的（下次的种子）
- Cooperative Vector 和过去手写 shader MLP 的本质区别？
- Tensor Core 物理结构到底长什么样？
- 车机 GPU（高通 8295）能多大程度跑 Neural Shader？
- 比如"Cooperative Vector 跟 SIMD 有什么区别？"
- 比如"Tensor Core 到底是什么物理结构？"