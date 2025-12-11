---
title: "Effective Modern C++ Study Notes"
date: 2016-06-14
slug: effective-modern-cpp-excerpt
tags:
- C++
---

{{<alert warning>}}
This article is translated from Chinese to English by ChatGPT. There might be errors.
{{</alert>}}

This article contains some study notes from Effective Modern C++.

<!--more-->

## Type Deduction Rules #1 #2

For code of the form

```cpp
template<typename T>
void f(ParamType param);

f(expr);
```

`T` and `ParamType` are deduced from `expr`. There are three cases:

- `ParamType` is a pointer or reference type, but not a universal reference (of the form `T&&`);
- `ParamType` is a universal reference;
- `ParamType` is neither a pointer nor a reference.

Basic rules:

- When `ParamType` is a reference, the reference is ignored when deducing `T`, while `const` and `volatile` are preserved;
- When `ParamType` is a universal reference, if `expr` is an lvalue, `T` and `param` will be deduced as lvalue reference types; if `expr` is an rvalue, the reference is ignored when deducing `T`, and `param` is deduced as an rvalue reference;
- When `ParamType` is neither a reference nor a pointer, references are ignored during deduction, and `const` and `volatile` are also ignored;
- When `expr` is an array or a function pointer, if `ParamType` is not a reference, the deduced `T` is a pointer type; if `ParamType` is a reference, the deduced `T` is an array type (for example `char[5]`) or a function reference (for example `void (&)(int)`).

In short, `T` is deduced as an lvalue reference type (such as `int&`) only in one situation: when `ParamType` is a universal reference and `expr` is an lvalue.

## `decltype` #3

In most cases, `decltype` returns the exact type of an expression, including reference types and `const volatile`.

In C++14, `decltype(auto)` can be used to deduce a function’s return type automatically.

Note that for a variable name whose type is `T`, `decltype` yields `T`; but for an expression of type `T` that is an *lvalue*, `decltype` yields the reference type `T&`. For example, with `int x = 0;`, `decltype((x))` yields `int&`.

## `auto` #5 #6

Using `auto` for type deduction reduces typing, avoids some subtle type issues, and lowers refactoring costs. In C++14, `auto` can be used in lambda parameter lists.

When declaring variables with `auto`, be careful about interfaces that use the Proxy design pattern, which may not return the actual object you want. In these cases you can use `static_cast` to convert the type, for example:

```cpp
auto x = static_cast<type_you_want>(proxy_expr);
```

## Uniform Initialization (Brace Initialization) #7

Initialization forms include:

- Copy (=) initialization: `int y = 0;`. Non-copyable objects cannot use copy initialization; for example `std::atomic<int> x = 0;` is **ill-formed**;
- Direct (parentheses) initialization: `int y(0);`. Cannot be used to specify default values for data members;
- Uniform (brace) initialization (with or without `=`): `int y{ 0 };`. Can be used in all situations.

When a class has a constructor overload taking `initializer_list<T>`, the compiler will **strongly** prefer this overload, even if other constructors would match the arguments better. Note the difference between the following two statements:

```cpp
std::vector<int> x1(10, 20);
std::vector<int> x2{10, 20};
```

When writing templates, you must choose between these two construction forms.

Using an empty initializer list calls the default constructor, for example `Widget w{};`. Parentheses initialization (`Widget w();`) cannot be used here, because the compiler tends to interpret the statement as a function declaration, which is the C++ “most vexing parse” problem. To select the `initializer_list` constructor, you can use `Widget w{{}};`.

In addition, `auto` can do two-step deduction with `initializer_list<T>`, i.e. `auto x = { 1, 2 };` is directly deduced as `initializer_list<int>`. But for template argument deduction, the C++ standard forbids deducing `initializer_list<T>` directly from an initializer list. When you need this, the template function should be declared as:

```cpp
template<typename T>
void func(const initializer_list<T>& il);
```

