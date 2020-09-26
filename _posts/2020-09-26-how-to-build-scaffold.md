---
layout: post
title:  "如何构建脚手架"
date:   2020-09-26 09:58:48 +0800
categories: how
tags: android, jinja, cookiecutter
---

# 背景

## 上下文

Android Project。

需要抽取项目中的部分代码为独立工程，方便后续 `git submodule` 依赖或直接发布 `aar`.

由于待抽取模块比较多，需要先搞一个`模版`工程出来，支持（或者后续支持）：

- 项目独立编译
- 常用库版本统一维护
- 方便发布
- 有 App 模块可以直接运行
- 单测
- check style

目前项目中的工具箱都是基于 `python` 开发，所以编程语言也是锁定 python。

交待完毕。

## 历程

用过 IDE 的同学都熟悉，新建一个 project，点几下，输入一些必要参数，一个项目骨架就搭建完毕了。

如果要自己写一个搭建工具，应该从哪里入手呢？

### 历程1: 脚本纯撸

好吧，自己写就从头来吧，大概需要定义清楚下面几个部分：

1. 哪些是变量？需要留出占位符占坑？
2. 项目结构如何定义，在哪里定义？
3. 文件类型不同，需要不同的模版，定义在哪里？

看似简单的事情要搞顺溜也是有点摸不到头脑，感觉上就是自己先建立一个工程，把需要设置参数的地方改为占位符，运行时用实际配置替换占位参数，就齐活了。

可是并不知道该怎么写好。。

### 历程2: 模版替换 jinja

先搞模版占坑动态替换吧，记得之前看一些早期的 web 站点返回的 html 就是这么搞的，搜索了一下 `jinja` 就出现了。

具体用法这里就不赘述了，这个 [Python Jinja tutorial] 就说得很详细了，贴一个例子感受一下。

#### jinja 示例

**for_expr.py**

```python
#!/usr/bin/env python3

from jinja2 import Environment, FileSystemLoader

persons = [
    {'name': 'Andrej', 'age': 34}, 
    {'name': 'Mark', 'age': 17}, 
    {'name': 'Thomas', 'age': 44}, 
    {'name': 'Lucy', 'age': 14}, 
    {'name': 'Robert', 'age': 23}, 
    {'name': 'Dragomir', 'age': 54}
]

file_loader = FileSystemLoader('templates')
env = Environment(loader=file_loader)

template = env.get_template('showpersons.txt')

output = template.render(persons=persons)
print(output)
```

**templates/showpersons.txt**

![showpersons.txt](/assets/images/scaffold/jinja_showpersons.png)

**目录结构**
```
dir
 |__ templates
 |        |__ showpersons.txt
 |__ for_expr.py
```

由此可见，`jinja`还是很强大的，不止简单的占位替换，还可以 `for/if` 放一些控制代码进去。

### 历程3：脚手架 cookiecutter

有了模版替换的工具，后一步就卡在了项目结构上。

- 如何描述工程的结构呢？

- 目录，文件的位置关系我是应该定义在一个配置文件中，运行时动态生成吗？

- 目录结构好说，不同类型的文件又应该组织在哪里呢？

真是没理清楚，继续求助搜索引擎。幸好，看到了`cookiecutter`.

#### cookiecutter

> Cookiecutter creates projects from project templates, e.g. Python package projects.

解释一下：
- 首先你先创建好一个项目模版
- 把需要留坑的地方按照 `jinja` 定义占位符的方式写出来
- 需要配置的信息定义到 `cookiecutter.json` 中
- 执行 cookiecutter，结合项目模版和配置信息，生成新的工程

简直就是 `历程1` 中预期的理想国嘛。

#### 示例

假设已经创建好了一个工程模版：
- 在本地目录下：cookiecutter-pypackage/ (目录名称随意)
- 或者在 remote git 仓库中：https://github.com/audreyr/cookiecutter-pypackage.git

同时，在工程模版中定义了创建时需要配置的参数信息（后面是默认值）：

**cookiecutter.json**

```json
{
    "full_name": "Audrey Roy",
    "email": "audreyr@gmail.com",
    "project_name": "Complexity",
    "repo_name": "complexity",
    "project_short_description": "Refreshingly simple static site generator.",
    "release_date": "2013-07-10",
    "year": "2013",
    "version": "0.1.1"
}
```

那么生成新项目只需要执行：

```shell
# 本地
cookiecutter cookiecutter-pypackage/

# 远端
cookiecutter https://github.com/audreyr/cookiecutter-pypackage.git
```

命令行会提示你输入 json 中配置的参数，最后回车即可完成。

#### 高级用法

`cookiecutter` 支持在构建前/后添加 hook 钩子的方式进行 pre/post 处理。

需要在模版项目中定义 hooks 目录，其中创建:
- pre_gen_project.py
- post_gen_project.py

**目录结构图**

![pre/post hooks](/assets/images/scaffold/cookiecutter_hooks_file_structure.png)

脚本不要求必须用 python 实现，shell 也可以，不过为了可移植性，还是用 python 更好些。

如果 `post_gen_project.py` 执行中出错了，会清空生成的项目。

目前的用法是根据输入的`package_name`参数创建 src/(main|test|androidTest)/java 下对应的目录结构，以及移动一些 stub 文件到对应目录下。

其他高级特性还未涉及，有待了解。

# 结语

- 看似容易的事情，等动手实现的时候还是有很多细节没有考虑清楚该怎么搞
- 你需要的轮子别人可能早就帮你造出来了，只是你不知道罢了，需要去好好找找
- 学习一下人家的轮子是怎么造的

# 参考资料

[Python Jinja tutorial]

[Cookiecutter: Better Project Templates]

[cookiecutter Documentation (Release 2.0.0)]

<!-- refs -->
[Python Jinja tutorial]: http://zetcode.com/python/jinja/
[Cookiecutter: Better Project Templates]: https://cookiecutter.readthedocs.io/en/1.7.2/index.html
[cookiecutter Documentation (Release 2.0.0)]: https://buildmedia.readthedocs.org/media/pdf/cookiecutter/latest/cookiecutter.pdf