---
title: "Electron安全学习笔记·上"
slug: "electron-study-notes-01"
description: "Electron架构、使用例、inspect abusing"
date: 2023-08-12T23:29:47+08:00
categories: ["NOTES&SUMMARY"]
series: []
tags: ["Electron"]
draft: false
toc: true
---

## 什么是Electron

> Chromium+NodeJS≈Electron!

简单来说，就是用web前端开发的技术栈来写跨平台的桌面端应用，运行环境是Chromium，NodeJS负责调用系统API、与操作系统底层进行交互，实现了仅靠前端无法实现的功能

虽然降低了开发门槛 很好上手，但缺点也很明显——体积很大、占用很大，每一个Electron应用就带了一个Chromium内核，代码几十k 引擎几百M，还会释放用户的数据到AppData，一旦进程启动.... 想想浏览器恐怖的内存占用吧

有同样理念的还有tauri，使用的是Rust+原生webview，但Rust做后端就能明显看出二者的差距了，tauri是向传统后端项目中添加webview做GUI，而Electron则是Javascript走天下

*讲正事之前，来点前端开发的笑话

![image-20230805165307463](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230805165307463.png)

### Electrun Runtime的设想

既然每一个Electron应用都需要Chromium和NodeJS 那为什么不把这两个打包成类似JDK的Electron Runtime呢？

我们先讨论为什么不行；Electron和Node的版本滚动更新都很勤快，更别提依赖的前端库更是一个版本更新过后就要担心是否会造成冲突/报错，因此如果一个版本对应一个运行时 可能最终效果和现行方案是一样的

但这不耽误我们设想 如果真要做成Runtime，开发流程和使用环境会做出怎样的改变；以Runtime开发者的角度来说，我们需要为Runtime使用者（也就是Electron应用开发者）提供 **打包工具**，Runtime使用者可以使用打包工具将js等静态文件和 **执行程序**、 **卸载程序**一起打包为资源文件，这个资源文件会被封装到安装程序中，在修改图标、签名、文件名等资源信息后就得到了 **安装程序**

这个安装程序在用户电脑上运行时会自动检测是否安装了Electron Runtime，如果未安装会自动下载发行版 并记录到注册表上，之后在用户指定的安装目录下释放资源文件，向注册表中写下 **卸载程序**的位置（使用户可以通过控制面板卸载程序），按用户需求创建开始菜单等等，这样程序就安装完成了

 **执行程序**在运行时会检测用户的注册表，找到Electron Runtime的位置，并把当前入口程序作为参数传给Runtime

 **卸载程序**会删除安装目录下的文件、删除注册表的信息

## 调试

### 开启调试

我们可以在以Chromium为内核的任意浏览器中直接调试Electron应用，比如纯净的Chromium内核

![image-20230804173904739](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230804173904739.png)

或混沌中立的edge

![image-20230804173951759](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230804173951759.png)

一般情况下可以在运行程序时加上这样的参数来开启debug

```
--remote-debugging-port=9222
```

![image-20230805145641055](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230805145641055.png)

但很显然这个flag非常醒目，很多开发者会在初始化阶段对其进行检测 并终止程序运行

![image-20230804174325497](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230804174325497.png)

虽然但是 家贼难防啊——NodeJS可以仍可开启动态调试，直接锁定PID开启调试

```
kill -SIGUSR [pid]
```

windows可以在NodeJS中调用`_debugProcess`

```js
process._debugProcess(pid)
```

![image-20230804175313142](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230804175313142.png)

![image-20230810124339767](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230810124339767.png)

一旦允许调试 那就意味着打开了潘多拉的魔盒，debugger对于NodeJS的执行环境有完全的权限（没有什么上下文隔离之类的阻拦），如果这一调试功能被滥用 就可直接导致RCE（详情见后）

### 源码

