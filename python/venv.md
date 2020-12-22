# venv原理浅析

## 1. [venv](https://docs.python.org/3/library/venv.html)是什么

简单的说，就是为每个应用提供了独立的lib，应用之间独享自己的lib

官方解释:

The venv module provides support for creating lightweight “virtual environments” with their own site directories, optionally isolated from system site directories. Each virtual environment has its own Python binary (which matches the version of the binary that was used to create this environment) and can have its own independent set of installed Python packages in its site directories.

## 2. 创建venv

```bash
python -m venv <name> # 创建
source <name>/bin/activate # 激活虚拟环境
```

这里的**name**一般来说大家都是命名为".venv"或者"venv", 创建虚拟环境的方式不只这一种，但是在python3.6之后，官方只推荐大家这么做，具体原因可以看一下[官方解释](https://docs.python.org/dev/whatsnew/3.6.html#id8)

## 3. 浅析

现在已经创建好虚拟环境，可以进去虚拟环境的目录看一下究竟

```bash
cd <venv_name>
tree -L 2 # 查看目录结构
    .
    ├── bin
    │   ├── activate
    │   ├── activate.csh
    │   ├── activate.fish
    │   ├── easy_install
    │   ├── easy_install-3.6
    │   ├── pip
    │   ├── pip3
    │   ├── pip3.6
    │   ├── python -> /Users/beanan/.asdf/installs/python/3.6.8/bin/python
    │   └── python3 -> python
    ├── include
    ├── lib
    │   └── python3.6
    └── pyvenv.cfg

    4 directories, 11 files
```

上面我们进行了一步激活虚拟的环境的操作，可以看看这个activate文件里面做了什么样的事情

```bash
VIRTUAL_ENV="/Users/beanan/py_workspace/study/venv"
export VIRTUAL_ENV
_OLD_VIRTUAL_PATH="$PATH"
PATH="$VIRTUAL_ENV/bin:$PATH"
export PATH
```

这段命令也很简单，就是将$name/bin目录添加到了PATH这个环境变量上，并放到首位，这样的话，只要我们在激活的虚拟环境中，我们输入 "python" 这样的命令，那么python这个命令位置肯定是来自我们虚拟环境目录的python执行文件。

```bash
which python
/<project_path>/<venv_name>/bin/python
```

从上面venv目录结构可以看到venv下的python命令是我们系统全局python解释器的一个软链接，这样侧面就说明一件事情，虚拟环境只是提供了独立的lib包，我们用的python解释器还是用的我们系统中的。

这里有个知识点需要了解下，在python中解释器是如何寻找包的，答案是`sys.path`，python根据`sys.path`里面的路径来依次找包，现在我们打印一下全局环境和虚拟环境中sys.path的值

全局环境

```bash
python -c "import sys;print(sys.path)"
output:
[
    '',
    '/Users/beanan/.asdf/installs/python/3.6.8/lib/python36.zip',
    '/Users/beanan/.asdf/installs/python/3.6.8/lib/python3.6',
    '/Users/beanan/.asdf/installs/python/3.6.8/lib/python3.6/lib-dynload',
    '/Users/beanan/.asdf/installs/python/3.6.8/lib/python3.6/site-packages'
]
```

虚拟环境

```bash
python -c "import sys;print(sys.path)"
output:
[
    '',
    '/Users/beanan/.asdf/installs/python/3.6.8/lib/python36.zip',
    '/Users/beanan/.asdf/installs/python/3.6.8/lib/python3.6',
    '/Users/beanan/.asdf/installs/python/3.6.8/lib/python3.6/lib-dynload',
    '/Users/beanan/py_workspace/study/venv/lib/python3.6/site-packages'
]
```

通过对比，我们可以发现，两者的差异在于列表的最后一位，全局环境使用的是全局的site-package，虚拟环境中使用的venv site-package，那么python是怎么做到的，答案在python的site.py文件里面，简而言之，当python解释器启动时候，解释器是知道自己当前启动的工作目录是在哪个位置，他会找父目录下有没有"pyenv.cfg"文件，如果有，添加虚拟环境的site-package， 否则添加global site-package, 具体逻辑可以看下site.py文件中的内容，site.py文件在python解释器加载的时候就会执行，改变我们的sys.path,同时sys.prefix也会改变。

## 4. 参考

1. [https://docs.python.org/3/library/venv.html](https://docs.python.org/3/library/venv.html)
2. [https://docs.python.org/dev/whatsnew/3.6.html#id8](https://docs.python.org/dev/whatsnew/3.6.html#id8)
3. [https://docs.python.org/3/library/site.html](https://docs.python.org/3/library/site.html)