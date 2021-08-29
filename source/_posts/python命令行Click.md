---
title: python命令行Click
tags:
  - Python
categories: Python
encrypt: 
enc_pwd: 
abbrlink: 49550
date: 2020-05-02 22:33:59
summary_img:
---

# 一 click

## 1介绍

[click](https://click.palletsprojects.com/) 是一个以尽可能少的代码、以组合的方式创建优美的命令行程序的 Python 包。它有很高的可配置性，同时也能开箱即用。

它旨在让编写命令行工具的过程既快速又有趣，还能防止由于无法实现预期的 CLI API 所产生挫败感。它有如下三个特点：

- 任意嵌套命令
- 自动生成帮助
- 支持运行时延迟加载子命令

## 2 快速开始

### 2.1 业务逻辑

假设命令行程序的输入是 `name` 和 `count`，功能是打印指定次数的名字。

那么在 `hello.py` 中，很容易写出如下代码：

```python
def hello(count, name):
    """Simple program that greets NAME for a total of COUNT times."""
    for x in range(count):
        click.echo('Hello %s!' % name)
```

这段代码的逻辑很简单，就是循环 `count` 次，使用 `click.echo` 打印 `name`。其中，`click.echo` 和 `print` 的作用相似，但功能更加强大，能处理好 Unicode 和 二进制数据的情况。

### 2.2 定义参数

很显然，我们需要针对 `count` 和 `name` 来定义它们所对应的参数信息。

- `count` 对应为命令行选项 `--count`，类型为数字，我们希望在不提供参数时，其默认值是 1
- `name` 对应为命令行选项 `--name`，类型为字符串，我们希望在不提供参数时，能给人提示

使用 `click`，就可以写成下面这样：

```python
from click import click

@click.command()
@click.option('--count', default=1, help='Number of greetings.')
@click.option('--name', prompt='Your name',
              help='The person to greet.')
def hello(count, name):
    """Simple program that greets NAME for a total of COUNT times."""
    for x in range(count):
        click.echo('Hello %s!' % name)              
```

在上面的示例中：

- 使用装饰器的方式，即定义了参数，又将之与处理逻辑绑定，这真是优雅。和 `argparse`、`docopt` 比起来，就少了一步绑定过程
- 使用 `click.command` 表示 `hello` 是对命令的处理
- 使用

```
  click.option
```

   

  来定义参数选项

- 对于 `--count` 来说，使用 `default` 来指定默认值。而由于默认值是数字，进而暗示 `--count` 选项的类型为数字
- 对于 `--name` 来说，使用 `prompt` 来指定未输入该选项时的提示语
- 使用 `help` 来指定帮助信息

不论是装饰器的方式、还是各种默认行为，`click` 都是像它的介绍所说的那样，让人尽可能少地编写代码，让整个过程变得快速而有趣。

### 2.3 代码梳理

使用 `click` 的方式非常简单，我们将上文的代码汇总下，以有一个更清晰的认识：

```python
# hello.py
import click

@click.command()
@click.option('--count', default=1, help='Number of greetings.')
@click.option('--name', prompt='Your name',
              help='The person to greet.')
def hello(count, name):
    """Simple program that greets NAME for a total of COUNT times."""
    for x in range(count):
        click.echo('Hello %s!' % name)

if __name__ == '__main__':
    hello()
```

若我们指定次数和名字：

```shell
$ python3 hello.py --count 2 --name Eric
Hello Eric!
Hello Eric!
```

若我们什么都不指定，则会提示输入名字，并默认输出一次：

```shell
$ python3 hello.py
Your name: Eric
Hello Eric!
```

我们还可以通过 `--help` 参数查看自动生成的帮助信息：

```shell
Usage: hello.py [OPTIONS]

  Simple program that greets NAME for a total of COUNT times.

Options:
  --count INTEGER  Number of greetings.
  --name TEXT      The person to greet.
  --help           Show this message and exit.
```

## 3 深入click

在概念上， `click` 把命令行分为 3 个组成：参数、选项和命令。

- `参数` 就是跟在命令后的除选项外的内容，比如 `git add a.txt` 中的 `a.txt` 就是表示文件路径的参数
- `选项` 就是以 `-` 或 `--` 开头的参数，比如 `-f`、`--file`
- `命令` 就是命令行的初衷了，比如 `git` 就是命令，而 `git add` 中的 `add` 则是 `git` 的子命令

### 3.1 参数

#### 3.1.1 基本参数

`基本参数` 就是通过位置里指定参数值。

比如，我们可以指定两个位置参数 `x` 和 `y` ，先添加的 `x` 位于第一个位置，后加入的 `y` 位于第二个位置。那么在命令行中输入 `1 2`的时候，分别对应到的就是 `x` 和 `y`：

```python
@click.command()
@click.argument('x')
@click.argument('y')
def hello(x, y):
    print(x, y)
```

#### 3.1.2 参数类型

`参数类型` 就是将参数值作为什么类型去解析，默认情况下是字符串类型。我们可以通过 `type` 入参来指定参数类型。

`click` 支持的参数类型多种多样：

- `str` / `click.STRING` 表示字符串类型，这也是默认类型
- `int` / `click.INT` 表示整型
- `float` / `click.FLOAT` 表示浮点型
- `bool` / `click.BOOL` 表示布尔型。很棒之处在于，它会识别表示真/假的字符。对于 `1`、`yes`、`y` 和 `true` 会转化为 `True`；`0`、`no`、`n` 和 `false` 会转化为 `False`
- `click.UUID` 表示 UUID，会自动将参数转换为 `uuid.UUID` 对象
- `click.FILE` 表示文件，会自动将参数转换为文件对象，并在命令行结束时自动关闭文件
- `click.PATH` 表示路径
- `click.Choice` 表示选择选项
- `click.IntRange` 表示范围选项

同 `argparse` 一样，`click` 也支持自定义类型，需要编写 `click.ParamType` 的子类，并重载 `convert` 方法。

官网提供了一个例子，实现了一个整数类型，除了普通整数之外，还接受十六进制和八进制数字， 并将它们转换为常规整数：

```python
class BasedIntParamType(click.ParamType):
    name = "integer"

    def convert(self, value, param, ctx):
        try:
            if value[:2].lower() == "0x":
                return int(value[2:], 16)
            elif value[:1] == "0":
                return int(value, 8)
            return int(value, 10)
        except TypeError:
            self.fail(
                "expected string for int() conversion, got "
                f"{value!r} of type {type(value).__name__}",
                param,
                ctx,
            )
        except ValueError:
            self.fail(f"{value!r} is not a valid integer", param, ctx)

BASED_INT = BasedIntParamType()
```

#### 3.1.3 文件参数

在基本参数的基础上，通过指定参数类型，我们就能构建出各类参数。

`文件参数` 是非常常用的一类参数，通过 `type=click.File` 指定，它能正确处理所有 Python 版本的 unicode 和 字节，使得处理文件十分方便。

```python
@click.command()
@click.argument('input', type=click.File('rb'))  # 指定文件为二进制读
@click.argument('output', type=click.File('wb'))  # 指定文件为二进制写
def inout(input, output):
    while True:
        chunk = input.read(1024)  # 此时 input 为文件对象，每次读入 1024 字节
        if not chunk:
            break
        output.write(chunk)  # 此时 output 为文件对象，写入上步读入的内容
```

#### 3.1.4文件路径参数

`文件路径参数` 用来处理文件路径，可以对路径做是否存在等检查，通过 `type=click.Path` 指定。不论文件名是 unicode 还是字节类型，获取到的参数类型都是 unicode 类型。

```python
@click.command()
@click.argument('filename', type=click.Path(exists=True))  # 要求给定路径存在，否则报错
def hello(filename):
    click.echo(click.format_filename(filename))
```

如果文件名是以 `-` 开头，会被误认为是命令行选项，这个时候需要在参数前加上 `--` 和空格，比如

```
$ python hello.py -- -foo.txt
-foo.txt
```

#### 3.1.5选择项参数

`选择项参数` 用来限定参数内容，通过 `type=click.Choice` 指定。

比如，指定文件读取方式限制为 `read-only` 和 `read-write`：

```python
@click.command()
@click.argument('mode', type=click.Choice(['read-only', 'read-write']))
def hello(mode):
    click.echo(mode)
```

#### 3.1.6可变参数

`可变参数` 用来定义一个参数可以有多个值，且能通过 `nargs` 来定义值的个数，取得的参数的变量类型为元组。

若 `nargs=N`，`N`为一个数字，则要求该参数提供 N 个值。若 `N` 为 `-1` 则接受提供无数量限制的参数，如:

```python
@click.command()
@click.argument('foo', nargs=-1)
@click.argument('bar', nargs=1)
def hello(foo, bar):
    pass
```

如果要实现 `argparse` 中要求参数数量为 1 个或多个的功能，则指定 `nargs=-1` 且 `required=True` 即可：

```python
@click.command()
@click.argument('foo', nargs=-1, required=True)
def hello(foo, bar):
    pass
```

#### 3.1.7从环境变量读取参数

通过在 `click.argument` 中指定 `envvar`，则可读取指定名称的环境变量作为参数值，比如：

```python
@click.command()
@click.argument('filename', envvar='FILENAME')
def hello(filename):
    print(filename)
```

执行如下命令查看效果：

```shell
$ FILENAME=hello.txt python3 hello.py
hello.txt
```



### 3.2 选项

通过 `click.option` 可以给命令增加选项，并通过配置函数的参数来配置不同功能的选项。

#### 3.2.1 给选项命名

`click.option` 中的命令规则可参考[参数名称](https://click.palletsprojects.com/en/7.x/parameters/#parameter-names)。它接受的前两个参数为长、短选项（顺序随意），其中：

- 长选项以 “--” 开头，比如 “--string-to-echo”
- 短选项以 “-” 开头，比如 “-s”

第三个参数为选项参数的名称，如果不指定，将会使用长选项的下划线形式名称：

```python
@click.command()
@click.option('-s', '--string-to-echo')
def echo(string_to_echo):
    click.echo(string_to_echo)
```

显示指定为 string

```python
@click.command()
@click.option('-s', '--string-to-echo', 'string')
def echo(string):
    click.echo(string)
```

#### 3.2.2 基本值选项

值选项是非常常用的选项，它接受一个值。如果在命令行中提供了值选项，则需要提供对应的值；反之则使用默认值。若没在 `click.option` 中指定默认值，则默认值为 `None`，且该选项的类型为 [STRING](https://click.palletsprojects.com/en/7.x/api/#click.STRING)；反之，则选项类型为默认值的类型。

比如，提供默认值为 1，则选项类型为 [INT](https://click.palletsprojects.com/en/7.x/api/#click.INT)：

```python
@click.command()
@click.option('--n', default=1)
def dots(n):
    click.echo('.' * n)
```

如果要求选项为必填，则可指定 `click.option` 的 `required=True`：

```python
@click.command()
@click.option('--n', required=True, type=int)
def dots(n):
    click.echo('.' * n)
```

如果选项名称和 Python 中的关键字冲突，则可以显式的指定选项名称。比如将 `--from` 的名称设置为 `from_`：

```python
@click.command()
@click.option('--from', '-f', 'from_')
@click.option('--to', '-t')
def reserved_param_name(from_, to):
    click.echo(f'from {from_} to {to}')
```

如果要在帮助中显式默认值，则可指定 `click.option` 的 `show_default=True`：

```python
@click.command()
@click.option('--n', default=1, show_default=True)
def dots(n):
    click.echo('.' * n)
```

在命令行中调用则有：

```bash
$ dots --help
Usage: dots [OPTIONS]

Options:
  --n INTEGER  [default: 1]
  --help       Show this message and exit.
```

#### 3.2.3多值选项

有时，我们会希望命令行中一个选项能接收多个值，通过指定 `click.option` 中的 `nargs` 参数（必须是大于等于 0）。这样，接收的多值选项就会变成一个元组。

比如，在下面的示例中，当通过 `--pos` 指定多个值时，`pos` 变量就是一个元组，里面的每个元素是一个 `float`：

```python
@click.command()
@click.option('--pos', nargs=2, type=float)
def findme(pos):
    click.echo(pos)
```

在命令行中调用则有：

```bash
$ findme --pos 2.0 3.0
(1.0, 2.0)
```

有时，通过同一选项指定的多个值得类型可能不同，这个时候可以指定 `click.option` 中的 `type=(类型1, 类型2, ...)` 来实现。而由于元组的长度同时表示了值的数量，所以就无须指定 `nargs` 参数。

```python
@click.command()
@click.option('--item', type=(str, int))
def putitem(item):
    click.echo('name=%s id=%d' % item)
```

在命令行中调用则有：

```bash
$ putitem --item peter 1338
name=peter id=1338
```

#### 3.2.4多选项

不同于多值选项是通过一个选项指定多个值，多选项则是使用多个相同选项分别指定值，通过 `click.option` 中的 `multiple=True` 来实现。

当我们定义如下多选项：

```python
@click.command()
@click.option('--message', '-m', multiple=True)
def commit(message):
    click.echo('\n'.join(message))
```

便可以指定任意数量个选项来指定值，获取到的 `message` 是一个元组：

```bash
$ commit -m foo -m bar --message baz
foo
bar
baz
```

#### 3.2.5 计值选项

有时我们可能需要获得选项的数量，那么可以指定 `click.option` 中的 `count=True` 来实现。

最常见的使用场景就是指定多个 `--verbose` 或 `-v` 选项来表示输出内容的详细程度。

```python
@click.command()
@click.option('-v', '--verbose', count=True)
def log(verbose):
    click.echo(f'Verbosity: {verbose}')
```

在命令行中调用则有：

```bash
$ log -vvv
Verbosity: 3
```

通过上面的例子，`verbose` 就是数字，表示 `-v` 选项的数量，由此可以进一步使用该值来控制日志的详细程度。

### 2.6 布尔选项

布尔选项用来表示真或假，它有多种实现方式：

- 通过 `click.option` 的 `is_flag=True` 参数来实现：

```python
import sys

@click.command()
@click.option('--shout', is_flag=True)
def info(shout):
    rv = sys.platform
    if shout:
        rv = rv.upper() + '!!!!111'
    click.echo(rv)
```

在命令行中调用则有：

```bash
$ info --shout
LINUX!!!!111
```

- 通过在 `click.option` 的选项定义中使用 `/` 分隔表示真假两个选项来实现：

```python
import sys

@click.command()
@click.option('--shout/--no-shout', default=False)
def info(shout):
    rv = sys.platform
    if shout:
        rv = rv.upper() + '!!!!111'
    click.echo(rv)
```

在命令行中调用则有：

```bash
$ info --shout
LINUX!!!!111
$ info --no-shout
linux
```

在 Windows 中，一个选项可以以 `/` 开头，这样就会真假选项的分隔符冲突了，这个时候可以使用 `;` 进行分隔：

```python
@click.command()
@click.option('/debug;/no-debug')
def log(debug):
    click.echo(f'debug={debug}')

if __name__ == '__main__':
    log()
```

在 cmd 中调用则有：

```bash
> log /debug
debug=True
```

#### 3.2.7 特性切换选项

所谓特性切换就是切换同一个操作对象的不同特性，比如指定 `--upper` 就让输出大写，指定 `--lower` 就让输出小写。这么来看，布尔值其实是特性切换的一个特例。

要实现特性切换选项，需要让多个选项都有相同的参数名称，并且定义它们的标记值 `flag_value`：

```python
import sys

@click.command()
@click.option('--upper', 'transformation', flag_value='upper',
              default=True)
@click.option('--lower', 'transformation', flag_value='lower')
def info(transformation):
    click.echo(getattr(sys.platform, transformation)())
```

在命令行中调用则有：

```bash
$ info --upper
LINUX
$ info --lower
linux
$ info
LINUX
```

在上面的示例中，`--upper` 和 `--lower` 都有相同的参数值 `transformation`：

- 当指定 `--upper` 时，`transformation` 就是 `--upper` 选项的标记值 `upper`
- 当指定 `--lower` 时，`transformation` 就是 `--lower` 选项的标记值 `lower`

进而就可以做进一步的业务逻辑处理。

#### 3.2.8 选择项选项

`选择项选项` 和 上篇文章中介绍的 `选择项参数` 类似，只不过是限定选项内容，依旧是通过 `type=click.Choice` 实现。此外，`case_sensitive=False` 还可以忽略选项内容的大小写。

```python
@click.command()
@click.option('--hash-type',
              type=click.Choice(['MD5', 'SHA1'], case_sensitive=False))
def digest(hash_type):
    click.echo(hash_type)
```

在命令行中调用则有：

```bash
$ digest --hash-type=MD5
MD5

$ digest --hash-type=md5
MD5

$ digest --hash-type=foo
Usage: digest [OPTIONS]
Try "digest --help" for help.

Error: Invalid value for "--hash-type": invalid choice: foo. (choose from MD5, SHA1)

$ digest --help
Usage: digest [OPTIONS]

Options:
  --hash-type [MD5|SHA1]
  --help                  Show this message and exit.
```

#### 3.2.9 提示选项

顾名思义，当提供了选项却没有提供对应的值时，会提示用户输入值。这种交互式的方式会让命令行变得更加友好。通过指定 `click.option` 中的 `prompt` 可以实现。

- 当 `prompt=True` 时，提示内容为选项的参数名称

```python
@click.command()
@click.option('--name', prompt=True)
def hello(name):
    click.echo(f'Hello {name}!')
```

在命令行调用则有：

```bash
$ hello --name=John
Hello John!
$ hello
Name: John
Hello John!
```

- 当 `prompt='Your name please'` 时，提示内容为指定内容

```python
@click.command()
@click.option('--name', prompt='Your name please')
def hello(name):
    click.echo(f'Hello {name}!')
```

在命令行中调用则有：

```bash
$ hello
Your name please: John
Hello John!
```

基于提示选项，我们还可以指定 `hide_input=True` 来隐藏输入，`confirmation_prompt=True` 来让用户进行二次输入，这非常适合输入密码的场景。

```python
@click.command()
@click.option('--password', prompt=True, hide_input=True,
              confirmation_prompt=True)
def encrypt(password):
    click.echo(f'Encrypting password to {password.encode("rot13")}')
```

当然，也可以直接使用 `click.password_option`：

```python
@click.command()
@click.password_option()
def encrypt(password):
    click.echo(f'Encrypting password to {password.encode("rot13")}')
```

我们还可以给提示选项设置默认值，通过 `default` 参数进行设置，如果被设置为函数，则可以实现动态默认值。

```python
@click.command()
@click.option('--username', prompt=True,
              default=lambda: os.environ.get('USER', ''))
def hello(username):
    print("Hello,", username)
```

详情请阅读 [Dynamic Defaults for Prompts](https://click.palletsprojects.com/en/7.x/options/#dynamic-defaults-for-prompts)。

#### 3.2.10 范围选项

如果希望选项的值在某个范围内，就可以使用范围选项，通过指定 `type=click.IntRange` 来实现。它有两种模式：

- 默认模式（非强制模式），如果值不在区间范围内将会引发一个错误。如 `type=click.IntRange(0, 10)` 表示范围是 [0, 10]，超过该范围报错
- 强制模式，如果值不在区间范围内，将会强制选取一个区间临近值。如 `click.IntRange(0, None, clamp=True)` 表示范围是 [0, +∞)，小于 0 则取 0，大于 20 则取 20。其中 `None` 表示没有限制

```python
@click.command()
@click.option('--count', type=click.IntRange(0, None, clamp=True))
@click.option('--digit', type=click.IntRange(0, 10))
def repeat(count, digit):
    click.echo(str(digit) * count)

if __name__ == '__main__':
    repeat()
```

在命令行中调用则有：

```bash
$ repeat --count=1000 --digit=5
55555555555555555555
$ repeat --count=1000 --digit=12
Usage: repeat [OPTIONS]

Error: Invalid value for "--digit": 12 is not in the valid range of 0 to 10.
```

#### 3.2.11 回调和优先

**回调** 通过 `click.option` 中的 `callback` 可以指定选项的回调，它会在该选项被解析后调用。回调函数的签名如下：

```python
def callback(ctx, param, value):
    pass
```

其中：

- ctx 是命令的上下文 [click.Context](https://click.palletsprojects.com/en/7.x/api/#click.Context)
- param 为选项变量 [click.Option](https://click.palletsprojects.com/en/7.x/api/#click.Option)
- value 为选项的值

使用回调函数可以完成额外的参数校验逻辑。比如，通过 --rolls 的选项来指定摇骰子的方式，内容为“d”，表示 M 面的骰子摇 N 次，N 和 M 都是数字。在真正的处理 rolls 前，我们需要通过回调函数来校验它的格式：

```python
def validate_rolls(ctx, param, value):
    try:
        rolls, dice = map(int, value.split('d', 2))
        return (dice, rolls)
    except ValueError:
        raise click.BadParameter('rolls need to be in format NdM')

@click.command()
@click.option('--rolls', callback=validate_rolls, default='1d6')
def roll(rolls):
    click.echo('Rolling a %d-sided dice %d time(s)' % rolls)
```

这样，当我们输入错误格式时，变会校验不通过：

```bash
$ roll --rolls=42
Usage: roll [OPTIONS]

Error: Invalid value for "--rolls": rolls need to be in format NdM
```

输入正确格式时，则正常输出信息：

```bash
$ roll --rolls=2d12
Rolling a 12-sided dice 2 time(s)
```

**优先** 通过 `click.option` 中的 `is_eager` 可以让该选项成为优先选项，这意味着它会先于所有选项处理。

利用回调和优先选项，我们就可以很好地实现 `--version` 选项。不论命令行中写了多少选项和参数，只要包含了 `--version`，我们就希望它打印版本就退出，而不执行其他选项的逻辑，那么就需要让它成为优先选项，并且在回调函数中打印版本。

此外，在 `click` 中每个选项都对应到命令处理函数的同名参数，如果不想把该选项传递到处理函数中，则需要指定 `expose_value=True`，于是有：

```python
def print_version(ctx, param, value):
    if not value or ctx.resilient_parsing:
        return
    click.echo('Version 1.0')
    ctx.exit()

@click.command()
@click.option('--version', is_flag=True, callback=print_version,
              expose_value=False, is_eager=True)
def hello():
    click.echo('Hello World!')
```

当然 `click` 提供了便捷的 `click.version_option` 来实现 `--version`：

```python
@click.command()
@click.version_option(version='0.1.0')
def hello():
    pass
```

#### 3.2.12 Yes 选项

基于前面的学习，我们可以实现 Yes 选项，也就是对于某些操作，不提供 `--yes` 则进行二次确认，提供了则直接操作：

```python
def abort_if_false(ctx, param, value):
    if not value:
        ctx.abort()

@click.command()
@click.option('--yes', is_flag=True, callback=abort_if_false,
              expose_value=False,
              prompt='Are you sure you want to drop the db?')
def dropdb():
    click.echo('Dropped all tables!')
```

当然 `click` 提供了便捷的 `click.confirmation_option` 来实现 Yes 选项：

```python
@click.command()
@click.confirmation_option(prompt='Are you sure you want to drop the db?')
def dropdb():
    click.echo('Dropped all tables!')
```

在命令行中调用则有：

```bash
$ dropdb
Are you sure you want to drop the db? [y/N]: n
Aborted!
$ dropdb --yes
Dropped all tables!
```

#### 3.2.13 其他增强功能

`click` 支持从环境中读取选项的值，这是 `argparse` 所不支持的，可参阅官方文档的 [Values from Environment Variables](https://click.palletsprojects.com/en/7.x/options/#values-from-environment-variables) 和 [Multiple Values from Environment Values](https://click.palletsprojects.com/en/7.x/options/#multiple-values-from-environment-values)。

`click` 支持指定选项前缀，你可以不使用 `-` 作为选项前缀，还可使用 `+` 或 `/`，当然在一般情况下并不建议这么做。详情参阅官方文档的 [Other Prefix Characters](https://click.palletsprojects.com/en/7.x/options/#other-prefix-characters)

### 3.3 命令和组

`Click` 中非常重要的特性就是任意嵌套命令行工具的概念，通过 [Command](https://click.palletsprojects.com/en/7.x/api/#click.Command) 和 [Group](https://click.palletsprojects.com/en/7.x/api/#click.Group) （实际上是 [MultiCommand](https://click.palletsprojects.com/en/7.x/api/#click.MultiCommand)）来实现。

所谓命令组就是若干个命令（或叫子命令）的集合，也成为多命令。

#### 3.3.1回调调用

对于一个普通的命令来说，回调发生在命令被执行的时候。如果这个程序的实现中只有命令，那么回调总是会被触发，就像我们在上一篇文章中举出的所有示例一样。不过像 `--help` 这类选项则会阻止进入回调。

对于组和多个子命令来说，情况略有不同。回调通常发生在子命令被执行的时候：

```python
@click.group()
@click.option('--debug/--no-debug', default=False)
def cli(debug):
    click.echo('Debug mode is %s' % ('on' if debug else 'off'))

@cli.command()  # @cli, not @click!
def sync():
    click.echo('Syncing')
```

执行效果如下：

```bash
Usage: tool.py [OPTIONS] COMMAND [ARGS]...

Options:
  --debug / --no-debug
  --help                Show this message and exit.

Commands:
  sync

$ tool.py --debug sync
Debug mode is on
Syncing
```

在上面的示例中，我们将函数 `cli` 定义为一个组，把函数 `sync` 定义为这个组内的子命令。当我们调用 `tool.py --debug sync` 命令时，会依次触发 `cli` 和 `sync` 的处理逻辑（也就是命令的回调）。

#### 3.3.2嵌套处理和上下文

从上面的例子可以看到，命令组 `cli` 接收的参数和子命令 `sync` 彼此独立。但是有时我们希望在子命令中能获取到命令组的参数，这就可以用 [Context](https://click.palletsprojects.com/en/7.x/api/#click.Context) 来实现。

每当命令被调用时，`click` 会创建新的上下文，并链接到父上下文。通常，我们是看不到上下文信息的。但我们可以通过 [pass_context](https://click.palletsprojects.com/en/7.x/api/#click.pass_context) 装饰器来显式让 `click` 传递上下文，此变量会作为第一个参数进行传递。

```python
@click.group()
@click.option('--debug/--no-debug', default=False)
@click.pass_context
def cli(ctx, debug):
    # 确保 ctx.obj 存在并且是个 dict。 (以防 `cli()` 指定 obj 为其他类型
    ctx.ensure_object(dict)

    ctx.obj['DEBUG'] = debug

@cli.command()
@click.pass_context
def sync(ctx):
    click.echo('Debug is %s' % (ctx.obj['DEBUG'] and 'on' or 'off'))

if __name__ == '__main__':
    cli(obj={})
```

在上面的示例中：

- 通过为命令组 `cli` 和子命令 `sync` 指定装饰器 `click.pass_context`，两个函数的第一个参数都是 `ctx` 上下文
- 在命令组 `cli` 中，给上下文的 `obj` 变量（字典）赋值
- 在子命令 `sync` 中通过 `ctx.obj['DEBUG']` 获得上一步的参数
- 通过这种方式完成了从命令组到子命令的参数传递

#### 3.3.3 不使用命令来调用命令组

默认情况下，调用子命令的时候才会调用命令组。而有时你可能想直接调用命令组，通过指定 `click.group` 的 `invoke_without_command=True` 来实现：

```python
@click.group(invoke_without_command=True)
@click.pass_context
def cli(ctx):
    if ctx.invoked_subcommand is None:
        click.echo('I was invoked without subcommand')
    else:
        click.echo('I am about to invoke %s' % ctx.invoked_subcommand)

@cli.command()
def sync():
    click.echo('The subcommand')
```

调用命令有：

```bash
$ tool
I was invoked without subcommand
$ tool sync
I am about to invoke sync
The subcommand
```

在上面的示例中，通过 `ctx.invoked_subcommand` 来判断是否由子命令触发，针对两种情况打印日志。

#### 3.3.4 自定义命令组/多命令

除了使用 [click.group](https://click.palletsprojects.com/en/7.x/api/#click.group) 来定义命令组外，你还可以自定义命令组（也就是多命令），这样你就可以延迟加载子命令，这会很有用。

自定义多命令需要实现 `list_commands` 和 `get_command` 方法：

```python
import click
import os

plugin_folder = os.path.join(os.path.dirname(__file__), 'commands')

class MyCLI(click.MultiCommand):

    def list_commands(self, ctx):
        rv = []  # 命令名称列表
        for filename in os.listdir(plugin_folder):
            if filename.endswith('.py'):
                rv.append(filename[:-3])
        rv.sort()
        return rv

    def get_command(self, ctx, name):
        ns = {}
        fn = os.path.join(plugin_folder, name + '.py')  # 命令对应的 Python 文件
        with open(fn) as f:
            code = compile(f.read(), fn, 'exec')
            eval(code, ns, ns)
        return ns['cli']

cli = MyCLI(help='This tool\'s subcommands are loaded from a '
            'plugin folder dynamically.')

# 等价方式是通过 click.command 装饰器，指定 cls=MyCLI
# @click.command(cls=MyCLI)
# def cli():
#     pass

if __name__ == '__main__':
    cli()
```

#### 3.3.5 合并命令组/多命令

当有多个命令组，每个命令组中有一些命令，你想把所有的命令合并在一个集合中时，`click.CommandCollection` 就派上了用场：

```python
@click.group()
def cli1():
    pass

@cli1.command()
def cmd1():
    """Command on cli1"""

@click.group()
def cli2():
    pass

@cli2.command()
def cmd2():
    """Command on cli2"""

cli = click.CommandCollection(sources=[cli1, cli2])

if __name__ == '__main__':
    cli()
```

调用命令有：

```bash
$ cli --help
Usage: cli [OPTIONS] COMMAND [ARGS]...

Options:
  --help  Show this message and exit.

Commands:
  cmd1  Command on cli1
  cmd2  Command on cli2
```

从上面的示例可以看出，`cmd1` 和 `cmd2` 分别属于 `cli1` 和 `cli2`，通过 `click.CommandCollection` 可以将这些子命令合并在一起，将其能力提供个同一个命令程序。

Tips：如果多个命令组中定义了同样的子命令，那么取第一个命令组中的子命令。

#### 3.3.6 链式命令组/多命令

有时单级子命令可能满足不了你的需求，你甚至希望能有多级子命令。典型地，`setuptools` 包中就支持多级/链式子命令： `setup.py sdist bdist_wheel upload`。在 click 3.0 之后，实现链式命令组变得非常简单，只需在 `click.group` 中指定 `chain=True`：

```python
@click.group(chain=True)
def cli():
    pass


@cli.command('sdist')
def sdist():
    click.echo('sdist called')


@cli.command('bdist_wheel')
def bdist_wheel():
    click.echo('bdist_wheel called')
```

调用命令则有：

```bash
$ setup.py sdist bdist_wheel
sdist called
bdist_wheel called
```

#### 3.3.7 命令组/多命令管道

链式命令组中一个常见的场景就是实现管道，这样在上一个命令处理好后，可将结果传给下一个命令处理。

实现命令组管道的要点是让每个命令返回一个处理函数，然后编写一个总的管道调度函数（并由 `MultiCommand.resultcallback()` 装饰）：

```python
@click.group(chain=True, invoke_without_command=True)
@click.option('-i', '--input', type=click.File('r'))
def cli(input):
    pass

@cli.resultcallback()
def process_pipeline(processors, input):
    iterator = (x.rstrip('\r\n') for x in input)
    for processor in processors:
        iterator = processor(iterator)
    for item in iterator:
        click.echo(item)

@cli.command('uppercase')
def make_uppercase():
    def processor(iterator):
        for line in iterator:
            yield line.upper()
    return processor

@cli.command('lowercase')
def make_lowercase():
    def processor(iterator):
        for line in iterator:
            yield line.lower()
    return processor

@cli.command('strip')
def make_strip():
    def processor(iterator):
        for line in iterator:
            yield line.strip()
    return processor
```

在上面的示例中：

- 将 `cli` 定义为了链式命令组，并且指定 invoke_without_command=True，也就意味着可以不传子命令来触发命令组
- 定义了三个命令处理函数，分别对应 `uppercase`、`lowercase` 和 `strip` 命令
- 在管道调度函数 `process_pipeline` 中，将输入 `input` 变成生成器，然后调用处理函数（实际输入几个命令，就有几个处理函数）进行处理

#### 3.3.8 覆盖默认值

默认情况下，参数的默认值是从通过装饰器参数 `default` 定义。我们还可以通过 `Context.default_map` 上下文字典来覆盖默认值：

```python
@click.group()
def cli():
    pass

@cli.command()
@click.option('--port', default=8000)
def runserver(port):
    click.echo('Serving on http://127.0.0.1:%d/' % port)

if __name__ == '__main__':
    cli(default_map={
        'runserver': {
            'port': 5000
        }
    })
```

在上面的示例中，通过在 `cli` 中指定 `default_map` 变可覆盖命令（一级键）的选项（二级键）默认值（二级键的值）。

我们还可以在 `click.group` 中指定 `context_settings` 来达到同样的目的：

```python
CONTEXT_SETTINGS = dict(
    default_map={'runserver': {'port': 5000}}
)

@click.group(context_settings=CONTEXT_SETTINGS)
def cli():
    pass

@cli.command()
@click.option('--port', default=8000)
def runserver(port):
    click.echo('Serving on http://127.0.0.1:%d/' % port)

if __name__ == '__main__':
    cli()
```

调用命令则有：

```bash
$ cli runserver
Serving on http://127.0.0.1:5000/
```

首先介绍了命令的回调调用、上下文，再进一步介绍命令组的自定义、合并、链接、管道等功能，了解到了 `click` 的强大。而命令组中更加高阶的能力（[如命令返回值](https://click.palletsprojects.com/en/7.x/commands/#command-return-values)）则可看官方文档进一步了解。

我们通过介绍 `click` 的参数、选项和命令已经能够完全实现命令行程序的所有功能。而 `click` 还为我们提供了许多锦上添花的功能，比如实用工具、参数自动补全等

## 4 click增强

### 4.1 Bash 补全

Bash 补全是 `click` 提供的一个非常便捷和强大的功能，这是它比 `argpase` 和 `docopt` 强大的一个表现。

在命令行程序正确安装后，Bash 补全才可以使用。而如何安装可以参考 [setup 集成](https://click.palletsprojects.com/en/7.x/setuptools/#setuptools-integration)。Click 目前仅支持 Bash 和 Zsh 的补全。

#### 4.1.1 补全能力

通常来说，Bash 补全支持对子命令、选项、以及选项或参数值得补全。比如：

```plain
$ repo <TAB><TAB>
clone    commit   copy     delete   setuser
$ repo clone -<TAB><TAB>
--deep     --help     --rev      --shallow  -r
```

此外，`click` 还支持自定义补全，这在动态生成补全场景中很有用，使用 `autocompletion` 参数。`autocompletion` 需要指定为一个回调函数，并且返回字符串的列表。此函数接受三个参数：

- `ctx` —— 当前的 click 上下文
- `args` 传入的参数列表
- `incomplete` 正在补全的词

这里有一个根据环境变量动态生成补全的示例：

```python
import os

def get_env_vars(ctx, args, incomplete):
    return [k for k in os.environ.keys() if incomplete in k]

@click.command()
@click.argument("envvar", type=click.STRING, autocompletion=get_env_vars)
def cmd1(envvar):
    click.echo('Environment variable: %s' % envvar)
    click.echo('Value: %s' % os.environ[envvar])
```

在 `ZSH` 中，还支持补全帮助信息。只需将 `autocompletion` 回调函数中返回的字符串列表中的字符串改为二元元组，第一个元素是补全内容，第二个元素是帮助信息。

这里有一个颜色补全的示例：

```python
import os

def get_colors(ctx, args, incomplete):
    colors = [('red', 'help string for the color red'),
              ('blue', 'help string for the color blue'),
              ('green', 'help string for the color green')]
    return [c for c in colors if incomplete in c[0]]

@click.command()
@click.argument("color", type=click.STRING, autocompletion=get_colors)
def cmd1(color):
    click.echo('Chosen color is %s' % color)
```

#### 4.1.2 激活补全

要激活 Bash 的补全功能，就需要告诉它你的命令行程序有补全的能力。通常通过一个神奇的环境变量 `__COMPLETE` 来告知，其中 `` 是大写下划线形式的程序名称。

比如有一个命令行程序叫做 `foo-bar`，那么对应的环境变量名称为 `_FOO_BAR_COMPLETE`，然后在 `.bashrc` 中使用 `source` 导出即可：

```bash
eval "$(_FOO_BAR_COMPLETE=source foo-bar)"
```

或者在 `.zshrc` 中使用：

```bash
eval "$(_FOO_BAR_COMPLETE=source_zsh foo-bar)"
```

不过上面的方式总是在命令行程序启动时调用，这可能在有多个程序时减慢 shell 激活的速度。另一种方式是把命令放在文件中，就像这样：

```bash
# 针对 Bash
_FOO_BAR_COMPLETE=source foo-bar > foo-bar-complete.sh

# 针对 ZSH
_FOO_BAR_COMPLETE=source_zsh foo-bar > foo-bar-complete.sh
```

然后把脚本文件路径加到 `.bashrc` 或 `.zshrc` 中：

```bash
. /path/to/foo-bar-complete.sh
```

### .2 实用工具

#### 4.2.1 打印到标准输出

[echo()](https://click.palletsprojects.com/en/7.x/api/#click.echo) 函数可以说是最有用的实用工具了。它和 Python 的 `print` 类似，主要的区别在于它同时在 Python 2 和 3 中生效，能够智能地检测未配置正确的输出流，且几乎不会失败（除了 Python 3 中的[少数限制](https://click.palletsprojects.com/en/7.x/python3/#python3-limitations)。）

`echo` 即支持 unicode，也支持二级制数据，如：

```python
import click

click.echo('Hello World!')

click.echo(b'\xe2\x98\x83', nl=False) # nl=False 表示不输出换行符
```

#### 4.2.2 ANSI 颜色

有些时候你可能希望输出是有颜色的，这尤其在输出错误信息时有用，而 `click` 在这方面支持的很好。

首先，你需要安装 `colorama`：

```bash
pip install colorama
```

然后，就可以使用 [style()](https://click.palletsprojects.com/en/7.x/api/#click.style) 函数来指定颜色：

```python
import click

click.echo(click.style('Hello World!', fg='green'))
click.echo(click.style('Some more text', bg='blue', fg='white'))
click.echo(click.style('ATTENTION', blink=True, bold=True))
```

`click` 还提供了更加简便的函数 [secho](https://click.palletsprojects.com/en/7.x/api/#click.secho)，它就是 `echo` 和 `style` 的组合：

```python
click.secho('Hello World!', fg='green')
click.secho('Some more text', bg='blue', fg='white')
click.secho('ATTENTION', blink=True, bold=True)
```

#### .2.3 分页支持

有些时候，命令行程序会输出长文本，但你希望能让用户盘也浏览。使用 [echo_via_pager()](https://click.palletsprojects.com/en/7.x/api/#click.echo_via_pager) 函数就可以轻松做到。

例如：

```python
def less():
    click.echo_via_pager('\n'.join('Line %d' % idx
                                   for idx in range(200)))
```

如果输出的文本特别大，处于性能的考虑，希望翻页时生成对应内容，那么就可以使用生成器：

```python
def _generate_output():
    for idx in range(50000):
        yield "Line %d\n" % idx

@click.command()
def less():
    click.echo_via_pager(_generate_output())
```

#### 4.2.4 清除屏幕

使用 [clear()](https://click.palletsprojects.com/en/7.x/api/#click.clear) 可以轻松清除屏幕内容：

```python
import click
click.clear()
```

#### .2.5 从终端获取字符

通常情况下，使用内建函数 `input` 或 `raw_input` 获得的输入是用户输出一段字符然后回车得到的。但在有些场景下，你可能想在用户输入单个字符时就能获取到并且做一定的处理，这个时候 [getchar()](https://click.palletsprojects.com/en/7.x/api/#click.getchar) 就派上了用场。

比如，根据输入的 `y` 或 `n` 做特定处理：

```python
import click

click.echo('Continue? [yn] ', nl=False)
c = click.getchar()
click.echo()
if c == 'y':
    click.echo('We will go on')
elif c == 'n':
    click.echo('Abort!')
else:
    click.echo('Invalid input :(')
```

#### 4.2.6 等待按键

在 Windows 的 cmd 中我们经常看到当执行完一个命令后，提示按下任意键退出。通过使用 [pause()](https://click.palletsprojects.com/en/7.x/api/#click.pause) 可以实现暂停直至用户按下任意键：

```python
import click
click.pause()
```

#### 4.2.7 启动编辑器

通过 [edit()](https://click.palletsprojects.com/en/7.x/api/#click.edit) 可以自动启动编辑器。这在需要用户输入多行内容时十分有用。

在下面的示例中，会启动默认的文本编辑器，并在里面输入一段话：

```python
import click

def get_commit_message():
    MARKER = '# Everything below is ignored\n'
    message = click.edit('\n\n' + MARKER)
    if message is not None:
        return message.split(MARKER, 1)[0].rstrip('\n')
```

`edit()` 函数还支持打开特定文件，比如：

```python
import click
click.edit(filename='/etc/passwd')
```

#### .2.8 启动应用程序

通过 [launch](https://click.palletsprojects.com/en/7.x/api/#click.launch) 可以打开 URL 或文件类型所关联的默认应用程序。如果设置 `locate=True`，则可以启动文件管理器并自动选中特定文件。

示例：

```python
# 打开浏览器，访问 URL
click.launch("https://click.palletsprojects.com/")

# 使用默认应用程序打开 txt 文件
click.launch("/my/downloaded/file.txt")

# 打开文件管理器，并自动选中 file.txt
click.launch("/my/downloaded/file.txt", locate=True)
```

#### 4.2.9 显示进度条

`click` 内置了 [progressbar()](https://click.palletsprojects.com/en/7.x/api/#click.progressbar) 函数来方便地显示进度条。

它的用法也很简单，假定你有一个要处理的可迭代对象，处理完每一项就要输出一下进度，那么就有两种用法。

用法一：使用 `progressbar` 构造出 `bar` 对象，迭代 `bar` 对象来自动告知进度：

```python
import time
import click

all_the_users_to_process = ['a', 'b', 'c']

def modify_the_user(user):
    time.sleep(0.5)

with click.progressbar(all_the_users_to_process) as bar:
    for user in bar:
        modify_the_user(user)
```

用法二：使用 `progressbar` 构造出 `bar` 对象，迭代原始可迭代对象，并不断向 `bar` 更新进度：

```python
import time
import click

all_the_users_to_process = ['a', 'b', 'c']

def modify_the_user(user):
    time.sleep(0.5)

with click.progressbar(all_the_users_to_process) as bar:
    for user in enumerate(all_the_users_to_process):
        modify_the_user(user)
        bar.update(1)
```

#### 4.2.10 更多实用工具

- [打印文件名](https://click.palletsprojects.com/en/7.x/utils/#printing-filenames)
- [标准流](https://click.palletsprojects.com/en/7.x/utils/#standard-streams)
- [智能打开文件](https://click.palletsprojects.com/en/7.x/utils/#intelligent-file-opening)
- [查找应用程序文件夹](https://click.palletsprojects.com/en/7.x/utils/#finding-application-folders)

## 5 总结

`click` 提供了非常多的增强型功能，本文着重介绍了它的 Bash 补全和十多个实用工具，这会让你在实现命令行的过程中如虎添翼。此外，`click` 还提供了诸如命令别名、参数修改、标准化令牌、调用其他命令、回调顺序等诸多[高级模式](https://click.palletsprojects.com/en/7.x/advanced/) 以应对更加复杂或特定的场景，我就不再深入介绍。

## 6 实战

## 6.1 使用click实现调用Java的jar包

```python
#!/usr/bin/python
#coding:utf-8
import click
import os
import getpass

# java -classpath table-meta-1.0-SNAPSHOT.jar com.lx.dw.utils.Runner -m lineage -r /Users/manzhong/Downloads/warehouse-master/dol
# "/data/dwdev/job-manager-1.0-SNAPSHOT-jar-with-dependencies.jar"
jar = "/data/dwdev/job-manager-1.0-SNAPSHOT-jar-with-dependencies.jar"
classname = "com.lx.dw.job.Runner"


jartab = "/data/dwdev/table-meta-1.0-SNAPSHOT.jar"
classnametab = "com.lx.dw.utils.Runner"


hdfspath={'lx':'/user/dwdev/sqlScript','wm':'/user/dwdev/wmsqlScript'}
gitpath={'lx':'dol','wm':'wmdol'}
project={'lx':'warehouse/dol/','wm':'warehouse/wmdol/'}
db={'lx':'dw','wm':'wmdw'}
@click.group()
def begin():
    pass

@click.command()
@click.option("--filename","-f",default="git filename",prompt='INPUT FILENAME:',help="文件名")
@click.option("--projecttype","-s",type=click.Choice(['lx', 'wm'], case_sensitive=False),prompt='INPUT wm or lx:',help="项目类型")
@click.option("--type","-g",default=False,type=bool,prompt='Input true or false(Publish to hdfs or not):',help="是否上传")
@click.option("--parse","-p",default=True,type=bool,prompt='Input true or false(Parse  blood or not):',help="是否解析血缘关系")
def publish(filename,type,projecttype,parse):
    """This script:git to hdfs"""
    if parse==True:
        dirs="/home/"+getpass.getuser()+"/"
        indb=True
        bloodparsedef(dirs,projecttype,filename,indb)
    if type==True:
        print("dwtools publish --filename=" + "{}".format(filename) + " --projecttype=" + "{}".format(projecttype) +" --type=true")
        command = "java -classpath"+" "+ jar +" " +classname
        arg0="-m"
        arg1="download"
        arg2="-f"
        arg3="{}".format(filename)
        arg4="-k"
        arg5=os.getenv('TOKEN')
        arg6="-s"
        arg7=hdfspath[projecttype]
        arg8="-i"
        arg9=gitpath[projecttype]
        arg10="-g"
        cmd = [command,arg0,arg1,arg2,arg3,arg4,arg5,arg6,arg7,arg8,arg9,arg10]
        new_cmd = " ".join(cmd)
        f = os.popen(new_cmd)
        d = f.read()
        print(d)
        f.close
    else:
        print("dwtools publish --filename=" + "{}".format(filename) + " --projecttype=" + "{}".format(projecttype) + " --type=false")
        command = "java -classpath"+" "+ jar +" " +classname
        arg0="-m"
        arg1="download"
        arg2="-f"
        arg3="{}".format(filename)
        arg4="-k"
        arg5=os.getenv('TOKEN')
        arg6="-s"
        arg7=hdfspath[projecttype]
        arg8="-i"
        arg9=gitpath[projecttype]
        cmd = [command,arg0,arg1,arg2,arg3,arg4,arg5,arg6,arg7,arg8,arg9]
        new_cmd = " ".join(cmd)
        f = os.popen(new_cmd)
        d = f.read()
        print(d)
        f.close
# sql测试运行

@click.command()
@click.option("--dir","-d",default="/home/"+getpass.getuser()+"/",prompt='Personal home directory',help="用户主目录")
@click.option("--projecttype","-s",type=click.Choice(['lx', 'wm'], case_sensitive=False),prompt='INPUT wm or lx:',help="项目类型")
@click.option("--filename","-f",prompt='Hql File[Example xxx.sql]:',help="文件名:")
@click.option('--message', '-p',prompt="Input parameters,[Example k:v,k:v]:",help="参数list")
@click.option("--engine","-e",type=click.Choice(['spark','hive'],case_sensitive=False),prompt='INPUT spark or hive:',help="执行sparksql或者hivesql")
@click.option("--type","-g",default=False,type=bool,prompt='Input true or false(Run task or only check syntax):',help="测试环境运行 or 语法检测")
def runJob(dir,projecttype,filename,message,engine,type):
    """This script:sql解析"""
    command = "java -classpath"+" "+ jar +" " +classname
    localpath="{}".format(dir)+project[projecttype]+"{}".format(filename)
    print (localpath)
    if type==True:
        print("dwtools runjob --dir=" + "{}".format(dir) + " --projecttype=" + "{}".format(projecttype) + " --filename=" + "{}".format(filename) +" --message="+"{}".format(message) + " --engine=" + "{}".format(engine) +  " --type=true")
        arg0="-m"
        arg1="parse"
        arg2="-l"
        arg3=localpath
        arg4="-p"
        arg5='{}'.format(message)
        arg6="-e"
        arg7='{}'.format(engine)
        arg8="-g"
        cmd = [command,arg0,arg1,arg2,arg3,arg4,arg5,arg6,arg7,arg8]
        new_cmd = " ".join(cmd)
        f = os.popen(new_cmd)
        d = f.read()
        print(d)
        f.close
    else:
        # properties=','.join(message)
        print("dwtools runjob --dir=" + "{}".format(dir) + " --projecttype=" + "{}".format(projecttype) + " --filename=" + "{}".format(filename) + " --message=" + "{}".format(message) + " --engine=" + "{}".format(engine) + " --type=false")
        arg0="-m"
        arg1="parse"
        arg2="-l"
        arg3=localpath
        arg4="-p"
        arg5='{}'.format(message)
        arg6="-e"
        arg7='{}'.format(engine)
        cmd = [command,arg0,arg1,arg2,arg3,arg4,arg5,arg6,arg7]
        new_cmd = " ".join(cmd)
        f = os.popen(new_cmd)
        d = f.read()
        print(d)
        f.close

@click.command()
@click.option("--dir","-d",default="/home/"+getpass.getuser()+"/",prompt='Personal home directory',help="用户主目录")
@click.option("--projecttype","-s",type=click.Choice(['lx', 'wm'], case_sensitive=False),prompt='INPUT wm or lx:',help="项目类型")
@click.option("--filename","-f",prompt='Hql File[Example xxx.sql]:',help="文件名:")
@click.option("--indb",default=True,type=bool,prompt='Input true or false(insert to mysql or only display result ):',help="是否入库 or 只显示结果")
def bloodparse(dir,projecttype,filename,indb):
    """This script:血缘关系解析"""
    command = "java -classpath"+" "+ jartab +" " +classnametab
    parsepath="{}".format(dir)+project[projecttype]+"{}".format(filename)
    if os.path.isdir(parsepath):
        if indb==True:
            print("dwtools bloodparse --dir=" + "{}".format(dir) + " --projecttype=" + "{}".format(projecttype) + " --filename=" + "{}".format(filename) +  " --indb=true")
            arg0="-m"
            arg1="lineage"
            arg2="-r"
            arg3=parsepath
            arg4="-g"
            arg5="-d"
            arg6=db[projecttype]
            cmd = [command,arg0,arg1,arg2,arg3,arg4,arg5,arg6]
        else:
            print("dwtools bloodparse --dir=" + "{}".format(dir) + " --projecttype=" + "{}".format(projecttype) + " --filename=" + "{}".format(filename) + " --indb=false")
            arg0="-m"
            arg1="lineage"
            arg2="-r"
            arg3=parsepath
            arg4="-d"
            arg5=db[projecttype]
            cmd = [command,arg0,arg1,arg2,arg3,arg4,arg5]
    else:
        if indb==True:
            print("dwtools bloodparse --dir=" + "{}".format(dir) + " --projecttype=" + "{}".format(projecttype) + " --filename=" + "{}".format(filename) + " --indb=true")
            arg0="-m"
            arg1="lineage"
            arg2="-f"
            arg3=parsepath
            arg4="-g"
            arg5="-d"
            arg6=db[projecttype]
            cmd = [command,arg0,arg1,arg2,arg3,arg4,arg5,arg6]
        else:
            print("dwtools bloodparse --dir=" + "{}".format(dir) + " --projecttype=" + "{}".format(projecttype) + " --filename=" + "{}".format(filename) + " --indb=false")
            arg0="-m"
            arg1="lineage"
            arg2="-f"
            arg3=parsepath
            arg4="-d"
            arg5=db[projecttype]
            cmd = [command,arg0,arg1,arg2,arg3,arg4,arg5]
    new_cmd = " ".join(cmd)
    f = os.popen(new_cmd)
    d = f.read()
    print(d)
    f.close



#  "-m","table","-d","dw","-t","mid_health_effective_movement_aggr_user_ss","-s","health","-o","yujing.zou"
@click.command()
@click.option("--db",type=click.Choice(['dw', 'wmdw'], case_sensitive=False),prompt='INPUT DB:',help="库名")
@click.option("--tablename",default="table name",prompt='INPUT TABLE NAME:',help="表名")
@click.option("--topicname",default="topic name",prompt='INPUT TOPIC NAME',help="主题名")
@click.option("--indb",default=True,type=bool,prompt='Input true or false(insert to mysql or only display result ):',help="是否入库 or 只显示结果")
@click.option("--on",default="yujing.zou",prompt='INPUT First responsible person:',help="第一责任人")
@click.option("--bn",default="hongfei.yang",prompt='INPUT Second responsible person:',help="第二责任人")
def metastore(db,tablename,topicname,indb,on,bn):
    """This script:元数据录入"""
    command = "java -classpath"+" "+ jartab +" " +classnametab
    if indb==True:
        print("dwtools metastore --db=" + "{}".format(db) + " --tablename=" + "{}".format(
            tablename) + " --topicname=" + "{}".format(topicname) + " --indb=true"+" --on="+"{}".format(on)+" --bn="+"{}".format(bn))
        arg0="-m"
        arg1="table"
        arg2="-d"
        arg3="{}".format(db)
        arg4="-t"
        arg5="{}".format(tablename)
        arg6="-s"
        arg7="{}".format(topicname)
        arg8="-g"
        arg9="-o"
        arg10="{}".format(on)
        arg11="-b"
        arg12="{}".format(bn)
        cmd = [command,arg0,arg1,arg2,arg3,arg4,arg5,arg6,arg7,arg8,arg9,arg10,arg11,arg12]
    else:
        print("dwtools metastore --db=" + "{}".format(db) + " --tablename=" + "{}".format(
            tablename) + " --topicname=" + "{}".format(topicname) + " --indb=false" + " --on=" + "{}".format(
            on) + " --bn=" + "{}".format(bn))
        arg0="-m"
        arg1="table"
        arg2="-d"
        arg3="{}".format(db)
        arg4="-t"
        arg5="{}".format(tablename)
        arg6="-s"
        arg7="{}".format(topicname)
        arg8="-o"
        arg9="{}".format(on)
        arg10="-b"
        arg11="{}".format(bn)
        cmd = [command,arg0,arg1,arg2,arg3,arg4,arg5,arg6,arg7,arg8,arg9,arg10,arg11]
    new_cmd = " ".join(cmd)
    f = os.popen(new_cmd)
    d = f.read()
    print(d)
    f.close




def bloodparsedef(dir,projecttype,filename,indb):
    """This script:血缘关系解析"""
    command = "java -classpath"+" "+ jartab +" " +classnametab
    parsepath="{}".format(dir)+project[projecttype]+"{}".format(filename)
    if os.path.isdir(parsepath):
        if indb==True:
            print("dwtools bloodparse --dir=" + "{}".format(dir) + " --projecttype=" + "{}".format(projecttype) + " --filename=" + "{}".format(filename) +  " --indb=true")
            arg0="-m"
            arg1="lineage"
            arg2="-r"
            arg3=parsepath
            arg4="-g"
            arg5="-d"
            arg6=db[projecttype]
            cmd = [command,arg0,arg1,arg2,arg3,arg4,arg5,arg6]
        else:
            print("dwtools bloodparse --dir=" + "{}".format(dir) + " --projecttype=" + "{}".format(projecttype) + " --filename=" + "{}".format(filename) + " --indb=false")
            arg0="-m"
            arg1="lineage"
            arg2="-r"
            arg3=parsepath
            arg4="-d"
            arg5=db[projecttype]
            cmd = [command,arg0,arg1,arg2,arg3,arg4,arg5]
    else:
        if indb==True:
            print("dwtools bloodparse --dir=" + "{}".format(dir) + " --projecttype=" + "{}".format(projecttype) + " --filename=" + "{}".format(filename) + " --indb=true")
            arg0="-m"
            arg1="lineage"
            arg2="-f"
            arg3=parsepath
            arg4="-g"
            arg5="-d"
            arg6=db[projecttype]
            cmd = [command,arg0,arg1,arg2,arg3,arg4,arg5,arg6]
        else:
            print("dwtools bloodparse --dir=" + "{}".format(dir) + " --projecttype=" + "{}".format(projecttype) + " --filename=" + "{}".format(filename) + " --indb=false")
            arg0="-m"
            arg1="lineage"
            arg2="-f"
            arg3=parsepath
            arg4="-d"
            arg5=db[projecttype]
            cmd = [command,arg0,arg1,arg2,arg3,arg4,arg5]
    new_cmd = " ".join(cmd)
    f = os.popen(new_cmd)
    d = f.read()
    print(d)
    f.close

begin.add_command(publish)
begin.add_command(runJob)
begin.add_command(bloodparse)
begin.add_command(metastore)
begin()#!/usr/bin/python
#coding:utf-8
import click
import os
import getpass

# java -classpath table-meta-1.0-SNAPSHOT.jar com.lx.dw.utils.Runner -m lineage -r /Users/manzhong/Downloads/warehouse-master/dol
# "/data/dwdev/job-manager-1.0-SNAPSHOT-jar-with-dependencies.jar"
jar = "/data/dwdev/job-manager-1.0-SNAPSHOT-jar-with-dependencies.jar"
classname = "com.lx.dw.job.Runner"


jartab = "/data/dwdev/table-meta-1.0-SNAPSHOT.jar"
classnametab = "com.lx.dw.utils.Runner"


hdfspath={'lx':'/user/dwdev/sqlScript','wm':'/user/dwdev/wmsqlScript'}
gitpath={'lx':'dol','wm':'wmdol'}
project={'lx':'warehouse/dol/','wm':'warehouse/wmdol/'}
db={'lx':'dw','wm':'wmdw'}
@click.group()
def begin():
    pass

@click.command()
@click.option("--filename","-f",default="git filename",prompt='INPUT FILENAME:',help="文件名")
@click.option("--projecttype","-s",type=click.Choice(['lx', 'wm'], case_sensitive=False),prompt='INPUT wm or lx:',help="项目类型")
@click.option("--type","-g",default=False,type=bool,prompt='Input true or false(Publish to hdfs or not):',help="是否上传")
@click.option("--parse","-p",default=True,type=bool,prompt='Input true or false(Parse  blood or not):',help="是否解析血缘关系")
def publish(filename,type,projecttype,parse):
    """This script:git to hdfs"""
    if parse==True:
        dirs="/home/"+getpass.getuser()+"/"
        indb=True
        bloodparsedef(dirs,projecttype,filename,indb)
    if type==True:
        print("dwtools publish --filename=" + "{}".format(filename) + " --projecttype=" + "{}".format(projecttype) +" --type=true")
        command = "java -classpath"+" "+ jar +" " +classname
        arg0="-m"
        arg1="download"
        arg2="-f"
        arg3="{}".format(filename)
        arg4="-k"
        arg5=os.getenv('TOKEN')
        arg6="-s"
        arg7=hdfspath[projecttype]
        arg8="-i"
        arg9=gitpath[projecttype]
        arg10="-g"
        cmd = [command,arg0,arg1,arg2,arg3,arg4,arg5,arg6,arg7,arg8,arg9,arg10]
        new_cmd = " ".join(cmd)
        f = os.popen(new_cmd)
        d = f.read()
        print(d)
        f.close
    else:
        print("dwtools publish --filename=" + "{}".format(filename) + " --projecttype=" + "{}".format(projecttype) + " --type=false")
        command = "java -classpath"+" "+ jar +" " +classname
        arg0="-m"
        arg1="download"
        arg2="-f"
        arg3="{}".format(filename)
        arg4="-k"
        arg5=os.getenv('TOKEN')
        arg6="-s"
        arg7=hdfspath[projecttype]
        arg8="-i"
        arg9=gitpath[projecttype]
        cmd = [command,arg0,arg1,arg2,arg3,arg4,arg5,arg6,arg7,arg8,arg9]
        new_cmd = " ".join(cmd)
        f = os.popen(new_cmd)
        d = f.read()
        print(d)
        f.close
# sql测试运行

@click.command()
@click.option("--dir","-d",default="/home/"+getpass.getuser()+"/",prompt='Personal home directory',help="用户主目录")
@click.option("--projecttype","-s",type=click.Choice(['lx', 'wm'], case_sensitive=False),prompt='INPUT wm or lx:',help="项目类型")
@click.option("--filename","-f",prompt='Hql File[Example xxx.sql]:',help="文件名:")
@click.option('--message', '-p',prompt="Input parameters,[Example k:v,k:v]:",help="参数list")
@click.option("--engine","-e",type=click.Choice(['spark','hive'],case_sensitive=False),prompt='INPUT spark or hive:',help="执行sparksql或者hivesql")
@click.option("--type","-g",default=False,type=bool,prompt='Input true or false(Run task or only check syntax):',help="测试环境运行 or 语法检测")
def runJob(dir,projecttype,filename,message,engine,type):
    """This script:sql解析"""
    command = "java -classpath"+" "+ jar +" " +classname
    localpath="{}".format(dir)+project[projecttype]+"{}".format(filename)
    print (localpath)
    if type==True:
        print("dwtools runjob --dir=" + "{}".format(dir) + " --projecttype=" + "{}".format(projecttype) + " --filename=" + "{}".format(filename) +" --message="+"{}".format(message) + " --engine=" + "{}".format(engine) +  " --type=true")
        arg0="-m"
        arg1="parse"
        arg2="-l"
        arg3=localpath
        arg4="-p"
        arg5='{}'.format(message)
        arg6="-e"
        arg7='{}'.format(engine)
        arg8="-g"
        cmd = [command,arg0,arg1,arg2,arg3,arg4,arg5,arg6,arg7,arg8]
        new_cmd = " ".join(cmd)
        f = os.popen(new_cmd)
        d = f.read()
        print(d)
        f.close
    else:
        # properties=','.join(message)
        print("dwtools runjob --dir=" + "{}".format(dir) + " --projecttype=" + "{}".format(projecttype) + " --filename=" + "{}".format(filename) + " --message=" + "{}".format(message) + " --engine=" + "{}".format(engine) + " --type=false")
        arg0="-m"
        arg1="parse"
        arg2="-l"
        arg3=localpath
        arg4="-p"
        arg5='{}'.format(message)
        arg6="-e"
        arg7='{}'.format(engine)
        cmd = [command,arg0,arg1,arg2,arg3,arg4,arg5,arg6,arg7]
        new_cmd = " ".join(cmd)
        f = os.popen(new_cmd)
        d = f.read()
        print(d)
        f.close

@click.command()
@click.option("--dir","-d",default="/home/"+getpass.getuser()+"/",prompt='Personal home directory',help="用户主目录")
@click.option("--projecttype","-s",type=click.Choice(['lx', 'wm'], case_sensitive=False),prompt='INPUT wm or lx:',help="项目类型")
@click.option("--filename","-f",prompt='Hql File[Example xxx.sql]:',help="文件名:")
@click.option("--indb",default=True,type=bool,prompt='Input true or false(insert to mysql or only display result ):',help="是否入库 or 只显示结果")
def bloodparse(dir,projecttype,filename,indb):
    """This script:血缘关系解析"""
    command = "java -classpath"+" "+ jartab +" " +classnametab
    parsepath="{}".format(dir)+project[projecttype]+"{}".format(filename)
    if os.path.isdir(parsepath):
        if indb==True:
            print("dwtools bloodparse --dir=" + "{}".format(dir) + " --projecttype=" + "{}".format(projecttype) + " --filename=" + "{}".format(filename) +  " --indb=true")
            arg0="-m"
            arg1="lineage"
            arg2="-r"
            arg3=parsepath
            arg4="-g"
            arg5="-d"
            arg6=db[projecttype]
            cmd = [command,arg0,arg1,arg2,arg3,arg4,arg5,arg6]
        else:
            print("dwtools bloodparse --dir=" + "{}".format(dir) + " --projecttype=" + "{}".format(projecttype) + " --filename=" + "{}".format(filename) + " --indb=false")
            arg0="-m"
            arg1="lineage"
            arg2="-r"
            arg3=parsepath
            arg4="-d"
            arg5=db[projecttype]
            cmd = [command,arg0,arg1,arg2,arg3,arg4,arg5]
    else:
        if indb==True:
            print("dwtools bloodparse --dir=" + "{}".format(dir) + " --projecttype=" + "{}".format(projecttype) + " --filename=" + "{}".format(filename) + " --indb=true")
            arg0="-m"
            arg1="lineage"
            arg2="-f"
            arg3=parsepath
            arg4="-g"
            arg5="-d"
            arg6=db[projecttype]
            cmd = [command,arg0,arg1,arg2,arg3,arg4,arg5,arg6]
        else:
            print("dwtools bloodparse --dir=" + "{}".format(dir) + " --projecttype=" + "{}".format(projecttype) + " --filename=" + "{}".format(filename) + " --indb=false")
            arg0="-m"
            arg1="lineage"
            arg2="-f"
            arg3=parsepath
            arg4="-d"
            arg5=db[projecttype]
            cmd = [command,arg0,arg1,arg2,arg3,arg4,arg5]
    new_cmd = " ".join(cmd)
    f = os.popen(new_cmd)
    d = f.read()
    print(d)
    f.close



#  "-m","table","-d","dw","-t","mid_health_effective_movement_aggr_user_ss","-s","health","-o","yujing.zou"
@click.command()
@click.option("--db",type=click.Choice(['dw', 'wmdw'], case_sensitive=False),prompt='INPUT DB:',help="库名")
@click.option("--tablename",default="table name",prompt='INPUT TABLE NAME:',help="表名")
@click.option("--topicname",default="topic name",prompt='INPUT TOPIC NAME',help="主题名")
@click.option("--indb",default=True,type=bool,prompt='Input true or false(insert to mysql or only display result ):',help="是否入库 or 只显示结果")
@click.option("--on",default="yujing.zou",prompt='INPUT First responsible person:',help="第一责任人")
@click.option("--bn",default="hongfei.yang",prompt='INPUT Second responsible person:',help="第二责任人")
def metastore(db,tablename,topicname,indb,on,bn):
    """This script:元数据录入"""
    command = "java -classpath"+" "+ jartab +" " +classnametab
    if indb==True:
        print("dwtools metastore --db=" + "{}".format(db) + " --tablename=" + "{}".format(
            tablename) + " --topicname=" + "{}".format(topicname) + " --indb=true"+" --on="+"{}".format(on)+" --bn="+"{}".format(bn))
        arg0="-m"
        arg1="table"
        arg2="-d"
        arg3="{}".format(db)
        arg4="-t"
        arg5="{}".format(tablename)
        arg6="-s"
        arg7="{}".format(topicname)
        arg8="-g"
        arg9="-o"
        arg10="{}".format(on)
        arg11="-b"
        arg12="{}".format(bn)
        cmd = [command,arg0,arg1,arg2,arg3,arg4,arg5,arg6,arg7,arg8,arg9,arg10,arg11,arg12]
    else:
        print("dwtools metastore --db=" + "{}".format(db) + " --tablename=" + "{}".format(
            tablename) + " --topicname=" + "{}".format(topicname) + " --indb=false" + " --on=" + "{}".format(
            on) + " --bn=" + "{}".format(bn))
        arg0="-m"
        arg1="table"
        arg2="-d"
        arg3="{}".format(db)
        arg4="-t"
        arg5="{}".format(tablename)
        arg6="-s"
        arg7="{}".format(topicname)
        arg8="-o"
        arg9="{}".format(on)
        arg10="-b"
        arg11="{}".format(bn)
        cmd = [command,arg0,arg1,arg2,arg3,arg4,arg5,arg6,arg7,arg8,arg9,arg10,arg11]
    new_cmd = " ".join(cmd)
    f = os.popen(new_cmd)
    d = f.read()
    print(d)
    f.close




def bloodparsedef(dir,projecttype,filename,indb):
    """This script:血缘关系解析"""
    command = "java -classpath"+" "+ jartab +" " +classnametab
    parsepath="{}".format(dir)+project[projecttype]+"{}".format(filename)
    if os.path.isdir(parsepath):
        if indb==True:
            print("dwtools bloodparse --dir=" + "{}".format(dir) + " --projecttype=" + "{}".format(projecttype) + " --filename=" + "{}".format(filename) +  " --indb=true")
            arg0="-m"
            arg1="lineage"
            arg2="-r"
            arg3=parsepath
            arg4="-g"
            arg5="-d"
            arg6=db[projecttype]
            cmd = [command,arg0,arg1,arg2,arg3,arg4,arg5,arg6]
        else:
            print("dwtools bloodparse --dir=" + "{}".format(dir) + " --projecttype=" + "{}".format(projecttype) + " --filename=" + "{}".format(filename) + " --indb=false")
            arg0="-m"
            arg1="lineage"
            arg2="-r"
            arg3=parsepath
            arg4="-d"
            arg5=db[projecttype]
            cmd = [command,arg0,arg1,arg2,arg3,arg4,arg5]
    else:
        if indb==True:
            print("dwtools bloodparse --dir=" + "{}".format(dir) + " --projecttype=" + "{}".format(projecttype) + " --filename=" + "{}".format(filename) + " --indb=true")
            arg0="-m"
            arg1="lineage"
            arg2="-f"
            arg3=parsepath
            arg4="-g"
            arg5="-d"
            arg6=db[projecttype]
            cmd = [command,arg0,arg1,arg2,arg3,arg4,arg5,arg6]
        else:
            print("dwtools bloodparse --dir=" + "{}".format(dir) + " --projecttype=" + "{}".format(projecttype) + " --filename=" + "{}".format(filename) + " --indb=false")
            arg0="-m"
            arg1="lineage"
            arg2="-f"
            arg3=parsepath
            arg4="-d"
            arg5=db[projecttype]
            cmd = [command,arg0,arg1,arg2,arg3,arg4,arg5]
    new_cmd = " ".join(cmd)
    f = os.popen(new_cmd)
    d = f.read()
    print(d)
    f.close

begin.add_command(publish)
begin.add_command(runJob)
begin.add_command(bloodparse)
begin.add_command(metastore)
begin()
```

## 7 将python写的命令行打包,并安装到linux上

## 1 安装setuptools

```
pip install setuptools
```

更新:

```
pip install --upgrade setuptools
```

## 2 新建python的项目

目录结构:

```
--python-dev
		--dwtools(文件夹)
				--__init__.py(空)
				--命令.py
		--setup.py
```

Setup.py:

```python
#coding:utf-8
from setuptools import setup,find_packages

setup(
    name = 'dwdevtools',
    version ='1.0',
    author = 'hongfei.yang',
    packages=find_packages(),
    include_package_data=True,
    description = '数据仓库开发系统',
    install_requires=[
        'Click',
    ],
    entry_points='''
        [console_scripts]
        dwtools=dwdevtools.dwtools:begin
    ''',

)
```

## 3 打包

```python
 python setup.py bdist_egg
```

## 4 Linux安装egg包

```python
easy_install *.egg 
```





