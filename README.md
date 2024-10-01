# <div align="center">TensorRT-LLM如何加速推理</div>

## 🫡简介

这篇文章的内容如标题所示，作者花时间对[TensorRT-LLM](https://github.com/NVIDIA/TensorRT-LLM)的原始仓库中，
以[llama2-7b-chat](https://huggingface.co/meta-llama/Llama-2-7b-chat-hf)模型作为案例，分析代码。

结合查询的资料分析出TensorRT-LLM如何优化推理。

## 🤔思路解析

TensorRT-LLM其实就是对常见的大模型如llama，gpt-j写了一段代码示例，使用TensorRT优化并推理。TensorRT-LLM核心的优化思路就是TensorRT对对大模型进行优化的思路。

所以我根据我查到的资料来说说TensorRT如何对模型进行优化，其中优化最重要的地方是这几点：

- 模型量化
- 层与张量融合
- GPU优化

下面我讲解一下😎：

### 模型量化：简单粗暴加快推理

一般来说，我们训练模型时会使用FP32的数据精度来进行训练，让模型得到不错的训练效果。但是在推理时，则**不需要**
这样高的数据精度，更高的数据精度需要更高的算力和内存😒。

在推理时，我们可以使用TF32，FP16，BF16和INT8这些数据精度来对模型进行量化。

> 数据精度比较
> TF32 > FP16 > BF16 > INT8

### 层与张量融合：特别秀的操作

在之前的TensorRT官网上有这些图，不过现在没有了，我重画了一下：

<div align="center">
    <img src="images\master_drawing.png" alt="描述" style="width:80%;"/>
</div>

> 图一：未压缩的原始网络结构，这应该是googlenet。

<div align="center">
    <img src="images\vertical_integration.png" alt="描述" style="width:80%;"/>
</div>

> 图二：对层进行垂直融合。CBR是**Conv、bias和relu的首字母**，CBR并不是什么特殊名词。

<div align="center">
    <img src="images\horizontal_fusion.png" alt="描述" style="width:80%;"/>
</div>

> 图三：对层进行水平融合。

示例网络如图一所示，这是个googlenet。在原始不变的网络中，除去输入和输出层这里一**共有19神经网络层**
。在对网络层进行两次融合之后，即垂直融合和水平融合，层数**减少至5层**😍。

减少层数有助于加速GPU底层张量的处理。除了层融合这种非常明显的操作，TensorRT还对神经网络做了更多优化。

对于新手来说，这部分复杂的优化很容易看了就让人心生退意，不用担心，我们在代码中调用api就好😚。

### GPU优化：针对设备进行底层优化

TensorRT毕竟是[NVIDIA](https://nvidia.com)自家的推理库，在自家公司对自家芯片做优化也是十分合理的。

在这个TensorRT-LLM中，我们需要为转换过后的模型制作一个针对与**当前设备**，**当前GPU的**，**深度优化过**
的引擎。这个引擎只能在当前的设备上使用，换到其他设备上，是没有优化效果的。

参考资料：

- [TensorRT模型加速 | 网络结构优化 | 低精度推理](https://blog.csdn.net/qq_41204464/article/details/124545576)
- [【模型压缩】网络层与算子融合](https://blog.csdn.net/qq_34106574/article/details/132445541)
