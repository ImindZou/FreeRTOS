# FreeRTOS软件定时器



[TOC]

## 一、软件定时器概述

### 1、软件定时器的特性

**软件定时器（Software Timer）是FreeRTOS中的一种对象**。

FreeRTOS中的软件定时器不直接使用任何硬件定时器或计数器，它依赖于系统中的 **时间服务任务（timer service task）**，或称为 **守护任务（daemon task）** 来工作。

软件定时器有一个 **定时周期**，还有一个 **回调函数** 。根据回调函数执行的调度，有两种类型的软件定时器。

- 单次定时器（one-shot-timer），回调函数被执行后，定时器就停止工作。
- 周期性定时器（periodic timer），回调函数每次执行都重装载定时器的计数值，使得每次定时器都可以重新开始倒计时。
- 定时器被创建后，有 **休眠**和 **运行**两种状态。
    - 休眠（dormant）状态
        - 处于休眠状态的定时器不会执行回调函数，但是可以使用其句柄对其进行操作，例如设置周期。
        - 定时器在以下几种情况处于休眠状态：
            - 定时器被创建后就处于休眠状态。
            - 单次定时器执行一次回调函数后进入休眠状态。
            - 定时器使用函数xTimerStop（）停止后进入休眠状态。
    - 运行（Running）状态
        - 处于运行状态的定时器在流逝的时间达到其定时周期就会执行其回调函数，不管是单次定时器，还是周期定时器。
        - 定时器在以下几种状态处于运行状态：
            - 使用函数xTimerStart（）启动后，定时器进入运行状态。
            - 定时器在运行状态时，被执行函数xTimerReset（）复位起始时间后依然处于运行状态。

![image-20240404215454361](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404042154515.png)

软件定时器的各种操作实际上是在系统时间服务任务里完成的。时间服务任务是FreeRTOS自动创建的一个任务。如下图系统内核中的任务列表所示（Tmr Svc    Timer Server）它是开启定时器配置后系统自动检测创建的，即使没有显式地调用或创建定时器任务。

![image-20240404215545027](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404042155082.png)

在用户任务里执行的各种指令都是通过一个队列发送给时间服务任务的，这个队列称为**定时器指令队列**。时间服务任务读取定时器指令队列里的指令，然后执行相应的操作。

![image-20240404215941613](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404042159659.png)

**时间服务任务和定时器指令队列是FreeRTOS自动常见的，其操作都是内核实现的。**

![image-20240404220032458](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404042200503.png)

**时间服务任务还在都是时间达到时执行定时器回调函数。**

由于FreeRTOS里的延时功能是由时间服务函数实现的，所以 **在定时器的回调函数里不能出现使系统进入阻塞状态的函数** 如vTaskDelay（）、vTaskDelayUntil（）等。

**回调函数里可以调用等待信号量、事件组等对象的函数，但是等待的节拍必须设置为0.**

回调函数中调用等待信号量、事件组等同步对象的函数时，等待的节拍数（tick count）必须设置为0的原因确实是为了避免阻塞。这是因为回调函数通常在特定的上下文中执行，比如中断服务例程（ISR）或者具有特定限制的任务上下文，其中不允许执行可能引起阻塞的操作。

以下是为什么在这些上下文中阻塞操作是不被允许或不推荐的几个原因：

> ### 1. 中断服务例程（ISR）中的阻塞

- **实时性**：中断服务例程应该尽可能快地执行完毕并返回，以确保系统能及时响应其他中断。阻塞操作可能导致ISR执行时间不确定，影响系统的实时性。
- **资源竞争**：在ISR中等待对象（如信号量、事件组）可能导致资源竞争和死锁，特别是当ISR优先级很高时，它可能阻塞等待一个低优先级任务释放的资源，而低优先级任务又得不到执行。
- **内核限制**：许多RTOS内核在中断上下文中有限制，不允许执行可能会阻塞的操作，因为这可能会破坏内核的调度和同步机制。

> ### 2. 特定任务上下文中的阻塞

在某些任务执行的回调函数中，尤其是设计为快速执行的任务，也不推荐进行阻塞操作，原因包括但不限于：

- **任务设计**：如果任务设计为执行周期性检查或快速响应外部事件，阻塞操作可能导致错过这些事件或延迟处理。
- **系统负载**：在这些上下文中执行阻塞操作，可能会不必要地增加系统负载，降低整体性能和响应能力。

