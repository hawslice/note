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