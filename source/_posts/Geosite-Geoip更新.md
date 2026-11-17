---
title: Geosite & Geoip更新
cover: /images/lwrc45_2.webp
date: 2024-01-15 12:27:56
tags: 
- 网络
- 经验
categories: 电脑
---
对于需要使用分流规则来进行个人定制化的网络广告拦截，还是很有必要定时更新Geosite和Geoip的数据的。
```bash
#!/bin/sh
wget https://github.com/Loyalsoldier/v2ray-rules-dat/releases/latest/download/geoip.dat -O /path/you/like/geoip.dat
wget https://github.com/Loyalsoldier/v2ray-rules-dat/releases/latest/download/geosite.dat -O /path/you/like/geosite.dat
```
还可以加入到`crontab`中来自动更新（至少我就是这么做的）