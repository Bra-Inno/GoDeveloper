



自己对异步的理解一直不是很深入，今天深入对比各种异步代码，发现异步的真正牛逼之处

首先，不是用了async和await这两个异步典型的操作关键字就能实现异步的效果，我觉得对于新手而言，非常容易犯的错误是以为用了async和await就实现了异步，可以加快执行速度



举一个例子

# 1 无法实现异步的情况

## 情况一：不是加了async和await 就是异步

我们假设有两个耗时任务，任务1——task1，执行需要消耗1s，任务2——task2，执行需要消耗2s，并且假设任务1和任务2之间和并行处理，没有先后关系（即不是任务2必须得等任务1完成才能执行）

```python
import asyncio
import time

async def task1():
    start_time = time.time()
    print("task1 start")
    time.sleep(1)
    print("task1 end")
    end_time = time.time()
    print(f"task1 executed in {end_time - start_time} seconds")

async def task2():
    start_time = time.time()
    print("task2 start")
    time.sleep(2)
    print("task2 end")
    end_time = time.time()
    print(f"task2 executed in {end_time - start_time} seconds")

```

然后在一个函数里执行这两个任务

```python
# 统计总的执行时间
async def main():
    start_time = time.time()

    await task1()
    await task2()

    end_time = time.time()
    print(f"Total executed in {end_time - start_time} seconds")

asyncio.run(main())
```

我们可以看到已经实现了两个

终端输出是这样的

```sh
task1 start
task1 end
task1 executed in 1.014275312423706 seconds
task2 start
task2 end
task2 executed in 2.0086023807525635 seconds
Total executed in 3.02388072013855 seconds
```

总共需要大约3s，可见这个并没有实现我们想要的效果，即task1和task2并行执行，总执行时间大大减少



这其实和同步没有任何区别，给大家看看同步的代码

```python
import asyncio
import time

def task1():
    start_time = time.time()
    print("task1 start")
    time.sleep(1)
    print("task1 end")
    end_time = time.time()
    print(f"task1 executed in {end_time - start_time} seconds")

def task2():
    start_time = time.time()
    print("task2 start")
    time.sleep(2)
    print("task2 end")
    end_time = time.time()
    print(f"task2 executed in {end_time - start_time} seconds")

# 统计总的执行时间
def main():
    start_time = time.time()

    task1()
    task2()

    end_time = time.time()
    print(f"Total executed in {end_time - start_time} seconds")

main()
```

终端输出

```python
task1 start
task1 end
task1 executed in 1.0130815505981445 seconds
task2 start
task2 end
task2 executed in 2.0044596195220947 seconds
Total executed in 3.0175411701202393 seconds
```

没有任何区别，甚至你会发现用了异步总执行时长反而可能更慢，笑死

## 情况二： 不是用了gather收集后就能实现异步（异步函数内部不能有阻塞操作）

async确实是申明函数异步，但是异步该怎么执行，还是得我们来亲自设计

网上查阅以后，有这样一种写法，说是需要一个gather函数，将任务收集起来，告诉系统这些任务需要并行处理

像下面这样

```python
asyncio.gather(task1(), task2())
```

OK，完整代码如下

```python
async def task1():
    start_time = time.time()
    print("task1 start")
    time.sleep(1)
    print("task1 end")
    end_time = time.time()
    print(f"task1 executed in {end_time - start_time} seconds")

async def task2():
    start_time = time.time()
    print("task2 start")
    time.sleep(2)
    print("task2 end")
    end_time = time.time()
    print(f"task2 executed in {end_time - start_time} seconds")

# 统计总的执行时间
async def main():
    start_time = time.time()

    asyncio.gather(task1(), task2())
	
    end_time = time.time()
    print(f"Total executed in {end_time - start_time} seconds")

asyncio.run(main())
```

发现输出是这样的

```python
Total executed in 0.0 seconds
task1 start
task1 end
task1 executed in 1.0034098625183105 seconds
task2 start
task2 end
task2 executed in 2.012523651123047 seconds
```

我靠！！！

