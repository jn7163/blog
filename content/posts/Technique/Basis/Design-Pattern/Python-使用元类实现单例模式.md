---
title: Python 使用元类实现单例模式
date: 2019-03-05 19:18:10
tags:
  - design pattern
  - python
---

上次在工厂模式中提到了单例模式，说有空再写（咕）。啊不过也没咕多久啦～这篇博客就介绍一下如何用 Python 元类简单优雅地实现单例模式 。

在一些使用场景里，我们会希望某些类只能创建一个实例，以避免重复造成的资源浪费或是冲突等情况 —— 例如 Windows 系统下的回收站无论如何只能打开一个窗口 —— 这就算是单例模式的应用 。我们通过使用 Python 元类就能非常优雅地实现单例模式 。

<!--more-->

**元类可以简单理解为类的类 。**在 Python 中一切皆对象，包括类也是对象；最常见的元类就是 `type` 。通过继承 `type`，我们可以实现自定义元类，从而控制类创建实例的行为 。直接说有点抽象，不如先看代码案例 。例子中的 `Connection` 代表了某种通过密钥获取的网络服务，而我们不希望看到重复认证连接该服务产生的带宽浪费，因此使用了单例模式 。

**源代码**：

```python
class Singleton(type):
    def __init__(cls, *args, **kwargs):
        print('__init__', cls, sep='\n')
        cls.instance = None
        super().__init__(*args, **kwargs)

    def __call__(cls, *args, **kwargs):
        print('__call__', cls, sep='\n')
        if cls.instance is None:
            cls.instance = super().__call__(*args, **kwargs)
        return cls.instance

class Connection(metaclass=Singleton):
    def __init__(self, secret_key):
        print('initializing connection...')
        self.secret_key = secret_key

if __name__ == '__main__':
    conns = [Connection(0), Connection(1)]
    print(conns[0] is conns[1])
```

**预期输出**：

```
__init__
<class '__main__.Connection'>
__call__
<class '__main__.Connection'>
initializing connection...
__call__
<class '__main__.Connection'>
True
```

这里的 `Singleton` 就是继承了 `type` 的单例元类 。通过指定 `metaclass=Singleton` 字段，我们让 `Singleton` 代替了默认的 `type` 作为 `Connection` 的元类 。整个运行过程是这样的：

-   当程序运行到第 13 行时，`Connection` 类作为 `Singleton` 的实例被创建，输出 `__init__`；
-   当程序运行到 `Connection(0)` 时，`Connection.__call__`（定义在第 7 行）被调用了 。由于此时 `instance` 仍然是 `None`，我们就通过 `super().__call__` 调用了 `__new__` 与 `__init__` 方法创建了目标实例，并输出 `initializing connection...`；
-   当程序运行到 `Connection(1)` 时，`Connection.__call__` 再次被调用 。不过此次直接返回了之前已经创建好的 `instance`，不再调用 `__init__`方法 。注意此时 `secret_key` 属性仍然是 0，没有发生改变；
-   最后判断的出 `conns[0]` 与 `conns[1]` 是两个完全相同的实例，输出 `True`；

如果你进入交互式环境试一试，就会发现有 `conns[*]` 与 `conn[*].instance` 与 `Connection.instance` 都指向同一个对象 。我们创建对象时就是通过判断 `Connection.instance` 是否存在来确定是否需要新建实例的 。

元类 `Singleton` 其实也可以这样改写：

```python
def Singleton(name, bases, attrs):
    class_ = type(name, bases, attrs)
    class_.instance = None
    class_.__new__ = __new__
    return class_

def __new__(cls, *args, **kwargs):
    if not hasattr(cls, 'instance'):
        cls.instance = super().__new__(cls, *args, **kwargs)
    return cls.instance
```

这就涉及到了元类的本质 —— 一个能接受三个参数并返回一个类的 callable 对象；这三个参数分别是类名，父类（元组）与属性（字典）。例如 `type('Foo', (object,), {'a': 1})` 会返回一个名为 `Foo`，继承自 `object`，有属性 `a=1`的类 。这里我们对 `type` 元类**做了一点微小的工作**，给返回的类添加了 `instance` 属性与 `__new__` 方法，使得通过 `Singleton` 新建的类在创建实例时会先检查该类是否存在 `instance` 属性，如果存在则直接返回 —— 从而阻止了新实例的创建，实现了单例模式 。

使用元类实现单例模式的过程还是蛮简单的，很大程度上你只要理解了**元类是类的类**就能弄明白了 。值得一提的是，尽管用元类实现单例模式很方便，但大多数情况下应该避免使用这种黑魔法 —— 尤其是在你还没彻底搞清楚它之前 。

---

参考文献：

-   [StackOverflow: What are metaclasses in Python?](https://stackoverflow.com/questions/100003/what-are-metaclasses-in-python)
-   [《Python3 Cookbook》：使用元类控制实例的创建](https://python3-cookbook.readthedocs.io/zh_CN/latest/c09/p13_using_mataclass_to_control_instance_creation.html)
-   [博客园：深刻理解Python中的元类(metaclass)以及元类实现单例模式](https://www.cnblogs.com/tkqasn/p/6524879.html)