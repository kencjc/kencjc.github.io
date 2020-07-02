---
title: 基于Jenkins的小程序CI
date: 2020-06-27 20:07:34
tags: jenkins 小程序 uni-app CI
categories: 技术
---

> CI,即持续交付。简化重复操作，避免人为操作出现的问题。

公司的业务主要作为第三方平台的小程序开发。每次花在构建代码，填写版本信息，上传代码的时间，平均一个环境需要2~10分钟，甚至更多。那时候就想着，既然每次操作都是固定流程式的，有没办法能够简化人力成本，将这套流程抽离出来。这时候就了解到了CI。  


## 相关介绍

### Jenkins

> Jenkins是基于Java开发的一种持续集成（CI）工具。

本次小程序CI实际运用了Jenkins里Job（任务）里面的Pipeline（流水线）。  
Pipeline通过识别Jenkinsfile文件内的脚本去执行步骤，Jenkinsfile一般放在仓库根目录下。  

### [miniprogram-ci](https://developers.weixin.qq.com/miniprogram/dev/devtools/ci.html){:target="_blank"}

> miniprogram-ci 是从微信开发者工具中抽离的关于小程序/小游戏项目代码的编译模块。开发者可不打开小程序开发者工具，独立使用 miniprogram-ci 进行小程序代码的上传、预览等操作

miniprogram-ci支持脚本调用和命令行调用两种方式，这里用到的是**命令行调用**的方式。


```bash
#upload
miniprogram-ci \
  upload \
  --pp ./demo-proj/ \               // 构建好的代码路径
  --pkp ./private.YOUR_APPID.key \  // 上传私钥，微信公众平台下载
  --appid YOUR_APPID \              // 小程序appId平台化这里填开发模板的小程序appId
  --uv PACKAGE_VERSION \            // 版本号
  --ud PACKAGE_DESCRIPTION \        // 版本描述
  -r 1 \                            // 定义CI机器人，可选1~30，可不填
```

## 流程

### 1、创建Job
选择流水线类型的Job进行创建
![](https://i.loli.net/2020/07/02/83hnL25xkBypVde.png)
