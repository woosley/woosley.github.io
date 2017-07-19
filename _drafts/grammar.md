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
 	
 	是一个或者多个X，其后为一个X或者Y的文法。
 	
 	```
 	<sentence> ::= X | Y | <sentence> X
 	```
 	
 	是一个X或者Y开头，其后跟多个X的文法。
 	
 - 这是一个X列表，并以；结尾的文法
 	
 	```
 	<sentence> ::= <listx> ";"
 	<listx> ::= X | X <listx>
 	```
 	
 - 现在我们可以将这些组合起来，形成一个“以；结尾的X句子”的列表
 	
 	```
 	<sentence> ::= <list> | <sentence> <list>
 	<list> ::= <listx>
 	<listx> ::= X | X <listx>
 	```
 	
 - 这是一个没有结尾；号但是有逗号分割的语言
 	
 	```
 	<listx> ::= X | X "," <listx>
 	```
 	 	
 - 这是一个每个X后都有一个；的语言
 	
 	有时候你必须问自己这是一个终止定界符还是一个分隔定界符


 	```
 	<listx> ::= X ";" | X ";" <listx>
 	```
 	再次比较分隔符的情形，每个X都被分隔开来。
 	
 	```
 	<listx> ::= X | X "," <listx>
 	```
 	
 - 由 X, Y, Z 组成的参数列表
 	
 	```
 	<arglist> ::= "(" ")" | "(" <varlist> ")"
 	<varlist> ::= <var> | <varlist> "," <var>
 	<var> ::= X | Y | Z
 	```
 	
 - 简单的类型定义
 
 	这是一个简单地类C语言类型定义语句的文法。它很具有层次感
 	
 	```
 	<tdecl> ::= <type> <varlist> ";"
 	<varlist> ::= <var> | <varlist> "," <var>
 	<var> ::= X | Y |Z
 	<type> ::= int | bool| string
 	```
 	
 - 包含关键字Static的类型文法
 	
 	```
 	<tdec> ::= <type> <varlist> ";"
 	<varlist> ::= <var> | <varlist> "," <var>
 	<var> ::= X | Y |Z
 	<type> ::= static <basictype> | <basictype>
 	<basictype> ::= int | bool | string
 	```
 	
 - 有嵌套括号的树状结构
 	
 	考虑这样一种表示树的方法：（ant）或者（cat，bat）或者（（cat），（bat））或者（（cat，bat，dog），ant）。注意有层次的处理各种特性。同时注意当<tree>从<thing>上升到文法顶端时，形成了一种嵌套效果。注意在顶端匹配<tree>的时候需要将括号移除是很重要的，这样可以防止无限循环。（我表示后面的都没有看懂）
 	
 	```
 	<tree> ::= "(" <list> ")"
 	<list> ::= <thing> | <list> "," <thing>
 	<thing> ::= <tree> | <name>
 	<name> ::= ant | bat | cow | dog
 	```
 	
###  结合性和优先级

当分隔符如圆括号没有的时候，将解析树恰当的分组需要我们理解结合性和优先级。现代C++的优先级多达十几层。这也是有一个分层设计大放光彩的地方。

- 算术表达式的文法，这里包含5个操作符：+，-，*，/，^(^是乘方)。操作符结合性决定了同质操作符的执行顺序。前面四个操作符从左至右求值，代表它们的结合性是从左至右或叫做左结合性。数学中的乘方是从右至左，也就是右结合性。在下面的文法中递归的地方，你可以看到这些。比如<exp>，递归发生在操作符的左边，而<multpart>，递归在右边。
 	
- 操作符的优先级决定在一个表达式中不同操作符的执行顺序。优先级通过将在同一个`production`中相同优先级的操作符分组来处理。+和-具有相同的优先级，*和/也一样。最高优先级的操作符出现在文法的更底端，比如一个表达式是首先是一堆product的集合，这些product才进而是一堆乘幂的集合。(an expression is a sum of products which is product of exponentiation)（啥玩意？）.

