---
title: "近期debug趣闻两则"
slug: "smth-about-debug"
description: "排错两小时，改错五分钟"
date: 2023-03-14T12:38:49+08:00
categories: []
series: []
tags: []
draft: false
toc: true
---

## 代理失效

科学上网一直是永恒的话题，为了保证学习过程中能流畅地访问google，之前我的方案是 学习的时候clash（不开启System Proxy）+proxifier（msedge.exe进程的流量全部走127.0.0.1:7890），需要刷b站或大量浏览国内网站时在proxifier中切换为全部Direct的rule

![image-20230314125208813](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230314125208813.png)

这样的方式比较符合我的使用习惯，我也不希望clash污染我的系统代理

然而大概从上周开始情况有变，在这样的策略下：

- 所有需要翻墙的网站全部无法打开、国内网站正常
- proxifier内server check正常，connections中需要翻墙的网站全部是517bytes send, 0 bytes received，其余网站正常

![image-20230314130544070](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230314130544070.png)

- clash Request Logs正常，有proxifier过代理的记录

![image-20230314130524562](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230314130524562.png)

然后开始排错，首先犯罪嫌疑人就是DNS，怀疑是虽然挂了代理但域名解析仍旧用的本地服务或缓存

![image-20230314130853400](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230314130853400.png)

proxifier有单独的Name Resolution设置，默认就是上图的Detect DNS settings automatically，我手动切换Resolve hostnames through proxy结果反而出错了

![image-20230314131107649](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230314131107649.png)

![image-20230314131128477](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230314131128477.png)

在我疑惑是不是错怪DNS的时候，刚把它切换回去瞬间正常了……成功恢复正常代理的情况

![image-20230314131240753](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230314131240753.png)

## hugo渲染过慢

差不多一两个月以前，我的博客每次push内容之后都build的特别特别慢，以前只要几十秒 现在成了动辄五六分钟

![image-20230314131613672](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230314131613672.png)

怎么会事呢？看看详情

![image-20230314131734902](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230314131734902.png)

![image-20230314131815397](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230314131815397.png)

开始排错……

1. 解决3个警告

警告的内容大致是要求从Node12升级到Node16，由于这个版本其实是actions执行时指定的，所以我们直接在actions的配置文件里升级对应的版本

![image-20230314132404309](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230314132404309.png)

还是很慢，继续找问题

2. 更新主题

既然是渲染问题，我考虑可能是主题的html相关文件存在冗余/小bug，再一看主题对应的github仓库 果然更新了几次，那这下不得不更新了

```bash
cd themes\hugo-theme-tokiwa
git pull
```

emm，还是很慢

3. 增加action cache

渲染，渲染，有没有缓存肯定影响很大，试图通过添加缓存来解决问题

![image-20230314132830101](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230314132830101.png)

（*注：这里也应该用@v3的最新版本）

emmmmmmmmm，还是很慢

4. 罪魁祸首

hugo一直以渲染速度快跟hexo正面刚，始终不明白到底是哪里出错了，直到我看了一下生成静态文件的页面大小……

是我冒犯了，我其中一篇博客光md就接近400kb，渲染成html有接近3M……把这篇博客删掉，瞬间渲染完成

![image-20230314134235268](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230314134235268.png)

![image-20230314134216556](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230314134216556.png)

但这个问题就很难解决了……那篇博客之所以大是因为有特别长的js代码，如果把代码都删了跟删掉整篇文章也没多大区别了

诶？删掉整篇博客？好方略！
