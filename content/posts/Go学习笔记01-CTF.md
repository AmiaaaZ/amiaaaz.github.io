---
title: "Go学习笔记Ⅰ"
slug: "go-study-notes-01"
description: "ctf中常见的Go安全问题，持续更新"
date: 2022-06-23T18:41:10+08:00
categories: ["NOTES&SUMMARY", "LTS"]
series: ["Go学习笔记"]
tags: ["Go"]
draft: false
toc: true
---

2022.06.23更新第二次

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

## 整数溢出

### [羊城杯2021]Checkin_Go

uint32的范围是0~4294967295

```
u1, err1 := strconv.ParseUint(nowMoney,10,32)
u2, err2 := strconv.ParseUint(addMoney,10,32)
....
newMoney = uint32(u1) + uint32(u2)
```

flag price是uint32，以admin身份（伪造cookie）加到溢出，变小，就可以买了

参考：[wp](https://yyz9.cn/2021/09/12/%E7%BE%8A%E5%9F%8E%E6%9D%AF2021-checkin_go%E8%AF%A6%E7%BB%86%E9%A2%98%E8%A7%A3/)

### [TStar 2022]不眠之夜

有购买的地方，付钱时的amount用了ParseInt，只有33和34满足条件

```
int8 : -128 to 127
int16 : -32768 to 32767
```

1000\*33或1000*34正好\> 32767 实现溢出，再大就库存不足了。

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

## http.Dir&.IsDir的技巧

### [JMCTF 2021]GoOSS

```go
package main

import (
   "bytes"
   "crypto/md5"
   "encoding/hex"
   "github.com/gin-gonic/gin"
   "io"
   "io/ioutil"
   "net/http"
   "os"
   "strings"
   "time"
)

type File struct {
   Content string `json:"content" binding:"required"`
   Name    string `json:"name" binding:"required"`
}
type Url struct {
   Url string `json:"url" binding:"required"`
}

func md5sum(data string) string {
   s := md5.Sum([]byte(data))
   return hex.EncodeToString(s[:])
}

func fileMidderware(c *gin.Context) { // 作中间件
   fileSystem := http.Dir("./files/")
   if c.Request.URL.String() == "/" {
      c.Next()
      return
   }
   f, err := fileSystem.Open(c.Request.URL.String()) // 打开与url同名的文件 当参数为`/xxx/...`表示打开fileSystem对应的文件夹自身 其中`xxx`部分任意
   if f == nil {
      c.Next()
   }
   //
   if err != nil {
      c.Next()
      return
   }
   defer f.Close()
   fi, err := f.Stat()
   if err != nil {
      c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
      return
   }

   if fi.IsDir() {

      if !strings.HasSuffix(c.Request.URL.String(), "/") { //
         c.Redirect(302, c.Request.URL.String()+"/") // 跳转到80端口的php服务
      } else {
         files := make([]string, 0)
         l, _ := f.Readdir(0)
         for _, i := range l {
            files = append(files, i.Name())
         }

         c.JSON(http.StatusOK, gin.H{
            "files": files,
         })
      }

   } else {
      data, _ := ioutil.ReadAll(f)
      c.Header("content-disposition", `attachment; filename=`+fi.Name())
      c.Data(200, "text/plain", data)
   }

}

func uploadController(c *gin.Context) {
   var file File
   if err := c.ShouldBindJSON(&file); err != nil {
      c.JSON(500, gin.H{"msg": err})
      return
   }

   dir := md5sum(file.Name)

   _, err := http.Dir("./files").Open(dir)
   if err != nil {
      e := os.Mkdir("./files/"+dir, os.ModePerm)
      _, _ = http.Dir("./files").Open(dir)
      if e != nil {
         c.JSON(http.StatusInternalServerError, gin.H{"error": e.Error()})
         return

      }

   }
   filename := md5sum(file.Content)
   path := "./files/" + dir + "/" + filename
   err = ioutil.WriteFile(path, []byte(file.Content), os.ModePerm)
   if err != nil {
      c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
      return
   }

   c.JSON(200, gin.H{
      "message": "file upload succ, path: " + dir + "/" + filename,
   })
}
func vulController(c *gin.Context) {

   var url Url
   if err := c.ShouldBindJSON(&url); err != nil {
      c.JSON(500, gin.H{"msg": err})
      return
   }

   if !strings.HasPrefix(url.Url, "http://127.0.0.1:1234/") { // 必须以这个开头
      c.JSON(403, gin.H{"msg": "url forbidden"})
      return
   }
   client := &http.Client{Timeout: 2 * time.Second}

   resp, err := client.Get(url.Url) // 访问url
   if err != nil {
      c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
      return
   }
   defer resp.Body.Close()
   var buffer [512]byte
   result := bytes.NewBuffer(nil)
   for {
      n, err := resp.Body.Read(buffer[0:])
      result.Write(buffer[0:n])
      if err != nil && err == io.EOF {

         break
      } else if err != nil {
         c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
         return
      }
   }
   c.JSON(http.StatusOK, gin.H{"data": result.String()})
}
func main() {
   r := gin.Default()
   r.Use(fileMidderware)
   r.POST("/vul", vulController)
   r.POST("/upload", uploadController)
   r.GET("/", func(c *gin.Context) {
      c.JSON(200, gin.H{
         "message": "pong",
      })
   })
   _ = r.Run(":1234") // listen and serve on 0.0.0.0:8080
}
```

```php
<?php
// php in localhost port 80
readfile($_GET['file']);    // 任意文件读取
?>
```

有两个端口的服务，默认是1234的Go，我们利用trick跳转到80端口的php 任意文件读取

## SSTI

[Doc: template](https://pkg.go.dev/text/template)  |  [Go SSTI初探](https://tyskill.github.io/posts/gossti/)  |  [Golang中的SSTI](https://blog.thekingofduck.com/post/ssti-in-golnag/)

语法类似jinja2，大差不差

- 基础语法

1. 模板的占位符为`{{语法}}`，这里的`语法`官方称之为`Action`，其内部不能有换行，但可以写注释，注释里可以有换行。
2. 特殊的`Action`：`{{.}}`，中间的点表示当前作用域的当前对象，类似`JAVA`中的`this`关键字。
3. `Action`中支持定义变量，命名以`$`开头,如`$var = "test"`，有一个比较特殊的变量`$`，代表全局作用域的全局变量，即在调用模板引擎的`Execute()`方法时定义的值，如`{{$}}`在上面的题目中获取到的值就是`Intigriti`
4. `Action`中内置了一些基础语法,如常见的语法,如判断`if else`,或且非`or and not`，二元比较`eq ne`,输出`print printf println`等等，除此之外还有一些常用的编码函数，如`urlescaper,attrescaper,htmlescaper`。
5. `Action`中支持类似`unix`下的管道符用法，`|`前面的命令会将运算结果(或返回值)传递给后一个命令的最后一个位置。

- 普通的信息泄露

模板支持`{{.Passwd}}`这样格式的内容来获得结构体Passwd字段的值

```jinja2
# 获得源码
{{printf "%+v" .}}
{{.}}
```

警惕直接的内容拼接

```go
// tpl := fmt.Sprintf(`<h1>Hi, ` + arg + `</h1> Your name is {{ .Name }}`)	// 哒咩
tpl := `<h1>Hi, {{ .arg }}</h1><br>Your name is {{ .Name }}`	// 可
data := map[string]string{
	"arg":  arg,
	"Name": user.Name,
}
html := template.Must(template.New("login").Parse(tpl))
html.Execute(w, data)
```

- XSS

Go的模板可以直接进行字符串打印，输出XSS语句

```jinja2
{{"<script>alert(/xss/)</script>"}}
{{print "<script>alert(/xss/)</script>"}}
```

修复：`text/template`内置html函数来转义特殊字符，js函数转义js代码

```jinja2
{{html "<script>alert(/xss/)</script>"}}
{{js "js代码"}}
```

这个包在模板处理阶段还有`template.HTMLEscapeString`等转义函数，也可以使用另一个模板包`html/template` 自带转义效果

- RCE / LFI

通过模板语法可知可以像`{{ .Name }}`一样调用对象方法，模板内部并不存在可以RCE的函数，所以除非有人为渲染对象定义了RCE或文件读取的方法，不然这个问题是不存在的

```go
func (u *User) System(cmd string, arg ...string) string {
	out, _ := exec.Command(cmd, arg...).CombinedOutput()
	return string(out)
}

func (u *User) FileRead(File string) string {
	data, err := ioutil.ReadFile(File)
	if err != nil {
		fmt.Print("File read error")
	}
	return string(data)
}
```

如果定义了就可以通过`{{.System "whoami"}}`和`{{.FileRead "filepath"}}`执行

### 某不知名题

```go
package main

import (
	"bufio"
	"html/template"
	"log"
	"os"
	"os/exec"
)

type Program string

func (p Program) Secret(test string) string {
	out, _ := exec.Command(test).CombinedOutput() // 目标
	return string(out)
}

func (p Program) Label(test string) string {
	return "This is " + string(test)
}

func main() {
	reader := bufio.NewReader(os.Stdin)
	text, _ := reader.ReadString('\n')
	tmpl, err := template.New("").Parse(text)
	if err != nil {
		log.Fatalf("Parse: %v", err)
	}
	tmpl.Execute(os.Stdin, Program("Intigriti"))
}
```

简单的模板注入

```jinja2
{{.Secret "whoami"}}
{{"whoami"| .Secret}}
```

### [LineCTF 2022]gotm

```go
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"os"
	"text/template"

	"github.com/golang-jwt/jwt"
)

type Account struct {
	id         string
	pw         string
	is_admin   bool
	secret_key string
}

type AccountClaims struct {
	Id       string `json:"id"`
	Is_admin bool   `json:"is_admin"`
	jwt.StandardClaims
}

type Resp struct {
	Status bool   `json:"status"`
	Msg    string `json:"msg"`
}

type TokenResp struct {
	Status bool   `json:"status"`
	Token  string `json:"token"`
}

var acc []Account
var secret_key = os.Getenv("KEY")
var flag = os.Getenv("FLAG")
var admin_id = os.Getenv("ADMIN_ID")
var admin_pw = os.Getenv("ADMIN_PW")

func clear_account() {
	acc = acc[:1]
}

func get_account(uid string) Account {
	for i := range acc {
		if acc[i].id == uid {
			return acc[i]
		}
	}
	return Account{}
}

func jwt_encode(id string, is_admin bool) (string, error) {
	claims := AccountClaims{
		id, is_admin, jwt.StandardClaims{},
	}
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	return token.SignedString([]byte(secret_key))
}

func jwt_decode(s string) (string, bool) {
	token, err := jwt.ParseWithClaims(s, &AccountClaims{}, func(token *jwt.Token) (interface{}, error) {
		return []byte(secret_key), nil
	})
	if err != nil {
		fmt.Println(err)
		return "", false
	}
	if claims, ok := token.Claims.(*AccountClaims); ok && token.Valid {
		return claims.Id, claims.Is_admin
	}
	return "", false
}

func auth_handler(w http.ResponseWriter, r *http.Request) {
	uid := r.FormValue("id")
	upw := r.FormValue("pw")
	if uid == "" || upw == "" {
		return
	}
	if len(acc) > 1024 {
		clear_account()
	}
	user_acc := get_account(uid)
	if user_acc.id != "" && user_acc.pw == upw {
		token, err := jwt_encode(user_acc.id, user_acc.is_admin)
		if err != nil {
			return
		}
		p := TokenResp{true, token}
		res, err := json.Marshal(p)
		if err != nil {
		}
		w.Write(res)
		return
	}
	w.WriteHeader(http.StatusForbidden)
	return
}

func regist_handler(w http.ResponseWriter, r *http.Request) {
	uid := r.FormValue("id")
	upw := r.FormValue("pw")

	if uid == "" || upw == "" {
		return
	}

	if get_account(uid).id != "" {
		w.WriteHeader(http.StatusForbidden)
		return
	}
	if len(acc) > 4 {
		clear_account()
	}
	new_acc := Account{uid, upw, false, secret_key}
	acc = append(acc, new_acc)

	p := Resp{true, ""}
	res, err := json.Marshal(p)
	if err != nil {
	}
	w.Write(res)
	return
}

func flag_handler(w http.ResponseWriter, r *http.Request) {
	token := r.Header.Get("X-Token")
	if token != "" {
		id, is_admin := jwt_decode(token)
		if is_admin == true {
			p := Resp{true, "Hi " + id + ", flag is " + flag}
			res, err := json.Marshal(p)
			if err != nil {
			}
			w.Write(res)
			return
		} else {
			w.WriteHeader(http.StatusForbidden)
			return
		}
	}
}

func root_handler(w http.ResponseWriter, r *http.Request) {
	token := r.Header.Get("X-Token")
	if token != "" {
		id, _ := jwt_decode(token)
		acc := get_account(id)
		tpl, err := template.New("").Parse("Logged in as " + acc.id)	// 直接拼接 ssti
		if err != nil {
		}
		tpl.Execute(w, &acc)
	} else {

		return
	}
}

func main() {
	admin := Account{admin_id, admin_pw, true, secret_key}
	acc = append(acc, admin)

	http.HandleFunc("/", root_handler)
	http.HandleFunc("/auth", auth_handler)
	http.HandleFunc("/flag", flag_handler)
	http.HandleFunc("/regist", regist_handler)
	log.Fatal(http.ListenAndServe("0.0.0.0:11000", nil))
}
```

大致审一下，我们需要一个is_admin==True的jwt即可拿flag

注意到root_handler在处理登录信息时用到了template，尝试ssti

```
/regist -> /auth -> /
id={{.}}&pw=ame
```

得到jwt和key

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6Int7Ln19IiwiaXNfYWRtaW4iOmZhbHNlfQ.0Lz_3fTyhGxWGwZnw3hM_5TzDfrk0oULzLWF4rRfMss
```

```
this_is_f4Ke_key
# 赵总都没改docker
```

jwt.io改is_admin为true，到/flag即可拿flag了

## url.URL结构体

https://www.ipeapea.cn/post/go-url/

```go
type URL struct {
	Scheme      string    // 协议
	Opaque      string    // 如果是opaque格式，那么此字段存储有值
	User        *Userinfo // 用户和密码信息
	Host        string    // 主机地址[:端口]
	Path        string    // 路径
	RawPath     string    // 如果Path是从转移后的路径解析的，那么RawPath会存储原始值，否则为空，见后面详解
	ForceQuery  bool      // 即便RawQuery为空，path结尾也有?符号
	RawQuery    string    // ?后面query内容
	Fragment    string    // #后面锚点信息
	RawFragment string    // 与RawPath含义一致
}

type Userinfo struct {
	username    string
	password    string
	passwordSet bool
}
```

示例：

```go
uStr := "http://root:password@localhost:28080/home/login?id=1&name=foo#fragment"
u, _ := url.Parse(uStr)
```

解析结果

```json
{
 "Scheme": "http",
 "Opaque": "",
 "User": {},
 "Host": "localhost:28080",
 "Path": "/home/login",
 "RawPath": "/home%2flogin",
 "ForceQuery": false,
 "RawQuery": "id=1\u0026name=foo",
 "Fragment": "fragment",
 "RawFragment": ""
}
```

- Opaque

为空，因为这个url是一个分层类型，只有当URL类型为不透明类型时才有意义

- RawPath

此时RawPath有值，为Path原始值 而Path存储的是将原始值反转义后的值

只有在原始`path`中包含了转移字符时才会有值，所以Go推荐我们使用`URL`的`EscapedPath`方法而不是直接使用`RawPath`字段
