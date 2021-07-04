---
title: "基于OpenAPI的测试代码生成"
date: "2021-07-04"
description: "本文介绍了从OpenAPI Spec文件自动生成接口测试的方法"
tags: [OpenAPI, Test, CodeGen]
---

### 背景介绍
OpenAPI[<sup>1</sup>](#refer-anchor)是一种用于描述REST接口,和语言无关的接口定义规范，可用于文档生成, 生成不同语言客户端等多种用途, 本文主要阐述利用OpenApi的接口规范来自动生成基于SuperTest[<sup>2</sup>](#refer-anchor)
的接口测试代码

### OpenAPI概览

#### 接口定义
```yaml
  /news/list
    get:                                                                      
     tags:                                                                   
       - news                                                                
     summary: 新闻列表                                                     
     parameters:                                                             
       - $ref: "#/components/parameters/userIdOptional"
       - $ref: "#/components/parameters/searchKw"                            
       - $ref: "#/components/parameters/page"                                
       - $ref: "#/components/parameters/pageSize"                          
     responses:                                                              
       "200":                                                                
         $ref: "#/components/responses/NewListRes"                         
```
该代码片段用OpenAPI描述了一个新闻列表接口,  接口路径为`/news/list`, 有四个参数, 返回为NewList结构的接口, 注意$ref语法， 为了复用参数和返回的结构定义, OpenAPI规范允许我们使用`$ref`语法引用在compoents模块里定义的组件定义

#### 引用的参数和返回结构

##### 参数片段
```yaml
  userIdOptional:
    name: userId
    in: query
    required: false
    description: 用户id
    schema:
      type: number
      example: 10
```
接下来我们看下参数的引用定义, 参数这里最重要的参数名字也就是name以及schema, name表示了参数的字段名，schema规范了参数的shape, 其他参数类似

##### 返回结构
```
  NewListRes:
    description: OK
    content:
      application/json:
        schema:
          type: object
          properties:
            code:
              type: number
              example: 0
            data:
              type: object
              properties:
                list:
                  type: array
                  description: 新闻列表
                  items:
                    $ref: "#/components/schemas/NewsItem"

```
返回结构例子如图，由此可知， 该返回为json结构，有code和data两个字段， data是个对象，对象里的list引用了一个NewsItem的schema结构

#### 期望生成的代码
```typescript
  it("should get newsList ok!", async () => {
    const qs: NewsReq.NewsList = {
      userId: 10,
      kw: "显卡福利",
      page: 1,
      pageSize: 10,
    };

    const res = await apiTest
      .get(prefix + "/news/list")
      .query(qs)
      .expect(200);

    const body = res.body;
    assert.equal(body.code, 0);

    const data: NewsRes.NewsList = body.data;
    assert.ok(data);
  });

```


### 需求分析
输入的是描述OpenAPI的yaml文件， 输出是基于supertest的测试用例ts代码,  可以作为测试的一部分跑在CI中

### 可行性分析
  从举例的文档结构可知，我们在每个接口描述中，详尽的描述了接口的HTTP Method, 接口的URL， 参数列表, 返回结构, 故而可以根据文档的描述自动按照文档中url来请求, 然后使用参数的scehma的example值来请求接口, 获取返回数据，从而自动生成一个测试用例, 也是说文档描述即为测试用例 从而摆脱人工编写测试的繁琐, 提高生产效率


### 难点分析
1. 参数类型多样， 参数不仅需要支持querystring， 还需要处理pathParams， 还需要处理reqBody格式
2. ref语义处理， schema结构中的ref可以递归引用，在生成测试参数的时候，需要resolve所有ref，并生成正确格式的测试例子
3. 容错处理，目前OpenApi是人工手写维护, 可能部分结构书写信息并不完善，这种情况下生成器需要容错

### 解决方法
1. 对于要支持的参数类型不同类型分别编写独立的unit test， 先完成一小部分在自底向上的集成到到整个codebase中
2. ref语义处理, 基于不同type对schema进行DFS遍历, 检测到ref结构后，根据ref路径查询整个文档并解析，之后不断重复，直到没有ref结构位置
3. 容错处理, 开启ts的strict选项， 对所有可能不存在的属性用编译器确保完成了nullcheck, 并且在使用生成器之前引入linter, 确保文档结构正确

### 完成效果展示

```bash
❯ openapi-code-gen ~/Workspace/fight-app/doc/swagger.json -o ~/Workspace/fight-app -t 'news' -i apiTest --dryRun
✨ openapi-component 1.0.24
🔭 Loading spec from /Users/zhenhua/Workspace/fight-app/doc/swagger.json…
generate component apiTest to /Users/zhenhua/Workspace/fight-app/test/components/news/index.test.ts
import { apiTest, prefix } from "../../testHelper";
import * as assert from "power-assert";
import { NewsReq, NewsRes } from "../../../src/components/news/type";

describe("Component#news", function () {
  beforeEach(async () => {});

  afterEach(async () => {});

  it("should get newsList ok!", async () => {
    const qs: NewsReq.List = {
      userId: 10,
      kw: "显卡福利",
      page: 1,
      pageSize: 10,
    };

    const res = await apiTest
      .get(prefix + "/news/list")
      .query(qs)
      .expect(200);

    const body = res.body;
    assert.equal(body.code, 0);

    const data: NewsRes.List = body.data;
    assert.ok(data);
  });

});

🚀 build component news /Users/zhenhua/Workspace/fight-app/doc/swagger.json -> /Users/zhenhua/Workspace/fight-app [122ms]

```

笔者在这里实现了一个OpenAPI的codegen， 用来帮助自动生成业务中接口的测试， 其中apiTest的代码生成只是其中一小部分, 目前codegen是基于组件来生产代码, 可以看到会自动提取schema的example然后生成测试逻辑，并检查返回是否200, 目前测试用例中并不会检查返回是否匹配定义的schema结构，这是因为笔者为OpenAPI生成的接口映射函数也生成了类型绑定，可以在编译期确保返回类型和文档结构匹配，所以测试这边就不再需要额外处理

### 总结
通过自动生成用例以及相关代码自动生成，笔者在新项目中实现了完全的Design First的API开发流程, 在完成最初的文档设计之后, 就可以和API的Consumer进行沟通协调，大大提高了开发效率， 并且通过在CI中集中测试，可以轻松确保每次修改调整都能符合文档设计,提高了整体的效率


### 扩展阅读
另一种有趣的思路，Rails社区的[rswag](https://github.com/rswag/rswag), 通过测试来自动生成文档


<div id="refer-anchor"></div>

### 参考
[1] [OpenAPI文档规范](https://swagger.io/specification/)

[2] [SuperTest库地址](https://github.com/visionmedia/supertest)
