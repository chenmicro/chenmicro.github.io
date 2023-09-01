---
title:  "Transformer相关的文章"
categories:
  - study
tags:
  - deep learning
---

**Transformer**[^Vaswani2017]本身的结构并不复杂，但是整个系统具有丰富的理论依据。文章中虽然介绍了该模型的主要内容和操作流程，但是并没有详细解释选择这些操作的原因（部分内容过于基础）。因此，需要阅读其它相关的文章作为补充：

* 在将文本数据输入到模型前，需要做**Word Embedding**。相比于**one-hot**编码，词向量[^Mikolov2013]能够增加数据的信息量，包括表示单词间的相关度等信息。在这里，虽然我们不需要这些额外的信息（因为没考虑语境所以可能是错的），但是可以通过这种方法得到指定维度的词向量，参考PyTorch的[Embedding](https://pytorch.org/docs/stable/generated/torch.nn.Embedding.html)实现。
* 给输入增加的位置信息并不是随意计算的，文章中选择的计算方式能够有效地计算出相对位置，见文章[Transformer 中的 positional embedding](https://zhuanlan.zhihu.com/p/359366717)。但是目前仍有部分trick未能得到确切的解释。
* 在上述对于数据预处理的基础上，**Transformer**才能够发挥最优秀的性能。整个模型的详细解释可阅读下列文章：[The Illustrated Transformer](https://jalammar.github.io/illustrated-transformer/)



[^Vaswani2017]: Vaswani, Ashish, et al. "Attention is all you need." Advances in neural information processing systems 30 (2017).

[^Mikolov2013]: Mikolov, Tomas, et al. "Efficient estimation of word representations in vector space." arXiv preprint arXiv:1301.3781 (2013).
