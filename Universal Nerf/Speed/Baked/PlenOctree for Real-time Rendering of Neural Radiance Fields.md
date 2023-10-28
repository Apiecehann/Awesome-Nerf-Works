# PlenOctree for Real-time Rendering of Neural Radiance Fields

本文需要关于球谐函数的一些知识：如果没有了解可以跳转

[球谐函数介绍（Spherical Harmonics） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/351289217)

[【论文复现】Spherical Harmonic Lighting:The Gritty Details - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/359856625)

[【论文复现】An Efficient Representation for Irradiance Environment Maps - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/363600898)

以及非常近似的球面高斯：

[球面高斯介绍（Spherical Gaussian） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/514017351)

### Abstract

在实际应用之中，NeRF的时间消耗是一件很严重的限制，由于其采样需求和神经网络的耗费，一般来说大概需要30秒来渲染一张800 $\times$ 800的图片，PlenOctree通过将NeRF蒸馏到一个层次化的3D体表示，并使得其能够在浏览器上做渲染。

### Introduction

作者认为原生的NeRF很慢因为它需要大量的采样，而且每次采样都需要跑一遍神经网络的推断，因为某个位置的颜色取决于观察方向和具体的空间位置，不可能缓存某个位置的全部方向颜色。

作者通过将NeRF转换为表格型的基于视角的体素方式解决了这个问题，通过一个稀疏的基于体素的八叉树（每个叶子节点存储该点模拟辐射场的外观和密度值），为了解决非朗伯材料（就是颜色跟观察方向有关的材料），建议用球谐来表示位置的RGB值，可以在任意方向查询方程以恢复视图的颜色。

训练了一个网络，该网络输出为**SH函数生成系数**，以便预测值直接存储在PlenOctree的叶子中，同样在NeRF训练过程中引入**稀疏性（有点迷糊，为啥引入稀疏性能够提高记忆效率）**，模型可以**早停**节省训练时间。

### Preliminaries

作者主要是认为NeRF均匀采样的话，每个光线都得采样192个样本点，**但是很多采样点就是空的，太低效了**。

![image-20231028151617114](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20231028151617114.png)

### Method

**NeRF-SH：NeRF with Spherical Harmonics**
$$
f(x) = (k, \sigma) \\ k = (k_l^m)_{l:0 \leq l \leq l_{max}}^{m: -l \leq m \leq l}
$$
其中$f$代表神经网络，然后$k_l^m$ 是对应RGB的一组的3个系数。
$$
c(d;k) = S(\sum_{l=0}^{l_{max}} \sum_{m=-l}^l k_l^m Y_l^m(d))\\
where \ S: x \rightarrow (1 + exp(-x))^{-1}
$$
$S$是Sigmoid函数，用来做归一化的，通过上述公式可以不将视角作为神经网络的输入。

$Y$是球谐波公式：$ Y^m_l：S^2 \rightarrow R$

作者提到对于SH公式的分量值也同样可以通过采样然后蒙特卡洛估计计算，但是这样太耗时了，也会产生2dB左右的损失。除了球谐波外，也尝试了球高斯，但是球谐波的方法更好。

**稀疏先验**

如果没有正则化，模型会在未观察到的区域自由生成任意几何图形，这额外的几何体会占用大量的体素空间。

**它希望空间空白和纯色中，鼓励NeRF选择空白空间**（不理解，只能说空间概率上大概率是这样的）
$$
L_{sparsity} = \frac{1}{K}\sum_{k=1}^K|1 - exp(-\lambda \sigma_k)
$$
而最终的训练损失函数是：
$$
L_{train}= \beta_{sparsity}L_{\beta_{sparsity}} + L_{RGB}
$$
这里面的$\beta_{sparsity}$是超参数。

### PlenOctree:Octree-based Radiance Fields

PlenOctree 存储密度和 SH 系数，以模拟每个叶子的视图相关外观。

**Rendering**  首先确定八叉树中的 光线和体素的交集，这样就会在体素之间生成一系列段，长度为$\left\{\delta_i\right\}_{i=1}^N$, 之后原本NeRF的方法来渲染这些光线段，这样的采样更加有效率，同时早停是如果透射率小于$\gamma < 0.01$。

**Conversion from NeRF-SH** 大概方法是在网格上通过神经网络进行测评，仅保留密度值，然后通过阈值进行过滤（主要是去除不可见的体素），最后对每个剩余的体素内随机点采样后进行平均，以获得存储在叶节点中的SH系数。

### Discussion

该方法在无界和仅前向的照片需要更进一步的工作，因为无界场景的分布并不一致，而前向场景不支持6自由度观看，可能MPI更合适进行实现。

