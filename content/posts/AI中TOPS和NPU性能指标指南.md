---
author: ["柿子"]
title: 'AI中TOPS和NPU性能指标指南'
date: 2024-06-07T11:54:44+08:00
summary: "解释TOPS是什么"
tags: ["AI","摘录","性能"]
typora-root-url: ../../static
typora-copy-images-to: ../../static/images
UseHugoToc: false
---

![科技感十足的图片，主题字符'NPU'](https://th.bing.com/th/id/OIG4.Q9ZK6muyAYLaz6ZQIqIj?pid=ImgGn)

在当今快速发展的技术环境中，[AI正在变革各行各业并推动创新](https://www.qualcomm.cn/news/blogs/2024/01/releases-2024-01-24)，理解AI性能指标的复杂性至关重要。过去许多AI模型需要在云端运行。当我们走向由终端侧生成式AI处理定义的未来时，我们必须能够评估计算平台可运行AI模型的性能、准确性和效率。如今，TOPS(每秒万亿次运算)是衡量处理器AI性能的主要方式之一。TOPS是基于处理器所需的架构和频率，衡量处理器潜在AI推理峰值性能的方法，比如神经网络处理器(NPU)。

**NPU是什么？**

在深入探讨TOPS的具体内容之前，让我们先看看NPU的重要性。对于终端侧AI处理，NPU在提高效率、为个人用户和企业提供创新的应用体验方面发挥着关键作用。评估这些专用处理器的性能需要全面了解其能力背后的关键指标。

[NPU的演进](https://www.qualcomm.cn/news/blogs/2024/03/blog-2024-3-6)改变了人们处理计算的方式。传统上，CPU负责执行AI算法。随着对处理性能的需求飙升，专用NPU应运而生，成为处理AI相关软件应用的专用解决方案。NPU旨在高效处理AI任务所需的复杂数学计算，提供出色的效率、性能和能效。

![img](https://www.qualcomm.cn/content/dam/qcomm-martech/dm-assets/images/blog/ai-tops/%E5%9B%BE%E7%89%872.png)

**AI TOPS是什么？**

TOPS作为展示处理器计算能力的指标，是衡量NPU性能的核心。

### **TOPS通过以万亿单位测量一秒钟内执行的运算（加法、乘法等）次数来量化NPU处理能力。**

这种标准化测量方式非常明确地显示了NPU的性能，可作为比较不同处理器和架构AI性能的关键指标。因为TOPS是针对NPU的基础性能指标，探索TOPS的计算参数以及它们如何决定性能至关重要，这有助于更深入地了解NPU的能力。

- **乘法累加(MAC)运算**执行AI工作负载中的核心数学公式。矩阵乘法由两类基础运算组成：累加器的乘法和加法。例如，一个MAC单元可在每个时钟周期内运行两类基础运算各一次，意味着它在每个时钟周期内执行两个运算。一个给定的NPU有一定数量的MAC单元，能够在不同精度级别进行运算，这取决于NPU架构。

- **频率**决定NPU及其MAC单元（以及CPU或GPU）运算的时钟速度（或每秒周期数），直接影响整体性能。更高的频率允许在单位时间内执行更多运算，从而提高处理速度。但是，提高频率也会导致更高功耗和发热，影响电池续航和用户体验。处理器TOPS计算通常使用峰值运行频率。

- **精度**指计算的颗粒度，通常精度越高模型准确性就越高，需要的计算强度也越高。最常见的高精度AI模型为32位和16位浮点精度，而速度更快的低精度低功耗模型通常使用8位和4位整数精度。当前行业标准为以INT8精度评估AI推理性能TOPS。

计算TOPS要从计算OPS开始，OPS等于MAC单元数乘以运行频率的两倍。TOPS数量是OPS除以一万亿的值，将公式更简单地列出，即：

### **TOPS = 2×MAC单元数×频率/1万亿。**


![img](https://www.qualcomm.cn/content/dam/qcomm-martech/dm-assets/images/blog/ai-tops/%E5%9B%BE%E7%89%873.png)

**TOPS和实际性能**

尽管TOPS提供了探索NPU能力的重要信息，我们仍必须将理论指标和实际应用联系起来。

### **毕竟，仅仅有高TOPS值并不能保证最佳的AI性能；各种因素协同作用的结果才能真正决定NPU实力。**

因此评估NPU性能时要考虑内存带宽、软件优化和系统集成等方面的因素。基准测试可以帮助我们超越数字，了解NPU在实际场景中的表现，其中时延、吞吐量和能效尤为重要。

[Procyon AI基准测试](https://benchmarks.ul.com/procyon)使用真实工作负载来帮助将理论性的TOPS评估转化为用户在使用AI推理的真实应用中对响应和处理能力的预期。它以多个精度运行六个模型，提供NPU不同性能表现的详细洞察。类似模型在生产力、媒体、创作者和其他应用中越来越常见。在Procyon AI和其他基准测试中有更快的性能表现，与实现更快推理和更好用户体验息息相关。

为此，分析实际性能可以为NPU的能力和局限性提供宝贵洞察。必须从可行性和实用性角度检验性能指标。

![img](https://www.qualcomm.cn/content/dam/qcomm-martech/dm-assets/images/blog/ai-tops/%E5%9B%BE%E7%89%874.png)

**根据用户需求评估NPU性能**

应对快速变化的NPU性能评估领域或许会让人望而生畏，但随着数字化转型（尤其是在AI领域）持续快速发展，深入了解TOPS对行业和个人来说都很重要。

最终，选择合适的系统级芯片(SoC)取决于用户、客户或组织的工作负载和优先级，而这一决策很可能需要取决于SoC中的NPU。

无论用户是优先考虑原始算力、能效还是模型准确度，骁龙X系列平台面向笔记本电脑，配备高达45TOPS的NPU，能够强力赋能PC，并将实际可用的AI体验引入用户的工作流程。

 ## 参考

- [深入了解面向计算的骁龙AI](https://www.qualcomm.cn/content/dam/qcomm-martech/dm-assets/documents/files/tong_guo_npu_he_yi_gou_ji_suan_kai_qi_zhong_duan_ce_sheng_cheng_shi_ai.pdf)

- [面向边缘终端优化生成式AI (qualcomm.cn)](https://www.qualcomm.cn/news/blogs/2024/05/blog-2024-05-30)
