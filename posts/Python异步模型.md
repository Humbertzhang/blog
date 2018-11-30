# Python异步模型



## async/await

现在的python语法中，使用`async def`来定义一个协程（coroutines）函数。而一个协程函数在其他协程函数中被调用的时候，需要使用`await`关键字来调用。但是await不仅仅只能await `async函数`.

我们知道，await 其实就是python3.5之后添加的用来替代yield from的关键字。而yield from的特性就是可以将程序的控制权进行传递，从而在主程序和最终的子生成器之间搭起一个程序控制权的通道，此处`await`关键字也是做的一样的事，但是它只可以在`async def`的函数中存在。可以被`await`的对象我们称为一个`awaitable`对象。

在asyncio包中，awaitable的对象有三类：**coroutines**, **Tasks**, **Futures**.



## Coroutine, Task 与 Future

#### `Coroutines`:

具体来说, coroutine可以代表这两类东西：

- a *coroutine function*: an [`async def`](https://docs.python.org/3/reference/compound_stmts.html#async-def) function;
- a *coroutine object*: an object returned by calling a *coroutine function*.

```python
import asyncio

async def nested():
    return 42

async def main():
    # Nothing happens if we just call "nested()".
    # A coroutine object is created but not awaited,
    # so it *won't run at all*.
    nested()

    a = nested()
    
    # Let's do it differently now and await it:
    print("directly")
    print(await nested())   # will print "42".
    print("a object")
    print(await a)			# will print "42".

asyncio.run(main())

```

如上，由`async def nested()`定义的函数即为coroutine function, 而在main() 中的`nested()`即为coroutine object.



#### `Tasks`

*Tasks* are used to schedule coroutines ***concurrently***.

即Task是在想要**并发**执行一些任务时来使用的。

常见的创建Task的方式：

```python
# In Python 3.7+
task = asyncio.create_task(coro())
# This works in all Python versions but is less readable
task = asyncio.ensure_future(coro())
```

create_task 源代码:

```python
def create_task(coro, *, name=None):
    """Schedule the execution of a coroutine object in a spawn task.
    Return a Task object.
    """
    loop = events.get_running_loop()
    task = loop.create_task(coro)
    _set_task_name(task, name)
    return task
```



而如果一个用`async def`定义的协程没有被注册到事件循环中，其实其运行过程就是类似于同步代码。 这是因为他们的运行其实并没有绑定在一个事件循环中，因此实际的执行是在

而在调用了`asyncio.create_tasks`之后，便会将其注册到loop事件循环中。等待第一次开始await时，便会激活事件循环，从而达到了并发执行的效果。需要注意的是，一旦你将一个东西使用create_task来注册为了一个Task，并且开始了event loop之后，被你注册的函数会立即开始执行，执行完所有的**同步**的代码，直到执行到了`asyncio.sleep()`或者网络IO处。

通过试验以下代码可以更加直观地了解具体的执行过程：

```python
import asyncio
import time
import os

async def say_after1(delay, what):
    print(f"delay:{delay}, what: {what}")
    await asyncio.sleep(delay)
    print(what)

async def say_after2(delay, what):
    print(f"delay:{delay}, what: {what}")
    for i in range(10):
        print(what)
    await asyncio.sleep(delay)

async def main():
    print(f"started at {time.strftime('%X')}")

    await say_after1(1, 'hello')
    await say_after1(2, 'world')

    print(f"finished at {time.strftime('%X')}")

# asyncio.create_task() can let u run coroutines **concurrently** as asyncio Tasks.
async def main2():
    task1 = asyncio.create_task(
            say_after2(1, "task1print"))
    task2 = asyncio.create_task(
            say_after2(1, "task2print")
            )
    task3 = asyncio.create_task(
            say_after2(1, "task3print")
            )
    time.sleep(1) 		# 你会发现在第一个await开始执行之前所有的task都还没开始执行
    print(f"task1 started at {time.strftime('%X')}")
    await task1			# 执行到了第一个await之后，所有之前create的task开始执行
    print(f"task1 finished at {time.strftime('%X')}")

    print(f"task2 started at {time.strftime('%X')}")
    await task2
    print(f"task2 finished at {time.strftime('%X')}")

    print(f"task3 started at {time.strftime('%X')}")
    await task3
    print(f"task3 finished at {time.strftime('%X')}")

if __name__ == '__main__':
    method = int(input())
    if method == 1:
        print("main1:")
        asyncio.run(main())
    else:
        print("main2:")
        asyncio.run(main2())
```



#### `Futures` 

我们看一下future对象的官方定义：

![](https://ww1.sinaimg.cn/large/007iUjdily1fxqdi3k9p6j319y0i4tcv)



也就是说，future是一次异步操作结果，更具体的说，future对象直接负责在event loop上去注册一个事件和管理其相应的callback和函数执行的结果。

需要注意的是，future对象其实不是由应用程序员进行编写的，而是由编写包的程序员来写的。在此处我们使用的包是asyncio。

我们可以知道，虽然python里面没有显式的callback，但python里面的协程就是基于event loop与callback来实现的。由event loop来管理事件循环。

另外一个需要探讨的是task和future之间的关系。

task是一个future like的对象，他实际上是future的子类。在event loop中，一些操作都是落实在了future中，如获取结果等等。



## Event Loop

较为底层的来看，python协程就是通过函数在yield处停止运行，而由event loop在运行时通过在合适的时刻激活停止的协程来进行调度。

Python异步模型如下图所示：

![img](https://lotabout.me/2017/understand-python-asyncio/asyncio-flow.png)



可以通过以下代码来模拟出一个event loop，特别需要主要的是其中的`run_until_complete`函数。

其会在开始时将所有task注册到队列中，在运行过程中通过检查时间是否到达了执行的时机来决定此次循环是执行一个任务还是空转。

```python
import datetime
import heapq
import types
import time

class Task:
    """Represent how long a coroutine should before starting again.
    Comparison operators are implemented for use by heapq. Two-item
    tuples unfortunately don't work because when the datetime.datetime
    instances are equal, comparison falls to the coroutine and they don't
    implement comparison methods, triggering an exception.
    Think of this as being like asyncio.Task/curio.Task.
    """
    def __init__(self, wait_until, coro):
        self.coro = coro
        self.waiting_until = wait_until

    def __eq__(self, other):
        return self.waiting_until == other.waiting_until

    def __lt__(self, other):
        return self.waiting_until < other.waiting_until

class SleepingLoop:
    """An event loop focused on delaying execution of coroutines.
    Think of this as being like asyncio.BaseEventLoop/curio.Kernel.
    """
    def __init__(self, *coros):
        self._new = coros
        self._waiting = []

    def run_until_complete(self):
        # Start all the coroutines.
        for coro in self._new:
            wait_for = coro.send(None)
            heapq.heappush(self._waiting, Task(wait_for, coro))
        # Keep running until there is no more work to do.
        while self._waiting:
            now = datetime.datetime.now()
            # Get the coroutine with the soonest resumption time.
            task = heapq.heappop(self._waiting)
            if now < task.waiting_until:
                # We're ahead of schedule; wait until it's time to resume.
                delta = task.waiting_until - now
                time.sleep(delta.total_seconds())
                now = datetime.datetime.now()
            try:
                # It's time to resume the coroutine.
                wait_until = task.coro.send(now)
                heapq.heappush(self._waiting, Task(wait_until, task.coro))
            except StopIteration:
                # The coroutine is done.
                pass

@types.coroutine
def sleep(seconds):
    """Pause a coroutine for the specified number of seconds.

    Think of this as being like asyncio.sleep()/curio.sleep().
    """
    now = datetime.datetime.now()
    wait_until = now + datetime.timedelta(seconds=seconds)
    # Make all coroutines on the call stack pause; the need to use `yield`
    # necessitates this be generator-based and not an async-based coroutine.
    actual = yield wait_until
    # Resume the execution stack, sending back how long we actually waited.
    return actual - now

async def countdown(label, length, *, delay=0):
    """Countdown a launch for `length` seconds, waiting `delay` seconds.
    This is what a user would typically write.
    """
    print(label, 'waiting', delay, 'seconds before starting countdown')
    delta = await sleep(delay)
    print(label, 'starting after waiting', delta)
    while length:
        print(label, 'T-minus', length)
        waited = await sleep(1)
        length -= 1
    print(label, 'lift-off!')

def main():
    """Start the event loop, counting down 3 separate launches.
    This is what a user would typically write.
    """
    loop = SleepingLoop(countdown('A', 5), countdown('B', 3, delay=2),
                        countdown('C', 4, delay=1))
    start = datetime.datetime.now()
    loop.run_until_complete()
    print('Total elapsed time is', datetime.datetime.now() - start)

if __name__ == '__main__':
    main()
```





Ref:

https://docs.python.org/3/library/asyncio.html

https://lotabout.me/2017/understand-python-asyncio/

https://juejin.im/entry/56ea295ed342d300546e1e22
