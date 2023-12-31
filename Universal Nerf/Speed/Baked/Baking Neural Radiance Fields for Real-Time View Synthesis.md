# Baking Neural Radiance Fields for Real-Time View Synthesis

### Abstract

Nerf中训练后的神经网络需要每条光线查询数百次多层感知机(MLP)，为解决该问题，提出了一种重新计算和存储训练后NeRF的方法，为了实现这一目标，主要有两种实现方式：1). NeRF架构的重新表述 2). 一种带有学习特征的稀疏体素表示方法。能够在保持精度的情况下，大幅提升速度。

### Introduction

在SNeRG的每个 active(不知道怎么翻译) 体素包含了一组特征，包括不透明度、固有颜色和对视图相关效果进行编码的学习特征向量。

具体的渲染过程为：首先对于每条光线**累加固有颜色和其他特征**，然后通过一个轻量化的MLP来产生一个跟视角(View)有关的残差，添加到累加的**固有颜色**上。

引入了两个重要的改进，确保NeRF能够被有效地被转化为 离散体素表示，1). 我们设计了一个视角依赖的MLP仅每个像素运行一次，而非如NeRF每次通过三维样本进行。2). 我们正则化了NeRF的预测的不透明场以鼓励稀疏性，这能有利于存储和渲染时间。

### Related work

对于基于八叉树结构的方法在面对细节几何和外貌 以及 渲染期间 不需要 可变细节（指的是不需要中间节点），查询非叶节点的树遍历会产生大量的内存和时间开销。

基于Hash map的方法在渲染期间遍历的时候，由于独立散列，很可能导致**内存提取不连贯**（然后咧？）。

### Method Overview

![image-20231025220512265](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20231025220512265.png)

在这篇文章中，作者把颜色分成固有颜色和镜面颜色，固有颜色与观察视角无关。网络输入3d坐标位置，输出体素密度，固有颜色，和镜面颜色特征向量。镜面颜色特征向量在通过一个小的网络，结合视角，解码成镜面颜色，加到最终的颜色上。与PlenOctrees一样，主干mlp网络都与视角解耦。PlenOctrees通过球协函数恢复视角颜色，SNeRG通过后接一个小MLP网络恢复镜面颜色，在叠加到固有颜色上。

### Modifying NeRF for Real-time Rendering

为了使得NeRF能够实时渲染，重新定义了NeRF, 从以下三个方面：

1). 因为NeRF本身的网络为了保持视角一致性，是先输出体素密度$\sigma$ ，而颜色则在之后的神经网络的输入$view$再计算，这里单独给了一个小的神经网络。

2). 引入了一个**bottlenect**，但是这个模块很小，可以被存储为8bit.

3). 引入了一部分新的损失函数，希望能够将不透明度保存在场景中的表面周围。

**大概的思想是，在训练的时候，相当于把体素内的信息用神经网络来表示，这样需要存储的空间少了几个数量级，但是每次查询都不是简单的Query，而是MLP(Query)，但是测试的时候就没必要了，可以把这样的值输入到某个确定的神经网络之中。**

**Deferred NeRF Architecture**
$$
\sigma(t), c_d(t), v_s(t) = MLP_{\theta}(r(t))
$$
其中$v_s$是特征值的意思, $c_d$是固有颜色。
$$
\hat C_d(r) = \sum_k T(t_k)\alpha(\sigma(t_k)\delta_k)c_d(t_k) \\
V_s(r) = \sum_k T(t_k)\alpha(\sigma(t_k)\delta_k)v_s(t_k) \\
\hat C(r) = C_d(r) + MLP_\Phi(V_s(r),d)
$$
$d$是View-direction。渲染一个像素，累计这条光线的所有特征值和颜色，之后传输到另外一个小的神经网络中和原先预测的固有颜色加和得到这个像素的渲染颜色。

**Opacity Regularization**
$$
L_s = \lambda_s \sum_{i,k}log(1+\frac{\sigma(r_i(t_k))^2}{c})
$$
其中$\lambda_s$和$c$分别代表着超参数对损失进行控制，$r_i$代表第$i$条线，然后$t_k$代表这第$i$条光线的第$k$个采样点。

**看实验结果来说，这个正则化还蛮有用的**

### Sparse Neural Radiance Grids

这篇的核心思想是用计算换取存储，从而大大减少渲染帧所需的时间。换句话说，希望用预先计算的数据结构中的快速查找来取代NeRF中的MLP评估。我们通过预计算和存储（即烘焙）、漫反射颜色 $c_d$、体积密度$σ$和 4 维特征向量与体素网格数据结构中的对比来实现这一点。

注意这个方法仅存储即可见又被占用的体素。

**SNeRG Data Structure**

用两个小的稠密的列表来存储三维网格信息。

第一个列表是一个3D纹理图集（一组2D纹理图像），相当于把所有又内容的体素拿出来转换成了3D纹理图集，这个方法的主要优点是保持了空间相邻（和Hash表这个数据集相比）

第二个列表是一个索引列表，用来指向这个像素对应是否为空以及对应这个像素的3D 纹理图集的索引内容。

![image-20231027170311003](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20231027170311003.png)

通过文章的模型图，我个人理解主要的加速在于渲染单个像素的时候，如果这个物体相较于整个空间来说是比较稀疏的（也就是空白像素多），那么应该会效果比较好。

另外存储好了该体素的特征向量和固有颜色，这样就可以少跑一个大的神经网络，可以直接跑一个小的神经网络就可以了。

光线行进过程中如果不透明度饱和后，就停止前进。

注意实际实验处理中采用了剔除空白空间 和 剔除可见性较低的 块来进行稀疏。

**需要注意但是在通过这样方法存储的延迟渲染的图像质量是低于直接延迟渲染的（为什么？个人认为是从体素到3D纹理图集）需要微调来达成近似的结果**