---
title: "Effective Modern C++学习笔记"
date: 2016-06-14
slug: effective-modern-cpp-excerpt
tags:
- C++
---

本文是Effective Modern C++的一些学习笔记。

<!--more-->

## 类型推导规则 \#1 \#2

形如

```cpp
template<typename T>
void f(ParamType param);

f(expr);
```

通过`expr`推导出`T`和`ParamType`两个类型。区分以下3种情况：

- `ParamType`是指针或引用类型，但不是通用引用（形如`T&&`）；
- `ParamType`是通用引用；
- `ParamType`既不是指针也不是引用。

基本规则为：

- 当`ParamType`是引用时，推导`T`时引用会被忽略，`const`和`volatile`会被保留；
- 当`ParamType`是通用引用时，如果`expr`是左值，推导`T`和`param`的类型时会推导出左值引用类型，如果`expr`是右值，推导`T`时引用会被忽略，`param`的类型被推导为右值引用；
- `ParamType`既不是引用也不是指针时，推导时引用会被忽略，`const`和`volatile`也会被忽略；
- 当`expr`是数组或者函数指针时，如果`ParamType`不是引用，推导出的`T`是指针类型，如果`ParamType`是引用，推导出的`T`是数组类型（比如`char[5]`）或函数引用（比如`void (&)(int)`）

简而言之，只有一种情况下，`T`会被推导为左值引用类型（例如`int&`），那就是当`ParamType`是通用引用并且`expr`是左值的时候。

## `decltype` \#3

`decltype`在大多数情况下返回表达式的确切类型，包含引用类型和`const volaitle`。

在C++14中，`decltype(auto)`可以自动推导函数的返回类型。

注意一点，对于类型为`T`的变量名，`decltype`会返回`T`，但对于具有类型`T`的**左值**表达式，`decltype`会返回引用类型`T&`，例如`int x = 0;`，`decltype((x))`会返回`int&`

## `auto` \#5 \#6

使用`auto`自动推导类型可以减少录入，规避一些为妙的类型问题，减少重构的代价。在C++14中，`auto`可以用在lambda表达式的参数列表中。

使用`auto`声明变量时要注意一些使用了Proxy设计模式的接口，它们返回的并不是真正需要的对象。这时可以使用`static_cast`做类型转换，例如：

```cpp
auto x = static_cast<type_you_want>(proxy_expr);
```

## 统一初始化（花括号初始化） \#7

初始化有以下几种方式：

- 等号初始化，`int y = 0;`。不可拷贝的对象无法使用等号初始化，如`std::atomic<int> x = 0;`是**错误**的；
- 括号初始化，`int y(0);`。无法用于指定成员变量的默认值；
- 统一初始化（花括号初始化、花括号与等号共用），`int y { 0 };`。任何情况下都能使用。

当类的构造函数中有`initializer_list<T>`的重载时，编译器会**强烈**地选择这个重载，无论是否有更好的其它构造函数可以匹配参数。注意以下两个语句的区别：

```cpp
std::vector<int> x1(10, 20);
std::vector<int> x2{10, 20};
```

编写模板类时，必须在这两种构造方式中做出抉择。

使用空的初始化列表会调用默认构函数，如`Widget w{};`，这里不能使用括号初始化（`Widget w();`），因为编译器会倾向于将语句解释为函数声明，这是C++的“most vexing parse”问题。如果想选用`initializer_list`的构造函数，可以使用`Widget w{{}};`

另外，`auto`在推导`initializer_list<T>`时可以进行两部推导，即`auto x = { 1, 2 };`可以直接推导为`initializer_list<int>`，但对于摸板函数的类型推导，C++标准规定不允许从初始化列表直接推导为`initializer_list<T>`，需要这样做时，摸板函数应声明为：

```cpp
template<typename T>
void func(const initializer_list<T>& il);
```

当用等号初始化和括号初始化对象时，两者有区别（\#42）：

```cpp
std::regex r1 = nullptr;       // copy initialization
std::regex r2(nullptr);        // direct initialization
```

C++标准规定拷贝初始化不允许使用`explicit`声明的构造函数，直接初始化可以。

## `nullptr` \#8

`nullptr`比`NULL`好的地方在于避免重载的歧义，例如：

```cpp
void f(int);
void f(void *);

f(NULL);    //可能无法编译，或者选用f(int)
f(nullptr); //ok，选用f(void *)
```

## 使用`using`同义声明 \#9

使用`using`定义变量替代`typedef`，好处有：

- `using`支持模板声明，如：
```cpp
template<typename T>
using MyAllocList = std::list<T, MyAlloc<T>>;
```
- 在模板中使用`using`定义的类型时，前面可以不用加`typename`关键字，因为编译器知道使用`using`声明的一定是类型。

