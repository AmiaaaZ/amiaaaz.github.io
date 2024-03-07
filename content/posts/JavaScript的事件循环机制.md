---
title: "JavaScript的事件循环机制"
slug: "event-loop-in-javascript"
description: "单线程的异步"
date: 2024-03-07T23:38:41+08:00
categories: ["NOTES&SUMMARY"]
series: []
tags: ["js", "Event Loop"]
draft: false
toc: true
---

JavaScript的应用场景天然具有高交互性，同时为了平衡效率和减少并发问题，js以单线程运行，任务的执行则使用Event Loop事件循环机制，具体实现根据Runtime不同而不同

## 浏览器中的Event Loop

```js
console.log('Start');

setTimeout(() => {
  console.log('Async task 1');
}, 0);

setTimeout(() => {
  console.log('Async task 2');
}, 0);

Promise.resolve().then(() => {
  console.log('Microtask 1');
});

Promise.resolve().then(() => {
  console.log('Microtask 2');
});

console.log('End');
```

```
# 执行结果
Start
End
Microtask 1
Microtask 2
Async task 1
Async task 2
```

以上述代码为例，console.log函数是synchronous同步任务，会按顺序在主线程上执行，而夹在中间的setTimeout和Promise均属于asynchronous异步任务，不进入主线程，而是位于task queue任务队列，只有当任务队列通知主线程可以执行了，才会排到执行栈中等待执行（而不是立刻执行，需要等待执行栈空了或是轮到它）

```js
console.log('Start');

setTimeout(() => {
  console.log('Async task 1');
}, 10000);

Promise.resolve().then(() => {
  console.log('Microtask 1');
});

console.log('End');
```

异步任务之间亦有差距，Promise的优先级高于setTimeout（原因稍后再解释），所以上述代码执行结果是Start-> End-> Micro-> Async，即使将setTimeout改为0，也只意味着可以在当前执行栈做完后最早执行，仍然是最后输出结果

主线程从任务队列中读取事件是个循环的过程，这个机制就叫Event Loop；在js代码编写过程中，如果同步代码和异步代码混着写，就可能出现意料之外的执行结果：最初定义的变量在if循环中不断+1结果最终输出的还是初始值

### 宏任务&微任务

回到开头实例代码中的Micro&Async，Event Loop会将任务队列分为Micro Task Queue宏任务队列和Micro Task Queue微任务队列，Promise就属于宏任务，setTimeout就属于微任务

- 微任务
  - Promise中的then, catch, finally
  - MutationObserver：监视DOM变化的API
  - Object.observe(deprecated)：监视对象的变化
  - Process.nextTick：Node中的API
- 宏任务
  - DOM操作
  - 用户交互操作
  - 网络请求
  - 定时器相关的setTimeout, setInterval
  - script中的代码

前面说过，主线程对任务队列的读取时是循环进行的，先同步再异步，而每一次循环都是先从宏任务开始，微任务结束，而宏任务、微任务中又可以区分异步与同步

宏任务中的同步代码优先进入主线程，按顺序执行完毕，当主线程空闲时执行同层的微任务；首层Event Loop结束后进入第二层事件循环，如此继续looplooploop

举例：

![image-20240307213004050](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20240307213004050.png)

![image-20240307213855122](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20240307213855122.png)

需要注意的是，Promise中只有then, catch, finally是异步执行代码，Promise中传入的回调函数属于同步执行代码

## Node中的Event Loop

注意：在执行结果上，除Node特有的process对象及相关API外，基本与浏览器环境下没有区别。

### 六阶段

与浏览器区别最大的地方在于Node事件循环不是一层一层的，而是按顺序在6阶段中循环

- timers: 执行setTimeout和setInterval中到期的callback
- pending callback: 上一轮循环中被推迟的callback，如某些系统操作的回调（比如TCP错误）
- idle, prepare: 仅在内部使用
- poll: 检索新的I/O事件，执行I/O有关的回调（几乎所有回调都不包括关闭回调、被计时器调度的回调以及`setImmediate()`），node会在此处适时阻塞（重要）
- check: `setImmediate()`的callback
- close callbacks: 一些close事件的callback，例如`socket.on('close', ...)`

每一个阶段都是FIFO的队列，当事件循环进入给定阶段后将执行该阶段的操作，然后执行该阶段队列中的回调，直到队列尽头或达到回调限制，然后进入下一个阶段，以此类推

### timers

timers中回调的实际执行时间会和定时器的间隔有一定出入，因为虽然定时器回调函数尽可能快的安排被执行，但操作系统的调度或其它回调函数的运行可能会推迟这个时间，poll阶段会对此进行控制

```js
const fs = require('node:fs');
function someAsyncOperation(callback) {
  // Assume this takes 95ms to complete
  fs.readFile('/path/to/file', callback);
}
const timeoutScheduled = Date.now();
setTimeout(() => {
  const delay = Date.now() - timeoutScheduled;
  console.log(`${delay}ms have passed since I was scheduled`);
}, 100);
// do someAsyncOperation which takes 95 ms to complete
someAsyncOperation(() => {
  const startCallback = Date.now();
  // do something that will take 10ms...
  while (Date.now() - startCallback < 10) {
    // do nothing
  }
});
```

![image-20240307231905230](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20240307231905230.png)

第7行的setTimeout定时器设定是100ms，但实际运行结果在100~120ms之间波动

### poll

没有`setImmediate()`的话poll阶段将是最终阶段，会控制timers何时被执行，它会计算为了I/O该阻塞多久，并依次处理此队列中的事件

如果当事件循环进入到poll阶段却根本没有计时器，将会按以下规则对号入座：

- poll队列非空：事件循环将进行迭代队列中的回调函数并同步执行它们，直到队列被清空或者达到系统限制为止
- poll队列为空：
  - 如果有`setImmediate()`：事件循环将结束poll阶段，进入check阶段
  - 没有`setImmediate()`：等待回调加入队列，一旦加入就立刻执行

### setImmediate vs setTimeout

当它们不在一个I/O周期中时，两个计时器先后不确定

```js
setTimeout(() => {
    console.log('setTimeout');
},0)
setImmediate(() => {
    console.log('setImmediate');
})
```

以上代码运行多次会有不同的输出结果

当两者被放入同一个I/O循环中时，`setImmediate()`总是优先调用

```js
const fs = require('fs');
fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log('timeout');
  }, 0);
  setImmediate(() => {
    console.log('immediate');
  });
});
```

----

参考链接：

[從「為什麼不能用這個函式」談執行環境（runtime）](https://blog.huli.tw/2022/02/09/javascript-runtime/)

[JavaScript 运行机制详解：再谈Event Loop](https://www.ruanyifeng.com/blog/2014/10/event-loop.html)

[Event Loop到底是个啥？](https://zhuanlan.zhihu.com/p/388710842)

doc: [The Node.js Event Loop](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick)

