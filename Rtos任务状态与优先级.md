# FreeRtos的任务管理

[TOC]

# 一、多任务基本运行机制

**在单核处理器上，任何时刻只能有一个任务占用CPU并运行。** RTOS的任务调度使得多个任务对CPU实现了分时复用的功能。

**在一个时间片内会有一个任务占 用CPU并执行，**在一个时间片结束 时（实际是基础时钟定时器发生中 断时）进行任务调度

当多个任务的优先级不同时， FreeRTOS还会使用基于优先级的抢 占式任务调度方法

![image-20231205095446852](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312050954979.png)

## 1、任务的状态

![image-20231205085148357](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312050851525.png)

### **1、就绪状态（Ready）**

**任务被创建之后就处于就绪状态。**根据抢占式任务调度的特 点，任务调度的结果有以下几种情况：

-  如果当前没有其他处于运行状 态的任务，就绪的任务就进入 运行状态
- 如果就绪任务的优先级高于或 等于当前运行任务的优先级， 就绪的任务就进入运行状态
- 如果就绪任务的优先级低于当 前运行任务的优先级，就绪的 任务继续处于就绪状态

### **2、运行状态（Running）**

**占有CPU并运行的任务就处于运行状态。**处于运行状态的任 务在空闲的时候应该让出CPU的使用权。

处于运行状态的任务有两种 方法主动让出CPU的使用权

- 执行函数vTaskSuspend()进入 挂起状态
- 执行阻塞类函数进入阻塞状态 运行的任务交出CPU使用权 后，任务调度器可以使其他就绪 状态的任务进入运行状态。

### 3、阻塞状态（Blocked）

**阻塞状态就是任务暂时让出CPU的使用权，处于一种等待的 状态。**运行状态的任务可以调用两类函数进入阻塞状态。

- **时间延迟函数**，如 vTaskDelay()
- **用于进程间通讯的事件请 求函数，**如请求信号量的 函数xSemaphoreTake()

### 4、挂起状态（Suspended）

**挂起状态的任务就是暂停的任务，挂起状态的任务不参与调 度器的调度。**其他3种状态的任务都可以通过调用函数 vTaskSuspend()进入挂起状态。

**挂起后的状态不能自动退出 挂起状态，**需要在其他任务里 调用vTaskResume()函数使一 个挂起的任务变为就绪状态

## 2、 任务的优先级

-  在FreeRTOS中，每个任务都必须设置一个优先级
- 总的优先级个数由宏configMAX_PRIORITIES定义，缺省值是 56
- 优先级数字越低，优先级别越低**，所以最低优先级是0**，最高 优先级是（configMAX_PRIORITIES-1）
- 在创建任务时就必须为任务设置初始的优先级，在任务运行起 来后还可以修改优先级别
- 多个任务可以具有相同的优先级

## 3、空闲任务

osKernelStart()启动FreeRTOS的任务调度器时，**会自动创 建一个空闲任务（Idle task），空闲任务的优先级别为0。**

**与空闲任务相关的几个主要配置参数是：**

- **configUSE_TICK_HOOK**，是否使用空闲函数的钩子函数， 若配置为1，则可以利用空闲任务的钩子函数，系统空闲时做 一些处理	
- **configIDLE_SHOULD_YIELD**，空闲任务是否对同优先级的 用户任务主动让出CPU使用权，这会影响任务调度结果
- **configUSE_TICKLESS_IDLE**，是否在空闲任务时关闭基础 时钟，若设置为1，可实现系统的低功耗

# 二、 FreeRTOS的任务调度

## 1、任务调度方法概述

FreeRTOS有两种任务调度算法，**基于优先级的抢占式** （pre-emptive）调度算法和**合作式**调度（co-operative）算法， **其中抢占式调度算法又可以使用时间片或不使用时间片。**

