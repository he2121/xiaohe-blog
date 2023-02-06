---
title: "如何在微信公众号上搭建一个属于自己的 chatgpt 服务"
date: 2023-02-06T00:08:32+08:00
tags: ["chatgpt","golang","微信公众号"]

---

> chatgpt 刚出时就体验了一下，并且写了一个飞书机器人再飞书上使用至今，感觉还挺方便。最近想起自己还有一个 n 年没用过的阿里云服务器和一个已冻结的微信公众号，便想着废物利用，在微信上也搞一个 chatgpt 的机器人。

# 通过本文你能获得什么

- 属于自己的 chatgpt 微信机器人

- 熟悉微信公众号的 golang 服务开发过程

# 前置条件

1. [服务器一台](https://www.aliyun.com/product/ecs?spm=5176.19720258.J_3207526240.47.542176f43hVLBb)
2. [chatgpt 注册账号](https://readdevdocs.com/blog/makemoney/%E4%B8%AD%E5%9B%BD%E5%8C%BA%E6%B3%A8%E5%86%8COpenAI%E8%B4%A6%E5%8F%B7%E8%AF%95%E7%94%A8ChatGPT%E6%8C%87%E5%8D%97.html#%E6%B3%A8%E5%86%8C-openai-%E8%B4%A6%E5%8F%B7)
3. [申请一个微信公众号](https://kf.qq.com/faq/120911VrYVrA151009eIrYvy.html)
4. 在服务器安装好 Git 和最新 Go 版本

以上每一个步骤对于小白来说都会比较繁琐，但这就是折腾的乐趣

# 部署服务

1. 克隆仓库到服务器，[仓库地址](https://github.com/he2121/chatgpt_weixin) 

   ```
   git clone https://github.com/he2121/chatgpt_weixin.git
   ```

   代码其实很简单，只有三个文件

   1. main.go: 使用 gin 起一个 HTTP 服务，作为微信公众号的回调地址
   2. weixin.go: 使用 [微信SDK](github.com/silenceper/wechat/v2) 监听消息，返回消息
   3. gpt.go: 封装了下 open-ai 的 HTTP 请求

   发送消息的总链路：用户私聊公众号 -> 公众号后台 -> 自己的服务器 -> open-ai

   open-ai 的返回结果则逆向翻译

2. 代码配置设置

weixin.go 文件里有如下四个参数需要设置

![image-20230207000454800](https://ganghuan.oss-cn-shenzhen.aliyuncs.com/img/image-20230207000454800-2023-02-07.png)

从微信公众号后台 [基础配置页面](https://mp.weixin.qq.com/advanced/advanced?action=dev&t=advanced/dev&token=1030165537&lang=zh_CN) 可以获取到所有参数

![image-20230207001036442](https://ganghuan.oss-cn-shenzhen.aliyuncs.com/img/image-20230207001036442-2023-02-07.png)

1. 这里可以加白名单，把自己服务器的 IP 加上
2. token 随便填，只要和页面一致就行，不是指 access_token
3. URL 端口号必须 80

chatgpt.go 文件里有如下参数需要设置

![image-20230207001118045](https://ganghuan.oss-cn-shenzhen.aliyuncs.com/img/image-20230207001118045-2023-02-07.png)

在 chatgpt [setting 页面]() 获取

3. 在克隆下来的仓库内，执行以下命令，启动服务

```
go build
./chatgpt_weixin
```

4. 从微信公众号后台 [基础配置页面](https://mp.weixin.qq.com/advanced/advanced?action=dev&t=advanced/dev&token=1030165537&lang=zh_CN) 可以获取到所有参数设置好服务器 URL，通过校验并启动。

![image-20230207003732285](https://ganghuan.oss-cn-shenzhen.aliyuncs.com/img/image-20230207003732285-2023-02-07.png)

5. 私聊公众号测试，通过服务日志排查问题。

# 总结

1. 公众号回复必须 5 s 内，chatgpt 国内访问又慢，经常超时，微信公众号使用异步接口发送消息又比较复杂。只能说微信公众号真难用，还是飞书香

2. 关注 **小贺coding** 公众号，可体验一下 chatgpt，免费额度不多了，还是推荐自己搭建哦
