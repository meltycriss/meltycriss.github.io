---
title: 学习笔记《A Tour of C++》下
date: 2024-08-18 18:22:16
tags:
	- A Tour of C++
categories:
	- 学习笔记
description:
	- 后面几章的笔记
---

# 前言

书接上回（{% post_link note-a-tour-of-cpp-1 学习笔记《A Tour of C++》上 %}），这一部分由于都是一些STL库的使用，感觉通读意义不大，所以采取了一种哪里感兴趣就读哪里的方式。这也就导致了这篇文章不会像上一篇那么规整，属于会长期更新的类型。

下面开始正文部分。老规矩，正文部分按照书的顺序展开，通过[✓]标记有较大实战价值的内容。

---

# 第十五章：并发编程

- 多线程基础：创建线程/入参/返回结果
    - 对于变长参数，需要使用`ref(object)`来指定是pass by reference
- 线程间通信：核心是通过锁对象`mutex`来实现合作竞争，`mutex`的底层实现跟OS相关
    - 无条件锁
        - 几个操作`mutex`的manipulator（RAII，不希望显式调用`m.lock()`和`m.unlock()`）
            - `lock_guard`：最朴素简单的RAII，开销最低
            - `scoped_lock`：支持一次性获取多个锁，避免由获取锁顺序引起的死锁
            - `shared_lock`：读写锁
            - `unique_lock`：支持底层上锁解锁操作
        - 书里面提到了“非必要不通信”，不要低估上锁解锁的开销，也不要高估copy数据的开销
    - 有条件锁
        - motivation：实现`if (condition) lck.acquire()`，之所以不能直接这么写，是因为判断lck.acquire的时候，condition可能已经不满足了
        - 一种暴力的做法就是`do {lck.release(); sleep; lck.acquire()} while (!condition)`，但这样子会一直吃cpu
        - `condition_variable`的实现跟这个其实是类似的，只不过没有通过`while`去重试，而是依赖事件触发（OS提供的线程原语）
```c++
void consumer()
{
    unique_lock lck {mmutex}; // acquire mmutex
    // release lck and wait;
    // re-acquire lck upon wakeup
    // don’t wake up unless mqueue is non-empty
    mcond.wait(lck,[] { return !mqueue.empty(); });

    // do sth
}

void producer()
{
    Message m;
    // ... fill the message ...
    scoped_lock lck {mmutex}; // protect operations
    mqueue.push(m);
    mcond.notify_one(); // notify
} // release lock (at end of scope)
```
- 更便捷的多线程编程方式
    - motivation：代码里面不要显式地出现`mutex`和`thread`这种东西
    - `future`和`promise`：return值的getter和setter，隐藏了用于读写同步的`mutex`
        - `promise`本身包含一个`get_future()`方法以获取setter对应的getter
<div style="width:400px; margin-left:auto; margin-right:auto;" >
  {% asset_img future_promise.png future和promise %}
</div>
```c++
void f(promise<X>& px) // a task: place the result in px
{
    // ...
    try {
        X res;
        // ... compute a value for res ...
        px.set_value(res);
    }
    catch (...) { // oops: couldn’t compute res
        px.set_exception(current_exception()); // pass the exception to the future’s thread
    }
}

void g(future<X>& fx) // a task: get the result from fx
{
    // ...
    try {
        X v = fx.get(); // if necessary, wait for the value to get computed
    // ... use v ...
    }
    catch (...) { // oops: someone couldn’t compute v
        // ... handle error ...
    }
}
```
    - `packaged_task`：将`future`和`promise`，以及实际的function联系起来，隐藏了跟`promise`相关的语句（比如`promise.set_value(res)`），仍然需要`thread`来启动
        - `packaged_task`要`move`：担心资源泄漏
```c++
double comp2(vector<double>& v)
{
    using Task_type = double(double*,double*,double); // type of task
    packaged_task<Task_type> pt0 {accum}; // package the task (i.e., accum)
    packaged_task<Task_type> pt1 {accum};
    future<double> f0 {pt0.get_future()}; // get hold of pt0’s future
    future<double> f1 {pt1.get_future()}; // get hold of pt1’s future
    double* first = &v[0];
    thread t1 {move(pt0),first,first+v.size()/2,0}; // start a thread for pt0
    thread t2 {move(pt1),first+v.size()/2,first+v.size(),0}; // start a thread for pt1
    // ...
    return f0.get()+f1.get(); // get the results
}
```
    - `async`：进一步隐藏了`thread`
        - 书里面提到了不要在`async`执行的task之间做数据通信，我看下来感觉原因是`async`创建的线程数是不确定的，不一定能做到所有task并发
```c++
double comp4(vector<double>& v)
// spawn many tasks if v is large enough
{
    if (v.size()<10000) // is it worth using concurrency?
        return accum(v.begin(),v.end(),0.0);
    auto v0 = &v[0];
    auto sz = v.size();
    auto f0 = async(accum,v0,v0+sz/4,0.0); // first quarter
    auto f1 = async(accum,v0+sz/4,v0+sz/2,0.0); // second quarter
    auto f2 = async(accum,v0+sz/2,v0+sz*3/4,0.0); // third quarter
    auto f3 = async(accum,v0+sz*3/4,v0+sz,0.0); // fourth quarter
    return f0.get()+f1.get()+f2.get()+f3.get(); // collect and combine the results
}
```