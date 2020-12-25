---
title: "利用PEG.JS从SQL生成模型类型签名"
date: "2020-07-04"
description: "介绍Swagger以及OpenAPI简易教程"
tags: [OpenAPI]
---

## 背景介绍
项目从JS迁移到TS后，我们需要为项目中大量的model类手写类型签名, 比如对于如下profile表

```sql
CREATE TABLE `profile` (
  `roleId` bigint(20) NOT NULL COMMENT 'playerId',
  `location` varchar(50) DEFAULT NULL COMMENT 'country-province-city',
  `signature` varchar(100) DEFAULT NULL COMMENT 'signature text',
  `avatar` varchar(255) DEFAULT NULL COMMENT 'user avatar url',
  PRIMARY KEY (`RoleId`),
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='user profile';
```
需要声明ProfileRecord为如下类型, 本文介绍如何使用PEG.JS来解析SQL的建表语句来自动生成类型签名
```ts
interface profile {
  roleId: number // playerId
  location: string // country-province-city
  signature: string // signature text
  avatar: string // user avatar url
}
```

### PEG.JS 简介
PEG（英语：Parsing Expression Grammar），解析表达语法, 是一种分析型形式文法, PEG.JS 是利用PEG文法来标示语言规则自动帮我们生成parser的工具

### 从例子中学习

打开网站PEG.JS[在线版本](https://pegjs.org/online)

#### 识别简单算术表达式
下面这个例子可以识别类似 2*(3+4) 的简单的算术表达式, 并计算他们的值
```ts
start         //语法标示的起点, 文本将从这个规则开始被解析
  = additive

additive // 规则开头表面该规则的名字
  /* 
  * left被规则名修饰匹配到的变量可用于后续方括号内的计算处理
  */
  = left:multiplicative "+" right:additive { return left + right; } 
  / multiplicative   // 符号 / 代表或的关系

multiplicative
  = left:primary "*" right:multiplicative { return left * right; }
  / primary

primary
  = integer
  / "(" additive:additive ")" { return additive; }

/** 
 * 这里第二个"integer"字符串的用于生成的parser中输出更加友好的错误信息
 */
integer "integer"  
  = digits:[0-9]+ { return parseInt(digits.join(""), 10); } 
```

让我们脑内模拟下此语法规则下 `2*(3+4)`解析和计算流程, 大致过程如下
```ts
// 向下传播
↓ start => additive:['2*(3+4)']

↓ additive:['2*(3+4)'] => multiplicative => left:[2] '*' right:[(3+4)] => 2 * right:[(3+4)]

↓ right:[(3+4)] => additive:[(3+4)] => "(" additive:[3+4] ")" => additive:[3+4]

↓ additive:[3+4] => left:[3] "+" right:[4] => 3+4 => 7

// 向上传播
↑ 7 => additive:[3+4]

↑ 7 => additive:[3+4] => right:[(3+4)] 

↑ 2 * right:[(3+4)] => 2 * 7 => 14 => additive:['2*(3+4)'] 

↑ 14 => additive:['2*(3+4)'] => start
```



### 参考
- [1] [PEG文法](https://en.wikipedia.org/wiki/Parsing_expression_grammar)
- [2] [PEG.JS官方文档](https://pegjs.org/documentation)
