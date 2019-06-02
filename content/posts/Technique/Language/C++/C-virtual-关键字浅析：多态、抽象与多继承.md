---
title: C++ virtual 关键字浅析：多态、抽象与多继承
date: 2019-03-29 17:36:26
tags:
  - c++
  - virtual
---

C++ 的**虚函数**算是比较高级的语法特性之一，通过 `virutal` 关键字来定义，主要用于实现面向对象编程中的多态；`virtual` 关键字还可以用来定义**纯**虚函数，主要用于实现面向对象编程中的抽象；而且多继承的实现也需要用到 `virtual` 关键字 。既然 `virutal` 关键字这么有用，那就写（水）篇博客记录一下吧！

<!--more-->

## 虚函数与运行时多态

### 基本实现

多态（polymorphism），正如其字面意思，是指一个实例具有多种状态 。在具体编程实现中体现为**不同的数据类型实现了相同的接口**，从而我们可以调用统一的接口来触发一个行为 —— 具体是哪一种状态下的行为则有类型系统来为我们决定 。多态为我们提供了一定编程上的便利性 。

这么说或许有些抽象，下面举个实际的例子：

```c++
#include <iostream>

using std::cout;
using std::endl;

class Person {
  public:
    void say() {
      cout << "I am a person." << endl;
    }
};

class Student : public Person {
  public:
    void say() {
      cout << "I am a student." << endl;
    }
};

class Teacher : public Person {
  public:
    void say() {
      cout << "I am a teacher." << endl;
    }
};

int main() {
  return 0;
}
```

上面的代码中定义了三个类：一个基类 `Person` 与两个派生类 `Student` 和 `Teacher` 。它们都实现了相同的接口 `say`，但接口的具体行为是不同的 。

而我们可能有这样一个函数，它能接收一个 `Person` 类或其派生类对象作为参数，并调用其 `say` 接口，再执行某些操作后返回 —— 对于不同的类，`say` 接口的具体行为是不同的 。但我们当然不希望为每一种新增的派生类重载一个函数，这样做大大增加了代码的复杂性并使之不易于维护 —— 因为我们只需要一个会说话（`say`）的人（`Person`），而不关心他/她具体是学生或教师或其他什么职业的人 。多态的作用就在于优雅地处理这类问题：我们可以不关心对象的具体状态（status agnostic）而直接调用其统一接口，而系统会为我们自动确定对象的状态并调用相应的接口 。

那么代码具体要怎么实现呢？你可能会这样写：

```c++
void person_say(Person p) {
  p.say();
}

int main() {
  Student s; Teacher t;
  person_say(s);
  person_say(t);
  return 0;
}
```

但运行结果是这样的：

```
I am a person.
I am a person.
```

显然这是多态的错误实现 。这是因为代码中存在两个问题：

-   `say` 方法需要被定义为虚函数，即 `virtual void say() {...}`
-   `person_say` 的 `Person` 参数类型必须为引用或指针，即 `Person& p` 或 `Person* p`

修复这两个问题之后的代码实现大概是这样的：

```c++
#include <iostream>

using std::cout;
using std::endl;

class Person {
  public:
    virtual void say() {
      cout << "I am a person." << endl;
    }
};

class Student : public Person {
  public:
    void say() {
      cout << "I am a student." << endl;
    }
};

class Teacher : public Person {
  public:
    void say() {
      cout << "I am a teacher." << endl;
    }
};

void person_say(Person& p) {
  p.say();
}

int main() {
  Student s; Teacher t;
  person_say(s);
  person_say(t);
  return 0;
}
```

而运行结果是：

```
I am a student.
I am a teacher.
```

这与我们的预期是一致的 。我们的代码终于能够自行确定参数的实际类型并调用相应的接口了！但又有几个新问题尚未解决：

-   `virutal` 关键字就能实现多态的原理是什么？
-   为什么上述实现只在基类中使用了 `virtual` 关键字？
-   为什么 `person_say` 函数的 `Person` 参数类型必须为引用或指针？

第一个问题暂且不表，看看第二个与第三个问题 。

首先，因为在基类中用 `virtual`将某个方法定义为虚函数会导致所有派生类中**签名相同**的函数都被自动定义为虚函数 —— 注意这里的签名相同不仅仅是函数名相同，还包括参数类型与返回值类型等完全相同 —— 也就是说，你完全可以在派生类中也添加相应的 `virtual` 关键字，但是没有必要 。

其次就涉及到了 C++ 求值策略的问题 。如果将参数定义为 `Person` 类型，函数在传参时就采用**按值传递**（pass by value）的策略，也就是说函数内部的 `p` 变量将是与实参完全不同的一个彻底的 `Person` 对象，因此系统也就无法在运行时确定实参原本究竟是什么类型 。但当你采用**按引用或指针传递**（pass by reference/pointer）的时候，系统就可以根据该引用/指针实际引用/指向的对象的类型（而非该引用/指针的类型）来确定所需要的行为 。顺便一提，在目前绝大多数 C++ 编译器底层，引用也是通过指针来实现的 。

### 运行时多态与动态绑定

现在来解答第一个问题 。虚函数是 C++ **运行时多态**的实现方式，也就是说，程序在运行时才能确定参数的实际类型 。程序在运行期间（而非编译期）判断所引用对象的实际类型的技术被成为**动态绑定**（dynamic binding） 。使用 `virtual` 关键字相当于告知程序在运行时进行动态绑定，以调用实际所需的方法 。

