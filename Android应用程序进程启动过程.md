# Android应用程序进程启动过程
## 1、应用程序进程启动过程
AMS在启动应用程序时会检查这个应用程序需要的应用程序进程是否存在，不存在就会请求Zygote进程启动需要的应用程序进程。在Zygote的Java框架层中会创建一个Server端的Socket，这个Socket用来等待AMS请求Zygote来创建新的应用程序进程。Zygote进程通过fock自身创建应用程序进程，这样应用程序进程就会获得Zygote进程在启动时创建的虚拟机实例。在应用程序进程创建过程中，除了获取虚拟机实例外，还创建了Binder线程池和消息循环，这样运行在应用进程中的应用程序就可以方便地使用Binder进行进程间通信以及处理消息了。

### 1.1 AMS发送启动应用程序进程请求

- AMS要启动应用程序进程，就需要向Zygote进程发送创建应用程序的请求，AMS会通过调用startProcessLocked方法向Zygote进程发送请求。

	- 获取要创建的应用程序进程的用户ID
	- 对用户组ID（gids）进行创建和赋值
	- 如果entryPoint为null，则赋值为`android.app.ActivityThread`，即应用程序进程主线程的类名
	- 调用Process的start方法，将应用程序进程用户ID和用户组ID传进去
- Process的start方法只调用了ZygoteProcess的start方法，ZygoteProcess类用于保持与Zygote进程的通信状态。最后调用zygoteSendArgsAndGetResult方法。
- zygoteSendArgsAndGetResult方法的主要作用就是将传入的应用进程的启动参数argsForZygote写入ZygoteState中，ZygoteState是ZygoteProcess的静态内部类，用于表示Zygote进程通信的状态。ZygoteState是由openZygoteSocketIfNeeded方法返回。
- 由于在Zygote的main方法中会创建name为"zygote"的Server端Socket，在openZygoteSocketIfNeeded方法中，首先调用ZygoteState的connect方法与Server端Socket建立连接。如果连接Zygote主模式返回的ZygoteState与启动应用程序进程所需的ABI不匹配，就会尝试连接Zygote的辅模式。如果辅模式返回的ZygoteState与启动应用程序进程所需的ABI不匹配，则抛出ZygoteStartFailedEx异常。

### 1.2 Zygote接收请求并创建应用程序进程
- ZygoteInit通过registerZygoteSocket方法创建一个name为"zygote"的Server端的Socket，然后预加载类和资源，启动SystemServer进程，最后调用ZygoteServer的runSelectLoop方法等待AMS请求创建新的应用程序进程。
- 当有AMS的请求数据到来时，会调用ZygoteConnection的runOnce方法来处理请求数据。

	- 调用readArgumentList方法获取应用程序进程的启动参数
	- 将readArgumentList方法返回的字符串数组args封装到Arguments类型的parseArgs对象中
	- 调用Zygote的forkAndSpecialize方法来创建应用程序进程，fockAndSpecialize方法主要是通过fork当前进程来创建一个子进程，如果pid等于0，则说明当前代码逻辑运行在新创建的子进程中，这时会调用handleChildProc方法来处理应用程序进程。
	- handleChildProc方法调用了ZygoteInit的zygoteInit方法，在新创建的应用程序进程中创建Binder线程池，初始化。
	- 通过抛出MethodAndArgsCaller异常，调用ActivityThread的main方法，运行主线程的管理类ActivityThread。

## 2、Binder线程池启动过程
- ZygoteInit的zygoteInit方法中会调用nativeZygoteInit方法创建Binder线程池，对应的jni函数是com_android_internal_os_ZygoteInit_nativeZygoteInit

		const JNINativeMethod methods[] = {
			{"nativeZygoteInit", "()V", 
			(void*) com_android_internal_os_ZygoteInit_nativeZygoteInit},
		};
		
- 函数内部最后调用ProcessState的startThreadPool函数启动Binder线程池
- 支持Binder通信的进程中都有一个ProcessState类，它里面有一个**mThreadPoolStarted**变量，用来表示Binder线程池是否已经被启动过，默认值是false。每次调用startThreadPool函数时都会先检查这个标记，从而确保Binder线程池只会被启动一次。
- Binder线程为一个PoolThread，PoolThread类继承了Thread类。通过调用IPCThreadState的joinThreadPool函数，将当前线程注册到Binder驱动程序中，这样我们创建的线程就加入了Binder线程池中，新创建的应用程序进程就支持Binder进程间通信了。我们只需要创建当前进程的Binder对象，并将它注册到ServiceeManager中就可以实现Binder进程间通信。

## 3、消息循环创建过程
应用程序进程启动后会自动创建消息循环，在RuntimeInit的invokeStaticMain方法最后，抛出一个MethodAndArgsCaller异常，这个异常会被ZygoteInit的main方法捕获。

```java
public static void main(String argv[]) {
	...
	try {
		...
	} catch (MethodAndArgsCaller caller) {
		caller.run();
	}
}

public static class MethodAndArgsCaller extends Exception implements Runnable {
	private final Method mMethod;
	private final String[] mArgs;
	
	public void run() {
		try {
			mMethod.invoke(null, new Object[] {mArgs});
		} catch (IllegalAccessException ex) {
			throw new RuntimeException(ex);
		}
	}
}
```
mMethod就是ActivityThread的main方法，mArgs就是应用程序进程的启动参数。ActivityThread的main方法如下：

```java
public static void main(String[] args) {
	...
	// 创建主线程Looper
	Looper.prepareMainLooper();
	ActivityThread thread = new ActivityThread();
	thread.attach(false);
	if (sMainThreadHandler == null) {
		sMainThreadHandler = thread.getHandleer();
	}
	...
	Looper.loop();
}
```
ActivityThread类用于管理当前应用程序进程的主线程，main方法中创建主线程的消息循环Looper，创建ActivityThread。判断Handler类型的sMainThreadHandler是否为空，如果为空则获取H类并赋值给sMainThreadHandler。这个H类继承自Handler，是ActivityThread的内部类，用于处理主线程的消息循环。最后调用Looper的loop方法，使得Looper开始处理消息。

因此，系统在应用程序进程启动完成后，就会创建一个消息循环，这样运行在应用程序进程中的应用程序可以方便地使用消息处理机制。
