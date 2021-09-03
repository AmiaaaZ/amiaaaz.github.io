---
title: "GraphQL学习笔记"
slug: "graphql-study-notes"
description: "做题中遇到了，拿来整体的学习一下"
date: 2021-09-03T20:56:54+08:00
categories: ["NOTES&SUMMARY"]
series: ["SQLi"]
tags: ["GraphQL"]
draft: false
toc: true
---

## GraphQL简述

GraphQL是一种针对Graph（图状数据）查询很有优势的Query Language（查询语言），而涉及到存储时可以选择NoSQL, SQL或其它任意存储方式（例如文本文件、存内存里等）；这是一门便于前后端交互的语言，而不是便于后端和数据库交互的语言。

应用GraphQL的一个很重要的前提是后端数据已经以图的结构进行保存，（并且一定情况下已经设置好基于隐私的访问控制 授权与鉴权，否则会直接被攻击者执行高危操作）。每次查询或更新都有自己的根节点，得到的数据是树状结构；如果希望以图的形式展示则前端不能简单的对其进行缓存，那必须使用相应的存储数据库，通过顶点的ID把不同节点之间的某些边重新连接起来。

并不是所有场景都需要迁移到GraphQL，如果RESTful API已经能满足需求的话。

> [GraphQL is basically just sugar for a simply typed lambda calculus.](https://twitter.com/jlouis666/status/1018140132153667584)

### 变更 - Mutations

>  [three important things:](https://medium.com/@HurricaneJames/graphql-mutations-fb3ad5ae73c4)
>
> mutations are just queries in different namespace, but do NOT mix them;
>
> arguments require Input Objects, not normal Objects;
>
> use xyzAttributes for anything you want to link, then let your backend sort out how to do the linking(just like any other system we currently use)

### 内省 - introspection

GraphQL 允许在查询的任何位置请求 `__typename`，一个元字段(Meta fields)，以获得那个位置的对象类型名称。

![image-20210902180824151](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210902180824151.png)

我们也可以通过查询 `__schema` 字段来向 GraphQL 询问哪些类型是可用的，类型有以下这些：

- Query, Character, Human, Episode, Droid - 这些是我们在类型系统中定义的类型。
- String, Boolean - 这些是内建的标量，由类型系统提供。
- `__Schema`, `__Type`, `__TypeKind`, `__Field`, `__InputValue`, `__EnumValue`, `__Directive` - 这些有着两个下划线的类型是内省系统的一部分。

## 敏感信息泄露&越权

自动文档生成/解析 - [graphdoc](https://github.com/2fd/graphdoc)  |  [graphql-playground](https://github.com/graphql/graphql-playground)  |  [graphql-voyager](https://apis.guru/graphql-voyager/) ......

![image-20210902183558042](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210902183558042.png)

由于对对象或属性的权限控制不完善，导致信息泄露，案例：[hackerone 一系列信息泄露漏洞](https://hackerone.com/reports/310946)

在objects.types中寻找敏感信息，如email, password, secretkey, token, licensekey, session等，多多关注废弃字段（deprecated fields)。当字段被废弃后直接用`__type`做内省确实查找不到，但当指定`includeDreprecated: true`时，`__type`仍然可以将废弃字段暴露出来。

## GraphQL的认证方式

GraphQL并没有规定任何身份认证和权限控制的相关内容，因此我们可以更灵活的在应用中实现各种粒度的认证和权限；但是也很容易写出一些“裸奔”的接口或无效认证无效的接口。

### 独立认证终端 (RESTful)

通用且官方推荐的方式，如果后端本身支持RESTful或有专门的认证服务器，可以修改少量代码实现GraphQL接口的认证。

举例：添加jwt认证

![image-20210902200434456](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210902200434456.png)

### 在GraphQL内认证

如果GraphQL的后端支持GraphQL不能支持RESTful，或全部请求都需要使用GraphQL，也可以用构造相关的Query Schema接口返回token的形式。

举例：构造login的Query Schema，在返回值中携带token

```
type Query{
	login(
		username: String!
		password: String!
	): LoginMsg
	type LoginMsg{
		message: String
		token: String
	}
}
```

在resolver中提供登录逻辑

```
import bcrypt from 'bcrptjs';
import jsonwebtoken from 'jsonwebtoken';
export const login = async(_, args, context) => {
	const db = await context.getDb();
	const{username, password} = args;
	const user = await db.collection('User').findOne({username: username});
	if(await bcyrpt.compare(password, user.password)){
		return{
			message: 'Login success',
			token: jsonwebtoken.sign({
				user: user,
				exp: Math.floor(Date.now() / 1000) + (60 * 60),
			}, 'your secret'),
		};
	}
}
```

登录成功后 我们把token设置在请求头中，继续请求GraphQL的其他接口，这时需要对ApolloServer进行如下配置

```
const server = new ApolloServer({
	typeDefs: schemaText,
	resolvers: resolverMap,
	context: ({ ctx }) => {
		const token = ctx.req.headers.authorization || '';
		const user = getUser(token);
		return{
			...user,
			...ctx,
			...app.context
		};
	},
});
```

实现getUser函数

```
const getUser = (token) => {
	let user = null;
	const parts = token.split(' ');
	if(parts.length === 2){
		const scheme = parts[0];
		const credentials = parts[1];
		if(/^Bearer$/i.test(scheme)){
			token = credentials;
			try{
				user = jwt.verify(token, JWT_SECRET);
				console.log(user);
			}catch(e){
				console.log(e);
			}
		}
	}
	return user
}
```

配置好ApolloServer后，在resolver中校验user

```
import {ApolloError, ForbiddenError, AuthenticationError} from 'apollo-server';
export const blogs = async(_, args, context) => {
	const db = await context.getDb();
	const user = context.user;
	if(!user){
		throw new AuthenticationError('You must be logged in to see blogs');
	}
	const {blogId} = args;
	const cursor = {}；
	if(blogId){
		cursor['_id'] = blogId;
	}
	const blogs = await db
		.collection('blogs')
		.find(cursor)
		.sort({publishedAt: -1})
		.toArray();
	return blogs;
}
```

![image-20210902203603699](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210902203603699.png)

## 更多安全漏洞

Express-GraphQL：

- 框架默认无防护
- 自带GraphiQL

Graphene-Django：

- 依赖Django的安全配置（Secure As Default）
- 自带GraphiQL

GraphQL-PHP

- 无关框架

### Express-GraphQL Endpoint CSRF漏洞

```
{"query":"mutation {\n editProfile(name:\"hacker\", age: 5) {\n name\n  age\n }\n}","variables":null}
```

![image-20210902205831949](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210902205831949.png)

将Content-Type修改为application/x-www-form-urlencode，仍可成功执行

```
query=mutation%20%7B%0A%20%20editProfile(name%3A%22hacker%22%2C%20age%3A%20
5)%20%7B%0A%20%20%20%20name%0A%20%20%20%20age%0A%20%20%7D%0A%7D
```

![image-20210902210011480](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210902210011480.png)

直接配合burp自带的Generate CSRD POC

![image-20210902210221214](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210902210221214.png)

### GraphiQL Clickjacking 漏洞

参见：https://github.com/graphql/graphiql/issues/683

可以配合burp自带的Clickbandit进行攻击

### GraphQL injection 漏洞

[这是一个相当全的payloads&exps](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/GraphQL%20Injection#enumerate-database-schema-via-introspection)  |  [这是一个自省payload](https://gist.github.com/craigbeck/b90915d49fda19d5b2b17ead14dcd6da)

p神ppt里的示意图直接搬过来了

![image-20210902211535481](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210902211535481.png)

![image-20210902211551814](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210902211551814.png)

仍然是拼接了恶意的GraphQL语句导致漏洞的发生，本质还是对用户输入的控制不严格；同类的漏洞还有xss, rce等等

> [有语法就有解析，有解析就会有结构和顺序，有结构和顺序就会有注入。](https://www.freebuf.com/articles/web/184040.html)

用“参数化查询”的方式来解决上述问题时，要确保后端的解析引擎没有大病

### 通过Custom Scalar的注入 (JSON)

> [NoSQL Injection is entirely possible when using GraphQL, and can creep into your application through the use of 'custom scalar types'](http://www.petecorey.com/blog/2017/06/12/graphql-nosql-injection-through-json-types/)



————更多的GraphQLi相关问题可参见[这个git仓库](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/GraphQL%20Injection#enumerate-database-schema-via-introspection)，一本满足（

### 拒绝服务

GraphQL中的query和mutation的返回结果都是可以有嵌套的对象的，如果不对嵌套深度进行限制，有可能被利用从而进行拒绝服务攻击。

一个举例：

定义了Blog和Author:

```
type Blog{
	_id: String!
	type: BlogType
	avatar: String
	title: String
	content: [String]
	author: Author
	....
}
type Author{
	_id: String!
	name: String
	blog: [Blog]
}
```

都有各自的Query:

```
extend type Query{
	blogs(
		blogId: ID
		systemType: String!
	): [Blog]
}
extend type Query{
	author(
		_id: String
	): Author
}
```

我们可以构造这样的查询，无限套娃导致dos

```
query GetBlogs($blogId: ID, $systemType: String!) {
    blogs(blogId: $blogId, systemType: $systemType) {
        _id
        title
        type
        content
        author {
            name
            blog {
                author {
                    name
                    blog {
                        author {
                            name
                            blog {
                                author {
                                    name
                                    blog {
                                        author {
                                            name
                                            blog {
                                                author {
                                                    name
                                                    blog {
                                                        author {
                                                            name
                                                            blog {
                                                                author {
                                                                       name
                                                                       # and so on...
                                                                }
                                                            }
                                                        }
                                                    }
                                                }
                                            }
                                        }
                                    }
                                }
                            }
                        }
                    }
                }
                title
                createdAt
                publishedAt
            }
        }
        publishedAt
    }
}
```

解决这个问题我们需要在GraphQL服务器上限制查询深度，同时设计GraphQL接口时尽量避免出现此类问题，以Node.js为例，graphql-depth-limit就可以解决这样的问题

```
// ...
import depthLimit from 'graphql-depth-limit';
// ...
const server = new ApolloServer({
	typeDefs: schemaText,
	resolvers: resolverMap,
	context: ({ ctx }) => {
		const token = ctx.req.headers.authorization || '';
		const user = getUser(token);
		console.log('user', user)
		return{
			...user,
			...ctx,
			...app.context
		};
	},
	validationRules: [ depthLimit(10) ]
});
// ...
```

### Graphene-Django DEBUG模式下的安全问题

![image-20210903201318096](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210903201318096.png)

## 在CTF中的表现

[[HITB CTF Singapore 2017]Blog](https://tsublogs.wordpress.com/2017/08/25/hitb-ctf-singapore-2017-web-512-blog/)

[[SECT 2017]Dark Market](https://godot.win/2017/GraphQl/#SEC-T-2017-Dark-Market)  |  [wp2](https://github.com/reznok/CTFWriteUps/tree/master/SEC-T_2017/DarkMarket)

[[Hack in Paris CTF 2019]Meet Your Doctor 1 2 3](https://swisskyrepo.github.io/HIP19-MeetYourDoctor/)  |  [wp2](https://jaimelightfoot.com/blog/hack-in-paris-2019-ctf-meet-your-doctor-graphql-challenge/)

[[VolgaCTF 2020]Library](https://www.gem-love.com/ctf/2198.html)

[corCTF 2021]devme

## 结尾

[Damn Vulnerable GraphQL Application](https://github.com/dolevf/Damn-Vulnerable-GraphQL-Application)-> 一个漏洞复现的靶场，包含了上面提到和没提到的GraphQL存在的洞

```
docker pull dolevf/dvga
docker run -d -p 5000:5000 -e WEB_HOST=0.0.0.0 dolevf/dvga
```

![image-20210903203245165](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210903203245165.png)

已经有[写好的wp](https://mp.weixin.qq.com/s/WoHEC50u7KACLLafZL5tww)了 不向互联网产出湿垃圾 从我做起

------

{{% spoiler "以下是本文中涉及到的 和我学习时看过的所有文章的链接 每日感谢互联网的丰富资源（" %}}

[中文官网](https://graphql.cn/)

[在线GraphiQL](http://graphqlapp.herokuapp.com/)

[learn-graphql](https://github.com/zhouyuexie/learn-graphql)



[什么是 GraphQL？](https://www.zhihu.com/question/264629587)

[玩转graphQL](https://www.secpulse.com/archives/148242.html)

[GraphQL 从入门到实践](https://juejin.cn/post/6844903795407716366)

[【CuteJavaScript】GraphQL真香入门教程](https://juejin.cn/post/6844903841813495822)



[攻击GraphQL](https://xzfile.aliyuncs.com/upload/zcon/2018/7_%E6%94%BB%E5%87%BBGraphQL_phithon.pdf)

[GraphQL安全指北](https://www.freebuf.com/articles/web/184040.html)



[UWP GraphQL数据查询的实现](https://www.shuzhiduo.com/A/RnJW3BRoJq/)

[GraphQL Mutations](https://medium.com/@HurricaneJames/graphql-mutations-fb3ad5ae73c4)



[GraphQL NoSQL Injection Through JSON Types](http://www.petecorey.com/blog/2017/06/12/graphql-nosql-injection-through-json-types/)

[GraphQL Injection](https://godot.win/2017/GraphQl/)



[【安全记录】玩转GraphQL - DVGA靶场（上）](https://mp.weixin.qq.com/s/WoHEC50u7KACLLafZL5tww)

[【安全记录】玩转GraphQL - DVGA靶场（下）](https://mp.weixin.qq.com/s/mmJrE6uIBC-4ztr5PFZasA)



[[HITB CTF Singapore 2017]Blog](https://tsublogs.wordpress.com/2017/08/25/hitb-ctf-singapore-2017-web-512-blog/)

[[SECT 2017]Dark Market](https://godot.win/2017/GraphQl/#SEC-T-2017-Dark-Market)  |  [wp2](https://github.com/reznok/CTFWriteUps/tree/master/SEC-T_2017/DarkMarket)

[[Hack in Paris CTF 2019]Meet Your Doctor 1 2 3](https://swisskyrepo.github.io/HIP19-MeetYourDoctor/)  |  [wp2](https://jaimelightfoot.com/blog/hack-in-paris-2019-ctf-meet-your-doctor-graphql-challenge/)

[[VolgaCTF 2020]Library](https://www.gem-love.com/ctf/2198.html)
{{% /spoiler %}}

------

开学了，不摆烂从我做起