相应运行时多态的就是**编译时多态** 。这种多态性在编译期就被确定，因而也不需要动态绑定 —— C++ 中模板函数就是典型的编译期多态，但函数重载也是编译期多态 。

```c++
template <typename T>
T add(T a, T b) {
  return a + b;
}
```

### 常见误区

#### 虚函数的默认参数

考虑以下代码：

```c++
#include <iostream>

using std::cout;
using std::endl;

class Base {
  public:
    virtual void test(int val=10) {
      cout << val << endl;
    }
};

class Derived : public Base {
  public:
    void test(int val=20) override {
      cout << val << endl;
    }
};

int main() {
  Derived a;
  Base* b = dynamic_cast<Base*>(&a);
  a.test();
  b->test();
  return 0;
}
```

依照我们之前对运行时多态与动态绑定的讨论，第 23 行与第 24 行代码应该都输出 20 。但实际上第 24 行代码输出了 10 —— 这是因为虚函数是动态绑定的，但默认参数是静态绑定的 。由于 `b` 是 `Base` 类型的指针，所以缺省参数的默认值被静态绑定到 `Base` 类型的默认值，即 10 而非 20 。顺便一提，**我们应该尽可能避免重新定义继承来的默认参数** 。它会导致一系列奇怪的难以调试的问题，包括我们正在讨论的这一个 。

#### 虚静态/构造/析构函数

-   **静态函数不可以声明为虚函数 。**静态函数不属于任一实例，因此将其声明为虚函数也没有意义 。

-   **构造函数不可以声明为虚函数 。**除了 inline 之外，构造函数不允许使用其它任何关键字。构造函数是用来创建实例的，被创建的实例必须有明确的类型，因此构造函数不能是虚函数 。

-   **析构函数可以声明为虚函数。**事实上，只要一个类有可能会被其它类所继承， 就应该声明虚析构函数，因为我们可能也需要动态确定被销毁对象的实际类型 。

## 纯虚函数与抽象类

基类一般比派生类更抽象，派生类一般比基类更具体 。而有些类型太过于抽象，以至于无法拥有具体的实例 —— 这种类被称作抽象类，它无法被实例化（instantiation）。抽象类一般用于定义统一的接口格式 。

在 C++ 中，拥有纯虚函数的类就是抽象类 。例如：

```c++
class Base {
  public:
    virtual void test() = 0;
};

class Derived : public Base {
  public:
    void test() {
      cout << "in derived" << endl;
    }
};
```

这里 `Base` 类就是一个抽象类；使用 `virtual` 修饰符，并在函数头部后写上 `=0` 就能定义纯虚函数 。我们在子类 `Derived` 中给出了纯虚函数的具体实现 。当然，如果你试图通过 `Base instance;` 或其他方式为抽象类创建实例，那么编译器就会抛出错误 。

## 多继承/环状继承

`virtual` 关键字除了用于实现多态与抽象之外，还常用与实现多继承 。多继承是指一个派生类有多个基类 。在下面的例子中，`Child` 类就实现了继承了 `ParentA` 与 `ParentB` 。

```c++
#include <iostream>

using std::cout;
using std::endl;

class Base {
  public:
    Base() { cout << "Base()" << endl; }
    Base(int x) { cout << "Base(int)" << endl; }
};

class ParentA : public Base {
  public:
    ParentA() : Base(0) { cout << "ParentA" << endl; }
};

class ParentB : public Base {
  public:
    ParentB() : Base(1) { cout << "ParentB" << endl; }
};

class Child : public ParentA, public ParentB {
  public:
    Child() { cout << "Child()" << endl; }
};

int main() {
  Child instance;
  return 0;
}
```

程序执行输出结果为：

```
Base(int)
ParentA()
Base(int)
ParentB()
Child()
```

这里面看起来有点小问题 。由于多继承，`Base` 类的有参构造函数被调用了两次，因而一个 `Child` 对象将会有两份 `Person` 类成员的拷贝 —— 这通常不是我们所希望看到的 。理想状态下，由于这种环状继承中 `ParentA` 与 `ParentB` 都有共同的基类 `Base`，我们会期望 `Base` 只被初始化一次，然后就依次初始化 `ParentA` 与 `ParentB` 。这时候就需要用到 `virtual` 关键字来消除这种歧义性 。

```c++
class ParentA : virtual public Base {...};
class ParentB : virtual public Base {...};
```

这样修改后重新编译运行会得到如下输出：

```
Base()
ParentA()
ParentB()
Child()
```

问题似乎得到了解决 。但还有一个问题：这里程序调用了 `Base` 类的无参构造函数，可是我们在程序里已经显示调用了有参构造函数 。这是由于如果在多继承中使用了 `virtual` 关键字，**即使显式调用了有参构造函数，也会继续调用无参构造函数**；理由也不难理解，如果多个父类调用了祖父类的有参构造函数并且传入了不同的参数，就会造成歧义性 。

如果你需要调用祖父类的有参构造函数，你需要在子类里显式调用，这样就不会造成歧义性了 。对 `Child` 类构造函数进行如下修改：

```c++
Child() : Base(0) { cout << "Child()" << endl; }
```

最后编译运行会得到如下输出：

```
Base(int)
ParentA()
ParentB()
Child()
```

这下是按照我们的预期运行了 w