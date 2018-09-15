---
layout:     post
title:      hive编译过程
subtitle:   hive源码学习之-hql到ast
date:       2018-09-08
author:     鲁湘宁
header-img: img/doc1-bg-alone.jpg
catalog: true
tags:
    - hive 入门 3.1.0 源码 hql解析 hql ast
---
## 前言
hql在用户写入之后就要进行解析，主要包括词法解析、语法解析、语义解析等。完成解析以后，根据语义解析的结果生成对应的map-reduce过程以获取数据。本节主要介绍从hql到抽象语法树的过程。hive从hql到ast的具体过程将会在本节详细的展开。

## 预备知识
### 巴科斯范式
>
定义：BNF范式是一种用递归的思想来表述计算机语言符号集的定义规范，又称巴科斯范式(Backus-Naur form)。  
内容：
>尖括号“< >” 内包含的为必选项。  
>方括号“[ ] ”内包含的为可选项。  
>大括号“{ } ”内包含的为可重复0至无数次的项。  
>竖线“| ”表示在其左右两边任选一项，相当于"OR"的意思。  
>“::= ”是“被定义为”的意思。  
>引号里面的内容代表其本身。  

为了方便,我们举个例子来说明  
>
在双引号中的字("word")代表着这些字符本身。而double_quote用来代表双引号。  
在双引号外的字（有可能有下划线）代表着语法部分。  
尖括号( < > )内包含的为必选项。  
方括号( [ ] )内包含的为可选项。  
大括号( { } )内包含的为可重复0至无数次的项。  
竖线( | )表示在其左右两边任选一项，相当于"OR"的意思。  
::= 是“被定义为”的意思。  
巴科斯范式示例  
下面是用BNF来定义的Java语言中的For语句的实例：  
FOR_STATEMENT ::=  
"for" "(" ( variable_declaration |  
( expression ";" ) | ";" )  
[ expression ] ";"  
[ expression ] ";"  
")" statement  
这是用BNF来定义的BNF本身的例子：  
syntax ::= { rule }  
rule ::= identifier "::=" expression  
expression ::= term { "|" term }  
term ::= factor { factor }  
factor ::= identifier |  
quoted_symbol |  
"(" expression ")" |  
"[" expression "]" |  
"{" expression "}"  
identifier ::= letter { letter | digit }  
quoted_symbol ::= """ { any_character } """  

然后我们尝试为java提供一下foreach的定义:  
FOREACH\_STATEMENT ::=  //即FOREACH\_STATEMENT定义为   
"foreach" "(" vars "as" var ")"  
statement  
接下来定义statement,为了简单方便，我们假设statement只有input和output 两种实现  
statement ::= "{" input | output "}"  
input ::= "readline(" file_name ");"  
file_name ::= """ { any_character } """   
output ::= "writeline(" file_name "," any_character ");"  

这样我们就实现了如下的语法:  
foreach(vars as var) {  
&nbsp;&nbsp;&nbsp;&nbsp;readline('thisfile.txt');  
&nbsp;&nbsp;&nbsp;&nbsp;writeline("thatfile.txt", var);  
}  

### 抽象语法树
>抽象语法树（abstract syntax code，AST）是源代码的抽象语法结构的树状表示，树上的每个节点都表示源代码中的一种结构，之所以说是抽象的，是因为抽象语法树并不会表示出真实语法出现的每一个细节，比如说，嵌套括号被隐含在树的结构中，并没有以节点的形式呈现。抽象语法树并不依赖于源语言的语法，也就是说语法分析阶段所采用的上下文无文文法，因为在写文法时，经常会对文法进行等价的转换（消除左递归，回溯，二义性等），这样会给文法分析引入一些多余的成分，对后续阶段造成不利影响，甚至会使合个阶段变得混乱。因些，很多编译器经常要独立地构造语法分析树，为前端，后端建立一个清晰的接口。

这里不再详细展开，后面会进行介绍。 
### ANTRL
君子性非异也，善假于物也。所以对于sql的解析，也有一个利器，也就是ANTRL。  
具体作用可以参考链接https://www.cnblogs.com/clonen/p/9083359.html  
总的来说，我们只需要按照ANTRL定义的方式定义hql的解析文档，ANTRL就可以按照定义进行词法分析、语法分析。可以参考一下thrift接口的生成，我们只需要定义thrift文件，运行生成程序，就可以生成我们可以使用的程序了。

