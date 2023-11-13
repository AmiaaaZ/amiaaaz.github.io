---
title: "对于chrome-read-later的小改进Ⅱ"
slug: "improvements-to-chrome-read-later-02"
description: "chrome插件开发锐意学习中"
date: 2023-11-13T21:22:23+08:00
categories: ["NOTES&SUMMARY"]
series: ["t00ls"]
tags: ["chrome-extensions"]
draft: false
toc: true
---

## 前置

毫无疑问，read-later已经成为我装机必备的浏览器插件了，虽然最后真正被阅读的文章只占很少的比例，但这极大的缓解了我的文章囤积症！原项目很好，但有一些定制化的需求还是要自己实现

废话就不多说了，[上一篇文章](https://amiaaaz.github.io/2023/08/28/improvements-to-chrome-read-later/)中对原项目改进了以下三点：

- 支持selectionText作为待读文章的标题
- 添加成功后浮现"done"
- 对接后端server，有独立dashboard页面进行管理

使用了两个月后又发现了一些新问题，于是再次进行改进（同时也学了不少chrome插件开发的技巧www

## 修正favicon

chrome插件对于v3版本提供了内置的favicon获取方式-> [doc](https://developer.chrome.com/docs/extensions/mv3/favicon/)

1. 在manifest文件中添加permission和web_accessible_resources

```js
{
  // omit
  "permissions": ["favicon"],
  "web_accessible_resources": [
    {
      "resources": ["_favicon/*"],
      "matches": ["<all_urls>"],
      "extension_ids": ["*"]
    }
  ]
}
```

2. 在new page时加入favicon

```js
get favIconUrl () {
	return `chrome-extension://${chrome.runtime.id}/_favicon/?pageUrl=${this.url}&size=32`
}
```

但需要注意这个API并不是百分百返回favicon，有一定概率仍然返回不了，此问题目前无解（搜了关键词，也没有很好的解决方案，猜测问题成因和同源策略和缓存有关

原项目中也使用了这样的获取方式，但`${this.url}`的部分是手动获取了url的origin、manifest也没有设置全面，实测不如直接用url的效果好、获取不到favicon的概率更大

## 优化popup页面

原项目中，popup默认reading list页面是对sync storage的展示，文章被点击后会有进程记录阅读情况，同时被默认列表和sync storage删除，历史reading list则是对local storage的展示，不会删除点击过的文章

....emmmm 是不是有哪里逻辑不对？为什么点一下列表内的文章就要被视作已读呢？断断续续几次才能看完也是很常见的情况，我们希望添加手动标记已读的功能，不要点击就删 ~~（什么赛博版阅后即焚~~

### 删去progress

略，删去所有相关代码

### 添加手动已读

这算是个小难点，因为涉及到了chrome插件开发中的通信问题

chrome插件中，popup页面（点击扩展图标会显示的页面）、background（后台工作的服务进程 可以发起网络请求）、content（和前台访问的页面同级）是无法直接互相通信的，需要借助chrome API

原项目中针对popup页面中open page是这样的：

- popup/action.js#open：向background发起`sendMessage`
- background/action.js#openPage：监听上面的消息，当tab加载完毕时向content发起`sendMessage`
- content/content.js：监听上面的消息，对点击过的文章更新位置信息用作progress显示

我们已经无需progress、但仍可以保留`onMessage.addListener`，将原本内容替换为针对web页面注入checkbox的js脚本，使用`sendResponse`处理checkbox被勾选后的情况，同时修改local storage中文章的状态，删去sync storage中对应的条目；修改如下：

- background/pageInfo.js

![image-20231113225457916](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20231113225457916.png)

- background/action.js

![image-20231113224914869](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20231113224914869.png)

- content/content.js

![image-20231113225315866](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20231113225315866.png)

在background中我复用了原项目中的异常处理代码，因为在background与content通信过程中，关于add checkbox的信息一定被发送，但read over的response不一定发送，如果此时再刷新页面，就会报no receive response的错误，而这其实是预期内的行为

“赛博阅后即焚”则很好处理，直接删去下面的判断即可

![image-20231113225935004](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20231113225935004.png)

## 未完待续

经过以上修改，现在popup默认列表点击文章后会在新开的页面右上角添加checkbox用于更新阅读状态，mark之后默认列表将删除该条记录，历史列表中则仍然存在

![image-20231113230237755](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20231113230237755.png)

之后的改进计划：

- 支持云端存储（后端server的data.db）和本地进行同步，新装插件也可以拉取远端的所有记录
- 优化dashboard的ui，现在还是太简陋了
