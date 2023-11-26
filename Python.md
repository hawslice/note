## 描述符
### 1. 什么是描述符

如果一个类实现了 `__get__` `__set__` `__delete__` 其中的任意一种方法， 用这个类创建的对象称为**描述符对象**。

一个类如果把一个**描述符对象**当作**类属性**，则称这个**类属性**为**描述符**。

通过描述符，能够实现调用属性时执行特定的方法。

**示例：**

```python
class Descriptor:
    def __get__(self, instance, owner):
        print('get called')

    def __set__(self, instance, value):
        print('set called')

    def __delete__(self, instance):
        print('delete item')


class Foo:
    bar = Descriptor()


if __name__ == '__main__':
    foo = Foo()
    foo.bar
    foo.bar = 10
    del foo.bar
```

### 2. 函数参数说明
`def __get__(self, instance, owner)`

- `self`： 当前描述符对象
- `instance`：调用该描述符的实例，或者 `None`（直接用类对象调用）
- `owner`：包含该描述符的类对象

`def __set__(self, instance, value)`

- `value`：要设置的值

### 3. 描述符调用案例

`Person` 类中包含一个描述符对象 `name`，要求为 `name` 设置值时只能是 **字符串** 类型，否则抛出异常。

```python
class NameDes:

    def __init__(self):
        self.__name = None

    def __get__(self, instance, owner):
        return self.__name

    def __set__(self, instance, value):
        if isinstance(value, str):
            self.__name = value
        else:
            raise TypeError('value must be str')

    def __delete__(self, instance):
        del self.__name


class Person:
    name = NameDes()


if __name__ == '__main__':
    person = Person()
    person.name = 'Mike'
    print(person.name)
    try:
        person.name = 2
    except TypeError as e:
        print(e)
```

由于 `name` 是一个类属性，因此如果我们创建多个 `Person` 对象，通过一个对象对 `name` 进行修改，所有对象在调用 `name` 时都将返回被修改后的值。