## hive cli环境sql提交流程
当我们提交sql后，首先会创建CliDriver的对象实例。该过程会获取当前session的相关状态。  
![](https://luxiangning.github.io/img/hive/hive-cli_01.png)  

session对象会存储当前会话的相关配置，包括hive从配置文件读取的信息、通过命令行设置的状态信息，还有当前查询执行的状态信息等。
然后会对sql进行预处理，包括移除注释信息和进行分词，当前的分词逻辑是使用空白字符分词（如select * from employee会被分解为[select, *, from, employee]）  
![](https://luxiangning.github.io/img/hive/hive-cli_02.png)


基于分词后的tokens判断当前的执行类型并执行。当前包括  
1、quit、exit  
2、source (重新加载配置信息)  
3、！cmd ，即shell指令  
4、local mode,即sql本地执行方案。  



然后会执行  
CommandProcessorFactory.get(tokens, (HiveConf) conf)->
getForHiveCommand->
getForHiveCommandInternal->
HiveCommand.find(cmd, testOnly)
select 执行后的结果为null  
这时候会执行DriverFactory.newDriver(conf)，  
getNewQueryState生成一个QueryState, 会创建一个QueryId来标志该查询。    
然后执行  
![](https://luxiangning.github.io/img/hive/hive-cli_03.png)  

即基于当前的queryState生成一个Driver实例。  
当前设定的一些策略会影响后面的执行，  
![](https://luxiangning.github.io/img/hive/hive-cli_04.png)  
至此，得到一个真正执行的Driver，即proc。  
执行 processLocalCmd  
这里进入到最重要的流程   
![](https://luxiangning.github.io/img/hive/hive-cli_05.png)→  
coreDriver.compileAndRespond(statement)->  
compileAndRespond(command, false)->  
compileInternal(command, false) ->  
compile(command, true, deferClose)  
compile过程中会先把相关的变量替换掉。
queryStr = HookUtils.redactLogString(conf, command)  
![](https://luxiangning.github.io/img/hive/hive-cli_06.png)     
然后会设置transaction manager等不同的上下文信息。  
然后是抽象逻辑树的生成。  
![](https://luxiangning.github.io/img/hive/hive-cli_07.png)    
然后使用ANTLR进行词法分析。  
![](https://luxiangning.github.io/img/hive/hive-cli_08.png)   
生成tokens如下：  
![](https://luxiangning.github.io/img/hive/hive-cli_09.png)   
然后执行   
HiveParser parser = new HiveParser(tokens)  
执行解析  
r = parser.生成的Tree为  
(tok_query (tok_from (tok_tabref (tok_tabname employee))) (tok_insert (tok_destination (tok_dir tok_tmp_file)) (tok_select (tok_selexpr tok_allcolref)))) <eof>


nil

    TOK_QUERY

        TOK_FROM

            TOK_TABREF

                TOK_TABNAME

                    employee

        TOK_INSERT

            TOK_DESTINATION

                TOK_DIR

                    TOK_TMP_FILE

            TOK_SELECT

                TOK_SELEXPR

                    TOK_ALLCOLREF

    <EOF>

支持抽象逻辑树已经生成完毕。  
然后是语义分析和执行计划生成  
BaseSemanticAnalyzer sem = SemanticAnalyzerFactory.get(queryState, tree)    
analyzeInternal(ASTNode ast, PlannerContextFactory pcf)开始进行语义分析->  
genResolvedParseTree->  
(SemanticAnalyzer.java) **doPhase1**->
[processTable,processSubQuery,processJoin]  


## hql到ast
hsql到ast的过程可以参考如下的样例。
首先是表的定义：  
CREATE TABLE IF NOT EXISTS employee ( eid int,  
 name String,  
 salary String, destination String)  
 COMMENT 'Employee details'  
 ROW FORMAT DELIMITED  
 FIELDS TERMINATED BY '\t'  
 LINES TERMINATED BY '\n'  
 STORED AS TEXTFILE;  
 
 
CREATE TABLE IF NOT EXISTS employee_cost ( eid int,   
 name String,  
 salary String, destination String, cost bigint)  
 COMMENT 'Employee details'  
 ROW FORMAT DELIMITED  
 FIELDS TERMINATED BY '\t'  
 LINES TERMINATED BY '\n'  
 STORED AS TEXTFILE;  
  
 CREATE TABLE IF NOT EXISTS performance ( eid int,  
 cost bigint)  
 COMMENT 'performance'  
 ROW FORMAT DELIMITED  
 FIELDS TERMINATED BY '\t'  
 LINES TERMINATED BY '\n'  
 STORED AS TEXTFILE;  
 
 CREATE TABLE IF NOT EXISTS performance_second ( eid int,  
  cost bigint)  
 COMMENT 'performance'  
 ROW FORMAT DELIMITED  
 FIELDS TERMINATED BY '\t'  
 LINES TERMINATED BY '\n'  
 STORED AS TEXTFILE;  
 
 
 CREATE TABLE IF NOT EXISTS extra_info ( eid int,  
 hometown String)  
 COMMENT 'extra_info'  
 ROW FORMAT DELIMITED  
 FIELDS TERMINATED BY '\t'  
 LINES TERMINATED BY '\n'  
 STORED AS TEXTFILE;  
 
对应的数据插入的sql为:  
```sql
insert overwrite table employee_cost  
select  
    emp.eid,  
    emp.name,  
    emp.salary,  
    emp.destination,  
    pf.cost as cost  
from  
    (  
        select  
            eid,  
            name,  
            salary,  
            destination  
        from  
            employee  
        where  
            eid > 0  
    ) emp  
    join (  
        select  
            eid,  
            count(distinct cost) as count,  
            sum(cost) cost  
        from  
            performance  
        where  
            cost > 0  
        group by  
            eid  
        union all  
        select  
            eid,  
            count(distinct cost) as count,
            sum(cost) cost
        from
            performance_second
        where
            cost > 0
        group by
            eid
    ) pf on pf.eid = emp.eid
    left join extra_info ei on ei.eid = emp.eid
where
    pf.cost > 0
order by
    cost desc
limit
    100;
```    

对应的ast为

![](https://luxiangning.github.io/img/hive/hive-cli_10.jpg)  
