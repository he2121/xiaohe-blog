---
title: "云服务器可以干什么系列：vscode 远程开发调试"
date: 2021-12-16T20:39:11+08:00
tags: ["云服务器"]
---

## 为什么需要远程开发调试

- 本地开发环境设置复杂
- 本地操作系统 windows/mac 开发体验、兼容不佳
- 多设备共享一个开发环境，切换设备开发体验好

## 如何设置

1. 本地已安装 vscode，并以安装好开发语言插件（代码提示，补全...）
2. ssh 能连接上远程开发机，并且远程开发机安装好开发环境（如安装好 Git，Go）
3. vacode 插件市场搜索安装 Remote Development

![image-20211216153214241](http://ganghuan.oss-cn-shenzhen.aliyuncs.com/img/image-20211216153214241-2021-12-16.png)

4. `cmd + shift + p` 搜索 `Remote-SSH: Add New SSH Host` 输入 `ssh user@IP` 添加 ssh 连接记录。
5. `cmd + shift + p` 搜索 `Remote-SSH: Connect to Host`，选择需要连接的 IP 进行连接。然后弹出的 vscode 界面即是远程开发的环境了

![image-20211216153632258](http://ganghuan.oss-cn-shenzhen.aliyuncs.com/img/image-20211216153632258-2021-12-16.png)

 