同时，由于 `name` 是一个[数据描述符](#7-描述符分类)，因此在实例中创建同名属性也会优先访问该描述符，即无法用同名属性覆盖该描述符。

为了解决上述问题，设想让各实例维护一个 `_Person__name` 实例属性。

```python
class MyProperty:

    def __init__(self, various, cls_name):
        self.key = '_{}__{}'.format(cls_name, various)

    def __get__(self, instance, owner):
        return instance.__dict__[self.key]

    def __set__(self, instance, value):
        if isinstance(value, str):
            instance.__dict__[self.key] = value
        else:
            raise TypeError('value must be a string')

    def __delete__(self, instance):
        del instance.__dict__[self.key]


class Person:
    name = MyProperty('name', 'Person')
```

虽然上述方法可以保证每个 `Person` 实例在调用 `name` 时得到的结果相互独立，但是，如果类内又使用了`self.__name` 就会造成冲突。

一个更好的解决方法是在描述符中维护一个 `dict` 对象来存储数据 ，以 `Person` 实例的引用作为键。

```python
class MyProperty2:

    def __init__(self, default):
        self.default = default
        self.data = dict()

    def __get__(self, instance, owner):
        return self.data.get(instance, self.default)

    def __set__(self, instance, value):
        if isinstance(value, str):
            self.data[instance] = value
        else:
            raise TypeError('value must be a string')

    def __delete__(self, instance):
        del self.data[instance]

class Person2:
    name = MyProperty2('')
```

### 4. 使用描述符的优势

如果存在多个属性拥有相同地获取与设置逻辑，使用 `property` 装饰器难免会产生很多冗余代码。而使用自定义描述符，仅需定义一个类对象，然后定义不同的类属性即可。

### 5. 使用描述符实现 `classmethod`

标准的 `classmethod`的行为如下：
```python
class Demo:
    @classmethod
    def func(cls, *args):
        """do something"""
```

经过 `classmethod` 装饰，函数 `func` 变成一个类方法，并且第一个参数为类本身。

为了实现上述行为，要先理解装饰器的行为。上面的例子等同于下面的代码：

```python
class Demo:
    def func(cls, *args):
        """do something"""
    func = classmethod(func)
```

我们自定义的装饰器类命名为 `clsmethod`，如果用他去装饰 `func`，效果等同于 `func = clsmethod(func)`，也就是 `func` 将指向 `clsmethod` 的一个实例对象，也就是一个描述符。

而我们在使用`func` 时会把他当作一个普通的函数，因此我们可以让 `__get__` 方法返回一个可调用对象，其行为与原始的 `func` 相同。这一部分可以利用闭包实现。

```python
class clsmethod:

    def __init__(self, func):
        self.func = func

    def __get__(self, instance, owner):
        def call(*args):
            return self.func(owner, *args)

        return call


class Foo:

    tool = 'pip'

    @clsmethod
    def get_tool(cls):
        return cls.tool
```

### 6. 描述符调用机制

在 python 中所有对象的所有属性以及方法，都存储在 `__dict__` 字典中，属性名或者方法名当作 `key`，而执行的内容当作 `value`。可以使用 `obj.__dict__` 查询。

通常情况下，实例的 `__dict__` 中只存储实例属性，类属性以及方法存储在类对象的 `__dict__` 中。

在调用对象 `obj` 的属性或者方法时，查询顺序如下：

1. 查询 `obj.__dict__`
2. 查询 `type(obj).__dict__`
3. 查询 `type(type(obj)).__dict__`
4. ...

期间，查询到普通值就返回，或者查询到一个描述符调用它的 `__get__` 方法。关于直接返回值还是调用 `__get__` 方法，这是由 `__getattribute__` 方法控制的。

### 7. 描述符分类

同时定义了 `__get__` 和 `__set__` 方法的描述符称之为 **数据描述符**；如果一个描述符只定义了 `__get__` 方法，则称之为非数据描述符。

当实例属性名和描述符名相同时，在访问该同名属性时，优先级从高到低分别为 数据描述符、实例属性、非数据描述符。

**示例：**

```python
class M:

    def __init__(self):
        self.x = 1

    def __get__(self, instance, owner):
        print('M.get()')
        return self.x

    def __set__(self, instance, value):
        print('M.set()')
        self.x = value


class N:

    def __init__(self):
        self.x = 1

    def __get__(self, instance, owner):
        print('N.get()')
        return self.x


class A:
    m = M()
    n = N()

    def __init__(self, m, n):
        self.m = m
        self.n = n


if __name__ == '__main__':
    print(A.__dict__)
    a = A(2, 2)
    print(a.__dict__)
```

**运行结果如下：**

```
{'__module__': '__main__', 'm': <__main__.M object at 0x0000023FFFF59910>, 'n': <__main__.N object at 0x0000023FFFD1CF90>, '__init__': <function A.__init__ at 0x0000023FFFE1AC00>, ...}
M.set()
{'n': 2}
```

以上结果证明了三者的优先级顺序，由于 `M` 实现了 `__get__` 方法和 `__set__` 方法，因此它的实例是一个数据描述符；而 `N` 仅仅实现了 `__get__` 方法，则它的实例是优先级最低的非数据描述符。

在 `A` 的 `__init__` 方法中， 我们分别为二者进行赋值，结果表明，`m` 的 `__set__` 方法被调用，未创建新的实例属性 `m`；而对 `n` 的赋值导致创建了一个新的实例属性 `n`。

如果想要构建一个**只读的数据描述符**，需要同时定义 `__get__` 和 `__set__` 方法，并且在 `__set__` 方法中抛出 `AttributeError`。通过这个方法定义 `__set__`， 就可以使描述符是数据描述符，并且只读。

### 8. 描述符注意点

1. 不可以使用实例属性定义描述符。如果将描述符赋值给实例属性，仅仅是将实例属性指向了描述符实例，在调用时不会自动调用 `__get__` 方法，起不到描述符的作用。
2. 描述符是类属性，所有由该类创建的实例共享一个描述符。因此在定义 `__get__` 和 `__set__` 方法时要格外注意，防止多个实例进行操作时发生冲突。一个解决方案是在描述符中用字典来为不同实例存储数据。

### 9. 描述符的应用案例

使用描述符实现惰性计算，要求在访问后进行求值，并且将所求得的值存储在实例中，避免重复计算。

````python
class LazyProperty:

    def __init__(self, func):
        self.func = func

    def __get__(self, instance, owner):
        print('get')
        if instance is None:
            return self
        value = self.func(instance)
        setattr(instance, self.func.__name__, value)
        return value


class ReadonlyProperty:

    def __init__(self, value):
        self.value = value

    def __get__(self, instance, owner):
        return self.value

    def __set__(self, instance, value):
        raise AttributeError


class Circle:
    pi = ReadonlyProperty(3.14)

    def __init__(self, radius):
        self.radius = radius

    @LazyProperty
    def area(self):
        return self.pi * (self.radius ** 2)


if __name__ == '__main__':
    circle = Circle(2)
    print(circle.__dict__)
    print(circle.area)
    print(circle.__dict__)
    print(circle.area)
````

**运行结果：**

```
{'radius': 2}
get
12.56
{'radius': 2, 'area': 12.56}
12.56
```

上面的代码中 `LazyProperty` 只定义了 `__get__` 方法，因此它的实例是一个非数据描述符，优先级低于实例属性，因此可以将运算结果存储在实例属性中用于后续访问。

为了避免 `pi` 值被修改，用之前提到的方法定义了一个只读的数据描述符。

上述的代码中存在一个问题，当 `radius` 被修改后，圆的面积不会被更新。有很多办法来解决这个问题，但这个例子只是用来说明描述符的应用。

## GIL(Global Interpreter Lock)

### 1. Python GIL 的概念以及对多线程的影响

- Pytho 语言和 GIL 没有关系，仅仅是因为历史原因在 CPython 虚拟机（解释器）中难以移除。
- GIL：全局解释器锁，每个线程在执行的时候都需要先获取 GIL，以此来保证每个时刻只有一个线程可以执行。
- 线程释放 GIL 的情况：在 IO 操作等可能引起阻塞的 System Call 之前，可以暂时释放 GIL，但在执行完毕之后必须重新获取。时间片用尽也会被强制释放。
- Python 多进程是可以利用 CPU 的多个核心的。
- 多线程爬取比单线程有提升，是因为遇到 IO 阻塞会自动释放 GIL。

### 2. 避免 GIL 的方式

调用其他语言的动态库函数作为子线程。

```c
// demo.c
#include <stdio.h>

void dead_loop() {
    while (true) {
        ;
    }
}
```

```bash
gcc -shared -fPIC -o demo.so demo.c
```

```python
from ctypes import cdll
from threading import Thread

lib = cdll.LoadLibrary('./demo.c')
t = Thread(target=lib.dead_loop)
t.start()

while True:
    pass
```

### 3. 为什么有 GIL 还要设置互斥锁

GIL 是为了解决资源竞争问题，但竞争的内容是 CPU，保证了同时只运行一个线程；互斥锁避免的是对数据资源的竞争，防止数据错乱，使得业务能够顺利进行。

## Garbage Collection(GC 垃圾回收)

现在的高级语言如 java，c# 等，都采用了垃圾收集机制，而不再是 c，c++ 里用户自己管理维护内存的方式。自己管理内存极其自由，可以任意申请内存，但如同一把双刃剑，为大量内存泄露，悬空指针等 bug 埋下隐患。 对于一个字符串、列表、类甚至数值都是对象，且定位简单易用的语言，自然不会让用户去处理如何分配回收内存的问题。 python 里也同 java 一样采用了垃圾收集机制，不过不一样的是：python采用的是**引用计数机制**为主，**分代回收机制**为辅的策略。

###  1. 引用计数机制

**引用计数机制：** 每一个对象维护一小块内存存放自己的引用数量，当有一个新的对象引用它时，引用计数会增加，当引用它的对象被删除时，引用计数减少。当引用计数变为 0 时，对象的生命周期结束。

**引用计数的优点：**

- 简单
- 实时性：一旦没有引用，内存就直接释放了。不用像其他机制等到特定时机。实时性还带来一个好处：处理回收内存的时间分摊到了平时。

**引用计数的缺点：**

- 维护引用计数消耗资源
- 循环引用

```python
# 循环引用
list1 = []
list2 = []
list1.append(list2)
list2.append(list1)
```

`list1` 与 `list2` 相互引用，如果不存在其他对象对它们的引用，`list1` 与 `list2` 的引用计数也仍然为 1，所占用的内存永远无法被回收，这将是致命的。 对于如今的强大硬件，消耗资源的缺点尚可接受，但是循环引用导致的内存泄露，使得 python 必将引入新的回收机制（分代回收）。

**导致引用计数增加的情况：**

- 对象被创建，例如 `a = 23`
- 对象被引用，例如 `b = a`
- 对象被作为参数，传入到一个函数中，例如 `func(a)`
- 对象作为一个元素，存储在容器中，例如 `list1 = [a, a]`

**导致引用计数减少的情况：**

- 对象的别名被显式销毁，例如 `del a`
- 对象的别名被赋予新的对象，例如 `a = 24`
- 一个对象离开它的作用域，例如函数执行完毕时，`func` 函数中的局部变量（全局变量不会）
- 对象所在的容器被销毁，或从容器中删除对象

**查看一个对象的引用计数：** 

```python
import sys
a = "hello world"
sys.getrefcount(a)
# 可以查看 a 对象的引用计数，但是比正常计数大 1，因为调用函数的时候传入 a，这会让 a 的引用计数 +1
```

### 2. 标记-清理

在介绍分代回收之前，先介绍标记删除机制。标记删除依赖两个容器来实现，分别是**死亡容器**和**存活容器**。

**清理过程：**

1. 对执行删除操作后的每个对象引用计数  -1，如果引用计数变为 0 将它们存储到死亡容器。
2. 遍历存活容器，查看存活容器中是否有对象引用了死亡容器中的对象，如果有，将该死亡容器中的对象存储到存货容器
3. 删除死亡容器中的所有对象

经过这上述三个步骤，可以解决循环引用对象无法被清理的问题。

### 3. 分代回收

使用**标记-清理**方法，已经可以保证对垃圾的回收了，但还有一个问题，**标记-清理**该什么时候执行。这就是分代回收机制要解决的问题。

- 分代回收是一种以空间换时间的操作方式，Python 将内存根据对象的存活时间划分为不同的集合，每个集合称为一个代，Python 将内存分为了 3 代，分别为年轻代（第 0 代）、中年代（第 1 代）、老年代（第 2 代），他们对应的是 3 个链表，它们的垃圾收集频率随着对象存活时间的增大而减小。
- 新创建的对象都会分配在**年轻代**，年轻代链表的总数达到上限时（内存分配数减去内存释放数达到阈值），Python 垃圾收集机制就会被触发，把那些可以被回收的对象回收掉，而那些不会回收的对象就会被移到**中年代**去，依此类推，**老年代**中的对象是存活时间最久的对象，甚至是存活于整个系统的生命周期内。
- 同时，分代回收是建立在标记清除技术基础之上。分代回收同样作为 Python 的辅助垃圾收集技术处理那些容器对象

### 4. 触发垃圾回收

1. 当 `gc` 模块的计数器达到阈值的时候，自动回收垃圾
2. 调用 `gc.collect()`，手动回收垃圾
3. 程序退出的时候，python 解释器来回收垃圾

## logging 日志模块

### 1. 输出到文件

```python
import logging

logging.basicConfig(level=logging.DEBUG,
                    filename = './demo.log'
                    filemode = 'a'
                    format='%(asctime)s - %(filename)s[line:%(lineno)d] - %(levelname)s: %(message)s')

logging.info('message')
```

### 2. 同时输出到终端和文件

```python
import logging  
  
# 第一步，创建一个logger  
logger = logging.getLogger()  
logger.setLevel(logging.INFO)  # Log等级总开关  
  
# 第二步，创建一个handler，用于写入日志文件  
logfile = './log.txt'  
fh = logging.FileHandler(logfile, mode='a')  # open的打开模式这里可以进行参考
fh.setLevel(logging.DEBUG)  # 输出到file的log等级的开关  
  
# 第三步，再创建一个handler，用于输出到控制台  
ch = logging.StreamHandler()  
ch.setLevel(logging.WARNING)   # 输出到console的log等级的开关  
  
# 第四步，定义handler的输出格式  
formatter = logging.Formatter("%(asctime)s - %(filename)s[line:%(lineno)d] - %(levelname)s: %(message)s")  
fh.setFormatter(formatter)  
ch.setFormatter(formatter)  
  
# 第五步，将logger添加到handler里面  
logger.addHandler(fh)  
logger.addHandler(ch)  
  
# 日志  
logger.debug('这是 logger debug message')  
logger.info('这是 logger info message')  
logger.warning('这是 logger warning message')  
logger.error('这是 logger error message')  
logger.critical('这是 logger critical message')
```

### 3. 日志格式说明

`logging.basicConfig` 函数中，可以指定日志的输出格式 `format` ，这个参数可以输出很多有用的信息，如下:

- `%(levelno)s`: 打印日志级别的数值
- `%(levelname)s`: 打印日志级别名称
- `%(pathname)s`: 打印当前执行程序的路径，其实就是 `sys.argv[0]`
- `%(filename)s`: 打印当前执行程序名
- `%(funcName)s`: 打印日志的当前函数
- `%(lineno)d`: 打印日志的当前行号
- `%(asctime)s`: 打印日志的时间
- `%(thread)d`: 打印线程 ID
- `%(threadName)s`: 打印线程名称
- `%(process)d`: 打印进程 ID
- `%(message)s`: 打印日志信息

在工作中常用的格式如下：

```python
format ='%(asctime)s - %(filename)s[line:%(lineno)d] - %(levelname)s: %(message)s'
```

