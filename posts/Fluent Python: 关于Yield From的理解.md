
## Fluent Python: 关于Yield From的理解

```Python
from collections import namedtuple

Result = namedtuple('Result', 'count average')

def averager():
    total = 0.0
    count = 0
    average = None
    print("averager started!")
    while True:
        term = yield
        print("averager:", term)
        if term is None:
            break
        total += term 
        count += 1
        average = total / count
    return Result(count, average)

def grouper(results, key):
    while True:
        print("group loop start.")
        results[key] = yield from averager()
        print("grouper:", results[key])

def main(data):
    results = {}
    for key, values in data.items():
        print("main big loop:", " key:", key, " values:", values)
        group = grouper(results, key)

        print("invoke next(group)")
        next(group)
        print("invoke ended")

        for value in values:
            print("main small loop:", "sended:", value)
            group.send(value)
        print("main send None")
        group.send(None)    
    report(results)

def report(results):
    for key, result in sorted(results.items()):
        group, uint = key.split(";")
        print('reporter: {:2} {:5} averaging {:.2f}{}'.format(
                result.count, group, result.average, uint))

data = {
    'boys;kg': [1,2,3],
}

if __name__ == '__main__':
    main(data)

```

output
```
group loop start.
averager started!
invoke ended
main small loop: sended: 1
averager: 1
main small loop: sended: 2
averager: 2
main small loop: sended: 3
averager: 3
main send None
averager: None
grouper: Result(count=3, average=2.0)
group loop start.
averager started!
reporter:  3 boys  averaging 2.00kg

```

通过上面的代码和输出，可以比较清晰的了解到整个代码的运行流程。

一. 首先在main函数处，拿到data的输入，开始遍历data dict.  打印  `main big loop:  key: boys;kg  values: [1, 2, 3]`

二. 之后group = grouper() 一句生成一个所谓 `委派生成器`，将results `(空字典，会在grouper函数之中填充数据)`和 key`(此处为boys;kg)`传入委派生成器中.

三. 之后通过`next(group)`预激委派生成器. 注意，此时**不仅激活了grouper 生成器，还激活了averager生成器**，即所谓 `子生成器`. 我们可以认为，此时next() 传到了grouper的`yield from averager()` 之后并没有结束，而是接着向下传递到了averager()中. 
`[参考16.8 yield from 意义第2条，传给委派生成器的send()如果是None, 则调用子生成器的.__next__(), 不是None则调用子生成器的.send()] ` 
averager() 被激活之后，第一次运行到了`term = yield` 之后便暂停(`GEN_SUSPENDED`)，并将控制权交换给了grouper(),  但grouper()并不能直接向下运行，因为他调用的 averager() 并没有返回，仅仅是yield了，所以还是维持在`GEN_SUSPENDED`状态，并将程序控制权上交，给了main处，main 得以继续运行，打印 `invoke ended`.

四. 之后程序进入 `main small loop`处.开始send数据. 此时因为grouper()其实仍在等待averager()的返回，所以send的数据也是直接传给了averager(). averager() 继续运行到下一个yield处，交还控制权给grouper()，group()进而交还给了main函数. 如此运行直到 `for value in values` 结束.

五.  此时，main函数给委派生成器send了None`（group.send(None)）`, 委派生成器照常没有保留，直接send给了子生成器. 此时`term is None `条件成立 , 子生成器中的while循环停止。子生成器return 了 `Result(count, averager)` . 注意，如16.6 16.7中所讲，协程返回值是以StopIteration：`return value` 形式来做的，此时grouper处的yield from 拿到了StopInteration Exception之后终于也返回了，并因yield from 的特性，捕获到了这个异常，并从异常的value中获取到了值. 
![](https://ww1.sinaimg.cn/large/005YhI8igy1fx9x2acmqtj30h803dmy3)


六.  yield from 表达式的值即为averager()返回的Result() namedtuple，因此results[key]得到了赋值。继续执行，打印`grouper: Result(count=3, average=2.0)` . 并继续while循环，继续打印了第二次`group loop start. ` 在运行到yield from averager() 时，与第一次一样，将程序控制权给了averager()，averager()得到了程序控制权之后，打印了第二次`averager started!`， 但是在运行到了`term = yield`时，将程序控制权上交给group, group继续上交给main, main 的`group.send(None)` 终于执行完毕。此时之前刚刚创建的 group() 与 averager() 被GC垃圾 回收，而main函数开始了下一次的迭代，重新创建了新的group() , averager() 等，继续运行下去。

七. 直到 `for key, values in data.items() ` 运行完毕后，main调用report()函数，打印出结果. 

![](https://ww1.sinaimg.cn/large/005YhI8igy1fx9x2mfmipj30h80440tl)


此时再看下书上归纳的yield from 语义(后两个与异常和终止有关):

+ 子生成器产出的值都直接传给委派生成器的调用方（即客户端代码）。
+ 使用 send() 方法发给委派生成器的值都直接传给子生成器。如果发送的值是None，那么会调用子生成器的 __next__() 方法。如果发送的值不是 None，那么会调用子生成器的 send() 方法。如果调用的方法抛出 StopIteration 异常，那么委派生成器恢复运行。任何其他异常都会向上冒泡，传给委派生成器。
+ 生成器退出时，生成器（或子生成器）中的 return expr 表达式会触发StopIteration(expr) 异常抛出。
+ yield from 表达式的值是子生成器终止时传给 StopIteration 异常的第一个参数。
+ 传入委派生成器的异常，除了 GeneratorExit 之外都传给子生成器的 throw() 方法。如果调用 throw() 方法时抛出 StopIteration 异常，委派生成器恢复运行。 StopIteration 之外的异常会向上冒泡，传给委派生成器。
+ 如果把 GeneratorExit 异常传入委派生成器，或者在委派生成器上调用 close() 方法，那么在子生成器上调用 close() 方法，如果它有的话。如果调用 close() 方法导致异常抛出，那么异常会向上冒泡，传给委派生成器；否则，委派生成器抛出GeneratorExit 异常。

大家应该有了新的理解.
