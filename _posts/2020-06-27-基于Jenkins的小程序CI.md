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

### 创建Job

选择流水线类型的Job进行创建，如下图选项  

![流水线类型](https://i.loli.net/2020/07/02/83hnL25xkBypVde.png)

### 配置流水线Job

我这里用的方法是，把Jenkinsfile脚本文件放在仓库的develop分支  
Jenkins首先会拉取仓库develop分支的代码，去读取Jenkinsfile，如下图所示  

![流水线配置](https://i.loli.net/2020/07/02/woTZmEO5IJbqNlp.png)


### 编写流水线脚本
  
重头戏来了，这里的脚本类似于工作流，jenkins会根据脚本一步步执行下去。  
先根据我要实现的功能，整理出来以下几个步骤：  

1. 拉取代码（pipeline获取Jenkinsfile已经拉取代码）
2. 配置小程序上传版本信息
3. 小程序打包
4. 小程序上传

小程序每次上传代码，都有两个参数需要去填写
- 版本号
- 版本说明

**先放上基本的jenkinsfile**

```groovy
// 流水线
pipeline {
    // 代理
    agent any
    // 环境变量
    environment {}
    // 参数
    parameters {}
    // 工作任务
    stages {
        // 任务
        stage('构建') {
          // 步骤
          steps {

          }
        }
    }
}
```

流水线中提供了parameters(参数)来支持参数化构建，这就大大增加了我们可自定义的内容  

我在这里定义了下列几个参数
- 版本号
- 版本描述
- 构建模板（这个是公司业务相关）
- 构建环境（打包不同的环境）
- Git分支（适用于有特殊的场景，需要在不同的分支上进行打包上传）
- 是否安装依赖