因此，设置等待节拍数为0的目的是尝试立即获取同步对象（如信号量或事件组），如果无法立即获取，则不等待（非阻塞），从而保持回调函数的执行速度和系统的实时性。这种做法允许在不违反实时操作原则的情况下，安全地在特定上下文中使用同步机制。

```c
//这种设计方法通常被认为是等待节拍为0的回调函数，等待节拍可以理解为阻塞函数。如：vTaskDelay（）等会运气系统阻塞的API。	
void Timer1CallBack(TimerHandle_t xTimer)
	{
		TickType_t tick_num1;

		Timer1CallBackCount++;  //increase by 1 for each callback

		tick_num1 = xTaskGetTickCount();  //geting the current tick count

		HAL_GPIO_TogglePin(LED1_GPIO_Port,LED1_Pin);

		printf("Timer1CallBackCount:%d,tick_num1:%d\r\n",Timer1CallBackCount,tick_num1);
	}
```



### 2、软件定时器相关配置

![image-20240405154410783](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404051544873.png)

![image-20240405154431120](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404051544214.png)



### 3、时间服务函数的优先级

时间服务函数执行定时器指令队列中的定时器操作指令，或定时器回调函数。时间服务任务的优先级由参数 **configTIMER_TASK_PRIORITY设定**，至少要高于空闲任务的优先级，默认值为2.

使用定时器的用户任务优先级可能高于或低于时间服务任务的优先级，那么时间服务任务执行定时器操作指令的时间是不同的。

![image-20240405160341211](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404051603483.png)

![image-20240405160421433](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404051604498.png)

![image-20240405160429960](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404051604024.png)

![image-20240405160644299](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404051606364.png)

## 二、软件定时器相关API

![image-20240405160717357](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404051607423.png)

![image-20240405160731068](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404051607130.png)

![image-20240405160737055](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404051607120.png)

![image-20240405160751931](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404051607974.png)

![image-20240405160807924](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404051608006.png)

![image-20240405160841015](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404051608079.png)

**也就是我们可以自定义一个函数指针来对定时器到位之后的后续进行一些特定的操作。**（切记，不可阻塞）

![image-20240405161034355](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404051610414.png)

![image-20240405161142026](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404051611095.png)

![image-20240405161156102](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404051611156.png)

![image-20240405161221412](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404051612457.png)

![image-20240405161226697](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404051612743.png)

![image-20240405161240520](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404051612570.png)

# 过程中对RISC—V的学习

**这是一个任务在切换上下文常见的一个操作**，切换上下文首先就需要把当前状态写入到一个寄存器里边，通过调用msr来访问一些特殊寄存器，然后进行写入操作，这是一个中断保护现场的一个操作。主要是对下面这个变量的理解。理解其在操作系统中的应用以及底层实现机制。首先任务切换要等待最近的一次Tick中断的到来，这个测过大概60us（任务拿到状态字后进入就绪态调度到运行态的周期），tick中断来了后调用svc进行任务调度。

![image-20240405164712713](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404051647887.png)

![image-20240405162435405](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404051624447.png)

![image-20240405162117406](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404051621485.png)

![image-20240405162124512](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404051621571.png)

![image-20240405162129991](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404051621063.png)

## 对于上下文切换的理解及实际操作（附.经典问答）

```c
#if 0	//binary semaphore test
	portBASE_TYPE taskWoken = pdFALSE; 
	if (GPIO_Pin == KEY1_Pin)
	{
		xSemaphoreGiveFromISR(key1_sem, &taskWoken);
		//printf("Key1 Pressed and release the sem\r\n");
		if(taskWoken == pdTRUE){
			portYIELD_FROM_ISR(taskWoken);
		}
	}
	if (GPIO_Pin == KEY2_Pin)
	{
		xSemaphoreGiveFromISR(key2_sem, &taskWoken);
		//printf("Key2 Pressed and release the sem\r\n");
		if(taskWoken == pdTRUE){
			portYIELD_FROM_ISR(taskWoken);
		}
	}
	#endif


void key1_press_entry(void *p)
{
	 // 	simple semaphore trigger
	while(1)
	{
		xSemaphoreTake(key1_sem, portMAX_DELAY);
		printf("gain the key1 sem\r\n");
		vTaskDelay(100);
	}
}
#endif
#if 0
void key2_press_entry(void *p)
{
	while(1)
	{
		xSemaphoreTake(key2_sem, portMAX_DELAY);
		printf("gain the key2 sem\r\n");
		vTaskDelay(100);
	}
}
#endif
```