在`<type_traits>`中定义的一些用于类型操作的方法中，C++11版本仍在使用`typedef`，C++14加入了`using`的版本，如：

```cpp
std::remove_const<T>::type //C++11
std::remove_const_t<T>     //C++14
```

## 枚举类 \#10

使用`enum class`声明的枚举值在使用时需要加上名字空间，如`enum class Color { black, red }; auto c = Color::red;` 

使用`enum class`声明的枚举值不会自动转型成基本类型（`int`、`double`等），需要时必须显式转型。

旧式的`enum`和`enum class`都有其内部的表达类型，`enum class`默认使用`int`，两者都可以指定内部类型：

```cpp
enum class Status : std::uint32_t;
enum Color : std::uint8_t;
```

指定内部类型后，两者都支持前向声明。使用`<type_traits>`中的`std::underlying_type<E>::type`可以得到枚举类的内部类型。

## `delete` \#11

`delete`不仅可用在成员函数上，也可以用在任何函数上，甚至是特例化的模板函数，如：

```cpp
template<typename T>
void func(T t);

template<>
void func<int>(int t) = delete;
```

## 新的函数声明修饰符 \#12 \#14

C++11新增了如下函数声明标识符：

- `&`和`&&`，用于成员函数，可区分重载，当`*this`是左值或右值时选用；
- `override`，显示告诉编译器覆盖了父类方法。因为子类的函数要覆盖父类函数需要满足许多条件，很容易就会写出自以为覆盖了父类函数其实并没有的代码，使用`override`关键字可以让编译器帮助差错；
- `final`，用于类声明或函数声明，表示该类不能被继承或函数不能被覆盖；
- `noexcept`，声明函数不会抛出异常。C++11认为一个函数关于异常真正有用的信息是“它是否会抛出异常”。有许多函数在设计时要求被调用者不能抛出异常，如`std::vector::push_back`，仅当容器中的类的移动构造函数声明为`noexcept`时才会选用移动构造，否则为了异常安全，会选用拷贝构造。STL中的许多方法都需要这个属性。析构函数会默认为`noexcept`。标识了`noexcept`的函数如果抛出了异常会终止程序运行。

## 特殊函数生成规则 \#17

Big 5的原则：如果拷贝操作、移动操作、析构函数中有任一用户自定义的行为，那么编译器就不应该假设其自动生成的bitwise操作是正确的。C++11按照这个原则设计移动操作，但拷贝操作在C++98定制标准时原则还不明确，所以其行为与原则有差别。

- 默认构造函数：用户没有自定义默认构造函数时，编译器会自动生成；
- 析构函数：用户没有自定义默认析构函数时，编译器会自动生成；自带`noexcept`属性，默认**非**`virtual`;
- 拷贝构造函数：用户没有自定义拷贝构造函数和移动操作函数时，编译器会自动生成；拷贝赋值与用户自定义析构函数对拷贝构造函数的自动生成没有影响，但被标记为过时；
- 拷贝赋值函数：用户没有自定义拷贝赋值函数和移动操作函数时，编译器会自动生成；拷贝构造与用户自定义析构函数对拷贝赋值函数的自动生成没有影响，但被标记为过时；
- 移动构造函数和移动赋值函数：仅当用户没有自定义任何拷贝操作、移动操作和析构函数时，编译器会自动生成；
- 成员函数模板对特殊函数的生成没有任何影响。

## 智能指针 \#18-\#22

1. `std::unique_ptr`在性能方面都与裸指针一样。当使用默认的删除器（`delete`）或者lambda表达式作为删除器时，`std::unique_ptr`的大小不会改变，因为删除器是`std::unique_ptr`类型信息的一部分。使用函数指针或函数对象会增加其大小；
2. `std::unique_ptr`有专门为数组的特例化：`std::unique_ptr<T[]>`，数组版本不支持指针操作`operator*`和`operator->`，但支持索引操作`operator[]`（普通版本不支持索引操作）；
3. `std::shared_ptr`和`std::weak_ptr`通常占用两个指针的大小，一个指向数据，一个指向控制块。控制块包含对象引用计数和弱引用计数，当对象引用计数为0时，对象被析构，内存被回收（如果不是通过`std::make_shared`创建的话），当对象引用计数和弱引用计数都为0时，控制块内存被回收。所有引用计数的操作为原子操作，因此性能会受到影响。控制块中还存放自定义的分配器（通过`std::allocate_shared`）和删除器。`std::shared_ptr`的删除器是通过构造函数指定，不包含在模板类型信息中。
4. 在成员函数中如果需要获得对自身的`std::shared_ptr`，类，需要继承自`std::enable_shared_from_this<T>`，然后调用`shared_from_this()`。调用函数前该实例的控制块必须已经存在，即之前创建过`std::shared_ptr`的实例，否则会抛出异常；
5. 使用`pImpl`模式时，如果使用`std::unique_ptr`管理资源，由于删除器是`std::unique_ptr`类型的一部分，实现需要完整的类型，因此需要在头文件中声明Big 5，然后在实现文件中定义它们；如果使用`std::shared_ptr`，则不需要；
6. 以下情况慎用或不能使用`make_xxx`系列函数：
    - 需要自定义删除器时；
    - 需要使用`std::initializer_list`构造对象时；
    - 类自定义了`operator new`和`operator delete`时；
    - 对象本身占用很大空间且弱引用会存在很长时间时（由于控制块和对象一起分配导致对象的内存空间生存期与控制块生存期相同）。

