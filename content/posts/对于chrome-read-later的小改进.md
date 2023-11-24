---
title: "对于chrome-read-later的小改进"
slug: "improvements-to-chrome-read-later"
description: "_(:_|∠)_"
date: 2023-08-28T21:46:56+08:00
categories: ["NOTES&SUMMARY"]
series: ["t00ls"]
tags: ["chrome-extensions"]
draft: false
toc: true
---

## 起因

在我的日常工作流中，read-later-list是必备项，一直以来 我都是用Notion Page来记录和同步每天待看的文章

![image-20230828215240130](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230828215240130.png)

~~（有很多的未读）（但是不重要）~~

由于标记了日期，也在回溯文章时有迹可循

可能就会有人问：那你为什么不用收藏夹or集锦？答案是删除时费工夫、不能根据时间检索、不能多端同步，同时 过于独立的页面也很难让人有想打开看的欲望，久而久之积攒的更多了

错误范例：

![image-20230828215845182](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230828215845182.png)

~~（更多 更多的未读）~~

所以Notion Page姑且成为了最佳选择，它被我固定到标签栏最上边（我是用竖栏标签页的），闲着没事就会进去消灭几个未读

然而时间长了另一个痛点逐渐涌现：把文章作为超链接加到Page中真的好麻烦！我需要复制标题、复制网页链接、进入Notion Page、添加To do list block、添加超链接内容，在我经历这一过程1年多以后心态逐渐崩溃（不是），我开始寻找有没有更简单的方式——哪怕，哪怕只有一个前端简陋的dashboard page，只要能够显示类似Notion Page的内容就可以——于是有了下面的探索

## 前置

### read-later扩展的痛点

