---
title: 低秩自适应（LoRA）原理、应用以及发展
mathjax: true
date: 2024/4/17
tag:
    - AI
    - LoRA
category: 技术学习与分享
---

# 摘要

在人工智能领域，低秩自适应（LoRA）技术是一种通过使用低维结构来近似大型模型的高维结构，以降低模型复杂性的技术。它旨在识别和删除大型模型中的冗余或不相关信息，创建一个更小、更易于管理的模型表示，同时保留原始模型的性能。低秩自适应技术的优点包括减少模型参数数量、提高训练吞吐量、减少模型推理延迟等。本文将从大语言模型的训练入手介绍LoRA的概念，并介绍LoRA在大语言模型训练以及图像生成方面的发展现状，最后总结LoRA对当今人工智能发展的作用。

# 绪言

低秩自适应技术可以应用于各种机器学习任务，如图像分类、语音识别、自然语言处理等。它通常在预训练模型的权重矩阵中添加一个低秩矩阵，使模型能够更有效地学习特定于任务的信息。在训练过程中，低秩自适应层可以初始化随机值，并在微调过程中更新以学习特定任务的信息。

在众多人工智能应用中，大语言模型属于时下相对热门的一个分支，尤其是GPT-4的出现使得大语言模型的潜力被进一步挖掘。但同时，模型的参数量也不断增长；对于GPT-3，其模型的参数量已经能够达到1750亿，GPT-4模型的参数量达到了1.8万亿；在模型具有如此巨大的可调节参数的情况下，想要对模型进行微调变成了一件非常困难的事情，且需要消耗大量的硬件资源；在背景之下，LoRA（Low-Rank Adaptaion）应运而生,通过对原模型的矩阵进行冻结并使用分解后的低秩矩阵来代替Transformer中的权重矩阵并累加到原模型的方式来实现对模型的微调。

# LoRA出现之前相关领域的发展情况概述
## 面临的困境

问题可以描述为在机器学习过程中（在目前讨论的内容中，是大语言模型的训练）对预训练模型再训练时参数的调节问题。对于大语言模型来说，训练的参数量常常百亿甚至千亿级别的（GPT-3的参数量就达到了175 Billlion），要对如此大的模型进行调参，需要使用大量的运算资源；同时，即便运算资源充足，对如此多的参数进行调整也会耗费大量的精力，甚至是不可能的。因此，需要设法减少模型训练过程中需要调节的参数量。

## LoRA之前的解决方案有哪些
对大模型微调的参数进行修正的研究有很多，就大语言模型来说，先前的研究主要有两个方向：增加适应层（Adapter Layers）和优化输入层后的激活函数（input layer activations）。


然而，适应层的添加会导致推理延迟，因为其与大型神经网络依赖硬件并行计算的工作模式不同，必须依序进行训练；另一方面，优化输入层后的激活函数涉及到对彼此相关联的参数进行调整，很难找到一个最优的解。




# LoRA原理

神经网络中含有多个layer，这些layer之间会进行多次矩阵运算。在大语言模型中，每一层被称为一个Transformer。在以往的训练过程中，这些层的矩阵表是通常是满秩的。