| 调度方式               | 宏定义参数             | 取值 | 特点                                                         |
| ---------------------- | ---------------------- | ---- | ------------------------------------------------------------ |
| **抢占式(使用时间片)** | configUSE_PREEMPTION   | 1    | 基于优先级的抢占式任务调度，同优先级任务使用时间片轮流进入运行状态(默认模式) |
|                        | configUSE_TIME_SLICING | 1    |                                                              |
| 抢占式(不使用时间片)   | configUSE_PREEMPTION   | 1    | 基于优先级的抢占式任务调度，同优先级任务不使用时间片调度     |
|                        | configUSE_TIME_SLICING | 0    |                                                              |
| 合作式                 | configUSE_PREEMPTION   | 0    | 只有当运行状态的任务进入阻塞状态，或显式地调用要求执行任务调度的函数taskYIELD(FreeRTOS才会发生任务调度，选择就绪状态的高优先级任务进入运行状态 |
|                        | configUSE_TIME_SLICING | 任意 |                                                              |

###  1、使用时间片的抢占式调度方法

**FreeRTOS基础时钟的一个定时周期称为一个时间片（time  slice），默认值为1ms。**当使用时间片时，在基础时钟的每次中 断里会要求进行一次上下文切换（context switching），函数 xPortSysTickHandler()就是SysTick定时中断的处理函数

![image-20231205102525779](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312051025849.png)

这个函数的功能就是将**PendSV**（Pendable request for  system service，**系统可挂起服务请求**）中断的挂起标志位置位， 也就是发起上下文切换的请求，而进行上下文切换是在PendSV 的中断服务程序里完成的。

​									<img src="https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312051029923.png" alt="image-20231205102907855" style="zoom:;" />			

#### 任务运行时序图（使用时间片的抢占式调度方法）

- 横轴是时间轴
- 纵轴是系统中运行的任务
- 垂直方向的虚线表示发生任务切换的时间点
- 水平方向的实心矩形表示任务占据CPU处于运行状态的时间段
- 水平方向的虚线表示任务处于就绪状态的时间段
- 水平方向的空白段表示任务处于阻塞状态或挂起状态的时间段

![image-20231205103333589](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312051033662.png)

-  t1时刻开始是空闲任务在运行，这时候系统里没有其他任务处 于就绪状态。
-  在t2时刻进行调度时，Task1抢占CPU开始运行，因为Task1的 优先级高于空闲任务。
-  在t3时刻，Task1进入阻塞状态，让出了CPU的使用权，空闲 任务又进入运行状态。
- 在t4时刻，Task1又进入运行状态。
-  在t5时刻，更高优先级的Task2抢占了CPU开始运行，Task1进 入就绪状态。
- 在t6时刻，Task2运行后进入阻塞状态，让出CPU使用权， Task1从就绪状态变为运行状态。
-  在t7时刻，Task1进入阻塞状态，主动让出CPU使用权，空闲 任务又进入运行状态。

### 2、不使用时间片的抢占式调度方法

使用时间片的抢占式调度方法在每个SysTick中断里都进行 一次上下文切换请求，从而进行任务调度。而不使用时间片的 抢占式调度算法只在以下情况下才进行任务调度：

- 有更高级别的任务进入就绪状态时；
- 运行状态的任务进入阻塞状态或挂起状态时。

**所以，不使用时间片时，进行上下文切换的频率比使用时间 片时低，从而可降低CPU的负担。但是，对于同优先级的任务 可能会出现占用CPU时间相差很大的情况。**

#### 任务运行时序图（不使用时间片的抢占式调度方法）

![image-20231205105055460](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312051050563.png)

-  在t1时刻，空闲任务占用CPU，因为系统里没有其他任务处 于就绪状态。
- 在t2时刻，Task0进入就绪状态。但是Task0与空闲任务优先 级相同，且调度算法不使用时间片，不会让Task0和空闲任务 轮流使用CPU，所以Task0就保持就绪状态。
- 在t4时刻，高优先级的Task1抢占CPU。
- 在t5时刻，Task1进入阻塞状态，系统进行一次任务调度， Task0获得CPU的使用权。
- 在t6时刻，Task1再次抢占CPU，Task0又进入就绪状态。
-  在t7时刻，Task1进入阻塞状态，系统进行一次任务调度，空 闲任务获得CPU使用权。之后没有发生任务调度的机会，所 以Task0就一直处于就绪状态。

### 3、合作式任务调度方法

使用合作式任务调度方法时，**FreeRTOS不主动进行上下文 切换，而是运行状态的任务进入阻塞状态，或显式地调用 taskYIELD()函数让出CPU使用权时才进行上下文切换。**

任务不会发生抢占，所以也不使用时间片。

**函数taskYIELD()的作用就是主动申请进行一次上下文切换。**

#### 任务运行时序图（合作式任务调度方法）

![image-20231205105557561](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312051055632.png)

-  在t1时刻，低优先级的Task1处于运行状态。
- 在t2时刻，中等优先级的Task2进入就绪状态，但不能抢占CPU。
- 在t3时刻，高优先级的Task3进入就绪状态，但是也不能抢占CPU。
- 在t4时刻，Task1调用函数taskYIELD()主动申请进行一次上下文切 换，高优先级的Task3获得CPU使用权。
-  在t5时刻，Task3进入阻塞状态，就绪的Task2获得CPU的使用权。
- 在t6时刻，Task2进入阻塞状态，Task1又获得CPU使用权。

# 三、任务管理相关函数

## 1、相关函数概述

![image-20231205110021070](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312051100133.png)

### 文件task.h中还有几个常用的宏函数

![image-20231205110047843](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312051100907.png)

**这两个宏函数用于关闭和开启MCU的可屏蔽中断，用于界 定不受其他中断干扰的代码段。**

**只能关闭FreeRTOS可管理的中断优先级别，也就是参数 configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY定 义的最高优先级【下一章详细介绍】**

### 界定临界代码段

![image-20231205110237508](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312051102576.png)

这两个函数用于界定一个**临界（Critical）代码段**，**在临界代 码段内，任务不会被更高优先级的任务抢占，以保障代码执行 的连续性。**

### 主要函数功能说明

#### 1、创建任务

<u>CubeMX导出的代码中使用函数**osThreadNew()**创建任务， 它会根据任务的属性自动调用**xTaskCreate()**动态分配内存创建 任务，或调用**xTaskCreateStatic()**静态分配内存创建任务</u>

![image-20231206164255157](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312061642243.png)

动态分配内存创建任务的函数是xTaskCreate()

![image-20231205110516092](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312051105146.png)

##### 动态任务跟静态任务的区别

![image-20231205110839103](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312051108155.png)

#### 2、延时函数

延时函数vTaskDelay()用于延时一定节拍数，它会使当前任 务进入阻塞状态。

```c
void vTaskDelay( const TickType_t xTicksToDelay );
```

参数xTicksToDelay表示基础时钟的节拍数，一般会结合 宏函数pdMS_TO_TICKS()将一个时间转换为节拍数，然后 调用vTaskDelay()，如

```c
vTaskDelay(pdMS_TO_TICKS(500)); //延时500ms
```

#### 3、绝对延时函数

```c
void vTaskDelayUntil( TickType_t * const pxPreviousWakeTime, const TickType_t xTimeIncrement );
```

**<u>参数pxPreviousWakeTime表示上次任务唤醒时，基础时钟 计数器的值</u>；参数xTimeIncrement表示相对于上次唤醒时刻延 时的节拍数。**

**函数里每次会自动更新pxPreviousWakeTime的值，但是 在第一次调用时需要给一个初值**

![image-20231205111132801](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312051111854.png)

![image-20231205111157956](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312051111027.png)

一个周期内，高电平表示任务运行时间，低电平表示阻塞时 间，上跳沿表示唤醒时刻，下跳沿表示进入阻塞状态的时刻

```c
void AppTask_Function(void *argument)
{
TickType_t previousWakeTime=xTaskGetTickCount();
for(;;) /* 死循环 */
{
//死循环内的功能代码
//确保一个循环周期是1000ms
vTaskDelayUntil(&previousWakeTime, pdMS_TO_TICKS(1000));
}
}
```

# 4、 多任务编程示例一

示例Demo2_1MultiTasks的功能：设计两个任务，在任务1 里使LED1闪烁，在任务2里使LED2闪烁。

![image-20231205111505836](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312051115880.png)			

在FreeRTOS中创建两个任务

- 两个任务的优先级都设置为osPriorityNormal
- Task_LED2使用静态分配内存。使用静态分配内存时需要 设置作为栈空间的数组名称，以及控制块名称

![image-20231205111635944](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312051116009.png)

以静态分配内存方式创建任务时，需要定义Buffer Name 和Control Block Name

## 初始化代码

```c
//主程序
int main(void)
{
HAL_Init(); //HAL 初始化，复位所有外设，初始化Flash和SysTick
SystemClock_Config(); //系统时钟配置
/* 初始化所有已配置外设 */
MX_GPIO_Init(); //GPIO初始化，LED1和LED2的GPIO初始化
osKernelInitialize(); //RTOS内核初始化
MX_FREERTOS_Init(); //FreeRTOS对象初始化函数，在freertos.c中实现
osKernelStart(); //启动RTOS内核
/* We should never get here as control is now taken by the scheduler */
while (1)
{
}
}
```

```c
//1、任务的创建
//Task_LED1是动态分配内存，定义任务句柄和任务属性，任务属性里无需分配内存。
/* 任务 Task_LED1 的定义，动态分配内存方式 */
osThreadId_t Task_LED1Handle; //任务Task_LED1的句柄变量
const osThreadAttr_t Task_LED1_attributes = {//任务Task_LED1的属性
.name = "Task_LED1", //任务名称
.priority = (osPriority_t) osPriorityNormal, //任务优先级
.stack_size = 128 * 4 //栈空间大小， 128*4字节
};

```

```c
//2、任务的创建
//Task_LED2是静态分配内存，需要定义用作栈空间的数组，控制块变量。
typedef StaticTask_t osStaticThreadDef_t; //类型符号定义，
/* 任务 Task_LED2 的定义，静态分配内存方式 */
osThreadId_t Task_LED2Handle; //任务Task_LED2的句柄变量
uint32_t Task_LED2_Buffer[ 128 ]; //任务Task_LED2的栈空间数组
osStaticThreadDef_t Task_LED2_TCB; //任务Task_LED2的任务控制块
const osThreadAttr_t Task_LED2_attributes = { //任务Task_LED2的属性
.name = "Task_LED2", //任务名称
.stack_mem = &Task_LED2_Buffer[0], //栈空间数组
.stack_size = sizeof(Task_LED2_Buffer), //栈空间大小,字
.cb_mem = &Task_LED2_TCB, //任务控制块
.cb_size = sizeof(Task_LED2_TCB), //任务控制块大小
.priority = (osPriority_t) osPriorityNormal, //任务优先级
};

```

```c
//函数MX_FREERTOS_Init()中创建两个任务，都使用函 数osThreadNew()，这个函数内部会自动调用xTaskCreate() 或xTaskCreateStatic()
void MX_FREERTOS_Init(void)
{
/* 创建任务Task_LED1 */
Task_LED1Handle = osThreadNew( AppTask_LED1, NULL, &Task_LED1_attributes);
/* 创建任务 Task_LED2 */
Task_LED2Handle = osThreadNew( AppTask_LED2, NULL, &Task_LED2_attributes);
}
```

### 编写用户功能代码

为两个任务函数编写代码，并且对任务的属性稍微做些修 改，以测试带时间片的抢占式任务调度方法的特点，以及 vTaskDelay()和vTaskDelayUntil()等函数的使用方法。

####  **1、相同优先级的任务的执行**

两个任务的优先级相同，分别使LED1和LED2以不同的周 期闪烁。

注意，两个任务函数的for循环中都使用延时函数 HAL_Delay()，这个延时函数不会使任务进入阻塞状态，而是 一直处于连续运行状态。

```c
void AppTask_LED1(void *argument) //任务Task_LED1的入口函数
{
/* USER CODE BEGIN AppTask_LED1 */
for(;;)
{
HAL_GPIO_TogglePin(GPIOF, GPIO_PIN_9); //PF9=LED1
HAL_Delay(1000);
}
/* USER CODE END AppTask_LED1 */
}
void AppTask_LED2(void *argument) //任务Task_LED2的入口函数
{
/* USER CODE BEGIN AppTask_LED2 */
for(;;)
{
HAL_GPIO_TogglePin(GPIOF, GPIO_PIN_10); //PF10=LED2
HAL_Delay(500);
}
/* USER CODE END AppTask_LED2 */
}
```

下载到开发板运行，会发现LED1和LED2都能闪烁，两个 任务都被执行。

![image-20231205112437816](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312051124875.png)

由于两个任务具有相同的优先级，所以调度器使两个任务 轮流占用CPU。两个任务都是连续运行的，所以每个任务每次 占用CPU的时间是一个嘀嗒时钟周期。空闲任务总是无法获得 CPU的使用权。

#### 2、低优先级任务被“饿死”的情况

修改任务优先级，其他不做任何修改。下载运行，会 发现只有LED1闪烁，而LED2不闪烁。

![image-20231205112532202](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312051125252.png)

由于Task_LED2具有高优先级，且是连续运行的，不会进 入阻塞状态，所以低优先级的任务Task_LED1和空闲任务都无 法获得CPU的使用权，它们被“饿死”了。

![image-20231205113129944](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312051131099.png)

![rtos2](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312051129771.gif)

#### 3、高优先级任务主动进入阻塞状态

对前面的程序再稍作修改，保留一高一低的优先级设置， 将Task_LED1中的延时函数修改为vTaskDelay()，Task_LED2 中的延时函数仍然使用HAL_Delay()

```c
void AppTask_LED1(void *argument) //任务Task_LED1的入口函数
{
/* USER CODE BEGIN AppTask_LED1 */
for(;;)
{
HAL_GPIO_TogglePin(GPIOF, GPIO_PIN_9); //PF9=LED1
vTaskDelay(pdMS_TO_TICKS(1000));
}
/* USER CODE END AppTask_LED1 */
}
```

下载到开发板运行，会发现LED1和LED2都能闪烁，两个 任务都可以被执行。

![image-20231205113609318](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312051136365.png)

- 任务Task_LED1大部分时间处于阻塞状态
- 在任务Task_LED1处于阻塞状态时，任务Task_LED2可以 获得CPU的使用权
- 任务Task_LED1在延时结束后，可以抢占CPU的使用权
- 空闲任务还是无法获得CPU的使用权

![rtos1](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312051137207.gif)

#### 4、多任务系统一般的任务函数的设计

在使用抢占式任务调度方法时，要根据任务的重要性分配 不同的优先级，在任务空闲时要让出CPU的使用权，使其他 就绪状态的任务能获得CPU的使用权。

对前面的程序再稍作修改，

- Task_LED2的优先级为osPriorityBelowNormal
- 任务Task_LED2的优先级任然为osPriorityNormal
- 任务函数中都使用vTaskDelay()延时函数

下载运行，会发现LED1和LED2都能闪烁

![image-20231205114047635](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312051140688.png)

任务Task_LED1和Task_LED2大部分时间处于阻塞状态， 系统的空闲任务获得CPU的使用权。

![image-20231205114308022](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312051143079.png)

![rtos1](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312051143357.gif)

#### 5、使用vTaskDelayUntil()函数

如果要在任务函数的循环中实现严格的周期性，就应该使 用函数vTaskDelayUntil()。对上一步的程序稍做修改，在两个 任务的任务函数中使用函数vTaskDelayUntil()。

```c
void AppTask_LED1(void *argument) //任务Task_LED1的入口函数
{
/* USER CODE BEGIN AppTask_LED1 */
TickType_t ticks1=pdMS_TO_TICKS(1000); //时间（ms）转换为节拍数
TickType_t previousWakeTime= xTaskGetTickCount();
for(;;)
{
HAL_GPIO_TogglePin(GPIOF, GPIO_PIN_9); //PF9=LED1
vTaskDelayUntil(&previousWakeTime, ticks1); //循环周期1000ms
}
/* USER CODE END AppTask_LED1 */
}
```

使用函数vTaskDelayUntil()延时的时间是从任务上次被转入 运行状态开始的绝对时间。

 第一次执行时需要通过xTaskGetTickCount()获取基础时钟 计数器的当前值作为previousWakeTime的初值。

![image-20231205114531256](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312051145344.png)

![image-20231205114735778](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312051147905.png)

![rtos1](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312051148306.gif)

# 多任务编程示例二

项目设置：在本例中我们会用到OLED跟LED；在sys组件中设置TIM6为HAL基础时钟，这是因为要用到FreeRTOS，用TIM6作为它基础节拍

用到一个ADC1的IN5作为输入通道，读取单片机内部电压，下图为ADC1的设置

![image-20231206153048552](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312061621337.png)	

RTOS的设置

![image-20231206153120401](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312061621159.png)

```attention
下面栈空间大小并不是随意给出的，是在程序运行过程中通过统计栈空间的高水位值，给出一个比较安全合理的值。栈空间太小，会导致栈空间溢出，程序无法正常运行，栈空间太大，会浪费内存。要设置合理的栈空间内存，最好在调试阶段统计任务的高水位值。
```

#### 主程序，一般无多大变化

```c
#include "main.h"
#include "cmsis_os.h"
#include "adc.h"
#include "i2c.h"
#include "gpio.h"

void SystemClock_Config(void);
void MX_FREERTOS_Init(void);

int main(void)
{

  HAL_Init();

  SystemClock_Config();

  MX_GPIO_Init();
  MX_ADC1_Init();
  MX_I2C2_Init();
    
  osKernelInitialize();  /* Call init function for freertos objects (in freertos.c) */
  MX_FREERTOS_Init();

  osKernelStart();

  while (1)
  {

  }
}
```

#### FreeRTOS对象初始化

```c
#include "FreeRTOS.h"
#include "task.h"
#include "main.h"
#include "cmsis_os.h"

/* Private includes ----------------------------------------------------------*/
/* USER CODE BEGIN Includes */
#include "oled.h"
#include "keyled.h"
#include "adc.h"
/*USER CODE END PTD */

osThreadId_t Task_ADCHandle;
const osThreadAttr_t Task_ADC_attributes = {
  .name = "Task_ADC",
  .stack_size = 256 * 4,
  .priority = (osPriority_t) osPriorityNormal,
};
/* Definitions for Task_Info */
osThreadId_t Task_InfoHandle;
const osThreadAttr_t Task_Info_attributes = {
  .name = "Task_Info",
  .stack_size = 256 * 4,
  .priority = (osPriority_t) osPriorityLow,
};

void AppTask_ADC(void *argument);
void AppTask_Info(void *argument);

void MX_FREERTOS_Init(void); /* (MISRA C 2004 rule 8.1) */

void MX_FREERTOS_Init(void) {

  Task_ADCHandle = osThreadNew(AppTask_ADC, NULL, &Task_ADC_attributes);

  Task_InfoHandle = osThreadNew(AppTask_Info, NULL, &Task_Info_attributes);

}
```

#### 任务Task_ADC的实现

```c
/* USER CODE END Header_AppTask_ADC */
void AppTask_ADC(void *argument)
{
  /* USER CODE BEGIN AppTask_ADC */
	OLED_Clear();
	OLED_ShowString(0,0,"Task_ADC:",8);
	OLED_ShowString(0,1,"ADC_Value:",8);
	
	TickType_t previousWakeTime=xTaskGetTickCount();//获取滴答信号值
  /* Infinite loop */
  for(;;)
  {
		HAL_ADC_Start(&hadc1);
		if(HAL_ADC_PollForConversion(&hadc1,1000) == HAL_OK)
		{
			uint32_t val=HAL_ADC_GetValue(&hadc1);
			uint32_t Volt=3300*val;
			Volt=Volt>>12;
			OLED_ShowNum(100,0,Volt,3,16);
			LED2_Toggle();
		}
    vTaskDelayUntil(&previousWakeTime,pdMS_TO_TICKS(500));
  }
  /* USER CODE END AppTask_ADC */
}
```

#### 任务Task_Info的实现

```c
void AppTask_Info(void *argument)
{
  /* USER CODE BEGIN AppTask_Info */
//==============获取单个任务消息=========================	
//	TaskHandle_t taskHandle=xTaskGetCurrentTaskHandle();	//获取当前任务句柄
//	TaskHandle_t taskHandle=xTaskGetIdleTaskHandle();	//获取空闲任务句柄
//	TaskHandle_t taskHandle=xTaskGetHandle("AppTask_ADC");	//通过任务名称获取任务句柄
	TaskHandle_t taskHandle=Task_ADCHandle;		//直接获取任务句柄变量
	
	TaskStatus_t taskinfo;													//任务信息结构体
	BaseType_t getFreeStackSpace=pdTRUE;						//是否获取高水位值
	eTaskState taskState=eInvalid;									//当前任务状态
	vTaskGetInfo(taskHandle,&taskinfo,getFreeStackSpace,taskState);		//获取任务信息
//	vTaskGetInfo(taskHandle,&taskinfo,getFreeStackSpace,eInvalid);		//把参数改为eInvalid则函数返回的是任务所处的状态信息，但是不知道为什么在oled不显示信息，可能是函数编码问题

	
	taskENTER_CRITICAL();			//开始临界代码段，不允许任务调度
	OLED_ShowString(0,2,(char *)taskinfo.pcTaskName,8);
	OLED_ShowNum(60,2,taskinfo.xTaskNumber,3,8);
	OLED_ShowString(70,2,(char *)taskinfo.eCurrentState,8);//无状态显示
	OLED_ShowNum(80,2,taskinfo.uxCurrentPriority,2,8);
	OLED_ShowNum(95,2,taskinfo.usStackHighWaterMark,5,8);//获取任务高水位值，一般用于调试然后给出合理栈空间，避免内存溢出或浪费
	
//=============用uxTaskGetStackHighWaterMark()单独获取每个任务的高水位值=======
	taskHandle=xTaskGetIdleTaskHandle();	//获取空闲任务句柄
	UBaseType_t hwm=uxTaskGetStackHighWaterMark(taskHandle);
	OLED_ShowString(0,3,"idle task=",8);
	OLED_ShowNum(85,3,hwm,3,8);
	
	taskHandle=Task_ADCHandle;		//获取ADC任务句柄
	hwm=uxTaskGetStackHighWaterMark(taskHandle);
	OLED_ShowString(0,4,"ADC  task=",8);
	OLED_ShowNum(85,4,hwm,3,8);	
	
	taskHandle=Task_InfoHandle;		//获取Info任务句柄
	hwm=uxTaskGetStackHighWaterMark(taskHandle);
	OLED_ShowString(0,5,"Info task=",8);
	OLED_ShowNum(85,5,hwm,3,8);	
	
//===========获取内核消息=====================
	OLED_ShowString(0,6,"Kernel Info",8);
	UBaseType_t taskNum=uxTaskGetNumberOfTasks();//获取任务个数
	OLED_ShowNum(90,6,taskNum,2,8);
	
	taskEXIT_CRITICAL();		//结束临界代码段，重新允许任务调度
	
  /* Infinite loop */
	UBaseType_t loopCount=0;
  for(;;)
  {
		loopCount++;
		LED1_Toggle();
		vTaskDelay(pdMS_TO_TICKS(300));
		if(loopCount == 10)
			break;
  }
  /* USER CODE END AppTask_Info */
	OLED_ShowString(0,7,"Delete Self!",8);
	vTaskDelete(NULL);
}
```

#### 这段程序测试了以下几个功能。

- 使用函数vTaskGetInfo()获取一个任务信息。首先要获取任务句柄，程序用了多种方法获取任务句柄，即程序中的如下几行语句。

```c
//	TaskHandle_t taskHandle=xTaskGetCurrentTaskHandle();	//获取当前任务句柄
//	TaskHandle_t taskHandle=xTaskGetIdleTaskHandle();	//获取空闲任务句柄
//	TaskHandle_t taskHandle=xTaskGetHandle("AppTask_ADC");	//通过任务名称获取任务句柄
	TaskHandle_t taskHandle=Task_ADCHandle;		//直接获取任务句柄变量
```

只需要使用其中的一条语句获取任务句柄，其他语句需注释掉。另外，使用函数xTaskGetIdleTaskHandle()可能会发生编译错误，显示这个函数未定义。这是因为函数中有一个预编译条件，只有当参数	#define INCLUDE_xTaskGetIdleTaskHandle 为1时才能使用该函数。

![image-20231206160137789](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312061601860.png)

调用函数vTaskGetInfo()的语句如下：

```c
vTaskGetInfo(taskHandle,&taskinfo,getFreeStackSpace,taskState);		//获取任务信息
```

函数返回的任务信息存储在结构体变量taskInfo里。参数getFreeStackSpace确定是否获取任务栈空间的高水位值。若参数参数taskState指定为某种状态，就返回在这种状态下的参数，若taskState为eInvalid，就返回任务实际所处的状态信息。

vTaskGetInfo()获取的任务信息包括任务编号，名称，优先级等，还有栈空间的高水位值。高水位值表示任务栈空间的最小可用剩余空间，这个值越小，就说明任务栈空间越容易溢出。在本例调试程序的过程中我们发现，如果栈空间设置的太小，程序将无法正常运行。所以在程序调试阶段，检查任务的高水位值是非常必要的。

- 使用函数uxTaskGetStackHighWaterMark()获取一个任务的高水位值。用户通过uxTaskGetStackHighWaterMark()直接获取一个任务的高水位值（单位是字）。程序使用该函数分别获取了空闲任务，Task_ADC、Task_Info这三个任务高水位值并加以显示。
- 获取其他内核信息，函数uxTaskGetNumberOfTasks()可获取FreeRTOS中当前管理的任务数，本示例程序运行时，这个函数返回值是4，除了创建两个用户任务，还有系统自动创建的空闲任务和定时器服务任务。

![image-20231206160744700](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312061607751.png)

定义关键代码段。程序使用函数**taskENTER_CRITICAL(); 和 taskEXIT_CRITICAL();**定义了临界代码段。在OLED显示之前，使用**taskENTER_CRITICAL(); 定义了临界代码段的开始，这样会暂停函数调度**，使后面的代码段在执行时不会被其他任务打断；在进入for循环之前，使用函数	**taskEXIT_CRITICAL();	定义了代码段的结束，恢复任务调度。**

- 删除任务。任务函数的主体一般是个无限循环，在任务函数中不允许出现return语句。如果跳出了循环，需要在任务函数返回之前执行vTaskDelete(NULL)删除自己。
    - 程序中的for循环只执行了10次，使LED1闪烁，退出for循环后，打印字符串“Delete Self!”，执行vTaskDelete(NULL)删除了任务自己。

现象

![rots3](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312061621923.gif)









