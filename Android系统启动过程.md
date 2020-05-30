# Android系统启动过程
## 1、启动电源以及系统启动
当电源按下时，引导芯片代码从预定义的地方（固化在ROM）开始执行，加载引导程序BootLoader到RAM中，然后执行。

## 2、引导程序BootLoader<br>
引导程序BootLoader是在Android操作系统开始运行前的一个小程序，它的主要作用是把系统OS拉起来并运行。

## 3、Linux内核启动<br>
当内核启动时，设置缓存、被保护存储器、计划列表、加载驱动。在内核完成系统设置后，它首先在系统文件中寻找init.rc，并启动init进程。

## 4、init进程启动过程
init进程是Android系统中用户空间的第一个进程，进程号为1，是Android系统启动流程中的一个关键步骤。


init进程主要用来初始化和启动属性服务，也用来启动Zygote进程。

- 创建和挂载启动所需的文件目录
- 初始化和启动属性服务
- 解析init.rc配置文件并启动Zygote进程

属性服务记录存储了用户、软件的一些使用信息。
	
## 5、Zygote进程启动过程
在Android系统中，DVM和ART、应用程序进程以及运行系统的关键服务的SystemServer进程都是由Zygote进程来创建的。它通过fock进程的形式创建应用进程和SystemServer进程，由于Zygote进程在启动时会创建DVM或者ART，因此通过fock而创建的应用程序进程和SystemServer进程可以在内部获取一个DVM或者ART的实例副本。

- 创建AppRuntime并调用其start方法，启动Zygote进程。
- 创建JVM并为JVM注册JNI方法。
- 通过JNI调用ZygoteInit的main函数进入Zogote的Java框架层。
- 通过registerZygoteStart方法创建服务端Socket，并通过runSelectLoop方法等待AMS的请求来创建新的应用程序进程。
- 启动SystemServer进程

## 6、SystemServer处理过程
SystemServer进程主要用于创建系统服务，AMS、WMS和PMS都是由SystemServer进程创建的。

- 启动Binder线程池，这样就可以与其它进程进行通信。
- 创建SystemServiceManager，其用于对系统的服务进行创建、启动和生命周期管理。
- 启动各种系统服务。

## 7、Launcher启动过程
系统启动的最后一步是启动一个应用程序用来显示系统中已经安装的应用程序，这个应用程序叫Launcher，被SystemServer进程启动的AMS会启动Launcher。Launcher在启动过程中会请求PackageManagerService返回系统中已经安装的应用程序信息，并将这些信息封装成一个快捷图标列表显示在系统屏幕上。
