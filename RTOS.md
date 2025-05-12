# RTOS

### 1. FreeRTOS文件介绍

​	（1）port.c

​		针对于不同硬件平台的标准化接口

​	（2） 

### 2.多任务并发执行所造成的混乱

​	做一个实验：两个任务，同时向串口上分别打印输出字符串“HELLO RTOS”和“hello rtos”

​	实验结果，打印内容混乱。

<img src="C:\Users\abc18\AppData\Roaming\Typora\typora-user-images\image-20250512100556749.png" alt="image-20250512100556749" style="zoom:80%;" />

两个任务使用一个串口，调度器周期性的切换两个任务，造成以上现象。称这个串口为临界资源，所以基本**解决思想**时：一个任务用完，另一个才能用。

1.使用宏taskENTER_CRITICAL ()和taskEXIT_CRITICAL(); 被这两个宏夹在中间的这部分代码就被称为“**临界区**”

**2.临界区的原理是什么？**

**停止多任务的切换**，就解决了任务间竞争临界资源的问题， 怎么实现停止多任务的切换呢？--**直接把tick中断关了，或者把处理器的全部中断都关了**

执行临界区代码的时候，暂时停止系统的任务切换。

**3.那如果系统的tick中断走走停停，那RTOS的延时还准吗？调度器切换任务的周期还准吗？**

这个问题确实存在。

### 3.创建任务的API（应用编程接口）函数

```
BaseType_t xTaskCreat(	TaskFunction_t		pxTaskCode,			//指向任务函数的函数指针，这里直接写函数名fun void fun(void *p)
						const char *const	pcName,				//一个字符串，代表任务名“fun_name” 
																//任务名字还不能太长，FreeRTOS有宏定义规定了任务名字最大长度																				//#define configMAX_TASK_AME_LEN	()
						const uint16_t		usStackDepth,		//任务栈的深度
						void *const			pvParameters,		//(void *) 0
						UBaseType_t			uxPriority,			//任务优先级
						TaskHandle_t *const	pxCreatedTask)		//任务句柄&xHandle
//类型重定义使用#define和typedef（projdefs.h）
//意义：1.提高代码的可读性	2.提高代码可移植性 
//TaskFunction_t	->	void(*)(void *) 函数指针
//UBaseType_t		->	unsigned long	无符号长整型
//TaskHandle_t		->	struct tskTaskControlBlock *	结构体指针
```

**什么是句柄？**

定义：句柄是一个软件资源或者说是单元的唯一标识。

在FreeRTOS中，句柄实质上是取任务栈的首地址，这可以保证每一个任务的句柄值一定是唯一的，句柄降低研发难度，通过句柄对任务进行操作。

实验测试任务句柄的作用，使用vTaskGetInfo()这个API查询一个任务的信息，给出这个任务的句柄。

有些时候任务创建句柄是一个局部变量，不推荐让其变成全局变量供整个文件使用，所以使用xTaskGetHandle(“fun_name”)获取句柄，只需要传入任务名。 

![image-20250512105858654](C:\Users\abc18\AppData\Roaming\Typora\typora-user-images\image-20250512105858654.png)

### 4.RTOS的核心精髓，任务调度器

​	 启动调度器：vTaskStartScheduler(),简单理解为使能Tick定时器中断。 

![image-20250512111042371](C:\Users\abc18\AppData\Roaming\Typora\typora-user-images\image-20250512111042371.png)

**任务被切换后，真正执行上下文切换是什么时刻？**任务在延时函数发生时，立即切换上下文，但是在下一个Tick来临时开始计算延时。每一次Tick中断中都会调用FreeRTOS内核的相关函数，这些函数会去检测所有任务的状态，从而执行上下文切换。也就是要暂停哪些任务，要恢复哪些任务。又可以发现Delay也不准确。

![image-20250512111533494](C:\Users\abc18\AppData\Roaming\Typora\typora-user-images\image-20250512111533494.png)****

**Tick中断函数代码**

​	<img src="C:\Users\abc18\AppData\Roaming\Typora\typora-user-images\image-20250512112106053.png" alt="image-20250512112106053" style="zoom:50%;" />

<img src="C:\Users\abc18\AppData\Roaming\Typora\typora-user-images\image-20250512112336828.png" alt="image-20250512112336828" style="zoom:50%;" />

