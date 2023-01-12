---
title: "Python新特性"
slug: "whats-new-in-python"
description: "人生苦短，我用Python"
date: 2023-01-12T17:39:18+08:00
categories: ["LTS"]
series: []
tags: ["Python"]
draft: false
toc: true
---

摘自[What’s New in Python](https://docs.python.org/3/whatsnew/index.html#what-s-new-in-python)，仅摘取部分我觉得我平常可能会用到的新特性，如有缺漏请自行查阅官方doc

## 3.11

### 限制类型LiteralString

为了防止sqli，可以强制传入类型为LiteralString

```python
def run_query(sql: LiteralString) -> ...
    ...

def caller(
    arbitrary_string: str,
    query_string: LiteralString,
    table_name: LiteralString,
) -> None:
    run_query("SELECT * FROM students")       # ok
    run_query(query_string)                   # ok
    run_query("SELECT * FROM " + table_name)  # ok
    run_query(arbitrary_string)               # type checker error
    run_query(                                # type checker error
        f"SELECT * FROM students WHERE name = {arbitrary_string}"
    )
```

### 模块与库（部分）

- On Unix, [`time.sleep()`](https://docs.python.org/zh-cn/3/library/time.html#time.sleep) now uses the `clock_nanosleep()` or `nanosleep()` function, if available, which has a resolution of 1 nanosecond (10-9 seconds), rather than using `select()` which has a resolution of 1 microsecond (10-6 seconds). (Contributed by Benjamin Szőke and Victor Stinner in [bpo-21302](https://bugs.python.org/issue?@action=redirect&bpo=21302).)

  On Windows 8.1 and newer, [`time.sleep()`](https://docs.python.org/zh-cn/3/library/time.html#time.sleep) now uses a waitable timer based on [high-resolution timers](https://docs.microsoft.com/en-us/windows-hardware/drivers/kernel/high-resolution-timers) which has a resolution of 100 nanoseconds (10-7 seconds). Previously, it had a resolution of 1 millisecond (10-3 seconds). (Contributed by Benjamin Szőke, Dong-hee Na, Eryk Sun and Victor Stinner in [bpo-21302](https://bugs.python.org/issue?@action=redirect&bpo=21302) and [bpo-45429](https://bugs.python.org/issue?@action=redirect&bpo=45429).)

## 3.10

### 带圆括号的上下文管理器

```python
with (
    CtxManager1() as example1,
    CtxManager2() as example2,
    CtxManager3(),
):
```

支持写在圆括号里+折行

### switch case

通用语法

```python
match subject:
    case <pattern_1>:
        <action_1>
    case <pattern_2>:
        <action_2>
    case <pattern_3>:
        <action_3>
    case _:	# default
        <action_wildcard>
```

python中支持匹配的`case`是一个字面值（str, int....），对于 **tuple**和 **list**会自动映射变量和位置，即：

```python
def func1(t):
    match t:
        case (_, title, 500 | 501, *error):	# 通配符_ *, `|`代表or
            print(f"{title} 存在服务端错误")
        case (url, _, status_code) if status_code == 200 or status_code == 302:	# 支持if子句进行约束
            print(f"{url} 可以正常访问")
        case (url, title, status_code):
            print(f"{title} {url}的状态码为{status_code}")


func1(("https://example.com/", "测试网站", 200))
func1(("https://example.com/", "测试网站2", 500, "panic!"))
func1(("https://example.com/", "测试网站3", 500))
func1(["https://example.com/", "测试网站4", 404])	# 支持list
# output:
# https://example.com/ 可以正常访问
# 测试网站2 存在服务端错误
# 测试网站3 存在服务端错误
# 测试网站4 https://example.com/的状态码为404
```

模式可以任意地嵌套，如果我们的数据是一堆由点组成的列表

```python
match points:
    case []:
        print("No points in the list.")
    case [Point(0, 0)]:
        print("The origin is the only point in the list.")
    case [Point(x, y)]:
        print(f"A single point {x}, {y} is in the list.")
    case [Point(0, y1), Point(0, y2)]:
        print(f"Two points on the Y axis at {y1}, {y2} are in the list.")
    case _:
        print("Something else is found in the list.")
```

除此之外，还可以自定义类型：

```python
class Point:
    x: int
    y: int

def location(point):
    match point:
        case Point(x=0, y=0):
            print("Origin is the point's location.")
        case Point(x=0, y=y):
            print(f"Y={y} and the point is on the y-axis.")
        case Point(x=x, y=0):
            print(f"X={x} and the point is on the x-axis.")
        case Point():
            print("The point is located somewhere else on the plane.")
        case _:
            print("Not a point")
```

也支持命名常量：

```python
from enum import Enum
class Color(Enum):
    RED = 0
    GREEN = 1
    BLUE = 2

match color:
    case Color.RED:
        print("I see red!")
    case Color.GREEN:
        print("Grass is green")
    case Color.BLUE:
        print("I'm feeling the blues :(")
```

### 模块与库（部分）

- 加入了 [`sys.orig_argv`](https://docs.python.org/zh-cn/3/library/sys.html#sys.orig_argv) 属性：传给 Python 可执行文件的初始命令行参数列表。（由 Victor Stinner 贡献于 [bpo-23427](https://bugs.python.org/issue?@action=redirect&bpo=23427) ）
- 添加了 [`sys.stdlib_module_names`](https://docs.python.org/zh-cn/3/library/sys.html#sys.stdlib_module_names)，包含标准库模块名称的列表。 （由 Victor Stinner 在 [bpo-42955](https://bugs.python.org/issue?@action=redirect&bpo=42955) 中贡献。）
- 现在 [`os.path.realpath()`](https://docs.python.org/zh-cn/3/library/os.path.html#os.path.realpath) 可接受一个关键字参数 *strict*。 若设为 `True` ，则在路径不存在或遭遇循环符号链接时，会触发 [`OSError`](https://docs.python.org/zh-cn/3/library/exceptions.html#OSError)。 （由 Barney Gale 在 [bpo-43757](https://bugs.python.org/issue?@action=redirect&bpo=43757) 中贡献。）
- 增加 [`base64.b32hexencode()`](https://docs.python.org/zh-cn/3/library/base64.html#base64.b32hexencode) 和 [`base64.b32hexdecode()`](https://docs.python.org/zh-cn/3/library/base64.html#base64.b32hexdecode) 以支持带有扩展十六进制字母的 Base32 编码。
- 现在 `encodings.normalize_encoding()` 会忽略非 ASCII 字符。（由 Hai Shi 在 [bpo-39337](https://bugs.python.org/issue?@action=redirect&bpo=39337) 中贡献）
- Python 3.10 以下版本允许在 [`urllib.parse.parse_qs()`](https://docs.python.org/zh-cn/3/library/urllib.parse.html#urllib.parse.parse_qs) 和 [`urllib.parse.parse_qsl()`](https://docs.python.org/zh-cn/3/library/urllib.parse.html#urllib.parse.parse_qsl) 中同时使用 `;` 和 `&` 作为查询参数分隔符。出于安全考虑，并符合 W3C 最新的建议，这已经被修改为只允许用一种分隔符，默认值为 `&`。 这一改变也影响到了 [`cgi.parse()`](https://docs.python.org/zh-cn/3/library/cgi.html#cgi.parse) 和 [`cgi.parse_multipart()`](https://docs.python.org/zh-cn/3/library/cgi.html#cgi.parse_multipart)，因为他们内部用到了这些函数。更多细节，请参阅各自的文档。（由 Adam Goldschmidt 、 Senthil Kumaran 和 Ken Jin 贡献于 [bpo-42967](https://bugs.python.org/issue?@action=redirect&bpo=42967) ）

## 3.9

### 字典合并与更新

```python
>>> x = {"key1": "value1 from x", "key2": "value2 from x"}
>>> y = {"key2": "value2 from y", "key3": "value3 from y"}
>>> x | y
{'key1': 'value1 from x', 'key2': 'value2 from y', 'key3': 'value3 from y'}
>>> y | x
{'key2': 'value2 from x', 'key3': 'value3 from y', 'key1': 'value1 from x'}
```

同key情况下，跟随运算符后的字典

### 模块与库（部分）

- 增加了 [`str.removeprefix(prefix)`](https://docs.python.org/zh-cn/3/library/stdtypes.html#str.removeprefix) 和 [`str.removesuffix(suffix)`](https://docs.python.org/zh-cn/3/library/stdtypes.html#str.removesuffix) 用于方便地从字符串移除不需要的前缀或后缀。 也增加了 `bytes`, `bytearray` 以及 `collections.UserString` 的对应方法。 请参阅 [**PEP 616**](https://peps.python.org/pep-0616/) 了解详情。 （由 Dennis Sweeney 在 [bpo-39939](https://bugs.python.org/issue?@action=redirect&bpo=39939) 中贡献。）

- [`ipaddress`](https://docs.python.org/zh-cn/3/library/ipaddress.html#module-ipaddress) 现在支持 IPv6 作用域地址（即带有 `%<scope_id>` 前缀的 IPv6 地址）。

  IPv6 作用域地址可使用 [`ipaddress.IPv6Address`](https://docs.python.org/zh-cn/3/library/ipaddress.html#ipaddress.IPv6Address) 来解析。 作用域的区 ID 如果存在，可通过 [`scope_id`](https://docs.python.org/zh-cn/3/library/ipaddress.html#ipaddress.IPv6Address.scope_id) 属性来获取。 （由 Oleksandr Pavliuk 在 [bpo-34788](https://bugs.python.org/issue?@action=redirect&bpo=34788) 中贡献。）

  从 Python 3.9.5 开始 [`ipaddress`](https://docs.python.org/zh-cn/3/library/ipaddress.html#module-ipaddress) 模块不再接受 IPv4 地址字符串中有任何前缀的零。 （由 Christian Heimes 在 [bpo-36384](https://bugs.python.org/issue?@action=redirect&bpo=36384) 中贡献。）

- [`multiprocessing.SimpleQueue`](https://docs.python.org/zh-cn/3/library/multiprocessing.html#multiprocessing.SimpleQueue) 类新增了 [`close()`](https://docs.python.org/zh-cn/3/library/multiprocessing.html#multiprocessing.SimpleQueue.close) 方法用来显式地关闭队列。 （由 Victor Stinner 在 [bpo-30966](https://bugs.python.org/issue?@action=redirect&bpo=30966) 中贡献。）

- 早先的 Python 版本允许使用 `;` 和 `&` 作为 [`urllib.parse.parse_qs()`](https://docs.python.org/zh-cn/3/library/urllib.parse.html#urllib.parse.parse_qs) 和 [`urllib.parse.parse_qsl()`](https://docs.python.org/zh-cn/3/library/urllib.parse.html#urllib.parse.parse_qsl) 中 query 形参的分隔键。 出于安全考虑，也为了遵循更新的 W3C 推荐设置，这已被改为只允许单个分隔键，默认为 `&`。 这一改变还会影响 [`cgi.parse()`](https://docs.python.org/zh-cn/3/library/cgi.html#cgi.parse) 和 [`cgi.parse_multipart()`](https://docs.python.org/zh-cn/3/library/cgi.html#cgi.parse_multipart) 因为它们在内部使用了受影响的函数。 要了解更多细节，请查看它们各自的文档。 （由 Adam Goldschmidt, Senthil Kumaran 和 Ken Jin 在 [bpo-42967](https://bugs.python.org/issue?@action=redirect&bpo=42967) 中贡献。）

- 添加了新的函数 random.Random.randbytes()。该函数可用于生成随机字节。

## 3.8

### 赋值表达式

```python
if (n := len(a)) > 10:
    print(f"List is too long ({n} elements, expected <= 10)")
```

看到这个功能，我的泪 冲了出来

真的太需要这个功能了

### f-string扩展

```python
>>> user = 'eric_idle'
>>> member_since = date(1975, 7, 31)
>>> f'{user=} {member_since=}'
"user='eric_idle' member_since=datetime.date(1975, 7, 31)"
```

我的泪 又冲了出来

### 模块与库（部分）

- 现在 dict 和 dictview 可以使用 [`reversed()`](https://docs.python.org/zh-cn/3/library/functions.html#reversed) 按插入顺序反向迭代。 （由 Rémi Lapeyre 在 [bpo-33462](https://bugs.python.org/issue?@action=redirect&bpo=33462) 中贡献。）

- 添加了新的替代构造器 [`datetime.date.fromisocalendar()`](https://docs.python.org/zh-cn/3/library/datetime.html#datetime.date.fromisocalendar) 和 [`datetime.datetime.fromisocalendar()`](https://docs.python.org/zh-cn/3/library/datetime.html#datetime.datetime.fromisocalendar)，它们分别基于 ISO 年份、周序号和周内日序号来构造 `date` 和 [`datetime`](https://docs.python.org/zh-cn/3/library/datetime.html#module-datetime) 对象；这两者分别是其所对应类中 `isocalendar` 方法的逆操作。 （由 Paul Ganssle 在 [bpo-36004](https://bugs.python.org/issue?@action=redirect&bpo=36004) 中贡献。）

- 在 Windows 上添加了新函数 [`add_dll_directory()`](https://docs.python.org/zh-cn/3/library/os.html#os.add_dll_directory) 用于在导入扩展模块或使用 [`ctypes`](https://docs.python.org/zh-cn/3/library/ctypes.html#module-ctypes) 加载 DLL 时为本机依赖提供额外搜索路径 。 （由 Steve Dower 在 [bpo-36085](https://bugs.python.org/issue?@action=redirect&bpo=36085) 中贡献。）

  添加了新的 [`os.memfd_create()`](https://docs.python.org/zh-cn/3/library/os.html#os.memfd_create) 函数用于包装 `memfd_create()` 系统调用。 （由 Zackery Spytz 和 Christian Heimes 在 [bpo-26836](https://bugs.python.org/issue?@action=redirect&bpo=26836) 中贡献。）

  在 Windows 上，大部分用于处理重解析点,（包括符号链接和目录连接）的手动逻辑已被委托给操作系统。 特别地，[`os.stat()`](https://docs.python.org/zh-cn/3/library/os.html#os.stat) 现在将会遍历操作系统所支持的任何内容，而 [`os.lstat()`](https://docs.python.org/zh-cn/3/library/os.html#os.lstat) 将只打开被标识为“名称代理”的重解析点，而其要由 [`os.stat()`](https://docs.python.org/zh-cn/3/library/os.html#os.stat) 打开其他的重解析点。 在所有情况下，`stat_result.st_mode` 将只为符号链接而非其他种类的重解析点设置 `S_IFLNK`。 要标识其他种类的重解析点，请检查新的 `stat_result.st_reparse_tag` 属性。

  在 Windows 上，[`os.readlink()`](https://docs.python.org/zh-cn/3/library/os.html#os.readlink) 现在能够读取目录连接。 请注意 [`islink()`](https://docs.python.org/zh-cn/3/library/os.path.html#os.path.islink) 会对目录连接返回 `False`，因此首先检查 `islink` 的代码将连续把连接视为目录，而会处理 [`os.readlink()`](https://docs.python.org/zh-cn/3/library/os.html#os.readlink) 所引发错误的代码现在会把连接视为链接。

  （由 Steve Dower 在 [bpo-37834](https://bugs.python.org/issue?@action=redirect&bpo=37834) 中贡献。）

- 返回布尔值结果的 [`os.path`](https://docs.python.org/zh-cn/3/library/os.path.html#module-os.path) 函数例如 [`exists()`](https://docs.python.org/zh-cn/3/library/os.path.html#os.path.exists), [`lexists()`](https://docs.python.org/zh-cn/3/library/os.path.html#os.path.lexists), [`isdir()`](https://docs.python.org/zh-cn/3/library/os.path.html#os.path.isdir), [`isfile()`](https://docs.python.org/zh-cn/3/library/os.path.html#os.path.isfile), [`islink()`](https://docs.python.org/zh-cn/3/library/os.path.html#os.path.islink), 以及 [`ismount()`](https://docs.python.org/zh-cn/3/library/os.path.html#os.path.ismount) 现在对于包含在 OS 层级无法表示的字符或字节的路径将会返回 `False` 而不是引发 [`ValueError`](https://docs.python.org/zh-cn/3/library/exceptions.html#ValueError) 或其子类 [`UnicodeEncodeError`](https://docs.python.org/zh-cn/3/library/exceptions.html#UnicodeEncodeError) 和 [`UnicodeDecodeError`](https://docs.python.org/zh-cn/3/library/exceptions.html#UnicodeDecodeError)。 （由 Serhiy Storchaka 在 [bpo-33721](https://bugs.python.org/issue?@action=redirect&bpo=33721) 中贡献。）

  [`expanduser()`](https://docs.python.org/zh-cn/3/library/os.path.html#os.path.expanduser) 在 Windows 上现在改用 `USERPROFILE` 环境变量而不再使用 `HOME`，后者通常不会为一般用户账户设置。 （由 Anthony Sottile 在 [bpo-36264](https://bugs.python.org/issue?@action=redirect&bpo=36264) 中贡献。）

  [`isdir()`](https://docs.python.org/zh-cn/3/library/os.path.html#os.path.isdir) 在 Windows 上将不再为不存在的目录的链接返回 `True`。

  [`realpath()`](https://docs.python.org/zh-cn/3/library/os.path.html#os.path.realpath) 在 Windows 上现在会识别重解析点，包括符号链接和目录连接。

  （由 Steve Dower 在 [bpo-37834](https://bugs.python.org/issue?@action=redirect&bpo=37834) 中贡献。）

## 3.7

### 强制utf-8

新的 [`-X`](https://docs.python.org/zh-cn/3/using/cmdline.html#cmdoption-X) `utf8` 命令行选项和 [`PYTHONUTF8`](https://docs.python.org/zh-cn/3/using/cmdline.html#envvar-PYTHONUTF8) 环境变量可被用来启用 [Python UTF-8 模式](https://docs.python.org/zh-cn/3/library/os.html#utf8-mode)。

当处于 UTF-8 模式时，CPython 会忽略区域设置，并默认使用 UTF-8 编码。 用于 [`sys.stdin`](https://docs.python.org/zh-cn/3/library/sys.html#sys.stdin) 和 [`sys.stdout`](https://docs.python.org/zh-cn/3/library/sys.html#sys.stdout) 流的错误处理器将设置为 `surrogateescape`。

强制 UTF-8 模式可被用来在嵌入的 Python 解释器中改变文本处理行为，而不会改变嵌入方应用的区域设置。

### 内置breakpoint

类似js中的`debugger;`

### 模块与库（部分）

- 新的 [`UUID.is_safe`](https://docs.python.org/zh-cn/3/library/uuid.html#uuid.UUID.is_safe) 属性会从平台中继有关是否使用多进程安全模式来生成所需 UUID 的信息。 （由 Barry Warsaw 在 [bpo-22807](https://bugs.python.org/issue?@action=redirect&bpo=22807) 中贡献。）

  [`uuid.getnode()`](https://docs.python.org/zh-cn/3/library/uuid.html#uuid.getnode) 现在更倾向于统一管理的 MAC 地址而不是本地管理的 MAC 地址。 这样可以更好地保证从 [`uuid.uuid1()`](https://docs.python.org/zh-cn/3/library/uuid.html#uuid.uuid1) 返回的 UUID 的全局唯一性。 如果只有本地管理的 MAC 地址可用，则返回首个找到的此类地址。 （由 Barry Warsaw 在 [bpo-32107](https://bugs.python.org/issue?@action=redirect&bpo=32107) 中贡献。）



## 3.6

### 数字字面值中的下划线

```python
>>> 1_000_000_000_000_000
1000000000000000
```

下划线不打破连贯性，一行内不允许有开头、末尾或连续多个下划线

