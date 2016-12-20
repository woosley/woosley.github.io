# 文法指南
原文 http://marvin.cs.uidaho.edu/Teaching/CS445/grammar.html

文法（Grammar）用来指定一门语言的句法（syntax）。它能够回答：哪些句子属于这门语言，哪些不是？句子是有语言的字母表中的符号所组成的有穷序列。我们这里讨论的是“context free languages”的语言，其文法可称为“Context free Grammar”或者“CFG”。


### 一些定义

一个文法定义包括：

- **字母表**（ALPHABET）是出现在语言中的一组有限符号
- **NONTERMALS** 能够被Production中的一些符号所取代的符号
- **TERMINAL** 字母表中的符号
- **PRODUCTIONS** nonterminals的替换规则
- **Start Symbol** 一个初始符号，语言中所有句子都能从这个符号推导出来。

**Backus-Naur Form（BNF/巴科斯范式）**是用来描述CFG的系统记号，这些记号有：

- **NONTERMINALs**用<>包围，比如“<turtle>”
- **ALTERNATION**用|分割，比如“cats|dogs”
- **REPLACEMENT**（替换）用::=标注，这些就是**PRODUCTIONs**。比如 <dogs> ::= corgi | other
-  **BLANKS（空格）**忽略或者必须被转义。
-  **TERMINALS** 是未加装饰的符号。

以下是描述简单英语句子的一个BNF文法。

```
<sentence> ::= <subject> <predicate>
<subject> ::= <article> <noun>
<predicate> ::= <verb> <direct-object>
<direct-object> := <article> <noun>
<article> ::= THE | A
<noun> ::= MAN | DOG
<verb> ::= BITES | PETS
```

`THE DOG BITES A MAN`在属于这门语言吗？是的

```
<sentence> -> THE DOG BITS A MAN
```

MAN BITES DOG 不属于此文法

**解析树（parse tree）**是一组树型符号，它用来描述使用文法构造句子的方法。对于一个句子，可能有多余一组解析树（见不明确性）。对于`THE DOG BITES A MAN`，其解析树为

<img src=http://marvin.cs.uidaho.edu/Teaching/CS445/sentenceParse.png>

**Derivation（衍生）**是关于一系列步骤的有序列表，它用来从文法为一个句子生成特定的解析树

**Left most Derivation（左衍生）**是一种衍生的方法，在这种方法中最先替换最左边的nonterminal

**Parse（解析）**用来显示一个句子怎样从文法构建。

 **Metasymbols** are symbols outside the language to prevent circularity in talking about the grammar. 
 
 ### 一些例子
 
 计算机语言中许多语言学上的结构都源自一些基本方言的集合。这是一些常见的形式，你可能在你的语言中要用到:
 
 - **允许一个包含X的列表的文法**
 	
 	```
 	start symbol:
 	<sentence> ::= X | X <sentence>
 	```
 	或者
 	
 	```
 	<sentence> ::= X | <sentence> X
 	```
 	这也说明对于一门语言，他可能有多种正确的文法
 	
 	- **一个包含X或者Y的列表的语法**

 	```
 	<sentence> ::= X | Y | X <sentense> | Y <sentense>
 	```
 	以下是一个更有层次的写法
 	
 	```
 	<sentence> ::= <sub> | <sub> <sentence>
 	<sub> ::= X | Y
 	```
 	
 	注意第一个部分处理“包含子部分的列表”，子部分可以是X或者Y。
 	
 	这是一些和以上相似，但又不同的语言的文法
 	
 	```
 	<sentence> ::= X | Y | X <sentence>
 	```
 	
 	是一个或者多个X，其后为一个X或者Y的语法。
 	
 	```
 	<>
 	```