[Armen Aghajanyan等人的研究](http://arxiv.org/abs/2012.13255)表明，在某些神经网络训练过程中，预训练模型有较低的”内在维度（instrisic dimension）“，仅仅由这些内在维度，就能够较好地描述大模型的特征，意即用较低维度的数据代表更高维度的数据，并由此降低矩阵运算的成本。由此，作者提出了”内在秩“的概念。LoRA模型中的“内在秩”可以理解为模型在各类下游任务上泛化的过程中，优化各类任务的公共低维本征（low-dimensional intrinsic）子空间中非常少量的几个自由参数。

对于一个$W_0 ∈ R^{d×k}$的权重矩阵，训练过程实际上是把上一层的$W_0$变成了下一层的$W_0 + \Delta W$ ，而 $d\times k$维的$\Delta W$ 可以分解为$d\times r$以及$r\times k$维的两个矩阵（记为A,B）的积，其中$r$远远小于$d$和$k$，对于输入x，输出的结果h为：
$$
h = W_0x + ∆Wx = W_0x + BAx
$$

![lora用右边的矩阵之积替换了权重矩阵W](attachments/Pasted%20image%2020231129211210.png)

# LoRA优势以及在大语言模型中的具体部署效果

## 优势

### 模型微调的泛化实现

一种更通用的微调范式是只微调预训练参数的子集。LoRA进一步降低了约束，在训练中不要求满秩，可以更灵活应对新任务。总而言之，随着模型可训练参数的增加，LoRA大致收敛到原始模型，而基于Adapter的方法收敛到MLP，基于Prefix Tuning的方法收敛到不能接受长输入的模型。

### 没有推理延迟

在LoRA训练过程中，$W = W_0 + BA$，当需要切换到不同的下游任务时，只要将矩阵$W$和矩阵$BA$做减法就可以恢复$W_0$，大大减小了内存开销，并且不会引入额外的推理延迟

## 部署的具体效果以及限制
原则上，LoRA可以用于任何神经网络的子集中，以减少可训练参数。Transformer架构中，自注意力模块有四个权重矩阵（$W_q,W_k,W_v,W_o$），两个MLP模块；在只对下游任务注意力的权重进行调整的情况下，LoRA能够将显卡内存占用降低2/3；在GPT-3 175B的训练中，checkpoint的大小从350GB左右缩减到了35MB，这意味着LoRA可以使用更少的I/O资源，并降低GPU的使用率。而同时，作者经过一系列实验表明，在训练参数较少的情况下，LoRA的表现也可以媲美甚至超过传统的训练方法。


但同时，LoRA也存在限制。在批处理不同任务的数据时，需要对每个任务进行单独处理，实现复杂，但是当推理延迟可以被接受时，也可以实现根据任务动态选择LoRA模块。

# LoRA目前的应用

## 消费级设备上的大模型应用

如上所述，LoRA能够对大语言模型进行高效的微调，这使得大模型的调参变得更加容易，降低了大语言模型的训练门槛，也使得大语言模型的用户能够更加容易对大语言模型进行定制；同时，由于训练门槛的降低，使得像**Alpaca Lora**这样在消费级硬件上能够运行的大语言模型能有相对好的表现，伴随着目前硬件性能的不断提升，在客户端本地运行的大模型的发展会进一步加速。

LoRA能够用较少的参数描述一组特征，这在图像生成领域也能够有很广泛的应用。作为同样消耗硬件资源的图像生成任务，LoRA的应用也能够使其在性能相对较低的硬件平台进行训练；LoRA能够作为Stable Diffusion模型的一个插件运行，在不修改SD模型的情况下，使用较少的提示词或样本图片生成目标图片；同时，调优后的LoRA能够作为某种图像风格的适配器，实现图像风格的迁移。

## 自然语言转换为工具语言

把自然语言转换为某些有语法规定的语言，在某些情境下可以提高效率；但前提是

[Amine Rebei](https://arxiv.org/search/cs?searchtype=author&query=Rebei,+A)提出可以使用LoRA训练的三个大语言模型将自然语言翻译成SQL语句（作者称之为text-to-SQL）。只要向模型提供问题、数据库的细节以及SQL语法，模型就能够生成对应的SQL语句；训练好的模型在text-to-SQL不仅比GPT-4的速度快，还能有接近甚至高于GPT-4的准确率
![训练结果](attachments/Pasted%20image%2020231207010922.png)


## 提高训练结果的时效性
清华大学一个研究团队提出了**LCM LoRA**，可以实现实时的图像生成。
### LCM
基于OpenAI 的宋飏博士在提出的一致性模型（Consistency Model，CM），提出了“潜在一致性模型”（Latent Consistency Model,LCM）.潜在一致性模型支持给定条件的图像生成任务，并结合了潜在编码、无分类器引导等诸多在扩散模型中被广泛应用的技术，大大加速了条件去噪过程，为诸多具有实际应用意义的任务打开了一条通路。
具体实现上，不同于迭代求解这一常微分方程，潜在一致性模型要求对常微分方程进行直接的单步求解，直接预测方程的最终解，从而在理论上能够在单步内生成图片。而要训练这样的模型，可以对预训练的扩散模型（比如Stable Diffusion）进行参数微调，在极少的资源消耗下赋予模型快速生成的效果

### LCM LoRA
在潜在一致性模型的基础上，作者团队随后进一步发布了他们关于 LCM-LoRA 的技术报告。由于潜在一致性模型的蒸馏过程可以被视作是对于原有的预训练模型的微调过程，从而可以使用 LoRA 等高效微调技术来训练潜在一致性模型。得益于 LoRA 技术带来的资源节省，作者团队在 Stable Diffusion 系列中参数量最大的 SDXL 模型上进行了蒸馏，成功得到了能够在极少步数内生成与 SDXL 数十步相媲美的潜在一致性模型。

## LoRA的发展动向

### DyLoRA

LoRA能够减少训练参数方面的开销，但是SVD(上文的矩阵B和矩阵A的乘积)仍然有两个问题：其一，是SVD的大小是固定的，如果想要改变SVD的秩的话，就需要重新训练；其二，将SVD的秩调整到最优也是一件非常耗费精力的事。因此，动态低秩自适应(Dynamic Low-Rank Adaptation)应运而生。

与传统 LoRA不同的是，DyLoRA 在训练过程中针对一系列 rank 而不是单一 rank 进行训练。这意味着 DyLoRA 可以动态地适应不同的 rank，提供模型更大的灵活性和适应性。对多个rank进行训练，还能让DyLoRA知道在不同的设备或对于不同的任务应该使用什么秩来进行训练，从而达到相对好的性能发挥。

对比LoRA，在一些常见的自然语言模型（如RoBERTa,GPT等）的训练中，DyLoRA速度提高了4至7倍，并且在更多不同的模型上表现出一致性。
### S-LoRA

对于行业大模型来说，仅拥有强大的”底座“基础大型模型并不足够，更重要的是如何让这些基础模型能够为垂直领域的业务赋能。或者说，如何让一个基础的模型能够快速高效地微调而向不同的领域提供服务。

一个思路是，通过对一个基础大型模型使用不同领域的数据进行LoRA方式的微调，可以得到多个小型LoRA适配器作为微调结果。这些适配器的参数非常小（例如只有基础模型参数的1%），而且在部署阶段，适配器可以共享基础模型的权重。这样一来，不仅微调阶段的计算量很小，LoRA相比全量微调更加高效，而且在推理阶段具有更大的优势。

S-LoRA解决的问题是如何在单台机器上部署数千个同源的LoRA adapter。所谓同源就是这些LoRA adapter都是来自同一个base model的权重。针对从同一基础模型用LoRA微调出来的多个adapter结果，S-Lora的提出者提出了一套高效部署方案。通过扩展Batching策略，PageAttention内存管理策略和并行策略，实现单个GPU服务上千个LoRA adapter的效果。

# 总结

在人工智能对硬件要求日益提高的今天，LoRA的不断发展让相关领域的模型训练等活动更加能够突破性能瓶颈，将人工智能的应用领域进一步推广，并降低训练、使用这些模型的时间成本，使得人工智能能够更好地服务于人类社会。

# 参考

[1.LORA: LOW-RANK ADAPTATION OF LARGE LANGUAGE MODELS](https://arxiv.org/abs/2106.09685)

[2.知乎对LoRA的介绍](https://zhuanlan.zhihu.com/p/620928739)

[3. Alpaca LoRA](https://github.com/tloen/alpaca-lora)

[3.THU LCM LoRA](https://arxiv.org/abs/2310.04378)

[4.DyLoRA](https://arxiv.org/abs/2210.07558)

[5.Dynamic Retrival LoRA知乎介绍](https://zhuanlan.zhihu.com/p/623455592)

[6.Fine-Tuning Language Models for Context-Specific SQL Query Generation](https://arxiv.org/abs/2312.02251)

[7.S-LoRA: Serving Thousands of Concurrent LoRA Adapters](https://arxiv.org/abs/2311.03285)

[8.S-Lora 知乎介绍](https://zhuanlan.zhihu.com/p/667213961)