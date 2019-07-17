## 协程

协程从字面上来理解，就是可以实现协同的例程，也就是说，它可以用来实现多个例程之间的协同执行，比如使得例程A中的某个操作在例程B中的某个操作完成之后被执行。

协程大致经过了三个版本的迭代，yield版本、@asyncio.coroutine/yield from版本、以及async/await版本，其中前两个版本又叫做生成器版本，后一个版本叫做原生协程版本。

### yield版本

yield版本的协程是对生成器语法的简单改进，功能有限，只能用来实现两个函数之间的同步式的协同，而不能用来实现异步的功能。

具体来说，yield协程在生成器的基础上做了如下几个方面的改进：

* 添加了send()方法，可以在yield的位置传入值
* 添加了throw()方法，可以在yield的位置抛入异常；异常如果没有被处理，会传递回给调用方
* 添加了close()方法，使得协程可以被提前终止，下次再调用send()方法会抛出StopIteration异常
  * colse()方法事实上是在yield的位置抛入了一个GeneratorExit异常
  * 收到GeneratorExit异常的生成器一定不能产出新值，否则会触发RuntimeError
  * 该异常如果不处理，或者处理了之后重新抛出了StopIeration异常，异常不会向上传递
  * 抛出的其它异常会向上传递
* 允许返回值，但是是通过在StopIteration中设置value属性的方式，也就是说要通过捕获异常、然后从异常中获取value属性的方式来得到返回值

其中最重要的改变是添加了send()方法，使得从原来只能提供迭代输出的生成器，变成允许接受输入、在协程和协程的调用者之间实现了双向的输入输出，以此实现两者之间的有意义的协同。

### @asyncio.coroutine/yield from版本

*（关于@asyncio.coroutine的实际作用不是很清楚，这里略过）*

yield from有两个作用，一个是委托生成，一个是支持异步IO。

#### 委托生成

作为委托生成的功能，yield from后面可以跟上一个迭代器，或者是另一个是使用了yield或者yield from语法的生成器。

调用者调用委托生成器的send()方法，委托生成器将调用者的调用通过yield from委托给子生成器处理。因此yield from的作用，就好比是在调用者和子生成器之间架起了一条透明的沟通的管道。委托生成需要有一个终点，这个终点要么是一个生成器，要么是一个只使用了yield语法的生成器。*（委托生成的实现原理：在委托生成器中通过yield获取调用者的输入，然后在子生成器上调用send()传入输入，获取子生成器的输出之后再通过yield向调用者返回输出。）*

yield from会自动预激后面的生成器（调用next(gen)方法），然后在这里挂起，直到被子生成器的StopIteration异常唤醒。

yield from的一个重要的工作是处理嵌套的生成器之间的异常传递：

* 调用者通过throw()抛入的异常会直接传递到子生成器中
* 子生成器如果没有处理异常，异常会在嵌套的生成器中逐层冒泡向上传递；如果没有任何生成器处理了这个异常，则会一直传递回给调用者
* yield from会自动处理StopIteration异常，包括获取并返回value属性，也就是说，使用yield from可以直接获取子生成器的返回值
* close()方法传入的GeneratorExit异常会直接传入子生成器
  * 如果子生成器正常关闭（不向上抛出异常），委托生成器中也抛出GeneratorExit异常，换句话说，两者一起关闭
  * 如果子生成器抛出了异常，该异常向上冒泡传递

#### 支持异步IO

协程在配合异步IO使用的时候才发挥出最大的威力。

yield from后面可以跟上一个支持协程的异步IO函数，发起异步调用，并接收该异步IO函数的返回值。yield from后面还可以跟上一个future，future是对一个函数的封装，该函数被承诺接下来一定会被执行。在协程中，这种承诺来自事件循环。

协程可以被包装成一个task，加入事件循环，task是future的子类。包装成task的协程事实上已经加入了事件循环的调度，将被异步地执行。下面是一个实例：

```python
@asyncio.coroutine
def a():
  print('a start')
  yield from asyncio.sleep(1) # 1
  print('a finish')

@asyncio.coroutine
def b():
  print('b start')
  yield from asyncio.sleep(1) # 2
  print('b finish')
 
@asyncio.coroutine
def main():
  task1 = asyncio.create_task(a()) # 3
  task2 = asyncio.create_task(b()) # 4
  yield from task1 # 5
  yield from task2 # 6
  # yield from a() # 7
  # yield from a() # 8
  
asyncio.run(main()) # 9
```

