---
title: 记一次本地部署Copilot to GPT4
date: 2024-01-25 13:17:38
cover: https://raw.githubusercontent.com/Caismis/blog_img/main/images/2024-01-25212520.png
tags:
- 网络
- 杂记
categories: 其他
---
免费的ChatGPT 3.5用着实在是差强人意，但是奈何ChatGPT Plus价格高达20刀，只好折中一下，选择折腾Copilot。

<!-- more -->

## 原料准备

- 订阅了Copilot的Github账号；天下没有免费的午餐，还是得**10$**一个月
- [转换服务](https://github.com/aaamoon/copilot-gpt4-service)
- [本地后端](https://github.com/ChatGPTNextWeb/ChatGPT-Next-Web)

- 个人将转换服务和本地后端全部部署在了自用的NAS服务器DX4600上，免得电脑一关机服务就掉线

## 转换服务部署

直接拉取docker即可

```bash
docker pull aaamoon/copilot-gpt4-service
```

然后本地进行部署

```bash
docker run -d \
  --name copilot-gpt4-service \
  e HOST=0.0.0.0 \
  --restart always \
  -p 8080:8080 \
  aaamoon/copilot-gpt4-service:latest
```

然后运行即可，这里的HOST设置为`0.0.0.0`来监听所有。

## 本地服务端部署

同样先拉取docker

```bash
docker pull yidadaa/chatgpt-next-web
```

然后部署

```bash
docker run -d -p 3000:3000 \
   --net=host \
   -e PROXY_URL=http://#NAS的地址#:8080 \
   yidadaa/chatgpt-next-web
```

部署完成后，通过转换服务中提及的API Token[获取方法](https://raw.githubusercontent.com/aaamoon/copilot-gpt4-service/master/shells/get_copilot_token.py)获取到自己的令牌，然后打开NextChat的页面

![2024-01-25214518](https://raw.githubusercontent.com/Caismis/blog_img/main/images/2024-01-25214518.png)

打开设置找到上方的选项，填入自己的Token就可以开始愉快地使用了，也别忘了将下方地模型选择为“gpt-4”，毕竟微软不按Token收费，那就必然全上GPT-4。

## 最后

不得不说这么一操作确实折腾，但是也很值得，经过在VS Code中Copilot地使用，我认为光原版功能就已经能值回票价，转换成GPT-4更是好上加好。