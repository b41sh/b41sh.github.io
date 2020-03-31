---
title: "如何阅读paper"
description: 介绍一种阅读paper的方法
date: '2020-03-31'
tags:
  - papers
keywords:
  - read
  - papers
---

要想对一个领域有深入的了解，阅读paper是必不可少的，学会高效阅读paper非常重要。
[How to Read a Paper](http://web.stanford.edu/class/cs245/readings/how-to-read-a-paper.pdf)
介绍了一种阅读paper方法，帮助大家提高读paper的速度和效率。

## 三遍阅读法(THE THREE-PASS APPROACH)

paper需要读三遍，每一遍都有不同的目标。
第一遍建立一个大体的印象，第二遍把握重要内容，第三遍深入理解。

### 第一遍(The first pass)

第一遍的重点是要快，花五到十分钟快速浏览一遍，获得一个全景印象。
关注如下几个方面：

1. 仔细阅读标题、摘要、简介(the title, abstract, and introduction)
2. 阅读章节和子章节的标题(the section and sub-section headings)
3. 扫一眼数学内容(the mathematical content)，了解理论基础
4. 阅读结论(the conclusions)
5. 浏览参考文献(the references)，勾选读过的paper

结束之后，回答5个问题：

1. 类型(Category): 评估(measurement)、系统分析(analysis)、研究原型(research prototype)
2. 上下文(Context): 相关论文、理论基础
3. 正确性(Correctness): 假设都成立吗？
4. 贡献(Contributions)
5. 清晰度(Clarity): 写的好吗？

这个时候要决定是否继续阅读？是否感兴趣，是否关注这个研究领域，作者是否做了无效的假设。
同样的，如果你自己写paper，也要写好摘要和章节标题，如果读者在五分钟内找不到要点，可能就不会继续阅读了。

### 第二遍(The second pass)

第二遍的重点是要精，仔细阅读每一部分内容，但是注意不要陷入细节。
把自己当成评审员，记下要点、不理解的内容、想问作者的问题，在空白处写下评论。
重点关注图表和插图是否存在错误，这往往是决定一篇paper是否真正优秀的关键。
标注没有读过的参考文献，以便进一步阅读。

这一遍通常要花费一个小时以上，这个时候应该掌握了paper的内容，可以有足够的论据总结出paper的主旨。
如果这篇paper只是有兴趣，但不是重点研究的领域，到这个程度就可以了。

有时也有可能没有读懂，这有可能是因为不熟悉这个领域，也可能是论文本身写的不好，也可能是累了。
这个是时候可以有三种选择：

1. 暂时搁置
2. 了解背景材料后重新阅读
3. 坚持继续读

### 第三遍(The third pass)

第三遍的关键是尝试重新实现一遍(virtually reimplement the paper)。
按照作者的假设，提出自己的想法，重新设计实现方案，并与原文的方案进行比较。
关注每一个细节，挑战每一个假设，记录下整个思路，并将有用的证明和技术加入到自己的工具库中。

通过这样的方式，能够自己从头重建整个系统。
同时可以发现原文存在的问题：能指出隐含的假设，遗漏的实验和潜在的问题。

## 文献分析(literature survey)

想深入了解一个领域，仅仅阅读一篇paper是不够的，可能需要读数十篇。
这时需要通过文献分析的方法找到合适的paper，也有三个步骤。

1. 选择合适的关键词，使用[Google Scholar](https://scholar.google.com/)或[CiteSeer](https://citeseerx.ist.psu.edu/index) 找到该领域最近被引用最多的三到五篇paper
2. 找到参考文献中引用量最多的作者，他们是这个领域最重要的研究人员，关注他们的最近发布的paper和conferences
3. 访问这些top conferences的网站，找到最近的高质量paper

反复重复这几个步骤，不断找出这个领域的最重要paper并阅读，不断总结归纳。