## 移动语义和完美转发 \#23-\#30

`std::move`的简单实现，C++14版本：
```cpp
template<typename T>
decltype(auto) move(T&& param) {
    using ReturnType = remove_reference_t<T>&&;
    return static_cast<ReturnType>(param);
}
```

`std::forward`的简单实现，C++14版本：
```cpp
template<typename T>
T&& forward(remove_reference_t<T>& param) {
    return static_cast<T&&>(param);
}
```

`std::move`总是返回`param`的右值引用，`std::forward`保持`param`的左右性，如果`param`是左值，返回左值引用，如果`param`是右值，返回右值引用。

编程建议：
- 在右值引用的变量上调用`std::move`；
- 在通用引用（模板中）的变量上调用`std::forward`；
- 必须在确保变量不会继续使用时才能应用上述两条，防止对象被意外的移动掉。

以下几种情况下完美转发`std::forward`可能失败：
- 使用花括号初始化；
- 使用`NULL`或是`0`表示空指针，会与整型重载冲突。应换用`nullptr`；
- 传入仅声明而没有定义的`static const`变量时。因为完美转发转发的是引用，需要取变量地址，没有定义的变量没有地址，因此链接时会出错。需要定义该变量；
- 传入重载函数时，因为并不知道究竟需要哪个函数，会产生歧义。需要明确指定传入函数的签名；
- 使用位域（Bitfield）时。C++明确规定，非`const`引用不可绑定位域。即使参数为`const`引用，也是将位域所在的整型复制后绑定到引用上。因此解决办法就是先拷贝出一个变量，然后调用。

#### 引用规约（reference collapsing）

引用规约的规则：只要规约的引用中有左值引用，结果为左值引用；仅当所有的引用都为右值引用，结果才是右值引用。

以下情况下可能发生引用规约：

- 模板参数实例化时；
- 使用`auto`推导变量类型时；
- 模板中使用`typedef`定义类型时；
- 使用`decltype`时。

**通用引用的新定义：**通用引用实际上是以下两个条件都被满足的上下文中的右值引用：
- 类型推导区分左值和右值，类型`T`的左值被推导为`T&`，类型`T`的右值被推导为`T`；
- 发生了引用规约。

#### 通用引用与重载

当通用引用与重载一起使用时，会出现麻烦的事情，因为编译器倾向于使用模板产生一个完美匹配的重载，而不是通过类型转换去匹配另一个重载。可能遇到的情况有：

- 数值类型不会转换（例如从`short`提升到`int`），而是从模板中直接生成完美匹配；
- 使用非`const`对象调用拷贝构造时，不会选用拷贝构造，因为从模板生成的**非**`const`构造更加匹配；
- 子类调用父类的构造函数时，会使用从模板生成的构造函数，因为类型更加匹配。

解决办法有以下几种：

- 不使用重载，使用不同的函数名。构造函数的重载不适用这条；
- 使用`const T&`传递变量。无法享受通用引用带来的性能优势（移动操作、字面量字符串直接构造避免创建临时`std::string`等等）；
- 使用**Tag dispatch**。重载选择基于所有函数参数和实参的匹配结果，因此可以配合另一个参数进行重载选择，如`std::true_type`和`std::false_type`，这些参数只用于重载的类型匹配，不需要变量名，也不参与代码，因此称为**Tag**；
- 使用`std::enable_if`约束模板。编译器的SFINAE特性，匹配失败并不是一种错误(Substitution Failure Is Not An Error)。如解决上述3个问题的代码：

```cpp
class Person {
public:
    template<
        typename T,
        typename = std::enable_if_t<
        !std::is_base_of<Person, std::decay_t<T>>::value
        &&
        !std::is_integral<std::remove_reference_t<T>>::value
        >
    >
    explicit Person(T&& n);

    explicit Person(int idx);

};
```

这类代码如果出错，编译器会报出一大堆错误，找不着北，可以使用`static_assert`自行定位错误：
```cpp
    static_assert(
        std::is_constructible<std::string, T>::value,
        "Parameter n can't be used to construct a std::string"
    );
```

## lambda表达式 \#31-\#34

