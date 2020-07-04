
---
title: "Swagger简易指南"
date: "2020-07-04"
description: "介绍Swagger以及OpenAPI简易教程"
tags: [OpenAPI]
---

## 背景介绍
Web开发过程中， 文档的交付是非常重要的一环， 其中接口文档定义了前后端合作的边界，一个清晰简洁的文档能极大地提高双方合作的效率。 社区发展过程中，涌现了非常多的方案， 其中OpenAPI/Swagger是目前生态最为成熟的方案

## Swagger 简介
OpenAPI是一种用于描述REST API的规范格式， 一个完善的OpanAPI定义文件可以完整的表述出接口的路径， 请求参数，返回类型等信息

Swagger是指一套围绕OpenAPI规范构建的工具集， 其中比较常用的有

 * [Swagger Editor](https://editor.swagger.io/) 可以编写OpenAPI规范的**在线编辑器**
 * [Swagger UI](https://swagger.io/tools/swagger-ui/) 基于OpenAPI规范自动生成的**可交互的接口文档**

## 开始动手吧

### 从例子出发
开始编写swagger文档前，我们需要了解[OpenAPI规范](https://swagger.io/specification/),为了表示api设计的方方面面，规范描述了诸多细节，为了快速开始，让我们从一个简单的例子入手，比如设计一个朋友圈的列表接口，用OpenAPI可描述如下
```yaml

openapi: 3.0.2
info:
  title: 朋友圈接口
  description: 朋友圈接口，提供发表动态，评论，**点赞**等服务
  version: "1.0"

servers:
  - description: local-dev
    url: "http://local.dev.com:4001/md"
  - description: production
    url: "https://production.com/md"
    
tags:
  - name: "moment"
    description: 动态相关 
    
paths:
  /moments/list:
    get:
      tags: ["moment"]
      summary: Get Moment List
      parameters:
        - $ref: "#/components/parameters/roleid"
        
      responses:
        200:
          description: ok
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: "#/components/schemas/Moment"
              
components:
  parameters:
    roleid:
      name: roleid
      in: query
      description: 角色id
      schema:
        type: number
        example: 24017600001
        
  schemas:
    Moment:
      type: object
      properties:
        id:
          type: integer
          example: 1
        text:
          type: string
          example: "share moment"
        roleid:
          type: integer
          example: 24017600001
        createTime:
          type: integer
          example: 1593847438392
```

粘贴至到上文提到的[Swagger Editor](https://editor.swagger.io/), 我们就可以在右侧得到如下图一样的交互文档

![image.png](https://i.loli.net/2020/07/04/xg97ruvAipVCazt.png)

### 文档的层次结构
从上图的例子中，我们可以清晰的看出，文档大致分为6个不同的子单元

* openapi 版本声明  这关系到相关工具链对文档格式的解析
* info 声明该文档的一些基本信息，比如接口提供那些服务
* servers 用于声明不同环境下服务器的地址
* tags 用于给api提供分组， 可声明多个tag
* paths 最重要的部分，声明各个接口的具体规范
* components 可以复用的组件，主要包括请求的参数和返回定义的model

> components配合ref是一个很棒的设计，在一个业务的api中，不同接口往往需要共享参数列表和返回的model定义， 利用这个设计可以大幅度简化文档的冗余信息


### 最重要的PATH

Path单元下描述API具体结构，一个定义首先从路径出发， 如图中的moments/list, 路径下可以定义不同的HTTP方法， 使用不同的逻辑， 大致结构如下

* 路径 (moments/list)
    * HTTP方法 (GET)
        * 所属标签 (moment)
        * 接口描述 (Get Moment List)
        * 参数列表
            * 直接定义
            * 引用组件的定义
        * 返回数据
            * 返回码
                * 返回格式
                    *  返回结构体schema

关于参数定义， 上例中使用了query这个类型参数， 就是浏览器上的queryString
规范也允许声明位于url路径上的参数(类型为path). 对于更复杂的情况，比如web业务中常见的提交复杂的表单， 这时候可以使用描述requestBody配合schema精确定义

### Swagger UI
Swagger UI提供了从符合OpenApi规范的定义文件生成一个**交互文档**的能力，区别与传统的静态文档，我们可以直接在Swagger上发送请求，获取返回的数据，非常的方便

![image.png](https://i.loli.net/2020/07/04/6q5dzRCTU4Shiku.png)

点击每个路径旁边的Try it out按钮，就可以在页面上看到请求返回的数据， 可交互的文档带来了一个巨大的协作优势， 在问题不确定是客户端请求不规范还是服务器返回有问题时，可以用swagger请求一次解决很多沟通问题


## 出色的生态
相对于其他方案，OpenAPI有着非常好的生态，以及相关工具链, 如下网站索引了一系列和OpenAPI相关联的项目

[OpenAPITools] https://openapi.tools/

比如以下场景

* datavalidator 根据规范自动生产校验参数的代码
* doc  文档类，除了swagger UI方案，社区也有非常多其他ui方案来选择
* mock server 根据接口定义的schema， 自动生产mock数据
* tool vscode插件，编写openApi做lint校验和自动补全， 文件支持导入Postman测试工具
* sass服务 一些saas服务的接口管理平台都支持OpenAPI格式的导入与管理

## 总结
本文主要通过一个例子简单了介绍了openAPI规范的基础结构, 以及一些相关的生态, OpenAPI相对于其他方案，比较强大也略微复杂，初次引入可能带来一定的成本，但这种事情是短期内收益不高，长期收益明显的例子。笔者负责维护的项目中，已经把接口文档全部转移到该方案，相比于旧方案，swagger协作中明显更为顺畅。 清晰的文档对于交流协作非常重要， 与人方便，与己方便。