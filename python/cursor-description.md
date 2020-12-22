# 利用Cursor description确定SQL返回值类型

## 前言

最近工作上有个业务场景，简单描述就是用户在前端界面输入一条查询SQL，然后后端返回预览数据以及数据的schema，注意这里的schema要的是精确的schema，也就是说精确的schema信息必须由数据库层面给到我们，我们自己是确定不了的，我们只能拿到表结构基础字段的类型，但是用户填入的SQL是复杂多变的，查询出来的每列值类型我们不知道，但是数据库是知道的。
这个信息怎么拿，现在的python数据库驱动基本都实现[PEP 249](https://www.python.org/dev/peps/pep-0249/#cursor-objects)定义的规范，从规范中我们可以看到Cursor这个类型，这个类有一个属性 description，这个属性是由7个元素组成的元组，组成如下：

` (name, typecode, display_size, internal_size, precision, scale, null_ok) `

我们可以从这个属性中，拿到我们列类型，每个数据库驱动都维护了自己的typecode，我们要做的就是把typecode对应到我们应用中定义的类型
思路就是这样的一个思路，但是在具体实现的时候，还是踩了不少坑，目前踩的坑基本都来自各大驱动对于NUMBER类型的处理，下面依次介绍

## MySQL

这里我采用的驱动为[pymysql](https://github.com/PyMySQL/PyMySQL)，其他驱动也基本都是类似问题

### INT、LONG类型映射错误

INT类型的字段返回的typecode为3，这里pymysql会将他映射成LONG类型，LONG类型返回的typecode为252，在pymysql中会将他映射为BOLB类型，是不是有点神奇，一开始我以为是pymysql映射出错了，想着去MySQL官方那找点线索，然后我发现MySQL官方自己也维护着一个python驱动，[mysql-connector-python](https://github.com/mysql/mysql-connector-python)，翻了翻这个库的源码，发现他的映射和pymysql是一致的，emmmm，MySQL不讲武德，一个类型映射都映射成这样。那现在我们要解决这个问题，把INT和LONG类型映射正确，通过测试发现，INT类型的typecode应该为3，所以我们只需要typecode=3的类型映射到INT就行了。

接下来就是要解决LONG类型的问题，MySQL把long和BLOB类型的typecode都定义为252，通过观察发现可以通过precision这个值来进行区分，precision=67108860的type应为LONG

```python
from typing import Tuple

def convert_type(description: Tuple) -> str:
    typecode = description[1]
    if typecode == 3:
        return "INT"
    elif typecode == 252:
        precision = description[4]
        if precision == 67108860:
            return "LONG"
        else:
            return "STRING"
    else:
        return "user custom mapping"

```

### DECIMAL precision和scale值异常

当我们确定一个类型为DECIMAL之后，还想拿到它具体的precision和scale值，从上面的元组组成我们可以看到是有precision和scale的定义的，直接取值可以了，但是事情没有这么简单，拿出来的值和我数据库中定义的有偏差，于是我定义了很多DECIMAL类型，一番观察之后，发现了规律

```python
from typing import Tuple


def determine_decimal_type(description: Tuple) -> str:
    precision, scale = description[4:6]
    if scale == 0:
        return f"DECIMAL({precision - 1},{scale})"
    else:
        return f"DECIMAL({precision - 2},{scale})"

```

我只想说，MySQL不讲武德

## Oracle

Oracle这里采用的是官方驱动 [cx_Oracle](https://github.com/oracle/python-cx_Oracle)

### 所有数字类型都是NUMBER类型

Oracle将所有的数字类型全部转换成了NUMBER，也就说从typecode上，我们只能知道他是一个数字类型，但是不能区分出INT，FLOAT这些，这就有点难受，查了一些资料，基本上都是说从precision, scale这两个字段信息下手，这里插入一个 [链接](https://www.programcreek.com/python/example/2699/cx_Oracle.NUMBER)，里面有不少解决方案，这里我因为业务需要和Spark确定出来的类型一致，所以要将NUMBER推断成了DECIMAL，然后不同的类型使用不同的精度。

```python
from typing import Tuple

def determine_number_type(description: Tuple) -> str:
    precision, scale = description[4:6]

    if scale == 0:
        return f"DECIMAL({precision},{scale})"
    elif scale == -127:
        return f"DECIMAL(38,10)"
    else:
        return f"DECIMAL({precision},{scale})"

```

## SQL Server

SQL Server这里我采用的python驱动为 [pymssql](https://github.com/pymssql/pymssql)，这个库也是让人一言难尽，给我的感觉是社区基本已经不维护了，从18年到20年也就更新了一次，提的issue和pr基本没有人理会，现在说一下在使用pymssql中遇到一些问题。

### 数字类型细化不够细致

在pymssql中，数字类型只细化了NUMBER和DECIMAL类型，但是这对于我来说，不够，我想要更细化的数字类型，于是我改了他的代码[链接](https://github.com/BeanNan/pymssql/commit/cd5cab1ca3bf1600d84351c9b123557ea70d91dd)

pymssql中Cursor.description这个元组里面只填充了name和typecode，从规范来说，他这是符合规范，但是对于我来说，不够，只能通过改代码的方式，来拿到具体的precision, scale的值[链接](https://github.com/BeanNan/pymssql/commit/1647f8c8996bd37e0d4ae914cec618279c7506c4)

社区目前给我的感觉基本处于不维护状态，鉴于此，我针对我打的这两个补丁，在pypi上重新发了一个包，可以通过 `pip install pymssql-plus`来安装使用，[代码仓库地址](https://github.com/BeanNan/pymssql)
