---
layout: post
title:  "图解go增量编译"
date:   2022-11-21 22:37:19 +0800
categories: jekyll update
typora-copy-images-to: pic
---

# 为什么出触发写这样的一个主题的文章

# 图解go增量编译（go incremental builds）

为什么要写这篇文章？

因为最近在做的一个项目，被go 增量编译的速度吓得吃不下饭（哈哈，这本来是好事）。设计了一个算法，解决了一个问题，最后发现这个问题不存在（当然问题还是存在的，只是那个时候我对go的增量编译里面的实现细节不是很清楚）。

本文会先用图片的形式直接给出，go增量编译的关键技术细节，然后提两句，解决未知问题的时候，该如何降低未知方案的风险。

## 1. go的编译流程

**静态语言的构建就是将编译单元的源码编译为对应的中间目标文件(.o/.a/.class)，然后将这些目标文件通过链接器链接在一起形成最终可执行文件的过程**。

![image-20221122230722087](https://github.com/Baojia-Yang/Baojia-Yang.github.io/blob/main/_posts/pic/image-20221122230722087.png?raw=true)

解释一个更为细节的内容：

pkgMain依赖pkgA和pkgB为什么可以编译？这是因为go import的机制，编译器在编译pkg的时候，会根据import找到其所依赖的包的导出信息。

## 2. go的增量编译流程

![image-20221122231915575](https://github.com/Baojia-Yang/Baojia-Yang.github.io/blob/main/_posts/pic/image-20221122231915575.png?raw=true)

这是解释增量编译，专门建立的一个工程。其依赖关系和目录结构保持一直。

### 2.1 首次编译

这里只展示最关键的构建日志：

![image-20221122232623402](https://github.com/Baojia-Yang/Baojia-Yang.github.io/blob/main/_posts/pic/image-20221122232623402.png?raw=true)

![image-20221122235522847](https://github.com/Baojia-Yang/Baojia-Yang.github.io/blob/main/_posts/pic/image-20221122235522847.png?raw=true)

### 2.2 修改一个文件的编译流程

这个时候修改了pkgd：

![image-20221122233100741](https://github.com/Baojia-Yang/Baojia-Yang.github.io/blob/main/_posts/pic/image-20221122233100741.png?raw=true)

编译日志：

![image-20221122233536943](https://github.com/Baojia-Yang/Baojia-Yang.github.io/blob/main/_posts/pic/image-20221122233536943.png?raw=true)

![image-20221122233700351](https://github.com/Baojia-Yang/Baojia-Yang.github.io/blob/main/_posts/pic/image-20221122233700351.png?raw=true)

### 2.3 删除一个文件的编译流程

和2.2一样

### 2.4 增加一个包的编译流程

改动之后的流程：

![image-20221122234149539](https://github.com/Baojia-Yang/Baojia-Yang.github.io/blob/main/_posts/pic/image-20221122233700351.png?raw=true)

编译日志：

![image-20221122234425691](https://github.com/Baojia-Yang/Baojia-Yang.github.io/blob/main/_posts/pic/image-20221122234425691.png?raw=true)

![image-20221122234711668](https://github.com/Baojia-Yang/Baojia-Yang.github.io/blob/main/_posts/pic/image-20221122234711668.png?raw=true)

### 2.5 删除一个包的编译流程

和2.4一样

### 2.6 包的间接依赖关系，增量编译怎么处理的

![image-20221122235312482](https://github.com/Baojia-Yang/Baojia-Yang.github.io/blob/main/_posts/pic/image-20221122235312482.png?raw=true)

编译日志：

![image-20221122235157296](https://github.com/Baojia-Yang/Baojia-Yang.github.io/blob/main/_posts/pic/image-20221122235157296.png?raw=true)

![image-20221122235423016](https://github.com/Baojia-Yang/Baojia-Yang.github.io/blob/main/_posts/pic/image-20221122235423016.png?raw=true)

### 2.7 总结

嗯，就是这样的。。。

go的编译器的设计有哪些特点：

1， 不允许循环依赖。这让go编译的时候，pkg与pkg之间的网状结构更加的清晰，一张有向无环的图。这能带来很多好处，写编译器的逻辑更加的简单，并行编译的效率也奇高。

2， 每个包就说明了他依赖的包。这样包的编译的依赖就解释得很清楚。其中还会有一些奇怪的现在，为什么go build *.go编译一个文件的时候，有时候有用有时候没有用（大家自己去悟一下）。我也是现在才搞懂的。

。。。。

## 3. 被吓到的经历

最近在做一个项目，其中有个业界的技术难点，需要想办法优化编译速度。之前设计了一种算法，把编译的时间复杂度由O(n)降低到了O(1)。把算法写完之后，测提升效率的时候，自己列了一个公式，计算出来效率很喜人（想要多少倍的提升，我就把测试数据量增大就好，10倍，100倍都行）。然后本地go build了一下，把我吓得一激灵，为什么go的增量编译这么快，快到我想把我之前的实现方案全部删掉。

然后来回答一下为什么我观察到的go的增量编译这么快？

我测试的场景是这张图，每次增量编译，我会发现go build的时间 < 5s：

![image-20221122233100741](https://github.com/Baojia-Yang/Baojia-Yang.github.io/blob/main/_posts/pic/image-20221122233100741.png?raw=true)

但是真实的业务场景是这样的（绝大多数场景下）：

![image-20221122235312482](https://github.com/Baojia-Yang/Baojia-Yang.github.io/blob/main/_posts/pic/image-20221122235312482.png?raw=true)



留下点经验：彻底了解清楚一个技术细节，你才不会被吓到吃不下饭

