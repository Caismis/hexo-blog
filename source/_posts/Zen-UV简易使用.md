---
title: Zen UV简易使用
date: 2024-01-20 14:24:07
cover: https://raw.githubusercontent.com/Caismis/blog_img/main/images/zenuv_post.png
tags: 
- Blender
- UV
- 经验
categories: Blender
---
继续上次出现奇怪UV问题之后，尝试使用了Zen UV，发现这个插件真是好用。

<!-- more -->

- 模型拆解阶段

从拆解UV的最基础阶段，Zen UV就开始好用了。在Edit Mode中单击N呼出右侧面板，选择Zen UV栏即可开始使用，虽然说可以通过Alt+U调出饼状菜单，但是个人不太喜欢用这个。根据我在这次练习中拆解的习惯，通常是点击Unmark All来清除或许是Hardops带来的接缝，然后点击Mark by Angle来自动标记所有锐边，接下来再根据实际情况标记或者取消标记接缝。

<img src="https://raw.githubusercontent.com/Caismis/blog_img/main/images/zenuv1.png" alt="拆解子菜单" style="zoom:50%;" />

值得一提的是，再使用这个插件之前，我是借助创建材料里自带的UV Grid图像来检查UV是否出现扭曲的情况，但是在Zen UV中自带了一个检查图像，点一下就可以覆盖，效果和封面图一样。

<img src="https://raw.githubusercontent.com/Caismis/blog_img/main/images/zenuv2.png" alt="UV检查器" style="zoom:50%;" />

在遇到圆柱类的物体没展开为矩形时，可以很方便地点击Quadrify Island来快速转变为矩形，实测效果非常好

<img src="https://raw.githubusercontent.com/Caismis/blog_img/main/images/zenuv_trans.png" alt='"Quadrify"' style="zoom:50%;" />

<img src="https://raw.githubusercontent.com/Caismis/blog_img/main/images/zenuv_quad.png" alt="实际效果" style="zoom: 33%;" />

- UV打包

检查无误后就可以UV打包了，Zen UV也可以直接调用UVPackmaster的引擎进行打包，UV利用率不错，在我这个还没对模型UV进行优化的情况下空间利用率达到了40%左右，设置的Margin pixel直接参考官方的。

| 贴图大小 | Margin pixel |
| :------: | :----------: |
|   256    |      2       |
|   512    |      4       |
|   1024   |      8       |
|   2048   |      16      |
| **4096** |    **32**    |
|   8192   |      64      |

但是UV利用率还是差强人意，如果要进一步提升，就得试试个人认为特别厉害的堆叠功能，这个功能会使一致的UV区块直接重叠，使得空间利用率显著提升。

<img src="https://raw.githubusercontent.com/Caismis/blog_img/main/images/zenuv_stack.png" alt="Stack子菜单" style="zoom:67%;" />

我个人的使用方法是：**Pack**->**Stack**->将Pack里的Lock...改为exact->**Pack**。这样的操作下来打包在UV贴图里的区块利用率将会达到最高，下图中的覆盖率达到了60%多，增加了20%左右。期待后续在SP中的表现。

<img src="https://raw.githubusercontent.com/Caismis/blog_img/main/images/zenuv3.png" alt="Stack效率非常明显" style="zoom:40%;" />

- 总结

不得不说原版Blender和打了插件后的Blender真是天差地别，虽然说插件能做的你自己也能做，但是插件点一下顶你一分钟，差距就出了来，至今已经在插件上花了不少了，难道这就是开源的代价（？