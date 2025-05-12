# RTOS

### 1. FreeRTOS文件介绍

​	（1）port.c

​		针对于不同硬件平台的标准化接口

​	（2） 

#### 2.多任务并发执行所造成的混乱

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