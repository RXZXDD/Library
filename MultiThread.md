# MultiThread
## 标准多线程 FRunnable

1.1 Worker
```c++
	#include "HAL/Runnable.h"
```

```c++
//FRunnable 
	virtual bool Init() override;
	virtual uint32 Run() override;
	virtual void Stop() override;
	virtual void Exit() override;
```


1.2 Runnable Thread
```c++
	Create()创建Worker 
```

## AsyncTask

1.1 FNonAbandonableTask

```c++
	DoWork()函数 == FRunnable::Run()
```

继承FNonAbandonableTask的Task不可以在执行阶段终止，即使执行Abandon函数也会去触发DoWork函数

1.2 FQueuedThreadPool线程池

	维护FQueuedThread 和 任务队列 IQueuedWork

	专有服务器的线程池GThreadPool默认只开一个线程，非专有服务器的根据核数开（CoreNum-1）个线程。

	编辑器模式会另外再创建一个线程池GLargeThreadPool，包含（LogicalCoreNum-2）个线程，用来处理贴图的压缩和编码相关内容。

	FQueuedThread 有事件触发机制FEvent* DoWorkEvent

1.3 AsynTask && FAutoDeleteAsyncTask
	FAsyncTask有几个特点，

	1.FAsyncTask是一个模板类，真正的AsyncTask需要你自己写。通过DoWork提供你要执行的具体任务，然后把你的类作为模板参数传过去
	2.使用FAsyncTask就默认你要使用UE提供的线程池FQueuedThreadPool，前面代码里说明了在引擎PreInit的时候会初始化线程池并返回一个指针GThreadPool。在执行FAsyncTask任务时，如果你在执行StartBackgroundTask的时候会默认使用GThreadPool线程池，当然你也可以在参数里面指定自己创建的线程池
	3.创建FAsyncTask并不一定要使用新的线程，你可以调用函数StartSynchronousTask直接在当前线程上执行任务
	4.FAsyncTask本身包含一个DoneEvent，任务执行完成的时候会激活该事件。当你想等待一个任务完成时再做其他操作，就可以调用EnsureCompletion函数，他可以从队列里面取出来还没被执行的任务放到当前线程来做，也可以挂起当前线程等待DoneEvent激活后再往下执行

	FAutoDeleteAsyncTask与FAsyncTask是相似的，但是有一些差异，

	1.默认使用UE提供的线程池FQueuedThreadPool，可以通过参数指定使用其他线程池
	2.FAutoDeleteAsyncTask在任务完成后会通过线程池的Destroy函数删除自身或者在执行DoWork后删除自身，而FAsyncTask需要手动delete
	3.包含FAsyncTask的特点1和特点3

## 其他

1. FScopeLock

FScopeLock是UE提供的一种基于作用域的锁，思想类似RAII机制。在构造时对当前区域加锁，离开作用域时执行析构并解锁。UE里面有很多带有“Scope”关键字的类，如移动组件中的FScopedMovementUpdate，Task系统中的FScopeCycleCounter，FScopedEvent等，他们的实现思路是类似的。

# 并行编程设计模式

1. Future 异步执行，同步返回
2. Master-Worker
   1. Master类有： 任务容器，任务提交方法， worker容器，worker构造器（构造Master类时同时新开n个worker线程），任务执行器，任务完成判断方法，任务归并方法
   2. Worker类： task队列，结果队列
3. 生产者-消费者
4. fork/join


# LockFree
	主要技术点
	-CAS原子操作
	-ABA问题
	-内存屏障

	代码里面有CAS（Compare-And-Swap）循环操作，就得考虑ABA问题

	## ABA问题的解决方法：

		Pop操作加上一个计数器
		更新栈顶和Pop操作计数器，为一个原子操作
		这样每次Pop操作时就不再是ABA，而是A1BA2了，这样只要上个Pop没完成，这个Pop就得等待了。

	
	## 内存屏障
		解决内存访问乱序

		内存访问乱序问题

			主要原因：
				编译器优化---手动添加MB
				CPU乱序执行（out-of-order）执行-----因内存模型差异导致
				CPU缓存（Cache）不一致性

# TLS（Thread local Storage）

# 常用的并行编程模式
 1. Agent and Repository
    1. 针对这样一类问题：我们有一组数据，它们会随机的被一些不同的对象进行修改。解决这一类问题的方案是，创建一个集中管理的数据仓库（data repository），然后定义一组自治的agent来操作这些数据，可能还有一个manager来对agent的操作进行协调，并保证数据仓库中数据的一致性。
 2. Map Reduce
    1. 指的是这样一类问题的解决方案：我们可以分两步来解决这类问题。第一步，使用一个串行的Mapper函数分别处理一组不同的数据，生成一个中间结果。第二步，将第一步的处理结果用一个Reducer函数进行处理（例如，归并操作），生成最后的结果。
 3.  Data Parallelism
 4.  Task Parallelism
 5.  Recursive Splitting
 6.   Pipeline
 7.    Geometric decomposition
       1.    对于一些线性的数据结构（例如数组），我们可以把数据切分成几个连续的子集
 8.    Non-work-efficient Parallelism
       1.    有些问题的处理使用传统的方法，必须依赖于对数据进行有序的访问，例如深度优先搜索，这样就很难并行化。但是假如我们愿意花费一些额外的计算量，我们就能够采用并行的方法来解决这个问题。


# TaskGraph

	## 引擎中有大量逻辑的并行依赖于TaskGraph系统：

	① 执行RenderCommand（渲染线程中）

	② 执行RHICommand（RHI线程中）

	③ Actor及ActorComponent的Tick

	④ 遮挡剔除（Occlusion Culling）

	⑤ MeshDrawCommands生成

	⑥ GC Mark

	⑦ 更新动画骨骼

	⑧ 物理模拟


## TaskGraph管理两种类型的线程：外部线程（NamedThread）和内部线程（AnyThread）

- 外部线程：非TaskGraph内部创建的线程，包括GameThread、RenderThread、RHIThread、StatsThread和AudioThread。通过FTaskGraphInterface::Get().AttachToThread()函数来添加。

- 内部线程：TaskGraph在引擎初始化时创建的工作线程（TaskGraphThreadHP、TaskGraphThreadNP、TaskGraphThreadBP，优先级：HP > NP > BP），具体逻辑在FTaskGraphImplementation构造函数中。
	内部线程的每档数量由FPlatformMisc::NumberOfWorkerThreadsToSpawn()函数来决定。当然如果平台本身不支持多线程，TaskGraph执行的逻辑会放回GameThread中。具体逻辑详见FTaskGraphImplementation构造函数