```ABAP
中断这么设计在中断服务例程执行完后它的taskwoken是true还是false ？

这个设计用于在中断服务例程（ISR）中给出一个信号量，并且在必要时请求一个上下文切换。`taskWoken`变量的作用是记录是否需要进行上下文切换。

初始时，`taskWoken`被设置为`pdFALSE`。调用`xSemaphoreGiveFromISR()`时，如果这个操作导致一个更高优先级的任务被解阻塞，那么`xSemaphoreGiveFromISR()`会将`taskWoken`设置为`pdTRUE`。这表示在退出中断时，需要立即进行上下文切换到那个更高优先级的任务。

- 如果在调用`xSemaphoreGiveFromISR()`期间，没有任务因为此信号量的释放而变得就绪，`taskWoken`将保持为`pdFALSE`，不会请求上下文切换。
- 如果有任务因此变得就绪，并且它的优先级高于当前正在运行的任务，`taskWoken`将被设置为`pdTRUE`，这时`portYIELD_FROM_ISR()`将被调用来触发上下文切换。

所以，在中断服务例程执行完后，`taskWoken`的值取决于`xSemaphoreGiveFromISR()`调用的结果。如果此操作导致需要立即切换到一个更高优先级的任务，`taskWoken`将为`pdTRUE`；否则，它将保持`pdFALSE`。

中断不算任务吧，它是切换到需要这个信号的同种任务中最高优先级的任务中去嘛？

中断确实不算作任务。在RTOS（实时操作系统）中，中断服务例程（ISR）和任务是不同的执行上下文。中断是由硬件事件直接触发的，而任务是操作系统调度的软件线程。

当在ISR中给出信号量（通过`xSemaphoreGiveFromISR`等函数）时，这可能会解阻塞一个或多个等待该信号量的任务。如果解阻塞的任务中有任何一个的优先级高于当前正在运行的任务，RTOS会标记需要进行上下文切换。一旦ISR执行完毕，如果`taskWoken`被设置为`pdTRUE`（表示有更高优先级的任务需要运行），RTOS会在中断返回时切换到那个更高优先级的任务去执行。

所以，“切换到需要这个信号的同种任务中最高优先级的任务中去”这个描述是准确的。如果有多个任务因为信号量被释放而变得就绪，RTOS会选择这些就绪任务中优先级最高的那个来执行。这个机制确保系统可以及时响应高优先级任务的需求，是RTOS设计中用于实现任务间通信和同步的关键部分。
```

> **这是两个不同的信号量，因此不会分优先级高低，因为对每种信号量类型有需求的任务只有一个，因此这个任务就是最高优先级，任务释放直接获取，这是一个简单的案例，可以想象，放到大型工程中会有多少种可能性呢。而且这还是普通二值信号量，把那个不是计数信号量，更别提互斥锁这种带优先级继承的信号量了。玩法很多，也可以说是策略的设计有很多种。** 
>
> **关键API：portYIELD_FROM_ISR（taskWoken）  如果ISR中释放的信号量使某个任务进入了就绪状态，则立即调用这个API快速进行任务调度，使该任务得以运行起来。**

## 顶半操作跟底半操作的再回忆（附.经典问答）