Electron应用的js源码一般就位于程序目录的resources文件夹内，会有一份.asar格式的打包文件，我们可以用[asar](https://github.com/electron/asar)这个官方工具进行解压

```bash
asar extract app.asar asar/
```

![image-20230804181019523](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230804181019523.png)

可以发现.asar的内容实际是JSON+自定义编码，非常容易从中得到源码

*因此审计Electron项目约等于白盒审计x

### 源码加密

由于Electron官方没有在源码保护方面给出很好的解决办法，出现了开源的加密方案-> [electron-asar-encrypt-demo](https://github.com/toyobayashi/electron-asar-encrypt-demo)

## Electron的架构

可能由于Electron和Chromium深度绑定的原因，Electron也仿照了Chromium的多进程模式，即：主进程Main Process作为核心进程，一个窗口一个独立的渲染进程Renderer Process；主进程和渲染进程之间采取IPC通信

![image-20230809164606231](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230809164606231.png)

以Typora举例，打开一个Typora窗口会启动4个进程（所有Electron应用的表现都一样），用flag区分不同进程的职责

![image-20230808170518545](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230808170518545.png)

- 无flag：主进程
- type=utility：效率进程，可用于托管不受信的服务、CPU密集型服务或容易崩溃的组件，可以通过MessagePorts与渲染进程建立通信，当需要从主进程派生新的子进程时 Electron会优先选择效率进程API而不是NodeJS child_process.fork API
- type=renderer：渲染进程，再打开第二个Typora窗口会新增一个type=renderer的进程，其余进程不变；第三个进程

### 主进程&渲染进程

主进程的主要是使用`BrowserWindow`模块创建和管理窗口，一个实例对应一个app窗口，使用单独的渲染进程加载其中的网页，在主进程中可以用window的`webContents`对象与网页内容进行交互；由于`BrowserWindow`模块是一个`EventEmitter`，所以我们也可以处理程序的执行流、控制app的生命周期，当一个`BrowserWindow`实例被销毁时 对应的渲染器进程也会终止

如果想在`BrowserWindow`中集成第三方web内容（web-embeds），可以用`<iframe>`, `<webview>`（不建议）, `BrowserViews`；此外，渲染进程也会为web-embeds对象而<u>单独服务</u>

默认情况下**nodeIntegration**关闭，渲染器无权访问直接NodeJS的API，我们编写的用于交互的页面需要遵守一般网页开发的规范，使用npm包需要用和web开发时一样的打包工具（比如webpack或parcel）

### preload.js

预加载脚本会在渲染进程中会被优先于网页内容加载，虽然处于渲染进程中 但可以访问部分Polyfill形式实现的NodeJS API，只能载入部分模块和对象

从Electron12之后 默认情况下**contextIsolation**开启，preload.js和renderer process、Electron内部代码和renderer process之间都存在隔离，即使preload.js和浏览器/渲染进程共享全局的`window`对象，但在preload.js中对`window`对象做出的改动无法附加到真正的页面上

为安全考虑我们必须使用`contextBridge`以及IPC模块来进行交互，在preload中暴露API、在页面中调用API

```js
// preload.js
const {contextBridge} = require('electron')
contextBridge.exposeInMainWorld('myAPI', {
	doAThing:()=> {}
})
```

```js
// render.js in index.html (in renderer process)
window.myAPI.doAThing()
```

### sandbox

从Electron20开始 渲染进程默认启用**sandbox**，沙盒化的渲染器不会有NodeJS环境

- 当nodeIntegration启用时沙盒会被禁用
- 在`app.whenReady`前可以调用`app.enableSandbox()`强制沙盒化所有渲染器
- 可以用`--no-sandbox`来完全禁用Chromium沙盒

### IPC通信

主进程使用`ipcMain`，渲染进程使用`ipcRender`，preload.js通过`contextBridge.exposeInMainWorld()`向渲染进程暴露相关API

出于安全考虑，我们不会在preload.js中暴露整个`ipcRenderer.send`，而是有约束、限定功能

- 以渲染进程-> 主进程的通信举例代码：

ipcMain.on, ipcRenderer.send

```js
// main.js
const { app, BrowserWindow, ipcMain } = require('electron')
const path = require('path')

function createWindow () {
  const mainWindow = new BrowserWindow({
    webPreferences: {
      preload: path.join(__dirname, 'preload.js')
    }
  })

  ipcMain.on('set-title', (event, title) => {
    const webContents = event.sender
    const win = BrowserWindow.fromWebContents(webContents)
    win.setTitle(title)
  })

  mainWindow.loadFile('index.html')
}

app.whenReady().then(() => {
  createWindow()

  app.on('activate', function () {
    if (BrowserWindow.getAllWindows().length === 0) createWindow()
  })
})

app.on('window-all-closed', function () {
  if (process.platform !== 'darwin') app.quit()
})

```

```js
// preload.js
const { contextBridge, ipcRenderer } = require('electron')

contextBridge.exposeInMainWorld('electronAPI', {
  setTitle: (title) => ipcRenderer.send('set-title', title)
})
```

```html
<!--index.html-->
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <!-- https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP -->
    <meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'self'">
    <title>Hello World!</title>
  </head>
  <body>
    Title: <input id="title"/>
    <button id="btn" type="button">Set</button>
    <script src="./renderer.js"></script>
  </body>
</html>
```

```js
// renderer.js
const setButton = document.getElementById('btn')
const titleInput = document.getElementById('title')

setButton.addEventListener('click', () => {
  const title = titleInput.value
  window.electronAPI.setTitle(title)
})
```

- 渲染器和主进程可以双向通信，使用ipcMain.handle和ipcRenderer.invoke
- 主进程到渲染进程单向通信可以使用webContents.send发送消息，preload.js用ipcRenderer.on处理消息
- 不同渲染进程间的通信使用类似postMessgae的MessagePorts

## 使用例

1. 创建项目

```bash
npm init	# entry-point应为main.js(做为主进程), author和description为必填项
$env:ELECTRON_MIRROR="https://npm.taobao.org/mirrors/electron/"
npm install --save-dev electron
```

会自动生成项目的package.json文件，可在`scripts`字段中添加

```json
"scripts": {
    "start": "electron ."
}
```

2. 编写main.js

main.js即为程序的主进程，可以创建浏览窗口、加载html、设置preload.js，也将处理Electron初始化、窗口事件等，可以把所有主进程都写在这里 也可以拆分成几个文件 然后用require导入

```js
// main.js
const {app, BrowserWindow} = require('electron')
const path = require('path')

const createWindow = () => {
    const mainWindow = new BrowserWindow({
        width: 800,
        height: 600,
        webPreferences: {
            preload: path.join(__dirname, 'preload.js'),
            sandbox: false,
            nodeIntegration: true,
            contextIsolation: false
        }
    })
    // 创建渲染进程
    mainWindow.loadFile('index.html')
    // 打开开发工具
    // mainWindow.webContents.openDevTools()
}

// 管理窗口的生命周期
app.whenReady().then(() => {
    createWindow()
    app.on('activate', () => {
        // 对macOS 如果没有可用窗口 打开新窗口
        if (BrowserWindow.getAllWindows().length === 0) createWindow()
    })
})

// 除macOS外，当所有窗口都被关闭的时候退出程序
app.on('window-all-closed', () => {
    if (process.platform !== 'darwin') app.quit()
})

```

3. 编写preload.js

在主进程通过Node的全局`process`对象访问进程相关信息很简单，但我们不能直接在主进程中编写DOM，因为它和渲染进程是不同的上下文，这时就需要preload.js，它会在渲染进程加载之前加载，并有权访问两个渲染器全局（window和document）和NodeJS的API

```js
// preload.js
window.addEventListener('DOMContentLoaded', () => {
  const replaceText = (selector, text) => {
    const element = document.getElementById(selector)
    if (element) element.innerText = text
  }

  for (const type of ['chrome', 'node', 'electron']) {
    replaceText(`${type}-version`, process.versions[type])	// 访问NodeJS的process对象
  }
})

```

4. 编写网页（HTML）

可以正常使用任何前端常用的技巧——包括meta标签的CSP设置

```html
<!--index.html-->

<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <!-- https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP -->
    <meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'self'">
    <title>你好!</title>
  </head>
  <body>
    <h1>Hello Electron!</h1>
    正在使用 Node.js <span id="node-version"></span>,
    Chromium <span id="chrome-version"></span>,
    和 Electron <span id="electron-version"></span>.
  </body>
</html>
```

5. 运行

因为前面已经在package.json中添加了运行命令，直接用`npm start`即可执行

![image-20230807123403980](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230807123403980.png)

6. 打包分发

```bash
npm install --save-dev @electron-forge/cli
npx electron-forge import
npm run make
```

在./out/下可以看到打包好的二进制文件

## Electron inspect abusing

前面开了个小头，这里来详细说说调试这一功能存在的安全隐患

任意Electron应用都可以开启调试，期间会有类似这样的socket通信

```
DevTools listening on ws://127.0.0.1:9222/devtools/browser/80cd01d9-9426-4c79-a9ee-8dea8987c336
```

而这样的websocket连接我们自己就可以做到

![image-20230810123554023](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230810123554023.png)

2019年就有大佬完成了实现debug通讯协议的工具-> [cefdebug](https://github.com/taviso/cefdebug)，可以自动查找当前开启debug的Electron应用并像DevTools窗口一样执行命令

![image-20230810151209672](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230810151209672.png)

设想这样的攻击场景：受害者本地Electron应用开启调试，通过钓鱼手段让受害者运行cefdebug应用并自动连接ws，达成RCE，同时让js开启新的socket连接做到回显

但.... 我们怎么能保证受害者本地的Electron应用能开启调试呢？正常的开发者都不会在启动时留下这样的flag

我们考虑直接pkg打包一个带Node环境的恶意文件，在其中用`process._debugProcess`开启Electron应用的调试，再接上前面的设想，达成RCE

### Electron_shell

新鲜热乎的利用工具-> [Electron_shell](https://github.com/djerryz/electron_shell)

![image-20230810112949105](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230810112949105.png)

#### 源码分析

主要文件是\_inspect.js, \_inspect\_repl.js和\_inspect\_clients.js，其实这三个代码就是从Node源码中摘出来的

![image-20230810165543995](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230810165543995.png)

作者在_inspect.js的`startInspect`函数中加入了寻找pid的逻辑，并把原来从命令行读入的debug参数改为了手动指定pid

![image-20230810170126380](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230810170126380.png)

其中的`writeTofile`函数是向系统临时目录中写入RCE和回显的js payload

![image-20230810170659210](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230810170659210.png)

而执行payload被放在了inspect_repl.js中

![image-20230810170758498](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20230810170758498.png)

最终以inspect.js的`startInspect`作为入口，用pkg打包NodeJS运行环境

```shell
pkg.cmd -t node16-win-x64 cli.js
```

经过分析我们发现，cli.exe实际就是把node-cli内置的和inspect有关的部分拆了出来，并加入恶意的部分组合打包而成的

------

{{% spoiler "以下是本文中涉及到的 和我学习时看过的所有文章的链接 每日感谢互联网的丰富资源（" %}}

[如何评价 Electron？](https://www.zhihu.com/question/52761908/answer/2779839142)

[为什么 electron 不做成独立的 runtime？ - liulun的回答 - 知乎](https://www.zhihu.com/question/505356140/answer/2268992442)

[注入任意代码到运行中的 Electron 应用](https://mp.weixin.qq.com/s/faYOXbn_4fNotEa8gs0CVw)

[向Typora学习electron安全攻防](https://mp.weixin.qq.com/s/Ggx34AqhzsbBLkecv5RNJw)

[深入理解Electron（一）Electron架构介绍](https://zhuanlan.zhihu.com/p/604169213)

[quick-start](https://www.electronjs.org/zh/docs/latest/tutorial/quick-start)

[cefdebug](https://github.com/taviso/cefdebug)  |  [Electron_shell](https://github.com/djerryz/electron_shell)

{{% /spoiler %}}