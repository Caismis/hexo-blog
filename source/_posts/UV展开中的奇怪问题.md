---
title: UV展开中的奇怪问题
date: 2024-01-18 14:34:35
cover: https://raw.githubusercontent.com/Caismis/blog_img/main/images/2024-01-17231143.png
tags: 
- Blender
- 硬表面
- UV
- 杂记
categories: Blender
---
这次试试硬表面的建模练习，本来艰辛的建模完毕准备UV展开，却在这个破地方怎么也修不好UV。
<!--more-->

![左边UV处有褶皱](https://raw.githubusercontent.com/Caismis/blog_img/main/images/2024-01-17223945.png)

也不知道是我操作不行，反正调整了很久，还是有这个褶皱。不过我也试了试单独展开这个面，结果这就正常了。

![褶皱消失了](https://raw.githubusercontent.com/Caismis/blog_img/main/images/2024-01-17223957.png)

但是也不能就展开这个面不是，于是又是一番操作，还是无法消除掉这个褶皱，无奈只能去找找UV插件，试试能不能解决默认UV这个问题。很不幸，即使使用了[Zen UV](https://blendermarket.com/products/zen-uv)展开后依旧如此。无奈只能放弃在建模层面上的字符雕刻，当然我也很好奇教程作者为何不采用法向贴图而采用实际建模的方式，不过既然我无法解决，那也就只能依赖法向贴图了，好在去除掉字符的建模后UV展开一切正常。

不过倒也发现了Zen UV的好，这个插件用来展UV确实方便，后面展完了在说说这个插件。