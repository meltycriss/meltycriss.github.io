---
title: 学习笔记《A Tour of C++》上
date: 2024-08-11 21:18:04
tags:
	- A Tour of C++
categories:
	- 学习笔记
description:
	- 记录一下看C++之父写的《A Tour of C++》时，感到比较有意思的点
---

# 前言

这书是C++之父Bjarne Stroustrup写的，比较高屋建瓴地介绍了现代C++的一些特性，没有涉及太多的细节，主要还是思想理念上的一些东西，对于快速了解现代C++有比较大的帮助。

本文主要记录看此书1-7章时感到比较有意思的点，这7章的内容大致分为以下三个部分：
- 面向过程：第1-3章是一些比较基础的的内容；
- 面向对象：第4章介绍用户自定义的类型（类），第5章介绍一些核心算子（构造/析构/赋值）；
- 泛型编程：第6章介绍模板，第7章介绍如何为模板实例化添加约束（Concept）。

至于后面的第8-15章主要是以STL为例子去进一步介绍上述概念的，第16章则是介绍了这门语言的发展历史。

下面开始正文部分。正文部分按照书的顺序展开，通过[✓]标记有较大实战价值的内容。

---

# 面向过程

## 第一章：基础语法

- [✓] 推荐使用*{} initialization*，因为自带类型检查，可以保证类型（比如`int x{1.2}`会直接报错）
- 指定函数为*constexpr function*就可以用于初始化constexpr类型的变量
    - constexpr编译期确定，const运行期确定
- reference只能被初始化（init），不能被修改（assign），即不能修改指向谁，所以指针能做的还是比ref多
- 引入`nullptr`的原因在于兼顾类型匹配和方便使用
    - 用`int 0`指代空指针的问题：调用函数有可能出错，会混淆`func(class *ptr)`和`func(int x)`
    - 用`(void *) 0`指代空指针的问题：没法`class *ptr = (void *) 0`，因为C++禁止这种转换，得很麻烦地写`A *p = static_cast<A*>((void *)0)`
- [✓] `if`也能引入局部变量，比如`if (n = v.size(); n > 0)`
- [✓] 推荐使用`auto`

## 第二章：用户定义的类型

- `struct`默认`public`，`class`默认`private`
- `enum class`和`enum`的区别在于前者需要指定prefix，以及后者可以隐式地`enum`转`int`
- `union`: 每次被使用时是多种类型中的一种（比如说一个表的Entry，可以是`int / string / double`，但每一条Entry只能是其中一种），`union`是个底层类型，一般使用`variant`这个封装

## 第三章：模块化

- C++20引入了`Module`来解决原有头文件系统的以下问题
    - 重复编译
    - 顺序相关（比如`#include h1 #include h2`有可能会与`#include h2 #include h1`的结果不同）
- [✓] *structured binding*: `for (const auto [key,value] : map)`

---

# 面向对象

## 第四章：类

- [✓] *RAII (Resource Acquisition Is Initialization)*：利用离开scope会自动调destructor的特性来避免资源泄露
- 类的类型
    - *Concrete Type*（比如`Vector`）
    - *Abstract Type*（比如`Container`）：一般包含`=0`
- 一些符号
    - `virtual`：虚函数
    - `= 0`：纯虚函数
    - 显式写`override`：避免typo
- 继承的成本
    - 时间上：多一次查虚表
    - 空间上
        - 每个对象多一个指向虚表的指针
        - 每个类要有一个虚表
- `dynamic_cast`用于将基类转向子类

## 第五章：关键算子

- 关键算子
    - 增：构造(*constructor*)
        - from different type: `default` / `ordinary`
        - from same type:
            - non-temporary: `copy`
            - temporary: `move`
    - 删：析构(*destructor*)
    - 改：赋值(*assignment*)
        - from same type:
            - non-temporary: `copy`
            - temporary: `move`
- 一些符号
    - `= default`可以兼顾显式实现和复用默认实现，即可以在省略`{}`内容的情况下复用默认实现
    - `= delete`表明一定没有某个函数
    - `explicit`相当于不能将int当class使了，必须手动class(int)
- [✓] *default member initializer*: `int x = 0`
- 不会连续调两次构造函数：`A a = A(1)`只会调一次带参构造函数，不会先调带参构造函数再调拷贝构造函数
- `A a()`并没有调用A的默认构造函数，而是在声明一个不带参，返回值为A类型的函数，下面这三种才是调默认构造函数
    - `B b`
    - `B b{}`
    - `B b{} = B()`
