---
title: "关于Cookie的豆知识"
slug: "notes-about-cookie"
description: "会话固定攻击的实战利用可能"
date: 2024-03-07T20:47:43+08:00
categories: ["NOTES&SUMMARY"]
series: []
tags: ["js", "cookie"]
draft: false
toc: true
---

## 组成部分

| 属性     | 作用                                                         |
| -------- | ------------------------------------------------------------ |
| Domain   | 作用域，向上通配，子域可共享cookie                           |
| Path     | 作用的URL路径，更详细的path会更优先                          |
| Expires  | 过期时间，格式为GMT时间                                      |
| Max-Age  | 过期时间，单位为秒，比Expires更优先；未设置过期时间的cookie成为session-cookie 会在会话结束后删除 |
| Secure   | 只能通过https传输，该属性只能在https站点设置，此flag本身不会阻止对cookie中敏感信息的访问 |
| HttpOnly | 此类Cookie仅作用于服务器，会使document.cookie API无法访问，例如不需要对JavaScript可用的持久化服务器端会话的Cookie，此flag有助于防止XSS |
| SameSite | 分为Strict, Lax, None，一定程度防止跨站攻击                  |
| Name     | 键名                                                         |
| Value    | 键值                                                         |

键名键值均应不为空，服务端也应对内容做好合理的解析和处理，以免造成cookie冲突或解析问题，再结合针对子域名的XSS或子域接管漏洞可以打CSRF的组合拳

同一作用域下，cookie键值对有数量限制（在不同浏览器内核下有不同的表现），Chromium对于保留优先级有`Priority`作为flag的[提案](https://developer.chrome.com/docs/devtools/application/cookies?hl=zh-cn)（但未实装），因此最好在服务端就做出限制并保持读写的一致（cookie jar和document.cookie API），以免影响cookie的完整性

## 可选的键名前缀

`__Host`和`__Secure`是两个可选的键名前缀，会代替一部分cookie flag的功能

- `__Secure-`：必须与`secure`标志一同设置，同时必须应用于安全页面（HTTPS）
- `__Host-`：必须与`secure`标志一同设置，必须应用于安全页面（HTTPS），也禁止指定 domain 属性（也就不会发送给子域名），同时 path 属性的值必须为`/`

## 会话固定攻击与组合拳

会话固定的原理很好理解，但如何拿到用户的token很难搞，如果对cookie做了HttpOnly的限制 我们将无法利用XSS来获取token

usenixsecurity23-squarcina的paper展示了这样一种利用可控子域站点进行CORF token fixation attack的方式

![image-20240305110731190](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20240305110731190.png)

在图示场景中，位于site.tld的应用会对未登录用户分发session id，攻击者控制子域atk.site.tld，并访问应用得到这个session id，当受害者访问atk.site.tld时，攻击者可以指定Set-Cookie的session id和攻击者自己的一致、Domain为主域名site.tld，由于Domain属性向上通配，受害者可以带着这个session id登录site.tld，而登录成功后session id不变，攻击者可以用这个已知的session id接管受害者的登录状态

即使受害者已经处于登录状态，攻击者还可以结合Path属性，指定特定Path把受害者的cookie挤掉、让受害者强制登出，这样就又回到了会话固定的攻击场景

回到攻击成立之前，作为攻击者该如何控制atk.site.tld呢？如果session id就作为get的query参数则会简单很多，直接将链接发给受害者做fixation；如果session id在cookie中，HttpOnly没开启的话对atk.site.tld用XSS，开启的话就只能尝试拿下站点权限、或子域名接管了，部分场景下还可以尝试CRLF攻击控制Set-Cookie的响应头

### 缓解措施

如果对session id所在的cookie键名使用`__Host`将会抵抗图示fixation阶段的step2，即无法用子域的Domain覆盖主域名新产生的cookie，但对于需要在多个相关域名进行跳转的业务又造成了新的麻烦

一个通用性更强的解决方法是在应用端不要沿用登录状态改变的cookie，并增加session id刷新的频率