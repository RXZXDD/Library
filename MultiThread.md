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

