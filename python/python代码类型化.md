# python代码类型化

python是一门强类型的的动态语言，在运行的时候值的类型是确定的，但是之前很长一段时间，我们在写python代码的时候，尤其是接手别人代码的时候，大部分情况下我们是不知道这个值到底是什么类型，是int还是str，如果是dict，那么这个dict的key是什么类型，key的参数名称是什么，value的类型是什么，这些具体的信息随着人员的更替，到最后就全靠摸索，给开发带来了很大的困扰。

如何让python代码更加可读，这个不同的人可能有不同的理解，也可能会有不同的方法，比如写文档，多写注释此类的，这个看团队的情况来决定吧，只要是大家能认可且能落实执行下去的，就是好方法。

下面我会介绍我目前我在使用的方法，也是目前开源社区python项目大家都在用的方式，[Type Hints](https://www.python.org/dev/peps/pep-0484/) + [Mypy](https://github.com/python/mypy)

## Type Hints

Type Hints俗称类型注释，是python3.5引入进来的语法，使用方式也比较简单

1. 对于所有变量和方法形参, 语法如下

```python
a: int = 1
b: str = "b"

def demo(c: int, d: str):
    pass
```

可以看到只是名称后面加`name: {type}`

2. 对于方法返回值

```python

def demo(c: int, d: str) -> str:
    return "demo"

```

可以看到只需要加一个箭头后面跟上类型即可

### 为什么要用，用了有什么好处?

**1. IDE或者编辑器正确的自动提示**

最早写Java的时候，用的IDEA，虽然说Java本身写起来可能多少有点啰嗦，但是依赖于IDE的自动提示，基本不用自己默写API什么的，很方便。
一开始写python的时候，比较痛苦，因为你的类型是在运行时才确定，对于pycharm或者vscode来说，他们就是在推测你的类型是什么，然后给到你提示，但是很多时候，推测不准，就需要自己默写API了，很痛苦。
加上了Type Hints了之后，分析器就有足够的信息来确定这个类型，从而给到我们正确的提示。

**2. 代码即文档**

使用了Type Hints之后，你的变量，你的方法参数都是有具体类型的，通过合理的参数名称和类型注释，可以让人很清晰的知道这些参数的用法。

**3. 利用Mypy来做类型检查**

Mypy是一个静态类型检查器，可以帮助你发现你程序中因为类型错误可能导致的bug，我们可以先看一段程序

```python

def compute(a):
    if a == 0:
        return None

    return 1


result = 0
result += compute(a=1)
result += compute(a=0)
```

这是一段很简单的程序，试着运行一下就会报错`TypeError: unsupported operand type(s) for +=: 'int' and 'NoneType'`，原因很简单，int不能和None相加，可以想一下，这种错误，在我们接受的项目中是不是比较容易出现，当然有方法避免，比如写注释，告诉调用者可能的返回值是什么类型的，但是现在我们可以用更优雅的方式来解决这个问题。

```python
# a.py
from typing import Optional


def compute(a: int) -> Optional[int]:
    if a == 0:
        return None

    return 1


result = 0
result += compute(a=1)
result += compute(a=0)
```

我们声明了这个函数的返回值为Optional[int]，这个Optional[int]代表着你这个值可能为int类型，但是也可能为None，
在运行该代码之前，我们拿mypy跑一下这个文件 `mypy a.py`，我们会收到以下错误

```shell
a.py:12: error: Unsupported operand types for + ("int" and "None")
a.py:12: note: Right operand is of type "Optional[int]"
a.py:13: error: Unsupported operand types for + ("int" and "None")
a.py:13: note: Right operand is of type "Optional[int]"
Found 2 errors in 1 file (checked 1 source file)
```

> mypy可以和vscode进行集成，不需要进行run mypy的，在你写代码的时候，就能收到该提示，同时pycharm自带的类型检查工具也能发现该问题。同时我们可以在mypy加到CI中，做类型检查。

大概意思就是说Optional[int]不能和int相加，说明我们的程序存在出bug的风险，那么这个时候，我们应该进行修复，如下

```python

from typing import Optional


def compute(a: int) -> Optional[int]:
    if a == 0:
        return None

    return 1


result = 0

r1 = compute(a=0)

if r1 is not None:
    result += r1

```

可以根据我们的业务需求来对该None做出处理，这样就能避免运行时候因为类型不匹配导致的问题，

### 常用的类型

python的typing模块中提供了我们常用的类型，下面介绍一下该模块中我常用的

1. int, str, list, dict内建类型，不需要从typing中导入

1. Optional

这个类型很重要，推荐大家使用

2. List[other type]

比如List[int], List[str], List[List[int]]

3. Dict[key type, value type]

比如 Dict[str, int], Dict[str, List[int]]

4. TypedDict

这是一个让人很惊喜的类型，也可以称之为异构字典，在python3.8之后可以在typing包中找到，python3.8之前的版本可以`pip install typing-extensions`，然后从typing-extensions导入使用，简单介绍一下为什么我们需要这个类型，没有他之前，`{"a": 1, "b": "str"}`只能用Dict[str, Any]来声明，同时这个参数给到下游之后，下游方法也不知道这个字典里面都有哪些key，所以只能这样写`data.get(key)`，然后得到一个Optional类型，再去做特殊处理，比较麻烦，不够清晰，有了TypedDict之后，我们就可以这样做了

```python
from typing import TypedDict


class Data(TypedDict):
    a: int
    b: str


data: Data = {"a": 1, "b": "str"}

```

TypedDict本质上还是一个Dict，只不过允许我们更加细粒度去声明这个Dict里面的情况。

## 总结

本文只是简单介绍了一个python的Type Hints，具体更多用法需要自己下去多研究研究了，核心目的其实只有一个，让我们的代码的可读性和健壮性更好