### 球谐函数

最有名的球面基函数是球谐函数，球谐函数有很多好的性质，譬如**正交性**, **旋转不变性**。

<img src="C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20231028133604052.png" alt="image-20231028133604052" style="zoom:67%;" />

**球谐函数公式：**

<img src="C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20231028133645662.png" alt="image-20231028133645662" style="zoom: 67%;" />

如果从二维的角度来看的话：

<img src="C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20231028133926026.png" alt="image-20231028133926026" style="zoom: 67%;" />

如果一个函数可以被表示为：
$$
r = 0.5 + 0.1\cos\theta + 0.07\sin \theta + 0.05 \cos\theta\sin\theta + 0.3(2cos^2\theta -1)
$$
这个函数就被压缩成为了：0.5, 0.1, 0.07, 0.05, 0.3。 这几个值就是SH系数。

一般来说二阶有四个系数，三阶是九个系数，这样的话，RGB共27个系数，加一个偏差项，总共需要存储28个系数即可。

参考**An Efficient Representation for Irradiance Environment Maps**中的内容：

原初的球谐函数在展开之后仍然存在一个问题：即针对**每个方向的法线**都需要预计算出一组球谐系数。（因为位置和法线完全相关）
$$
L(p,w_0) = \sum_{j=0}\sum_{i=0}L_it_j\int_{\Omega} Y_i(w)Y_j(w)dw
$$
 其中$Y_i$, $Y_j$都是基函数，而$L_i$，$t_j$ 分别为光照和光照在该点的分量。

其中由于 $L_i$本身为全局光照，所以光照的强度是一致的，而$t_j$则是强相关的与该位置有关，因为是该点的法线方向。

为了解决这个问题，引入球坐标系，球谐函数经过化简后为：
$$
Y_l^m(R^{\alpha, \beta, \gamma }(u)) = \sum_{m^{'}=-l}^l D_{m^{'},m}^l(R^{\alpha, \beta, \gamma}Y_l^{m^{'}}(u))
$$

$$
D_{m^{'},m}^l(R^{\alpha, \beta, \gamma}) = e^{-im^{'}\alpha}d_{m^{'},m}^l(\beta)e^{-im^{'}\gamma}
$$

其中$d_{m^{'},m}^l$ 为维格纳$D$矩阵，这个式子提供了一个旋转，也就是只需要将这个原本计算好的$R(u)$                                           乘上一个旋转矩阵就可以得到一个$ R^{\alpha, \beta, \gamma}(u)$
$$
f(R^{\alpha, \beta, \gamma}(u)) = \sum_{l}^{\infty} \sum_{m=-l}^l(R^{\alpha, \beta, \gamma}(u))\\
=\sum_{l}^{\infty} \sum_{m=-l}^l g_l^{m ^{'}}Y^{m ^{'}}(u)\\
g_l^{m ^{'}} =  \sum_{m=-l}^l c_l^m D_{m^{'},m}^l(R^{\alpha, \beta, \gamma}Y_l^{m^{'}})
$$
这样就可以通过计算$c_l^m D_{m^{'},m}^l$ 仅对原来的球谐函数进行处理得到旋转后的结果。
$$
L(w_i) = L(R^{\alpha, \beta, \gamma}(w_i^{'})) = \sum_{l=0}^{\infty} \sum_{m=-l}^l L^m_l Y^m_l (R^{\alpha, \beta, \gamma}(w_i^{'}))
$$
而其中的球谐基函数又可以进一步进行拆分：
$$
Y^m_l (R^{\alpha, \beta, \gamma}(w_i^{'})) = \sum_{m=-l}^l D_{m^{'},m}^l(R^{\alpha, \beta, \gamma}Y_l^{m^{'}}(w_i^{'}))
$$
但是我们只关心 **光线**和**法线的夹角** $\alpha$， 最后经过一堆化简得到：
$$
L(n) = \sum_{l=0}^{\infty} \sum_{m=-l}^l \sqrt{\frac{4\pi}{2l+1}}L_l^mt_lY_l^m(n)
$$
注意$t$的下标是$l$的索引，对于相同的$l$不管$m$怎么变都相等。对于$L$仍需要预计算出来，最后实时计算球谐函数系数即可~。

**球谐函数有一些问题，第一只能开展到三项，再高会出现一些问题，第二个是因为在仅变换 $\alpha$ 过程中并非无偏（所以结果也不是无偏的）**

**但是球谐函数处理低频信息更好（意思是更适合处理漫反射或者是粗糙度比较大的光泽反射），因为对于纯镜面反射而言，我们需要非常高阶的球谐函数才能进行逼近，那计算量实在是太大了。**
