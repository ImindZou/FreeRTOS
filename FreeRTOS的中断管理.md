# FreeRTOS的中断管理

[TOC]

**FreeRTOS的任务有优先级，MCU的硬件中断有中断优先级，这是两个不同的概念。FreeRTOS的任务甘丽要用到硬件中断，使用FreeRTOS时也可以使用硬件中断，但是硬件中断ISR的设计要注意一些设计原则**。在本章中，我们将详细介绍FreeRTOS与硬件中断的关系，以及如何正确使用硬件中断。

## 一、FreeRTOS与中断

中断时MCU的硬件特性，STM32 MCU的 NVIC 管理硬件中断。STM32F4使用4个位设置优先级分组策略，用于设置中断的抢占优先级和次优先级，优先级数字越小，优先级越高。每个中断有一个中断服务例程，即ISR，用于对中断做出响应。

FreeRTOS的运行要用到中断，在前面介绍FreeRTOS的运行原理时已经讲过，**FreeRTOS的上下文切换就是在PendSV中断里进行的，**FreeRTOS还需要一个i初始中产生滴答信号，CubeMX中启用FreeRTOS后，系统会自动对NVIC做一些设置，例如，示例Demo中对NVIC的设置如下。

![image-20231206203441437](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312062034521.png)（来自CM3权威指南)

![image-20231206203822693](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312062038762.png)

启动FreeRTOS后，中断优先级分组策略自动设置为4位全部用于前抢占优先级，所以抢占优先级编号是0到15.这个设置对于文件**FreeRTOSConfig.h**中的参数**configPRIO_BITS**，默认定义如下：

![image-20231206204323719](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312062043779.png)

这个参数在CubeMX中不能修改，固定为4，也就是分组策略使用4位抢占优先级。

在CubeMX中设置FreeRTOS的“config”参数时，有2个与中断相关的参数设置，如下图所示。

![image-20231206205056359](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312062050423.png)

```c
/* The lowest interrupt priority that can be used in a call to a "set priority"
function. */
/*表示中断的对地优先级数值。因为中断分组策略是4位全用于抢占优先级，所以这个数值为15*/
#define configLIBRARY_LOWEST_INTERRUPT_PRIORITY   15

/* The highest interrupt priority that can be used by any interrupt service
routine that makes calls to interrupt safe FreeRTOS API functions.  DO NOT CALL
INTERRUPT SAFE FREERTOS API FUNCTIONS FROM ANY INTERRUPT THAT HAS A HIGHER
PRIORITY THAN THIS! (higher priorities are lower numeric values. */
/*表示FreeRTOS可管理的最高优先级，默认值为5.也就是说，只有在中断优先级等于或大于中断ISR里，才可以调用FreeRTOS的
中断安全API函数，也就是带“FromISR”后缀的函数，使用taskDISABLE_INTERRUPTS()函数也只能屏蔽优先级大于或等于5的中断（5~15）*/

#define configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY 5
```

```attention
configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY绝 不允许设置为0，绝对不要在大于此优先级的中断ISR函数里调用 FreeRTOS的API函数，即使带“FromISR”的中断安全函数也不可以。
```

![image-20231206210445441](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312062104509.png)

**PendSV（Pendable request for system service，可挂起系统服务请求）**中断用于上下文切换，也就是在这个中断ISR里决定那个任务占用CPU。**PendSV中断占用优先级为15，也就是最低优先级。所以，只有在没有其他ISR运行的情况下，FreeRTOS才会执行上下文切换。**

**SysTick 的中断优先级为15，是最低的。系统在SysTick中断里发出任务调度请求，所以，只有在没有其他中断ISR运行的情况下，任务调度请求才会被及时响应。根据NVIC管理中断的特点，同等抢占优先级是不能发生抢占的，所以，即使有一个抢占优先级为15的中断ISR在运行，SysTick和PendSV的中断就无法被及时响应，也就是不会发生任务调度，任务函数也不会被执行。**

### 说白了中断优先级小于5的都是不可屏蔽中断，系统会一直运行下去，运行的方式不同而已

在示例中，指定了定时器TIM6作为HAL基础时钟源。从下图可以看到，TIM6中断的抢占优先级为0，也就是最高优先级，所以FreeRTOS无法屏蔽HAL的基础时钟中断。

![image-20231206212657932](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312062126996.png)

