无意中搜到了[read-later](https://github.com/willbchang/chrome-read-later)这个浏览器扩展，安装后可以在当前页面右键（或超链接右键）将网页加入待读，在扩展的popup页面即可显示

![image-20230828220655472](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230828220655472.png)

对一个网页可以显示favicon、标题、阅读进度（部分），还可以方便的删除&同步（利用扩展的同步性），看似已经是完美符合我的需求，但实际体验一段时间后我发现了这样几个通点

- 无法在更多端进行同步：比如我的ipad mini6，比如我的firefox
- popup页面和收藏夹、集锦一样，侵入性太强
- 不能根据时间做分类
- 不能自定义添加网页的标题

于是开始了改造之路

### read-later的程序逻辑

让我们先对这个扩展进行简易的代码分析，核心的代码其实只有background文件夹中的4个js

![image-20230828221530088](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230828221530088.png)

下面简单说一下每个js的主要功能：

1. actions.js

包含添加页面、删除页面、更新storage的函数

2. background.js

注册contextMenus，监听鼠标右键操作 并把需要的信息交给action.js中处理，也监听runtime的消息（更新、安装）

3. pageInfo.js

定义了PageInfo, PositionInfo和SelectionInfo（后两个均基于PageInfo），分别表示页面信息、当前浏览的位置信息、超链接的选择信息，向外导出initPageInfo和completeInfo用于添加页面到storage

4. request.js

主要是针对超链接文章，用于获取未实际访问的超链接的标题

## 改造

### 自定义添加网页的标题

*私信认为这是个相当必需的需求，但不知为何作者并未做到这一点？

background.js原本的监听是这样做的

```js
contextMenus.onClicked(async (selection, tab) => {
    selection.linkUrl
        ? await action.saveSelection(tab, selection)
        : await action.savePage()
})
```

对点击区域按是否存在selection.linkUrl来区分即将添加的是超链接还是当前网页，如果是超链接 会默认不存在标题、交给request.js获取`<title>`中的内容作为标题；如果是网页，直接从tab.title获取

——怎么看都没有我们插手的地方，我选择首先把saveXXX的判断改掉

```js
contextMenus.onClicked(async (selection, tab) => {
	if (typeof selection.linkUrl === "undefined" && typeof selection.selectionText === "undefined"){
        await action.savePage()
    } else {
        await action.saveSelection(tab, selection)  // use selectionText as page.title
    }
})
```

当右键划动选中文字区域时，虽然没有selection.linkUrl，但是selection.selectionText仍然存在，依靠二者共同判断是添加Page还是Selection

之后修改SelectionInfo类的方法

![image-20230828223713035](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230828223713035.png)

我们把“在选中内容处右键”作为添加Page的一种方式 但仍然归类到SelectionInfo中，最小限度的修改代码、实现需求

### 修改"done"浮现的条件

![image-20230828224113911](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230828224113911.png)

添加成功就会浮现这个Badge，但并不是所有添加都会浮现！强迫症不服

![image-20230828224231991](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230828224231991.png)

同时我也删去了很奇葩的一个设置：将当前页面添加read-list后自动关闭当前页面，很智障，很让人窒息

### 对接后端server

因为涉及到了数据展示和更新，本想用我最喜欢的streamlit，但streamlit不能像正常的后端一样接收api请求，无奈选用了Flask；因为之前CTF接触了太多的Flask，代码部分倒是不难（还有chatgpt强力驱动）

```python
import sqlite3

from flask import Flask, render_template, request, redirect, jsonify, abort
from datetime import datetime
import os

app = Flask(__name__)


@app.before_request
def auth():
    if request.headers.get('Authorization') == os.getenv('seckey'):
        return None
    else:
        abort(403)


@app.route('/api/add', methods=['POST'])
def add():
    data = request.get_json()
    # print(data)
    conn = sqlite3.connect('data.db')
    c = conn.cursor()
    c.execute("INSERT INTO reading_list (title, url) VALUES (?, ?)",
              (data['title'], data['url']))
    conn.commit()
    conn.close()
    return jsonify({'message': 'Data added successfully'})


@app.route('/dashboard')
def dashboard():
    conn = sqlite3.connect('data.db')
    cursor = conn.cursor()
    cursor.execute(
        "SELECT id, title, url, strftime('%Y-%m-%d %H:%M:%S', create_time, '+8 hour'), status FROM reading_list ORDER BY create_time DESC")
    rows = cursor.fetchall()
    conn.close()

    data = {}
    for row in rows:
        record = {
            'id': row[0],
            'status': row[4],
            'title': row[1],
            'url': row[2],
            'create_time': datetime.strptime(row[3], "%Y-%m-%d %H:%M:%S")
        }
        create_time = record['create_time'].strftime("%m%d")
        if create_time not in data:
            data[create_time] = []
        data[create_time].append(record)
        # print(data)

    return render_template('dashboard.html', data=data, seckey=request.headers.get('Authorization'))


@app.route('/sync/<int:id>/<int:status>')
def sync_status(id, status):
    conn = sqlite3.connect('data.db')
    cursor = conn.cursor()
    cursor.execute("UPDATE reading_list SET status=? WHERE id=?", (status, id))
    conn.commit()
    conn.close()

    return redirect('/dashboard')


@app.route('/delete/<int:id>')
def delete_record(id):
    conn = sqlite3.connect('data.db')
    cursor = conn.cursor()
    cursor.execute("DELETE FROM reading_list WHERE id=?", (id,))
    conn.commit()
    conn.close()

    return redirect('/dashboard')


if __name__ == '__main__':
    app.run(host="0.0.0.0", port=10393)
```

模板渲染部分，用jinja2真的非常丝滑

![image-20230828224746021](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230828224746021.png)

![image-20230828224829221](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230828224829221.png)

![image-20230828224841198](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230828224841198.png)

经过前面的修改，现在页面右键、对超链接右键、选择内容右键都可以在原扩展的基础上请求自己搭建的server，并存到data.db数据库中，访问/dashboard会有类似这样的显示效果

![image-20230828225025036](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230828225025036.png)

checkbox和代表delete的×都可以点击，联动后端的存储

虽然一看是让前端er闻者落泪的显示效果……但这已经是我和chatgpt大战了500个回合+自己修改了N次的结果了（）能用就行嗯嗯嗯

可能细心的师傅已经注意到代码中频繁出现的`seckey`了，惭愧的承认 这时另一个较为失败的地方，在脑内思考了多种鉴权、认证方式后，选择了代码最少的方式——Flask自带的装饰器`@app.before_request`，服务端通过设置环境变量seckey，整个app靠Authorization头来做一刀切的鉴权；不过纵使有种种缺点，它最大的有点还是代码少，之后会进行修改的（迫真）

## 仍存在的问题

- 过于简陋的前端
- 需要在前端加入随便写的便签功能
- 修改认证/鉴权逻辑，做到多端丝滑访问
- 适当修改readme ~~（虽然根本不会有第二个人用就是了）~~