### 5.任务上下文切换

​	 任务上下文切换的实质：将CPU内核的寄存器组的值压到任务栈（PUSH）或者从任务栈中恢复到CPU内核的寄存器组中（POP）。

#### 	 在任务运行过程中突然出现中断会影响上下文切换吗？

​	这个时刻结束后，进行直接进行压栈操作，那CPU寄存器的值，应该是中断程序的现场，也就是说中断现场压入当前任务的栈中了，被中断打断的任务现场压入主栈中。然后再下一次运行该任务是，POP到CPU寄存器的值那就是中断的现场了。

​	**所以任务上下文的切换之前，还得充分考虑硬件中断的问题。**

​	RTOS虽然各任务都是基于自己的任务栈开发的，但是还是存在**主栈**。主栈用来运行非任务代码，出入栈由硬件或编译器来完成的。

​	所以在调度器切换上下文时，先检查有没有中断程序正在执行，如果有，就**等待中断程序运行完，再执行任务上下文切换**。保证切换操作都是在任务环境下进行的。

#### 	但是调度器怎么知道当前是不是有中断在执行呢？

 	假设有一种优先级最低的中断，等所有中断完成后才会去处理它，这个中断由我们主动控制，想让它什么时候产生，它就什么时候产生，我们把任务切换放在这个中断的服务程序中，就可以做到上下文切换时所有中断都已经执行完成了。

​	**所有处理器都有一个这样中断：ARM上叫做PendSV:可被悬挂的系统调用（缓期执行），RISC-V上叫ecall，专门为RTOS而生。**

​	RTOS自身用到的硬件中断，优先级基本都很低，比如PendSV、Tick中断

#### 	那为什么Tick中断的优先级要设置的很低呢？设置的高会怎么？

​	Tick中断需要频繁发生，优先级高会破坏实时性的特点。那Tick中断优先级降低了，说明Tick周期就不准了，那中断多起来，就会拖累整个系统，导致各个任务调度变慢。所以RTOS开发中不要有过多的硬件中断。但是在通信上，比如说串口接收，就是要连续不断的触发中断接收字节，那实际开发时可使用DMA，批量的处理数据收发，减少中断次数。



### 6.RTOS调度方式

##### 抢占式调度

创建函数时会有一个形参：uxPriority，值越大，优先级越高。可以在FreeRTOSConfig.h中使用configMAX-PRIORITIES修改系统优先级的层数。

FreeRTOS默认调度方式就是抢占式调度

如果优先级一样会是怎样？

调度器会把时间均匀分给就绪状态的任务，并且优先启动同优先级后创建的任务。

如果在高优先级的任务中，直接使用while(1)，就直接独占CPU了，在实际开发时，一定不要让任务独占CPU，要适当的主动让出CPU，尤其是高优先级的任务，要让别的任务也有时间执行。

##### 时间片轮转调度

如果FreeRTOS默认的情况下，当任务优先级相同时，则自动采用时间片轮转调度。

在FreeRTOS中时间片轮转是一个鸡助，因为在FreeRTOS中每个人物只有一个Tick的时间片长度，无法调整时间片，这样调度器就会一直进行切换。RTOS本身也要耗费CPU资源和时间，所以尽量不要切换任务。**如果不用时间片轮转，在FreeRTOS.h中关闭时间片轮转调度，会发生什么呢？**

这样只有一个任务在运行。

其他RTOS可以设置时间片大小



### 7.RTOS的四种状态

​	在任务中可以使用 vTaskDelay让任务暂停运行，实质是在等待资源，就是vTaskDelay被调用后，任务就进入了阻塞状态。

RTOS有：**运行态、阻塞态、就绪态和挂起态**。

​	任务运行起来的必需基本条件：当前任务处于就绪状态，在所有任务中，任务优先级最高。

​	任务的挂起状态，脱离调度器的管辖。

<img src="C:\Users\abc18\AppData\Roaming\Typora\typora-user-images\image-20250512161329323.png" alt="image-20250512161329323" style="zoom: 50%;" />



### 8.优先级反转

  低中高三个优先级任务，高优先级要等待低优先级资源，但是中优先级会抢占低优先级任务，从而低优先级释放不了资源，那么中优先级就会一直运行，高优先级得不到运行。

使用优先级继承，暂时提高低优先级任务的优先级，不让中优先级抢占。