- [✓] 关于`move`
    - motivation是**根据copy时src的类型，采用不同的copy方法，以优化copy的性能**。
    - 具体来说，**按照src后续是否还会被使用**，copy可以分成两种。针对不需要被使用的情况（比如说函数的值返回），有机会可以优化copy的性能（比如一种方法是偷，即不需要再申请内存，直接将dst的资源指针指向src的资源）。
    - 为了标识后续不会再被使用的src，引入了右指引用`&&`这种类型，其实也可以把右值引用这个拗口的名字直接理解成**将死类型**（这个称呼不是我首创的，但是忘了在哪看到过这个说法，没有引用敬请原谅），当然这只是方便理解，取名叫右指引用还是有他的道理的，只是初看容易懵
        - 右值：将死类型可以出现在等号右边，但等号右边的不一定是将死类型
        - 引用：因为我们要偷他的东西，必然是引用
    - 估计是因为针对该src类型（将死类型）的优化方法就是偷，而偷这个字眼不太正经，所以换了一个正经点的名字叫移动，也就是`move`
    - 于是`move`也就成了跟`copy`并列的一个概念，本质上还是copy，只不过手段不一样
    - 能够标记将死类型还不够，最后能起到优化效果还是依赖于针对将死类型如何操作，这也就是*move constructor*和*move assignment*要做的事了
    - misc
        - `std::move`干的事情只是一个简单的cast，把类型转成了将死类型
        - 那啥类型会是将死类型呢
            - `std::move`转换过的
            - 各种无名变量，比如函数返回值
            - 值得一提的是类型为右指引用的形参仍然是左指，要想以右指引用使用这个形参还得用`std::move`
                - 估计是为了保留调用常规函数的可能性
- [✓] *literal operator*：通过指定后缀将东西转成class
    - `"Surprise!"s` is a `std::string`
    - `"Surprise!"` is a `const char[10]`

---

# 泛型编程

## 第六章：模板

- [✓] 我自己关于模板的理解
    - 正常编程语法：面向运行期(program generation)，模板编程语法：面向编译期(code generation)
    - 运行期多态（继承）时为了不用写if xxx type call xxx function这种代码
    - 编译期多态（模板）是为了不用写只替换类型或者某些const的代码
- 类级别模板
    - `typename/class`: for all T
    - `Element`(e.g., `Sequence`)：for all T such that，跟第七章的*Concept*关系很大
    - 类里面搞个`using value_type = T`，方便外界通过`Class::value_type`拿到T，
    - *type deduction*可以直接`vector v{1,2,3}`，都不需要`<type>`了
- [✓] 函数级别模板
    - virtual function不可以是模板函数：估计主要考虑到虚表的复杂度
        - 虚表的复杂度正比于`type * virtual func num * subclass depth`
        - 类级别模板允许了`type`这个纬度的膨胀
        - 假如再允许`virtual function num`这个纬度的膨胀，就会是平方级别的复杂度了
    - 函数对象：带context的函数
    - lambda：`[context](parameter){body}`
        - context部分（也叫*capture list*）主要是描述lambda要用哪些local variable，`&`引用，`=`值
        - 一个很nice的应用场景是使用lamda将原来先构造再赋值的初始化合并成拷贝构造
- built-in级别模板
    - variable: `template <class T> constexpr T viscosity = 0.4`
    - alias: `template<typename C> using Value_type = typename C::value_type`
    - compile time if: `if constexpr(xxx)`，一种用途是根据不同类型选择不同实现
```c++
template<typename T>
void update(T& target)
{
    // ...
    if constexpr(is_pod<T>::value)
        simple_and_fast(target); // for "plain old data"
    else
        slow_and_safe(target);
    // ...
}
```

## 第七章：约束模板实例化时，类型要满足的条件

- *Concept*
    - motivation是单凭`template<typename T>`的约束太少了，`T`可以是任意类型，我们想限定当前模板只适用于满足某些条件的类型`T`
    - 关键词是`requires`，也支持重载，定义最基础的concept的关键词也是`requires`，比如以下，第一个`requires`是*requirements-clause*，表示需要满足什么条件；第二个`requires`是*requires−expression*。当然一般来说不太需要*requires−expression*，STL里面有现成的。
```c++
template<Forward_iterator Iter, int n>
    requires requires(Iter p, int i) { p[i]; p+i; } // Iter has subscripting and addition
void advance(Iter p, int n) // move p n elements forward
{
    p+=n; // a random-access iterator has +=
}
```
- 泛型编程的思路：先搞几个concrete，通过一直问哪些东西可以放宽限制的方法进行抽象，别一上来就想着复用，好的泛型编程会导致模板实例化出来的代码跟你要手写的代码是一样的
- 变长参数模板：默认的方法是通过拆解为`<first, left>`递归去做的，C++17支持fold expression一把梭
- 模板类里面支持再定义模板函数，配合`std::forward`（保留左指右指）很好用
- 使用concept的好处在于
    - 提早报错期（不需要等实例化后才知道错）
    - 更好的报错（实例化之后的报错很奇怪）
    - 避免*duck typing*（不基于operand，而是基于含义）
- *module*之后，模板也可以像普通代码一样分离声名和定义了
    - 模板的实现要放在.h主要因为缺乏沟通需要实例化哪些模板的渠道，导致需要将实例化放到普通的.cpp的编译过程中
    - module估计增加了这个通信渠道，导致模板.cpp可以知道要实例化哪些模板