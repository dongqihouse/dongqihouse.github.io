---
layout: post
title:  "python中的coroutine"
date:   2024-01-02 11:47:35 +0800
categories: technology
---

最靠谱的还是官方文档
> https://docs.python.org/zh-cn/3.12/library/asyncio-task.html#scheduling-from-other-threads

## 可以记录的点

1. coroutine的基本使用方法
2. coroutine如何和callback结合使用
3. event loop的概念

## 为什么需要Coroutine

### 回调地狱和线程阻塞

之前我们没有协程的时候，面对异步返回的数据都是使用callback处理。如果嵌套异步请求调用，势必会造成回调地狱。所以有了协程这种结构化编程的方式。其实在python中大多数代码都是同步的，callback的方式很少。所以说，python中的协程更多的是解决现成阻塞的问题。

但你说是不是非协程不可，那也大可不必。大不了回调地狱和多线程罢了。

## 基础概念

### 最基本的使用 

使用async包装异步函数，使用await等待异步函数的结果回调。

```python
async def await_fucntion():
    result = await async_function()
    print(result)

run(await_fucntion())
```

### await消费的对象

await 只能修饰 Awaitable的对象，重点就是Awaitable是什么？Awaitable顾名思义就是可等待的对象，只要继承Awaitable的父类都能被await消费。

```python
class Awaitable(metaclass=ABCMeta):
    __slots__ = ()

    @abstractmethod
    def __await__(self):
        yield

    @classmethod
    def __subclasshook__(cls, C):
        if cls is Awaitable:
            return _check_methods(C, "__await__")
        return NotImplemented
```

### 哪些对象是Awaitable

Coroutine, Task, Future 这些都是Awaitable对象。

熟悉源码的同学可能会觉得有问题，因为

```python
# 没有 Coroutine对象啊
def coroutine(func):
    """Convert regular generator function to a coroutine."""
# Task 和 Future 也不继承 Awaitable 啊

class Task(futures._PyFuture):  # Inherit Python Task implementation
                                # from a Python Future implementation.

class Future:
    """This class is *almost* compatible with concurrent.futures.Future.

```

但是我们知道python著名的鸭子类型：
*决定一个对象是否有正确的接口，关注点在于它的方法或属性，而不是它的类型*

```python
def __await__(self):
        if not self.done():
            self._asyncio_future_blocking = True
            yield self  # This tells Task to wait for completion.
        if not self.done():
            raise RuntimeError("await wasn't used with future")
        return self.result()  # May raise too.
```

这也是data class的编程概念。

既然我们知道现在有3种对象是awaitable的了，那后面重点就是如何使用这3种对象了。

### Coroutine, Task, Future 的使用场景

#### Coroutine

这个基本是我们平时最长使用的，coroutine是异步编程的最小使用单元。

```python
async def async_function():
    return 1

print(type(async_function()))
```

> /var/folders/jp/7nqpgjbj2mz03_tdhdy9cq4w0000gq/T/ipykernel_62298/3049755617.py:3: RuntimeWarning: coroutine 'async_function' was never awaited
  print(type(async_function()))

直接使用async修饰的函数，就是协程函数。
调用这个函数，返回协程对象，而不是返回函数的结果。
只要 await调用 才会返回数据的结果。

#### Task

Task是基于Coroutine的封装，首先coroutine最为异步编程的最小使用单元，肯定是不能解决所有问题的，那么coroutine的缺陷有哪些呢？

首先一个问题是，系统不能确定一个协程的边界，下面的例子，

单纯的await就阻塞中当前的执行了啊。

```python
import asyncio
import time

async def say_after(delay, what):
    await asyncio.sleep(delay)
    print(what)

async def main():
    print(f"started at {time.strftime('%X')}")

    await say_after(1, 'hello zhangsan')
    await say_after(2, 'hello lisi')

    print(f"finished at {time.strftime('%X')}")

asyncio.run(main())

# started at 17:13:52
# hello zhangsan
# hello lisi
# finished at 17:13:55
```