每个lambda表达式会生成一个lambda表达式类，闭包（closure）是lambda表达式类的实例。

lambda表达式捕获的主意事项：

- 默认的捕获会捕获`this`指针，有造成悬空指针的危险；
- 不能捕获静态生存周期的变量，如用`static`声明的变量。

C++14起支持初始化捕获，例如：

```cpp
[pw = std::move(pw)]() {
	...
}
```

使用初始化捕获可以将对象移动到闭包中。C++11不支持初始化捕获，要移动参数，需要使用`std::bind()`模拟。

C++14起支持泛lambda（generic lambda），可以在参数类型中使用`auto`，内部使用模板实现。当需要完美转发参数时，在变量名上使用`decltype`推导变量类型，如：

```cpp
auto f = [] (auto&& param) {
	return func(std::forward<decltype(param)>(param));
};
```

## 并发编程 \#35-\#40

#### `std::async()`

`std::async()`提供基于任务的并发编程模型。内置实现的调度器可以解决手写`std::thread`导致的线程过多、频繁上下文切换、（手写调度器时）平台移植等问题，还可以从返回的`std::future`获得返回值或发生的异常。当需要使用更加底层的API（如CPU亲和度、优先级）时，则只能使用`std::thread`。

`std::async()`的调度有两种模式，一种是并发运行（`std::launch::async`），一直是延后运行(`std::launch::deferred`)。指定为延后运行时，执行体会在调用`std::future::get()`时才在调用者的线程运行。默认值为两者都指定（`std::launch::async | std::launch::deferred`），由调度器根据运行时的负载决定使用哪种策略。这一随机特性使得只有满足以下条件时才能使用默认的调度策略，否则应该指定调度策略：

- 任务不需要与调用`get`或`wait`的线程并发运行；
- 并不需要读写指定线程的`thread_local`变量；
- 除非能保证在`std::async()`返回的`std::future`上调用`get()`或`wait()`，否则任务不一定会运行（负载高时调度器选择`deferred`策略）；
- 当调用`wait_for()`或者`wait_until()`函数时，要考虑到`std::future_status::deferred`状态。

#### `std::thread`

`std::thread`在销毁时，必须处于不可连接状态（unjoinable），即与底层的线程实现分离，有以下几种方式可让`std::thread`处于不可连接状态：

- 默认构造的`std::thread`；
- 被移动掉的`std::thread`；
- 已经调用过`join()`的`std::thread`；
- 已经调用过`detach()`的`std::thread`。

#### `std::future`

`std::future`只可被移动，不能被拷贝，且只能调用一次`get()`，再次调用会出错，通过调用`std::future::share()`可以创建`std::shared_future`，它可以被拷贝，且可以调用多次`get()`，可以在多个线程间分享，通过独立的`std::shared_future`对对象的访问是线程安全的。

`std::promise`也可以创建`std::future`，两者通过一个在堆上分配的shared state通信，同时包含引用计数。

`std::future`的析构函数通常仅销毁其成员变量，但是当`std::future`是通过`std::async()`以`async`策略调度运行（从而会创建新线程）所创建的最后一个指向shared state的`future`时，会阻塞直到任务完成（在线程上调用了`join()`）。

`std::future<void>`可以用来进行线程间的一次性通信。

#### `volatile`

`volatile`关键字用于禁止编译器对特定内存位置的优化。例如这样的代码：

```cpp
auto y = x;
y = x;       // redundant loads
x = 10;
x = 20;      // dead stores
```

很有可能被编译器优化为：

```cpp
auto y = x;
x = 20;
```

使用`volatile`可以禁用这样的优化。

## `emplace`系列函数 \#42

`emplace`系列函数的性能不是一定比`push`、`insert`系列好。满足以下几个条件时基本可以认为`emplace`的性能更好：

- 要插入的值是被构造进容器，而不是通过赋值（`operator=`）进容器的（例如插入到`vector`的队首时）；
- 传递参数的类型与容器的类型不同（因此通过`push`和`insert`函数插入时需要先构造临时变量）；
- 容器不太会因为值重复而拒绝插入新的值。

在向容器插入资源管理对象（如`std::shared_ptr`）时，一定要先创建好资源管理对象。当使用`emplace`系列函数时，获取资源与创建资源管理对象之间有间隔，一但发生了异常（如`vector`重新调整缓冲大小时内存不够），资源可能会泄露，如以下代码具有潜在的资源泄露风险：

```cpp
std::list<std::shared_ptr<Widget>> ptrs;
ptrs.emplace_back(new Widget, killWidget);
```

当使用`emplace`系列函数时，要注意因为参数是完美转发到构造函数，因此使用的是直接初始化，允许使用`explicit`声明的构造函数，这种情况在使用`push`和`insert`系列函数时不会发生，因为编译器不允许`explicit`构造函数隐式创建临时变量。