- 最后，可以将product放在括号里面

要获得这样的效果，括号包起来的表达式被放在任何变量可以出现的地方，为了回归到文法的顶端，括号被用作计数器，记录你回到文法顶端的次数。

```
<exp> ::= <exp> + <addpart> | <exp> - <addpart> | <addpart>
<addpart> ::= <addpart> * <multpart> | <addpart> / <mulpart>
<mulpart> ::= <group> ^ <mulpart> | <group>
<group> ::= <var> | ( <exp> )
<var> ::= X | Y |Z
```

让我们看一下使用上面文法的简化版，如何来解析表达式`Z-(X+Y)*X`

```
<exp> ::= <exp> + <addpart> | <exp> - <addpart> | <addpart>
<addpart> ::= <addpart> * <mulpart> | <mulpart>
<var> ::= X | Y | Z
```

<img src="http://marvin.cs.uidaho.edu/Teaching/CS445/calcParse.png">

由于这是一个CFG，上面的文法可以通过将所有<var>替换成production右边的 X|Y|Z获得简化.

#### 一元操作符


#### 一个例题


### 如何搞砸

文法最常见的问题有

- 在production里面使用了不在字母表中的符号。
- 不可的Production（inescapable production)，见下文
- 模糊的文法。下文会详细讨论

#### inescapable production

考虑如下两个简单地文法

```
<exp> ::= <exp> + <pound> | <exp> - <pound> | <mul>
<mul> ::= <mul> * <var> | <mul> / <var> | <var>
<var> ::= X | Y |Z
```

和

```
<exp> ::= <exp> + <pound> | <exp> - <pound> 
<mul> ::= <mul> * <var> | <mul> / <var> 
<var> ::= X | Y |Z
```

第二个文法砍死合理，但是等等...，由于右边都包含有<exp>，你怎样将它消除呢？这意
味着你永远无法生成没有包含<exp>的句子。第一个文法无效的原因是右边所有的选项都包
含有左边导致production `inescapable`

#### 模糊性

对语言中句子的处理来获得它的含义，一般使用解析树。解析树是一种理解句子的意思和
结构的方法。如果对于语法G，其对于语言中存在有多个解析树的句子，那么G就是模糊的
。证明这一点你只需要一个例子就可以了。注意有模糊性的是文法，而不是语言。同时注
意由于使用模糊文法，一个句子有多个解析是可以的。

```
<sentence> ::= X | <sentence> X | X <sentence>
```

这个文法就是模糊的，因为存在具有多个解析树的句子。比如：XX有两个不同的解析树。

<img src='http://marvin.cs.uidaho.edu/Teaching/CS445/ambiguousL.png'>


这是同一语言的另外一种模糊法。

```
<sentence> ::= X | <sentence> <sentence>
```

试试使用上面的文法解析句子**XXX**。你能找到两个解析树吗？解决这个问题只需要：

```
<sentence> ::= X | <sentence> X
```

这是另一个非常相似的模糊文法的例子：

```
<binary-string> ::= 0 | 1 | <binary-string> <binary-string>
```

这是修复方法

```
<binary-string> ::= 0 <binary-string> | 1 <binary-string> | 0 | 1
```

模糊性会影响文法的含义:

```
<sentence> ::= <expression>
<expression> ::= <expression> + <expression> |
                 <expression> * <expression> | 
                 <identifier>
                 
<identifier> ::= X | Y | Z
```

此语言中有以下句子

```
X
X + Y
X + Y * Z
```

头两个句子都没什么问题，但是 `X + Y * Z`有多个解析树。我们通过深度优先遍历解析
树来提取其结构的时候，我们希望先进行乘法。怎样做到这一点呢？我们需要控制操作符
的优先级。这里我们修改这个模糊的语法，在加法之前进行乘法

```
<sentence> ::= <expression>
<expression> ::= <term> | <expression> + <expression>
<term> ::= <identifier> | <term> * <term>
<identifier> ::= X | Y | Z
```

