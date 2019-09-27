---
title: Python 实现工厂模式
date: 2019-02-23 19:39:37
tags:
  - design pattern
  - python
---

前几天订阅的 RSS 给我推送了 [The Factory Method Pattern and Its Implementation in Python](https://realpython.com/factory-method-python/#supporting-additional-formats)，于是就研究了一番工厂模式 。工厂模式从简单到复杂大约可以分为**简单工厂模式**（又称静态工厂方法模式），**工厂方法模式**（又称多态工厂模式）与**抽象工厂模式**；都是工厂（Factory）—产品（Product）—客户（Client）这样的基础架构 。作为一种创建型模式（Creational Pattern），工厂模式的核心理念就是让某个工厂类来负责生产对象（即产品），客户端只需要提供特定的参数即可；将对象创建和业务处理分离降低系统的耦合度，使得两者修改起来都相对容易 。本文依旧采用 python 实现这一设计模式 。

<!--more-->

## 简单工厂模式

出现问题的场景是这样的：你现在正在开发一款 Markdown 文本编辑器，你需要根据用户的选择来实现导出不同格式的电子文档（例如 DOCX、HTML、PDF 等）。直接的想法就是使用 `if/elif/else` 或 `switch` 这样的选择结构来实现；但是业务逻辑一旦稍加复杂，代码大约就会变成这样：

```python
if option == 'docx':
    # some long hard-to-read code
elif option == 'html':
    # another piece of long hard-to-read code
elif option == 'pdf':
    # unbearable complex code
else:
    raise ValueError(option)
```

你的代码很有可能会变得混乱而难以理解；而且随着代码量的增长，修改或增添业务逻辑会变得异常复杂 —— 因为你把所有代码都写在一处，导致了系统的高度耦合 。这时候我们就需要引入简单工厂模式：

```python
class Factory:
    def get(self, option):
        if option == 'docx':
            return self.export_docx
        elif option == 'html':
            return self.export_html
        elif option == 'pdf':
            return self.export_pdf
        else:
            raise ValueError(option)

    @staticmethod
    def export_docx(*args):
        # logic of exporting docx file

    @staticmethod
    def export_html(*args):
        # logic of exporting html file

    @staticmethod
    def export_pdf(*args):
        # logic of exporting pdf file
```

这里 `Factory` 即工厂类，而静态方法即产品 。这里使用了 `@staticmethod` 这一装饰器表明了这些产品与工厂之间不存在依赖关系，事实上更推荐将产品函数作为外部函数来写，而非工厂类的静态方法（不过我觉得这样写好看 w）。

上面的例子中我们的产品是函数，这些函数包装了业务逻辑；客户端只需要通过 `factory.get(option)` 这样的形式就的到相应的产品函数，从而完成业务逻辑 —— 这样无疑降低了系统的耦合度并提高了代码的可读性 。但在更复杂的情况下，我们的产品可能是一系列拥有相似特征的对象：这时候我们就可以充分利用面向对象编程的 `继承` 特性，先定义一个抽象 `Product` 类来提供通用的接口，随后用子类 `ConcreteProductA`，`ConcreteProductB` 等实现其特有的接口 。例如某个工厂要生产越野车、小汽车、自行车，我们就可以这样定义产品：

```python
class Vehicle:
    def __init__(self, speed):
        self.speed = speed

    def run(self):
        print(f'running at {self.speed}km/h')

class Jeep(Vehicle):
    def __init__(self):
        super().__init__(90)

class Car(Vehicle):
    def __init__(self):
        super().__init__(60)

class Bike(Vehicle):
    def __init__(self):
        super().__init__(30)
```

在某些情况下，你的产品在创建过程中还用到其他对象 —— 例如 `Car` 会用到 `Engine`，`Wheel` 等 —— 把这些对象的创建全部写在业务逻辑里就会让代码更加难以理解 。在复杂的系统里工厂模式的优势就愈发体现出来了 。

简单工厂模式大概就是这样 。它的优点是显而易见的：工厂只负责生产产品，客户只负责消费产品，实现了职责分离与系统解耦；但其缺点也同样存在：

-   引入了新类，一定程度上增加了系统的复杂程度；

-   一旦工厂类出现错误，整个系统都会受到影响；
-   由于产品的生产逻辑仍是使用 `if/elif/else` 的方式硬编码，一旦想要增加新的产品就必须修改工厂类的源代码，不易于扩展

随后引入的工厂方法模式一定程度上客服了这些缺点 。源代码托管在 [Github Gist](https://gist.github.com/queensferryme/ef42ff59942da96652351055b0e54119#file-simple_factory-py)，欢迎查阅 。

## 工厂方法模式

简单工厂模式中有这么三个角色：工厂角色、抽象产品角色、具体产品角色 —— 其中工厂负责所有具体产品的生产；而抽象产品是所有具体产品的父类，提供统一的接口 。但在工厂方法模式中，我们**将工厂角色进一步细分为抽象工厂角色与具体工厂角色**，其中每种具体工厂仅负责一种具体产品的生产 。

现在假设我们有一项云存储服务，会同时用到 Google 与 AWS 。但是接入 AWS 需要 `app-key` 与 `secret-key`，而接入 Google 只需要 `secret-key` —— 如果我们依然只使用一个工厂类，就又得在 `authenticate` 方法里写出 `if/elif/else` 这样糟糕的代码 。所以我们将工厂也分为抽象工厂与具体工厂 —— 抽象工厂作为所有具体工厂的父类提供一些通用接口，而一种具体工厂只负责一种具体产品的生产；而客户只需要调用相应的具体工厂就可以获得具体产品 —— 很多地方将之称为“**将对象的创建延迟到子类**” 。下面放样例代码：

```python
class CloudFactory:
    def __init__(self, config:dict):
        self.config = config

class AWSCloudFactory(CloudFactory):
    def authenticate(self, app_key, secret_key):
        if app_key != 'aws-app-key':
            raise ValueError(app_key)
        elif secret_key != 'aws-secret-key':
            raise ValueError(secret_key)
        return AWSCloud()

    def create(self):
        conn = self.authenticate(
            self.config['aws-app-key'],
            self.config['aws-secret-key']
        )
        return conn

class GoogleCloudFactory(CloudFactory):
    def authenticate(self, secret_key):
        if secret_key != 'google-secret-key':
            raise ValueError(secret_key)
        return GoogleCloud()

    def create(self):
        conn = self.authenticate(self.config['google-secret-key'])
        return conn

class Cloud:
    pass

class AWSCloud(Cloud):
    def __repr__(self):
        return f'<AWSCloud {id(self)}>'

class GoogleCloud(Cloud):
    def __repr__(self):
        return f'<GoogleCloud {id(self)}>'

if __name__ == '__main__':
    config = {
        'aws-app-key': 'aws-app-key',
        'aws-secret-key': 'aws-secret-key',
        'google-secret-key': 'google-secret-key'
    }
    print(AWSCloudFactory(config).create())    # <AWSCloud 140047826359184>
    print(GoogleCloudFactory(config).create()) # <GoogleCloud 140047826359128>
```

这样一来，要增加产品也变得相对容易了：你只需要增加相应的具体工厂与具体产品即可 。

当然，你可能希望在应用的生命周期内不要重复认证并创建连接 —— 这也不难实现 。简单的方法就是将连接对象存进字典里，当字典中不存在需要的服务时再另行创建；更好的方式就是**单例模式**啦 —— 这是后话 。

再举一个例子，Python 中的 `list`，`dictionary`，`tuple`，`set` 数据结构都可以视作实现了 `Iterable` 接口的具体工厂，而这些具体工厂通过调用 `__iter__` 工厂方法返回一个 `Iterator`（即具体产品），最后客户使用 `for...in` 或 `next` 等方式来消费这一产品 。这其实也是一种工厂方法模式 。

经过一番修改，我们成功解决了新增产品（即系统可扩展性）的问题 。但也引入了一个显见的缺点：每次新增一个产品，系统中的类都会成对增加，这导致在产品数量大的情况下系统变得十分复杂 。源代码见 [Github Gist](https://gist.github.com/queensferryme/ef42ff59942da96652351055b0e54119#file-factory_method-py) 。

## 抽象工厂模式

抽象模式中又新增了**产品族**的概念，这使得我们能从两个维度来对产品进行分类 。例如我们开发的某件产品 UI 主要由两部分组成：文本和菜单栏；现在我们还要为用户提供白天/夜间主题选项，两种主题都有对应的不同风格的 UI 组件 —— 于是我们就能从主题与组件种类两个方面来组织产品 。

要实现抽象工厂模式也并不复杂 。原有的工厂方法模式里每个具体工厂只对应生产一种具体产品，而在抽象工厂模式里每个具体工厂对应生产同一族（系列）的多个某方面（如主题）相同的具体产品 ；而与此同时可能会有多个抽象产品类，每个抽象产品类从另一方面（如组件种类）描述了具体产品的一些接口和特性 。下面贴出例子（源代码依旧见 [Github Gist](https://gist.github.com/queensferryme/ef42ff59942da96652351055b0e54119#file-abstract_factory-py)）：

**工厂类**

```python
class UIFactory:
    def __init__(self, **builders):
        self.builders = builders

    def createMenu(self):
        return self.builders['menu']()

    def createText(self):
        return self.builders['text']()

class DayUIFactory(UIFactory):
    def __init__(self):
        super().__init__(
            menu=DayMenuComponent,
            text=DayTextComponent
        )

class NightUIFactory(UIFactory):
    def __init__(self):
        super().__init__(
            menu=NightMenuComponent,
            text=NightTextComponent
        )
```

**产品类**

```python
class Component:
    def __init__(self, category, theme):
        self.category = category
        self.theme = theme

    def __repr__(self):
        return '<{}{}Component {}>'.format(
            self.theme.title(),
            self.category.title(),
            id(self)
        )

class MenuCompnent(Component):
    def __init__(self, theme):
        super().__init__('menu', theme)

class DayMenuComponent(MenuCompnent):
    def __init__(self):
        super().__init__('day')

class NightMenuComponent(MenuCompnent):
    def __init__(self):
        super().__init__('night')

class TextComponent(Component):
    def __init__(self, theme):
        super().__init__('text', theme)

class DayTextComponent(TextComponent):
    def __init__(self):
        super().__init__('day')

class NightTextComponent(TextComponent):
    def __init__(self):
        super().__init__('night')
```

**运行案例**

```python
factory = DayUIFactory()
print(factory.createMenu()) # <DayMenuComponent 139692210593072>
print(factory.createText()) # <DayTextComponent 139692210593072>
factory = NightUIFactory()
print(factory.createMenu()) # <NightMenuComponent 139692210593016>
print(factory.createText()) # <NightTextComponent 139692210593016>
```

抽象工厂模式有如下优势：

-   只要拥有某一具体工厂类，我们就能生产出对应产品族的所有产品，从而降低了系统耦合度
-   确保同一产品族中的具体产品共同工作，不会出现混乱
-   具有部分可扩展性，例如在上例中要增加圣诞主题只需增添对应的具体工厂 `ChrisUIFactory` 与具体产品 `ChrisMenuComponent` `ChrisTextComponent` 即可

但同时也依然有这些不足：

-   不完整的可扩展性，例如在上例中若要增加 `ButtonComponent` 就不得不修改源码
-   系统中的类大量增加，层级结构相对复杂，复杂性导致了更多潜在的问题

## 总结

[![factory_relation](https://i.loli.net/2019/09/27/1KhGcuO5HCJrtq9.png)](https://i.loli.net/2019/09/27/1KhGcuO5HCJrtq9.png)

-   简单工厂模式只有**工厂**、**抽象产品**与**具体产品**三种角色，一个工厂负责生产所有的产品 。不易于扩展，无法增添新产品 。《设计模式》一书中将简单工厂模式归为工厂方法模式的特例；
-   工厂方法模式将工厂进一步细分为**抽象工厂**与**具体工厂**，一个具体工厂只负责生产一种具体产品 。具有较好的可扩展性，可以增添新产品；
-   抽象工厂模式又引入了**产品族**的概念， 一个具体工厂负责生产一族（类）产品 。具有部分可扩展性，即只支持增加新的产品族 。

---

**参考文献**：

- [RealPython: The Factory Method Pattern and Its Implementation in Python](https://realpython.com/factory-method-python/#supporting-additional-formats)
- [Graphic Design Pattern：简单工厂模式( Simple Factory Pattern )](https://design-patterns.readthedocs.io/zh_CN/latest/creational_patterns/simple_factory.html)
- [Graphic Design Pattern：工厂方法模式( Factory Method Pattern )](https://design-patterns.readthedocs.io/zh_CN/latest/creational_patterns/factory_method.html)
- [Graphic Design Pattern：抽象工厂模式( Abstract Factory )](https://design-patterns.readthedocs.io/zh_CN/latest/creational_patterns/abstract_factory.html)
- [简书：工厂方法模式（Factory Method）- 最易懂的设计模式解析](https://www.jianshu.com/p/d0c444275827)
- [Hollis' Blog：设计模式（八）——工厂模式总结](https://www.hollischuang.com/archives/1430)