When initializing an object with `=` and with parentheses, there is a difference (#42):

```cpp
std::regex r1 = nullptr;       // copy initialization
std::regex r2(nullptr);        // direct initialization
```

The C++ standard forbids calling `explicit` constructors for copy initialization, but allows them for direct initialization.

## `nullptr` #8

`nullptr` is better than `NULL` in avoiding overload ambiguities, for example:

```cpp
void f(int);
void f(void *);

f(NULL);    // may fail to compile, or choose f(int)
f(nullptr); // ok, chooses f(void *)
```

## Using `using` for Type Aliases #9

Use `using` instead of `typedef` to define aliases; advantages:

- `using` supports templates, for example:
```cpp
template<typename T>
using MyAllocList = std::list<T, MyAlloc<T>>;
```
- When using a type defined with `using` inside templates, you usually don’t need the `typename` keyword, because the compiler knows that a `using` declaration always introduces a type.

Some type utilities in `<type_traits>` still use `typedef` in C++11; C++14 adds `using`-based versions, for example:

```cpp
std::remove_const<T>::type // C++11
std::remove_const_t<T>     // C++14
```

## Scoped Enums #10

For enums declared with `enum class`, enumerators must be qualified with the enum’s scope, for example `enum class Color { black, red }; auto c = Color::red;`.

Enumerators declared with `enum class` are not implicitly convertible to fundamental types (`int`, `double`, etc.); explicit casts are required when needed.

Both old-style `enum` and `enum class` have an underlying type. The default underlying type for `enum class` is `int`, and both forms can specify an underlying type:

```cpp
enum class Status : std::uint32_t;
enum Color : std::uint8_t;
```

After specifying an underlying type, both support forward declarations. You can use `std::underlying_type<E>::type` in `<type_traits>` to obtain an enum’s underlying type.

## `delete` #11

`delete` can be used not only on member functions, but on any function, including specialized template functions, for example:

```cpp
template<typename T>
void func(T t);

template<>
void func<int>(int t) = delete;
```

## New Function Specifiers #12 #14

C++11 adds the following function specifiers:

- `&` and `&&` for member functions, enabling overloads that are selected depending on whether `*this` is an lvalue or rvalue;
- `override` explicitly informs the compiler that a function overrides a base-class method. Because overriding requires many conditions to hold, it’s easy to write code that appears to override but actually doesn’t; `override` lets the compiler catch such errors;
- `final` applied to a class or function, indicating that the class cannot be derived from, or the function cannot be overridden;
- `noexcept` declares that a function does not throw exceptions. In C++11, the only really useful information about exceptions is “whether a function may throw.” Many functions are designed with the requirement that callees must not throw, such as `std::vector::push_back`; it will use the move constructor only when the element type’s move constructor is declared `noexcept`, otherwise for exception safety it will use the copy constructor. Many STL functions depend on this property. Destructors are `noexcept` by default. If a function marked `noexcept` throws, the program is terminated.

## Rules for Generating Special Member Functions #17

The Big Five principle: if any of the copy operations, move operations, or destructor has user-defined behavior, the compiler should not assume its auto-generated bitwise operations are correct. C++11 designs move operations according to this principle, but in C++98, the principle was not yet clear when specifying copy operations, so their behavior differs somewhat.

- Default constructor: generated automatically if the user does not define one;
- Destructor: generated automatically if the user does not define one; implicitly `noexcept`, and by default **not** `virtual`;
- Copy constructor: generated automatically if the user does not define a copy constructor or any move operations; the presence of a user-defined destructor does not prevent generation of the copy constructor, but this behavior is deprecated;
- Copy assignment operator: generated automatically if the user does not define a copy assignment operator or any move operations; the presence of a user-defined destructor does not prevent generation of the copy assignment operator, but this behavior is deprecated;
- Move constructor and move assignment operator: generated automatically only if the user does not define any copy operations, move operations, or a destructor;
- Member function templates have no effect on the generation of these special member functions.

## Smart Pointers #18–#22

1. `std::unique_ptr` has performance comparable to raw pointers. When using the default deleter (`delete`) or a lambda expression as the deleter, the size of `std::unique_ptr` does not change, because the deleter is part of the `std::unique_ptr`’s type information. Using a function pointer or function object increases its size;
2. `std::unique_ptr` has a partial specialization for arrays: `std::unique_ptr<T[]>`. The array version does not support `operator*` and `operator->`, but does support `operator[]` (the non-array version does not support indexing);
3. `std::shared_ptr` and `std::weak_ptr` typically occupy the size of two pointers, one pointing to the object and one to the control block. The control block contains the strong reference count and weak reference count. When the strong count reaches 0, the object is destroyed and its memory is reclaimed (unless it was not allocated via `std::make_shared`); when both strong and weak counts are 0, the control block itself is deallocated. All reference count operations are atomic, which affects performance. The control block also stores custom allocators (via `std::allocate_shared`) and deleters. A `std::shared_ptr`’s deleter is specified via its constructor and is not part of the template type;
4. If a member function needs to obtain a `std::shared_ptr` to `this`, the class must inherit from `std::enable_shared_from_this<T>` and then call `shared_from_this()`. Before calling this function, the instance must already have an associated control block, i.e. at least one `std::shared_ptr` has been created for it; otherwise an exception is thrown;
5. When using the pImpl idiom, if you manage resources with `std::unique_ptr`, the deleter is part of the `std::unique_ptr`’s type, and thus the implementation type must be complete, so you must declare the Big Five in the header and define them in the implementation file; if you use `std::shared_ptr`, this is not required;
6. Use the `make_xxx` family with care or not at all in the following situations:
    - When a custom deleter is required;
    - When the object must be constructed with `std::initializer_list`;
    - When the class defines custom `operator new` and `operator delete`;
    - When the object itself is large and weak references will live for a long time (because the control block and object are allocated together, the object’s memory lifetime matches that of the control block).

## Move Semantics and Perfect Forwarding #23–#30

A simple C++14 implementation of `std::move`:

```cpp
template<typename T>
decltype(auto) move(T&& param) {
    using ReturnType = remove_reference_t<T>&&;
    return static_cast<ReturnType>(param);
}
```

A simple C++14 implementation of `std::forward`:

```cpp
template<typename T>
T&& forward(remove_reference_t<T>& param) {
    return static_cast<T&&>(param);
}
```

`std::move` always returns an rvalue reference to `param`; `std::forward` preserves the value category of `param`: if `param` is an lvalue, it returns an lvalue reference; if `param` is an rvalue, it returns an rvalue reference.

Guidelines:

- Call `std::move` on variables that are rvalue references;
- Call `std::forward` on variables that are universal references (in templates);
- Apply the above only when you are sure the variable will not be used afterward, to avoid accidentally moving from an object that is still needed.

Situations where perfect forwarding via `std::forward` may fail:

- Brace initialization;
- Using `NULL` or `0` for null pointers, which can conflict with overloads for integer types; use `nullptr` instead;
- Forwarding `static const` variables that are declared but not defined. Perfect forwarding forwards references and may require taking the variable’s address. Undeclared definitions have no address, leading to link errors. The variable must be defined;
- Forwarding overloaded functions, because it is unclear which overload is intended, leading to ambiguity. Specify the function signature explicitly;
- Using bitfields. C++ explicitly forbids binding non-`const` references to bitfields. Even if the parameter is a `const` reference, the bitfield is first copied into an integer, and the reference binds to that. The workaround is to copy into a variable first and then call the function.

#### Reference Collapsing

Reference collapsing rules: if any of the references being collapsed is an lvalue reference, the result is an lvalue reference; only if all references are rvalue references is the result an rvalue reference.

Reference collapsing can occur:

- During template argument instantiation;
- When deducing variable types with `auto`;
- When using `typedef` for types in templates;
- When using `decltype`.

**New definition of universal references:** a universal reference is, effectively, an rvalue reference in a context where both of the following hold:

- Type deduction distinguishes between lvalues and rvalues: an lvalue of type `T` is deduced as `T&`, an rvalue of type `T` as `T`;
- Reference collapsing occurs.

#### Universal References and Overloading

When universal references are combined with overloading, things can get tricky, because the compiler tends to instantiate a template to create a perfect match instead of choosing another overload that requires a conversion. Possible issues:

- Numeric types are not converted (e.g., `short` promoted to `int`); instead, a template is instantiated to create a perfect match;
- When calling a copy constructor with a non-`const` object, the copy constructor may not be chosen, because a **non-`const`** constructor generated from a template is a better match;
- When a derived class calls a base-class constructor, the constructor generated from a template may be chosen because its type match is better.

Possible solutions:

- Avoid overloading; use different function names instead. This does not apply to constructors;
- Pass variables by `const T&`. This forfeits the performance benefits of universal references (moves, constructing from string literals without creating temporary `std::string`s, etc.);
- Use tag dispatch. Overload resolution is based on matching all function parameters and arguments, so you can use an extra parameter to influence overload selection, such as `std::true_type` and `std::false_type`. These parameters are used only for overload resolution, need no variable name, and are not part of the runtime code, hence “tags”;
- Constrain templates with `std::enable_if`. Using SFINAE (Substitution Failure Is Not An Error), substitution failures do not cause errors. For example, code to address the three issues above:

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

If this kind of code fails, compilers can emit a huge amount of diagnostics. You can use `static_assert` to localize the error:

```cpp
    static_assert(
        std::is_constructible<std::string, T>::value,
        "Parameter n can't be used to construct a std::string"
    );
```

## Lambda Expressions #31–#34

Each lambda expression generates a unique lambda class; a closure is an instance of that class.

Notes on lambda captures:

- Default capture captures the `this` pointer, which may cause dangling pointer issues;
- Static-storage-duration variables, such as those declared with `static`, cannot be captured.

Since C++14, init-capture is supported, for example:

```cpp
[pw = std::move(pw)]() {
    ...
}
```

Init-capture allows moving objects into the closure. C++11 does not support init-capture; to move parameters into a closure, you must simulate it with `std::bind()`.

Since C++14, generic lambdas are supported; you can use `auto` in parameter types, implemented internally with templates. When you need perfect forwarding, use `decltype` on the parameter name to deduce its type, for example:

```cpp
auto f = [] (auto&& param) {
    return func(std::forward<decltype(param)>(param));
};
```

## Concurrency Programming #35–#40

#### `std::async()`

`std::async()` provides a task-based concurrency model. Its built-in scheduler avoids problems caused by manually using `std::thread`, such as too many threads, frequent context switching, and portability issues (when writing your own scheduler). It also lets you retrieve return values or exceptions from the associated `std::future`. When you need low-level APIs (CPU affinity, thread priority, etc.), you must use `std::thread`.

`std::async()` supports two launch policies: concurrent execution (`std::launch::async`) and deferred execution (`std::launch::deferred`). With deferred execution, the task body runs in the caller’s thread when `std::future::get()` is called. The default policy is both (`std::launch::async | std::launch::deferred`), leaving the scheduler to choose at runtime based on system load. Because of this nondeterminism, you should use the default only when all of the following hold; otherwise specify the launch policy explicitly:

- The task does not need to run concurrently with the thread calling `get` or `wait`;
- You don’t need to read or write `thread_local` variables of a specific thread;
- Unless you can guarantee that `get()` or `wait()` will be called on the `std::future` returned by `std::async()`, the task might never run (if the scheduler chooses `deferred` under heavy load);
- When calling `wait_for()` or `wait_until()`, consider the `std::future_status::deferred` state.

#### `std::thread`

When a `std::thread` is destroyed, it must be in an unjoinable state, i.e., detached from the underlying OS thread. A `std::thread` is unjoinable in the following situations:

- Default-constructed `std::thread`;
- A `std::thread` that has been moved from;
- A `std::thread` on which `join()` has already been called;
- A `std::thread` on which `detach()` has already been called.

#### `std::future`

`std::future` is movable but not copyable, and `get()` can be called only once; calling `get()` again is an error. Calling `std::future::share()` creates a `std::shared_future`, which is copyable and allows multiple calls to `get()`, enabling safe sharing among threads; concurrent access through separate `std::shared_future` objects is thread-safe.

`std::promise` can also create a `std::future`; they communicate via a heap-allocated shared state that also contains a reference count.

The destructor of `std::future` usually just destroys its data members, but when the `std::future` was created by `std::async()` with the `async` policy (which creates a new thread) and is the last `future` pointing to the shared state, its destructor will block until the task is finished (i.e., it effectively calls `join()` on the thread).

`std::future<void>` can be used for one-time communication between threads.

#### `volatile`

The `volatile` keyword prevents the compiler from optimizing away accesses to a specific memory location. For example, code like:

```cpp
auto y = x;
y = x;       // redundant loads
x = 10;
x = 20;      // dead stores
```

is likely to be optimized to:

```cpp
auto y = x;
x = 20;
```

Using `volatile` disables such optimizations.

## `emplace` Functions #42

The `emplace` family is not always faster than `push`/`insert`. Performance is usually better only when all of the following conditions hold:

- The value is constructed directly in the container, not inserted via assignment (`operator=`) (e.g., inserting at the front of a `vector`);
- The argument types differ from the container’s element type (so `push`/`insert` would need to construct a temporary object first);
- The container is unlikely to reject inserts due to duplicate values.

When inserting resource-managing objects (e.g. `std::shared_ptr`) into containers, you must create the resource-managing object first. With `emplace`, there may be a gap between acquiring a resource and creating its managing object; if an exception occurs in this interval (e.g. `vector` reallocation fails due to insufficient memory), the resource may leak. The following code has potential leakage risk:

```cpp
std::list<std::shared_ptr<Widget>> ptrs;
ptrs.emplace_back(new Widget, killWidget);
```

With `emplace`, arguments are perfectly forwarded to the constructor, which means direct initialization is used and `explicit` constructors are allowed. This does not happen with `push`/`insert`, because the compiler does not allow `explicit` constructors to be used implicitly when creating temporaries.