# 神经渲染笔记

## 定义

渲染通常是指将 3D 场景，转变为 2D 图像的过程。









------

## 可微分渲染(Differentiable Rendering)

简单的定义：

> 可微分渲染是指可以对各类三维表示（比如Mesh、Voxel）进行合理渲染，并能回传渲染颜色关于渲染参数的梯度的一类梯度方法，简单来说就是**可微分渲染就是计算渲染过程的导数**。

普通的渲染过程是将 3D 场景扔进渲染器得到 2D 图像，其逆向过程，称为逆向渲染(Inverse Rendering)，就是从 2D 图像得到获得生成这张图像所需要的 3D 信息，它们可以是 3D 几何、光照、材质、视角等其中的一种或者多种。

如下图展示了一个渲染和逆向渲染过程，图片来自 Mitsuba2 论文：

![来自 Mitsuba2](https://pica.zhimg.com/80/v2-5b181bba9ccbf5d0b3e6048f10736903_720w.jpg?source=1940ef5c)

整个过程包括如下步骤：

1. 定义一个目标图片 A，目标是找到能生成这张图像的 3D 场景 B；
2. 首先是会先做一个初始的 3D 场景 B1，这个场景不要求很准确，错的或者大致是相似的都行；
3. 将 B1 送到渲染器中，渲染得到图片 A1
4. 和神经网络类似，比较 A 和 A1 的区别，然后计算 Loss
5. 将 Loss 反向传播到场景 B1 的不同参数上，比如材质、光照、相机参数等，修改这些参数的值，得到新的场景 B2
6. 和训练神经网络一样，重复步骤 3-5 这个过程，直到满足某个终止条件，比如迭代次数

其中第 5 步就是可微分渲染的关键部分，这一步需要计算梯度，从传统角度来看:

1. 渲染的计算过程是 Monto Carlo Ray Tracing，光线在整个场景中跳来跳去，这个计算过程中很多数值是离散的，并不太好计算出合适的梯度，目前一个解决方法是采用 Edge Sampling；
2. 渲染中有很多效果，包括 BSDF 函数，这个是一个比较新的领域，一个解决方法是 Mitsuba2



另外，实际上渲染过程包含了不同的模型，比如 pipeline rendering(实时渲染）和 physics-based rendering (离线渲染)，因此其求导过程也差别很多，前者在视觉领域已经有一些实际的应用，主要是用光栅化的方法，典型的代表是Soft rasterizer(Soft Rasterizer: A Differentiable Renderer for Image-based 3D Reasoning)。后者主要是用光线追踪的方法，但由于是基于Monte Carlo，模型本身相对复杂一些，针对其求导的研究一直没有太大的进展，其难点包括：

- 与visibility相关的导数（比如Geometry）难以计算 
- 如何快速的计算导数。

对于第一个问题，Tzumao的论文 （Differentiable Monte Carlo Ray Tracing through Edge Sampling）是第一篇针对visibility的论文，并通过 edge sampling 来解决，但是它的局限性在于，为了正确的计算visility相关的导数，需要在渲染过程中实时的进行 silhouette detection，使得计算会很慢。之后的两年陆续有若干篇physics-based differentiable rendering的论文。包括

- Wenzel 的Reparameterization，提出了一种biased but fast的近似解法，不再需要edge sampling （Reparameterizing discontinuous integrands for differentiable rendering）
- DTRT, 将可微渲染拓展到体渲染 ( A Differential Theory of Radiative Transfer)
- PSDR, 将可微渲染拓展到Path Space, 并且不需要做 silhouette detection，因此对于performance也有很大的提高 (Path-Space Differentiable Rendering)
- MIT 组的论文 （Unbiased Warped-Area Sampling for Differentiable Rendering），提出了一种新的算法，不需要做edge sampling, 理论上也是unbiased。

对于第二个问题，目前的情况是：

1. Mitsuba2是目前最快速（但与 Tzumao 的 redner 差别不大） 的 physics-based differentiable renderer ，但是对于GPU memory需求极大（Mitsuba 2: A Retargetable Forward and Inverse Renderer）
2. 为了解决GPU Memory的问题，Wenzel 组提出了Radiative Backpropagation (Radiative Backpropagation: An Adjoint Method for Lightning-Fast Differentiable Rendering)










------

### 基于网格的可微渲染

网格模型的传统渲染流程大致分为如下两步：
- 为每个图片像素指定三角面片（光栅化）
- 根据像素对应的三角面片与光照、纹理等渲染参数，计算像素的颜色；



许多应用中，网格的几何也是优化对象，由于**光栅化本身是关于网格几何和像素位置的不连续函数，这导致了传统的网格渲染是不可微的**。



目前基于网格的可微渲染解决该问题的方法分两种：
- 保持渲染过程不变，近似梯度，典型的方法包括 OpenDR
- 修改渲染管道(pipeline)，绕过不可微的光栅化，比如 SoftRas



### 基于点云的可微渲染

传统点云渲染分为以下几步：
- 将点云转到相机系
- 计算每个点的颜色与影响范围；
- 对每个像素，根据点云的深度计算像素颜色



在许多实际应用中，会将点云转为高斯分布，扩大单个点的影响范围，此时点云的渲染和可微化和 SoftRas 的思想类似

基于网格的可微渲染和基于点云的可微渲染有相似性：**两种几何表示都有广泛的应用，且渲染过程中都需要为像素指定着色图元**；

应用点云的可微渲染时，如同 SoftRas一文，**渲染精度和点云影响区域大小是需要平衡的变量**；

点云可微渲染的一个额外应用是深度图的新视角渲染，该渲染方法允许单目深度估计的自监督；



### 基于体素的可微渲染

三维物体的体素表示是一系列以体素形式存储的三维表示，比如：

- Occupancy field，可以是 0/1 值，也可以是连续的浮点数；
- SDF，表示空间中任一点距离三维物体表面的有向距离

体素表示的渲染过程如下：

1. 确定每条光线经过哪些体素；
2. 选取或者融合每条光线上体素的信息，得到渲染结果

通常来说，**体素表示的可微渲染设置只要求对第二部分进行可微化，此时，光线颜色信息来自一些固定位置的体素，不存在网格渲染那样的图元位移问题**。



使用体素作为物体的三维表示的最大问题是在于**存储空间，存储空间的增加速度是分辨率的三次方**，在要求高精度渲染的应用中是不合适的。目前的解决方法有这几个方向：

1. 用八叉树等数据结构减小体素的空间需求；
2. 直接使用隐式表示来替代体素表示；



### 基于隐式表示的可微渲染

隐式表示和体素表示一样，都是一类存储方式特定的三维表示的统称，比较流行的隐式表示包括

- occupancy field
- SDF
- NeRF

隐式表示的可微渲染和体素表示的可微渲染有非常多相似之处，**其可微化大多数限于对固定点信息的优化，两者在渲染流程上的不同在于采样策略**。



隐式表示的优劣势：

- 隐式表示本身是三维信息的利器，它可以在较小的体积内达成高分辨率甚至 photo-realistic 的渲染，相对网格和体素有明显的优势；
- 基于隐式表示的可微渲染某种程度上已经成为了隐式表示的一部分，没有隐式表示的可微渲染，隐式表示的构造程度将会上升；
- 最大的劣势在于**渲染速度较慢**，其根源在于**隐式表示的渲染需要对表示网格进行密集求值**。









------

## 数据

### OBJ 网格模型

#### 实体建模 vs 表明建模

在说OBJ模型之前，先从大的写起，说说关于3D建模的分类。根据情况可以略过不看。

> 所有的3D cad软件从机制上来说其实可以分三类：
>
> - 实体建模（Solid Modeling）
> - 表面建模（Surface Modeling）
> - 实体建模+表面建模。
> 
> 即实际上建模方式只有两种：实体建模和表面建模。工程师用的软件偏向于实体建模，而艺术生用的软件偏向于表面建模。

实体建模：

> 实体建模的工具方式基本上有三种：扫掠，立体布尔运算和表面围成。
>
> 1. **扫掠**通常指的是将一个平面沿着一条指定曲线做扫掠操作，然后这个平面扫过的空间就成为了目标固体（所谓的平面拉伸，旋转，方样等都属于这个种类）。
> 2. 布尔运算是对两个固体来求和，求差或者求交。这里指的固体运算其实是指空间的运算。通常计算机都用一种被称为边界限定法来描述一个空间，布尔运算的实质是对这个被限定住的空间做逻辑运算。求和，指的是不同立体空间相加后构成包含了两个空间的新空间。类似的，求差指的是用一段空间去剪切另外一段空间等。
> 3. **表面围成**就比较直接了，这种方法直接通过几个特殊的表面来生成固体，这些表面必须能够围住一个封闭的空间。

表面建模：

> 表面建模可以用一句话概括：**添加或者修改点，让这些点能够约束一个表面**。
> 表面建模只关注表面的外形，不关注这些表面是否能够围成一个封闭的空间。表面建模的整个过程都像做一个泥娃娃，你可以给它添加泥巴，也可以拿下一些泥巴，还可以捏这些泥巴。
>
> 而maya这些软件就是给你提供各种工具来玩这个泥娃娃。同样布尔操作也可以用在表面建模中，但是形成的意义有所不同。实体建模中的布尔运算是真正的立体空间求和，求差，求交；而表面模型的布尔操作就像是做不同的表面缝合操作。（通常3D打印用的stl格式基本上就是表面模型描述文件）



OBJ 模型是属于表明建模模型的，然后它本身是ASCII格式，是可读的。

**OBJ 模型的优点**：

1. 文件格式被广泛支持；
2. 3D打印机只识别STL或OBJ格式。



#### 网格模型

表面建模模型中，两个点组成一条边，多条边组成一个面，多个面组成一个体，一个或多个体组成一个模型。可以说一个模型是由若干个面组成的。

在计算机图形学中，考虑到图形求交等计算的问题，这个面通常选择三角形，即模型的表面由若干三角面构成。一个模型文件需要记录的信息就是这些**三角面的信息**，进一步讲，一个三角面又需要有顶点、法线、连接顺序等这些信息。

##### 法线

顶点位置信息——x,y,z坐标值——描述了三角面的位置，但光有位置信息还不够，我们还需要知道一个三角面的正面与背面，为什么要这样，直观的理解就是，由于模型体只有表面，内部是空腔（如果模型是封闭的话），内部的面是不可见的，然后背面朝着我们（摄像机）的面也是不可见的。那怎么标注这个正面与背面呢？就是靠法线了。

（补充：一个简单的例子就是，比如我们有一个立方体模型，这个立方体表示一个魔方之类的物品，这时它可见的是外表面，但如果它表示一间房间，人在房间里面观察，那个此时它可见的应该是内表面。）

所谓**面的法线就是垂直于此面的线，用一个向量表示，向量指向的一侧为正面，另一侧为反面。进一步地说，法线用处还是在光照模型中确定一个面的颜色**。

下图展示了一个无顶的正方体模型的渲染情况，左图是一个正常的正方体，外表面为正面，右图中的模型与左图相同，只是有一面的法线翻转到内侧，可以看到两个模型虽然形状一样，但渲染效果是不同的。

（P.S. 那能不能让我们看到同时看到正反面呢，答案是可以的，3dsmax和three.js里都可以设置一个”双面“参数，就是适用于这种模型不封闭有缺口，但正反面都想看的这种情况的）

![img](https://pic1.zhimg.com/80/v2-884a6d5cb7839137e1508ad1eb2328a4_720w.jpg)



#### OBJ 文件格式

OBJ文件是一种被广泛使用的3D模型文件格式（obj为后缀名）。由Alias|Wavefront公司为3D建模和动画软件"Advanced Visualizer"开发的一种标准，适合用于3D软件模型之间的互导，也可以通过3dsmax、Maya等建模软件读写。

![img](https://pic3.zhimg.com/80/v2-da992e536b4af78023bfc6aedc03bcae_720w.jpg)

![img](https://pic1.zhimg.com/80/v2-f30e306cf70c3ce2fb643a9d9f65f63c_720w.jpg)

图上这个立方体的.obj文件内容大概就长下面的这个样子，相关解释已包含在注释中。

```python
# "#"号开头是注释行
# v（vertex）数据段: 模型顶点列表
# 顶点位置信息，是xyz三维坐标
# v开头的每一行描述一个顶点，行数等于顶点数。8个顶点所以有8行
v  1.00  -1.00  -1.00
v  1.00  1.00  1.00
......
# vt（vertex texture）数据段：模型顶点的纹理坐标列表
# 顶点的纹理坐标信息，是xy二维坐标
# vt开头的每一行描述一个纹理坐标，行数大于等于顶点数，因为一个模型顶点在贴图的UV坐标系中很可能对应多个顶点/纹理坐标。且坐标值范围是在0~1之间，这个模型中有14行。
# 关于纹理坐标看图，本文不多解释纹理坐标，可参考文献[2]或自行百度
vt  0.74  0.75
vt  0.29  0.55
......
# vn（vertex normal）数据段：顶点法线列表
# 三维法向量，xyz
# vn开头的每一行描述一个法向量，行数大于等于顶点数。 前面介绍了，法线是与面相关的概念，但是现在的面是靠顶点来描述，拿示意图中的点"1"为例，它与参与构成了三个面，所以"顶点1"对应有3条法线
# 可能你已经发现了，在这个立方体模型中，共面顶点的法向量的方向是相同的，也就是说这里面的数据会重复，所以在建模软件导出obj文件时有个优化的选项，勾选后的导出的法线列表数据中就不会有重复项，这里的例子优化后有6条法线*
vn  -1.00 0.00 0.00 
vn  1.00 0.00 0.00
vn  0.00 1.00 0.00
......
# f（face）：模型的三角面列表
# f开头的每一行描述一个面 ，关键的来了，三个点组成一个面，怎样拿到这三个点呢？通过从1开始的索引，去前面的v、vt、vn列表中去取。
# 总结一下就是：每一行定义1个面，1个面包含3个点，1个点具有“顶点/纹理坐标/法线”3个索引值，索引的是前面3个列表的信息。
f  1/1/1  2/2/1  3/3/1      # 顶点1、顶点2、顶点3 组成的面
f  2/2/1  3/3/1  4/4/1      # 顶点2、顶点3、顶点4 组成的面
f  1/1/1  5/10/1  8/14/6  # 顶点1、顶点5、顶点8 组成的面
......
```

以上便是obj 文件的核心部分，此外还有一些数据段，记录如下。

- o 对象名
- g 组名
- s 平滑组
- usemtl 材质名
- mtllib 材质库.mtl



#### 其他3D模型文件格式

> 通常，三维软件的输出格式分为三类。
> 第一类是商业内核级别的数据格式，如ACIS商业内核的sat数据格式(*.sat 扩展名为SAT的文件)；ParaSolid内核的X_T数据格式(*.X_T)。
> 第二类是公共级别的数据格式，如Step的*.stp；IGES的*.igs(爱鸡屎，强烈建议不要使用!)
> 第三类就是专用级的数据格式，就是每个软件自己特有的数据文件格式，如SW的SLDPRT，PROE的PRT等。

具体可以查看[Data Formats: 3D, Audio, Image，对模型文件格式总结，很全。







---

## 参考文章

1. [OBJ网格模型文件（上） - 学习随笔](https://zhuanlan.zhihu.com/p/38052123)
2. 《从传统渲染到可微渲染**:** 基本原理、方法和应用》
3. [[报告摘录] 可微渲染与神经渲染 简述](https://zhuanlan.zhihu.com/p/363358697)
4. [什么是可微分渲染（differentiable rendering）？](https://www.zhihu.com/question/364770565)