```ABAP
rtos中顶半操作跟底半操作的区别是什么，为什么说中断属于顶半操作，任务属于底半操作？


在实时操作系统（RTOS）的上下文中，顶半（Top Half）操作和底半（Bottom Half）操作是处理中断和任务的两种机制，它们共同协作以高效地处理系统中的事件和任务。这种划分主要用于管理中断服务和保持系统的实时响应性。

顶半操作（Top Half）
顶半操作主要指的是中断服务例程（ISR）的处理，这是中断发生时立即执行的部分。顶半处理通常需要快速执行并完成以下任务：

快速响应中断信号。
保存必要的状态或数据。
快速清除中断标志，以便硬件可以继续生成后续中断。
触发底半处理（例如，通过设置一个信号量、标志或发送消息）以进一步处理。
顶半处理的设计目标是尽量减少在中断上下文中花费的时间，以避免影响系统的实时性和响应其他中断的能力。

底半操作（Bottom Half）
底半操作是对中断处理的延续，但它在任务上下文中执行，而不是中断上下文。底半处理通常负责：

执行更复杂或耗时的数据处理。
完成由顶半触发的操作，如更新系统状态、处理接收到的数据等。
通过任务调度，底半操作可以被优先级管理，确保更重要的任务首先得到处理。
底半机制使得系统可以在不阻塞中断的情况下，安全地执行复杂或耗时的操作。

为什么说中断属于顶半操作，任务属于底半操作
中断属于顶半操作：因为中断的处理需要快速响应和处理，以释放中断线以便处理新的中断。中断处理的快速部分（顶半）负责处理最紧急的任务，比如读取数据或者清除中断标志。
任务属于底半操作：因为它们处理更复杂的逻辑，可能需要更多的时间，并且通常不需要像顶半那样的即时响应。底半操作使得系统可以在不影响中断响应的前提下，安排和执行这些较为复杂的任务。
通过这种顶半和底半的分工，RTOS能够保持良好的实时性，同时处理复杂的任务和事件，确保系统稳定高效地运行。
```

![image-20240405162537763](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404051625807.png)

![image-20240405162553200](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404051625260.png)

![image-20240405162558878](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404051625924.png)



代码示例：

```c
static uint32_t Timer1CallBackCount = 0;  //software timer1 callback count
static uint32_t Timer2CallBackCount = 0;  //software timer2 callback count

#if 1	//software timer callback task
	void Timer1CallBack(TimerHandle_t xTimer)
	{
		TickType_t tick_num1;

		Timer1CallBackCount++;  //increase by 1 for each callback

		tick_num1 = xTaskGetTickCount();  //geting the current tick count

		// HAL_GPIO_TogglePin(LED1_GPIO_Port,LED1_Pin);
		HAL_GPIO_WritePin(LED1_GPIO_Port,LED1_Pin,GPIO_PIN_SET);

		printf("Timer1CallBackCount:%d,tick_num1:%d\r\n",Timer1CallBackCount,tick_num1);
	}
    void Timer2CallBack(TimerHandle_t xTimer)
	{
		TickType_t tick_num2;

		Timer2CallBackCount++;  //increase by 1 for each callback

		tick_num2 = xTaskGetTickCount();  //geting the current tick count

		// HAL_GPIO_TogglePin(LED2_GPIO_Port,LED2_Pin);
		HAL_GPIO_WritePin(LED2_GPIO_Port,LED2_Pin,GPIO_PIN_SET);

		printf("Timer2CallBackCount:%d,tick_num2:%d\r\n",Timer2CallBackCount,tick_num2);
	}
#endif 


#if 1
	printf("strat to create the timer handler\r\n");
	static TimerHandle_t xtimer0 = NULL;	//sofware timer1 handler
	static TimerHandle_t xtimer1 = NULL;	//sofware timer2 handler
  #endif

#if 1
	/**
	 ***********************************************************
	 * @brief 创建 FreeRTOS 定时器的函数
	 * @param const char *const pcTimerName                  执行定时器的名称
	 * @param cconst TickType_t xTimerPeriodInTicks          定时器的周期
	 * @param UBaseType_t uxAutoReload                       表示定时器的自动重载标识，当其值为 pdTRUE 时，定时器在执行回调函数后会自动重新启动，否则只运行一次。
	 * @param void *const pvTimerID                          定时器的身份标识
	 * @param TimerCallbackFunction_t pxCallbackFunction     定时器到期时要调用的函数指针
	 * @return TimerHandle_t                                 定时器句柄
	 ***********************************************************
	 */
	printf("start to create the sofware timer task\r\n");
	xtimer0 = xTimerCreate("timer0",pdMS_TO_TICKS(1000),pdTRUE,(void *)1,Timer1CallBack);
	if(xtimer0 != NULL)
	{
		printf("create the timer0 success\r\n");
	}
	xTimerStart(xtimer0,0);

	xtimer1 = xTimerCreate("timer1",pdMS_TO_TICKS(2000),pdTRUE,(void *)2,Timer2CallBack);
	if(xtimer1 != NULL)
	{
		printf("create the timer1 success\r\n");
	}
	xTimerStart(xtimer1,0);

#endif
```

现象：

![image-20240405163032669](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404051630731.png)