最后应该输出的Total executed in 0.0 seconds 竟然最先输出，所以main函数执行顺序完全是混乱的

而如果轻轻给asyncio.gather(task1(), task2())前面加一个await

像下面这样

```python
import asyncio
import time


async def task1():
    start_time = time.time()
    print("task1 start")
    time.sleep(1)
    print("task1 end")
    end_time = time.time()
    print(f"task1 executed in {end_time - start_time} seconds")

async def task2():
    start_time = time.time()
    print("task2 start")
    time.sleep(2)
    print("task2 end")
    end_time = time.time()
    print(f"task2 executed in {end_time - start_time} seconds")

# 统计总的执行时间
async def main():
    start_time = time.time()

    await asyncio.gather(task1(), task2())

    end_time = time.time()
    print(f"Total executed in {end_time - start_time} seconds")

asyncio.run(main())
```

总的任务执行又变成同步的了，Task1执行完再执行Task2

```sh
task1 start
task1 end
task1 executed in 1.0091900825500488 seconds
task2 start
task2 end
task2 executed in 2.0130677223205566 seconds
Total executed in 3.0222578048706055 seconds
```

其实这是因为task1和task2内部有一个非异步操作，阻塞等待操作time.sleep,也可以叫同步操作time.sleep(1)，time.sleep(2)

正确应该是这样的

```python
import asyncio
import time


async def task1():
    start_time = time.time()
    print("task1 start")
    await asyncio.sleep(1)  # 非阻塞的等待
    print("task1 end")
    end_time = time.time()
    print(f"task1 executed in {end_time - start_time} seconds")

async def task2():
    start_time = time.time()
    print("task2 start")
    await asyncio.sleep(2)  # 非阻塞的等待
    print("task2 end")
    end_time = time.time()
    print(f"task2 executed in {end_time - start_time} seconds")

async def main():
    start_time = time.time()
    # 并发执行
    await asyncio.gather(task1(), task2())
    end_time = time.time()
    print(f"Total executed in {end_time - start_time} seconds")

asyncio.run(main())
```

最终输出如下：

```sh
task1 start
task2 start
task1 end
task1 executed in 1.014965534210205 seconds
task2 end
task2 executed in 2.01570987701416 seconds
Total executed in 2.01570987701416 seconds
```



分析下来，我觉得对于异步非常重要的几点

（1）不是加了async和await就是异步，对应情况一

（2）异步函数内部，如果有非异步操作，如这里的time.sleep(1)会导致整体异步失效，对应情况二



# 2. 如何设计异步

那大家可能就好奇，为啥要await，很简单

在父任务中（即这里的主函数，task1和task2是可以并行执行的）

那这时候有一个task3，必须得等task1和task2执行完毕后，才能执行，那这时候必须要用await了

await task1和task2

然后再执行task3

如下所示

```python
import asyncio
import time


async def task1():
    start_time = time.time()
    print("task1 start")
    await asyncio.sleep(1)  # 非阻塞的等待
    print("task1 end")
    end_time = time.time()
    print(f"task1 executed in {end_time - start_time} seconds")
    return 1

async def task2():
    start_time = time.time()
    print("task2 start")
    await asyncio.sleep(2)  # 非阻塞的等待
    print("task2 end")
    end_time = time.time()
    print(f"task2 executed in {end_time - start_time} seconds")
    return 2
#任务3必须得等待任务1和任务2执行完毕后才能执行,将任务1和2返回的结果加起来
async def task3(task1_number, task2_number):
    return task1_number + task2_number

async def main():
    start_time = time.time()
    # 并发执行
    return_list=await asyncio.gather(task1(), task2())
    end_time = time.time()
    result=await task3(return_list[0], return_list[1])
    print(f"Total executed in {end_time - start_time} seconds")
    print(result)


asyncio.run(main())
```

最终输出

```sh
task1 start
task2 start
task1 end
task1 executed in 1.0085046291351318 seconds
task2 end
task2 executed in 2.0127499103546143 seconds
Total executed in 2.0127499103546143 seconds
3
```

