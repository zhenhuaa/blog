---
title: "利用PEG.JS从SQL生成模型类型签名"
date: "2020-12-28"
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

#### SQL DDL的结构

大致上，建表语句的声明大致为三部分，
- 数据表本身相关声明 (表名， 所用字符集，备用注释等)
- 表中列的相关声明 (列明，使用类型，相关注释等)
- 需要建立的索引(索引名字, 索引类型)

对于我们的目标来说, 生成ts声明, 需要关注的只有表名，列名，列的类型, 由此思路，我们可以构造简单的如下PEGJS语法规则来制导

```ts
start=Sqls EOF

Sqls = (TableDef / CommentLine / NoCreateStat) +

TableDef
  = _ "CREATE"i _ "TABLE"i _ tbName:field _ "(" cols:ColsDef keys:KeyStats? ")" tbEnd:TableEnd ";" _ {
    return {tableName: tbName, cols: cols, keys: keys, comment: tbEnd.comment}
}
  

ColDef 
  = _ colName:field _ type:type typeModify? _ colsModify comment:comment? ","? _ {
  return {colName: colName, type:type, comment: comment || ""}
}

```
> [完整规则可见](https://raw.githubusercontent.com/zhenhuaa/sql2ts/master/src/lib/sql2ts.pegjs)

#### SQL规则的解释
这次我们利用PEGJS生成parser对识别后，直接返回一个AST来存储解析的结果, 对于create table语句的解析，最终我们会返回如下一个结构。
```js
TABLE_DEF
= {tableName: "profile", cols: COLS, keys: KEYS, comment: "user profile"}

COLS =  [
      {
          "colName": "roleId",
          "type": "bigint",
          "comment": "playerId"
      }
]
```

#### 遍历AST来生成
有了AST结构，最终生成interface就非常简单, 我们只要遍历ast结构, 对于列中不同的COL的类型做一次映射，转换成ts支持的类型即可, 比如bigint转换成number， varcahr转换成string等等

### 制作成web工具
由于PEGJS生产的parser是纯js，我们可以很方便的制作成web版工具，运行在浏览器中,
![image.png](https://i.loli.net/2020/12/28/Ag8lC2Y3ZIPzJRx.png)
我们用REACT工具栈左右设置各放置一个aceEditor组件， 然后自动监听坐标组件文本变化，来生成右边内容即可,制作完成后的的 [在线工具地址](https://zhenhuaa.github.io/sql2ts/)

### 参考
- [1] [PEG文法](https://en.wikipedia.org/wiki/Parsing_expression_grammar)
- [2] [PEG.JS官方文档](https://pegjs.org/documentation)
- [3] [AST抽象语法树](https://zh.wikipedia.org/wiki/%E6%8A%BD%E8%B1%A1%E8%AA%9E%E6%B3%95%E6%A8%B9)