* 语句9会自动生成并且启动一个事件循环，将main()协程加入调度，并在协程完成之后自动关闭事件循环
* 进入main函数之后，先是在语句3、4中将协程打包成一个task；注意这个时候协程已经被加入了事件循环，即使没有后面5、6的yield from语句，协程还是会被执行（不过在本例中，如果注释掉5、6，因为事件循环在main()协程结束之后提前关闭，所以两个协程事实上等不到被执行）
* 语句5的yield from导致main函数交出执行权
* 执行权回到事件循环，事件循环从等待列表中取出等待的任务task1开始执行
* 进入函数a，执行print('a start')，执行到语句1的时候，发起一个休眠1秒钟的异步调用，yield from交出执行权
* 执行权回到事件循环，事件循环从等待列表中取出等待的任务task2开始执行
* 进入函数a，执行print('b start')，执行到语句2的时候，发起一个休眠1秒钟的异步调用，yield from交出执行权
* 执行权回到事件循环，此时没有就绪的任务
* 过了1秒钟之后，协程a()和b()发起的异步调用返回，task1和task2进入就绪状态，被事件循环依次取出执行
* task1打印print('a start')，然后返回，task1完成；与此同时在yield from task1上休眠的main()协程随着task1的完成进入就绪状态
* task2打印print('b start')，然后返回，task2完成
* 事件循环取出就绪的main()协程，在语句5上恢复执行*（在语句6上的表现是怎么样的，这时候task2已经完成，是先交出执行权然后再次进入，还是直接往下执行？暂时还不清楚）*
* main协程完成，语句9返回，事件循环关闭
* 所以最终的输出顺序是
  * a start
  * b start
  * a finish
  * b finish

真正发起异步调用的是语句3、4，而语句5、6处的yield from实际上提供了一种类似“阻塞”的效果。注意这并不是真正的线程阻塞，只是在效果上“阻塞”了当前协程，而不会阻塞到当前线程。

协程也可以直接被放到yield from语句之后，像语句7、8那样，这种情况下协程会被自动打包成一个任务。但是这时的输出会变成如下的顺序：

* a start
* a finish
* b start 
* b finish

因为这时候协程a()和b()不是被同时加入事件循环的，语句7使得协程a()先被加入事件循环，在yield from上“阻塞”了1秒钟之后，接下来执行语句8，这个时候协程b()才被加入事件循环，所以两者实际上是被串行执行的。

除了提供“阻塞”效果外，yield from还可以获取协程的返回值、传递协程中抛出的异常。虽然协程被包装成任务之后就会加入事件循环被承诺执行，但是尽量不要使用这样的fire and forget的方式，而是总是通过yield from捕获或者向上传递异常，否则异常会被丢失。

### async/await版本

async/await版本是原生协程，不再基于生成器，也就是说没有实现\_\_iter\_\_和_\_next\_\_方法。

在语法上，async/await基本和@asyncio.coroutine/yield from没什么区别，可以简单地替换使用。

此外，原生协程支持异步生成器async for和异步上下文管理器async with。

### 阻塞函数的异步调用

协程因为是单线程的，所以在任何一个环节都不能调用阻塞的函数，否则会阻塞整个线程。

如果实在是没有可用的非阻塞函数，只能调用阻塞函数，这时候可以通过loop提供的run_in_executor(executor, func, *args)方法将阻塞操作加入线程池执行。其中executor必须是一个concurrent.futures.Executor实例，如果传入None，会使用默认的executor。run_in_executor()方法返回一个future，该future由线程池承诺执行。使用线程池的实例如下：

```python
def a():
  print('a start')
  time.sleep(1)
  print('a finish')

def b():
  print('b start')
  time.sleep(1)
  print('b finish')
 
@asyncio.coroutine
def main():
  loop = ayncio.get_running_loop()
  future1 = loop.run_in_executor(None, a)
  future2 = loop.run_in_executor(None, b)
  yield from future1
  yield from future2

loop = asyncio.get_event_loop()
loop.run_until_complete(main())
loop.close()
```

输出结果是：

* a start
* b start
* a finish
* b finish

 使用线程池的时候要注意，asyncio中大部分对象和方法都不是线程安全的，只有两个方法是线程安全的，asyncio.run_coroutine_threadsafe(coro, loop)和loop.call_soon_threadsafe(callback, *args)。如果要在其它线程（非event loop所在的线程）中使用其它对象和方法，需要先通过这两个方法将控制传递回event loop。

当需要动态地往event loop里加入协程的时候，可以在主线程中生成一个event loop，然后在一个新的线程中用loop.run_forever()启动该event loop，当需要动态加入协程时，可以在主线程中通过调用asyncio.run_coroutine_threadsafe(coro, loop)将协程跨线程地加入到event loop中。

### 协程和多线程的比较

协程可以在用户态完成切换，性能更高。

协程比线程占用的资源少，并发数量远高于多线程。

线程的调度是抢占式的，中断的时机不可控，而协程的调度是可控的，总是在可以预见的地方中断，所以在访问共享资源的时候对锁的依赖更小。