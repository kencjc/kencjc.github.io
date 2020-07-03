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

#### 环境变量
环境变量里面定义的参数是全局可用的
```groovy
...
environment {
  GIT_URL = 'git仓库地址'
  GIT_CERDENTIALS_ID = 'Jenkins中定义好的Git权限认证Id'
  BUILD_PATH = './dist/build/'    // 构建目录位置
  PRIVATE_KEY_PATH = './'   // 上传私钥路径，miniprogram-ci相关
}
...
```

#### 参数化构建

流水线中提供了parameters(参数)来支持参数化构建，这就大大增加了我们可自定义的内容  

我在这里定义了下列几个参数
- 版本号
- 版本描述
- 构建模板（这个是公司业务相关）
- 构建环境（打包不同的环境）
- Git分支（适用于有特殊的场景，需要在不同的分支上进行打包上传）
- 是否安装依赖

版本号和版本描述是在小程序上传时的参数  
构建环境，一般情况会区分测试、预发、生产环境，对应的小程序appId不一致，上传参数也不同。  
基于实际项目，会存在git工作流的情况，所有会有在不同分支上打包的情况。  

```groovy
...
parameters {
  string(name: 'VERSION', defaultValue: '1.0.0', description: '请输入版本号')
  string(name: 'VERSION_DESCRIPTION', defaultValue: '', description: '请输入版本描述，默认为：xx行业模板-xx环境-版本号（版本描述，不填则为空）')
  choice(name: 'ENV', description: '请选择构建环境', choices: ['dev', 'pre', 'prod'])
  gitParameter(name: 'BRANCH', description: '请选择构建分支', sortMode: 'NONE', quickFilterEnabled: true, defaultValue: 'develop', branchFilter: 'origin/(.*)', type: 'PT_BRANCH', )
  booleanParam(name: 'IF_INSTALL', description: '是否需要安装依赖（若无npm包更新,可以不安装,提高构建速度）', defaultValue: false )
}
...
```

具体的操作流程如下  
**PS: 第一次构建是没有Build with Parameters的选项的，需要先构建成功一次，让jenkins拉取代码识别到Jenkinsfile**

![参数化构建](https://i.loli.net/2020/07/03/LZN1MWbKxynpqQj.png)

![参数化构建图形界面](https://i.loli.net/2020/07/03/GQtXDu8my6VlFSK.png)

#### 构建流程

定义好参数后，就是构建的流程，整体思路是这样的
- 切换分支
- 安装依赖
- 构建打包
- 上传小程序

##### 切换分支
```groovy
...
stages {
  state('构建') {
    steps {
      script {  // 脚本语法
        // 切换构建分支
        git branch: "${params.BRANCH}", credentialsId: "${GIT_CERDENTIALS_ID}" , url: "${GIT_URL}"
      }
    }
  }
}
...
```

##### 安装依赖
这里多加了个参数IF_INSTALL是否安装，灵活搭配使用。
```groovy
...
stages {
  state('构建') {
    steps {
      script {  // 脚本语法
        // 安装
        if (params.IF_INSTALL == true) {    // 是否安装依赖
          echo '安装依赖'
          sh "npm install"
        }
      }
    }
  }
}
...
```

##### 构建打包
这里根据各自小程序框架的打包命令来执行就可以了  
我们这里用的是uni-app框架开发的小程序，需要把项目转成vue-cli版本后再通过命令行执行构建打包。  
项目内的配置参数我是用的gulp实现替换各环境的配置文件。
```groovy
...
stages {
  state('构建') {
    steps {
      script {  // 脚本语法
        // 构建打包
        sh "npm run build:mp-weixin"
      }
    }
  }
}
...
```

##### 上传小程序
--pp 是小程序打好包的路径  
--pkp 是小程序上传私钥路径，详情可以看miniprogram-ci文档，私钥我也是放在仓库内
--appid 是小程序的appId
--uv 是上传提交的版本号
--ud 是上传提交的描述
```groovy
...
stages {
  state('构建') {
    steps {
      script {  // 脚本语法
        // 上传小程序
        sh "miniprogram-ci upload --pp ${BUILD_PATH} --pkp ${PRIVATE_KEY_PATH}/private.${appId}.key --appid ${appId} --uv ${params.VERSION} --ud ${uploadDesc}"
      }
    }
  }
}
...
```

## 总结

搭建好这一套小程序CI之后，在日常测试、迭代中，确实极大的减少了人工操作的时间。更加友好的提测打包方式，对敏捷开发有一定的帮助。  
平时测试同事需要找前端开发打包新版本，需要打开工具配置环境信息，再去执行打包。有时候前端手里有活，再去处理打包效率会低下。如今支持了CI，效率提升杠杠的。  
纯开发者上传，同一个开发者只能显示一个开发版本（第三方平台草稿箱同样只能显示一个），运用CI上传，最多支持30个CI机器人传版本。