我们发现2个方法的执行是并不是并发的，
但是如果我想让这个2个函数并发执行怎么办？

await会阻塞中eventloop,能不能不使用await触发异步函数的执行？

答案就是`create_task`

上面的代码改成这样

```python
import asyncio
import time

async def say_after(delay, what):
    await asyncio.sleep(delay)
    print(what)

async def main():
    print(f"started at {time.strftime('%X')}")

    task1 = asyncio.create_task(say_after(1, 'hello zhangsan'))
    task2 = asyncio.create_task(say_after(2, 'hello lisi'))

    # 这里为了等待2个task执行的结束，并不是上面create_task的时候，
    # 已经在并发执行了
    await asyncio.gather(task1, task2)

    print(f"finished at {time.strftime('%X')}")

await main()

# started at 15:06:02
# hello zhangsan
# hello lisi
# finished at 15:06:04
```

我们发现这2个函数耗时仅有2s,证明是并发执行的。

```python
# 下面2种写法，都可以当成2个Task完成之后触发后续流程。
# ✅
await task1
await task2

# ✅
await asyncio.gather(task1, task2)
```


看起来coroutine和task已经满足了很多需求了，但是有个很重要的问题是，大部分框架和代码并不支持协程，那么如果把这些callback调用或者多线程同步调用改成协程使用了，这里就需要来我们Future了。

#### Future

Future其实比较底层的异步编程的api，但是面对之前多线程的api。我们就必须要使用了Future对象了。

```python
import asyncio
import threading
import requests
from concurrent import futures

def fetch_data():
    print(f"当前线程{threading.current_thread()}")

    url = "https://www.baidu.com"
    res = requests.get(url)
    time.sleep(3)
    print(f"百度的网页{res}")

def fetch_data_callback():
    print("done")


async def afetch_data():
    pool = futures.ThreadPoolExecutor(max_workers=1)
    pool.submit(fetch_data)

    loop = asyncio.get_event_loop()
    print("已经开始执行子线程任务了")
    future = loop.run_in_executor(pool, fetch_data_callback)
    await future

async def aother_task():
    for i in range(10):
        await asyncio.sleep(1)
        print(f"aother_task 开始工作{i}")

async def main():
    task2 = asyncio.create_task(aother_task())
    task1 = asyncio.create_task(afetch_data())

    await asyncio.gather(task2, task1)
    

await main()
```

```plain
当前线程<Thread(ThreadPoolExecutor-23_0, started 6305378304)>
已经开始执行子线程任务了
aother_task 开始工作0
aother_task 开始工作1
aother_task 开始工作2
百度的网页<Response [200]>
done
aother_task 开始工作3
aother_task 开始工作4
aother_task 开始工作5
aother_task 开始工作6
aother_task 开始工作7
aother_task 开始工作8
aother_task 开始工作9
```

## 补充点

### asyncio.run(main()) 是阻塞线程的

### create_task已经开始任务了，await只是等待获取结果罢了

```python
import asyncio

async def coroutine1():
    print("Coroutine 1 started")
    await asyncio.sleep(1)
    print("Coroutine 1 completed")

async def coroutine2():
    print("Coroutine 2 started")
    await asyncio.sleep(2)
    print("Coroutine 2 completed")

async def main():
    print("Main started")
    # 启动两个协程
    task1 = asyncio.create_task(coroutine1())
    task2 = asyncio.create_task(coroutine2())
    print("开始等待结果")

    # await task1
    # print("task1 结束")
    await task2
    print("task2 结束")

    print("Main completed")

# 在事件循环中运行主协程
asyncio.run(main())
```

```plain
Main started
开始等待结果
Coroutine 1 started
Coroutine 2 started
Coroutine 1 completed
Coroutine 2 completed
task2 结束
Main completed
```

没有await task1, 并不影响task1的执行。