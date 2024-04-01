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

**SysTick 的中断优先级为15，是最低的。系统在SysTick中断里发出任务调度请求，所以，只有在没有其他中断ISR运行的情况下，任务调度请求才会被及时响应。根据NVIC管理中断的特点，同等抢占优先级是不能发生抢占的，所以，即使有一个抢占优先级为15的中断ISR在运行，SysTick和PendSV的中断就无法被及时响应，也就是不会发生任务调度，任务函数也不会被执行。**（这句话比较关键，结合MCU硬件中断跟FreeRTOS任务优先级的概念，两者在执行权的概念上是相反的。）

### 说白了中断优先级小于5的都是不可屏蔽中断，系统会一直运行下去，运行的方式不同而已（要理清楚任务与中断的关系才能够真正理解这些微小的差别，不然失之毫厘，差之千里！！！仔细阅读第二节）

在示例中，指定了定时器TIM6作为HAL基础时钟源。从下图可以看到，TIM6中断的抢占优先级为0，也就是最高优先级，所以FreeRTOS无法屏蔽HAL的基础时钟中断。

当参数**configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY	 =  5 时**，系统各中断优先级如图所示，归纳要点如下：

![image-20231206212657932](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312062126996.png)

- HAL基础时钟（示例中是TIM6） 的中断优先级为0
-  PendSV和SysTick的中断优先 级为15，所以，只有在没有其 他中断需要处理的情况下才会 发生任务切换
- 中断分为2组，高优先级的一组中断不受FreeRTOS的管理， **称为FreeRTOS不可屏蔽中断**，低优先级的一组是FreeRTOS 可屏蔽中断，可以用函数**taskDISABLE_INTERRUPTS()**屏蔽 这些低优先级中断

注意：这里的不可屏蔽中断具体是指**FreeRTOS的不可屏蔽中断**，**不要与MCU硬件系统的不可屏蔽中断混淆**，STM32F4的中断向量表里有3个不可屏蔽中断，分别是**Reset**、**NMI**和**HardFault**。

## 二、任务与中断服务例程

### 1、任务与中断服务例程的关系

> ##### **MCU 的中断服务函数有中断优先级，有中断服务例程（ISR）；FreeRTOS 的任务有任务优先级，有任务函数。这两者的特点和具体区别具体如下。**

- 中断时 MCU 的硬件特性，由硬件事件或软件信号引起中断，运行那个 ISR 是由硬件决定的。中断额优先级数字越小，表示的优先级越高，所以中断最高优先级是0。
- FreeRTOS 的任务是一个纯软件的概念，与硬件系统无关。任务的优先级使开发者在软件事件中赋予的，**优先级数字越低，表示优先级越低，所以任务的最低优先级为 0 。**FreeRTOS的任务调度器决定哪个任务处于运行状态，FreeRTOS 在中断优先级为 15 的PenSV 中断里进行上下文切换，所以，只要中断 ISR 在运行，FreeRTOS 就无法进行任务切换。
- 任务只有在没有 ISR 运行的时候才能运行，**即使优先级最低的中断，也可以抢占高优先级的任务执行**，而任务不能抢占 ISR 的运行。【这一句需要重点解释】

> **注意对最后一条规则的理解。根据NVIC 管理中断的原则，同等抢占优先级的中断是不能发生抢占的。一个优先级为15的RTC唤醒中断是不能抢占优先级为15的SysTick 和 PendSV 中断的执行的，只是因为 SysTick 和 PendSV 中断的 ISR 运行时间很短，RTC唤醒中断的 ISR 才能被及时执行。但如果优先级为15的RTC 唤醒中断的 ISR 执行时间很长，那么 SysTick 和 PendSV 发生了中断也无法发生抢占，也就是说无法进行任务调度，任务函数也无法运行。**

#### 举个例子上面进行抽象理解

![image-20231207191401917](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312071914981.png)

- 在t2时刻发生了一个中断1，不管User Task的任务优先级有 多高，ISR1函数都会抢占CPU。ISR1执行完成后，User  Task才可以继续执行。
- 在t6时刻发生了中断2，ISR2函数同样抢占了CPU。**但是 ISR2占用CPU的时间比较长**，导致User Task执行时间变长， <u>从软件运行响应来说，可能就是软件响应变得迟钝了</u>。

> **从图中可以看出，在外部中断ISR执行时，就无法执行任务函数。所以如果一个 ISR 执行的时间比较长，任务函数就无法即使执行，FreeRTOS 也无法进行任务调度，就会导致软件响应变得缓慢 （这是从Cotx—M3权威指南里一直强调的一个结论，在执行完中断 ISR 后，程序还是会由 Handle 模式回退到 Thread 线程模式，并且该怎样还是怎样（<u>这也是中断保护现场的一个主要机制</u>），时间差多少而已，出错了的话就上硬Falut，或者其他Fault 这些都是其他中断，用于检查中断执行期间有无发生错误的。）**

**在实际的软件设计中，一般要尽量简化ISR函数的功能，使 其尽量少占用CPU的时间。**ISR函数一般只负责数据采集后收发， 将数据处理的任务放到任务函数里去执行。（这个例程跟我想的一样，任务函数就是处理数据的嘛，ISR 给个标志位就可以释放了，剩下的交给任务函数，随着ISR 的轻便，也给了任务函数留出了许多时间片可以去处理一些数据。）

### 2、中断屏蔽和临界代码段

一个任务在执行的时候，可能会被其他高优先级的任务抢占 CPU，也可能被任何一个中断 ISR 抢占 CPU 。（RTOS太惨了呀）在某些时候，任务的某段代码可能很关键，需要连续执行完，不希望被其他任务或中断打断，**这种程序代码段称为临界段（critical section）。**在FreeRTOS 中，有函数定义临界代码段，也可以屏蔽系统的部分中断。**这也可以理解为给特定区域的代码加上锁，跟互斥锁原理差不多，就一模一样好吧。**（一图胜千言）

![image-20231206212657932](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312071931708.png)

![image-20231207202138391](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312072021478.png)

> #### 看一下这些代码的底层实现原理

![image-20231207203350840](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312072033927.png)

从图中可以看出taskENTER_CRITICAL()   和  taskEXIT_CRITICAL() 使用了嵌套计数器，所以这一对函数可以嵌套使用。函数  taskDISABLE_INTERRUPTS() 和  taskENABLE_INTERRUPTS() 不能嵌套使用，只能成对使用。

![image-20231207204703739](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312072047818.png)

> **任务临界区使用了嵌套计数器，可以进行嵌套，这是属于软件互斥锁的一种机制；而中断屏蔽函数没有使用嵌套计数器，原因是中断属于硬件层面，它使用到了DSB，跟ISB两个关键字对一些重要数据进行了临时的保存。然后从任务程序切换到中断程序，中断执行完成后再切换回来任务程序，这时候把ISB跟DSB的数据弹出，恢复现场。**

### 3、在 ISR 中使用FreeRTOS API 函数

在中断的 ISR 里，有时会调用 FreeRTOS 的 API 函数，但是调用普通的API 函数可能会存在问题。例如，在 ISR 里调用 vTaskDelay() 就会出问题，因为**vTaskDelay() 会使任务进入阻塞状态**，而 ISR 根本就不是任务， ISR 运行的时候，也不能进行任务调度。

为此，FreeRTOS 的 API 函数分为两个版本：一个称为“**任务级**”，即普通名称的 API 函数；另一个称为“**中断级**”，即带后缀 “**FROM_ISR**” 的宏函数，中断级的 API 函数也称为**中断安全 API 函数**。

```ABAP
注意，在ISR中绝对不能使用任务级API函数，但是在任务函数 中可以使用中断级API函数。而且，在FreeRTOS不能管理的高优先 级中断的ISR里，连中断级API函数也不能用。
```

### 4、中断及其ISR设计原则

根据FreeRTOS 管理中断的特点，中断优先级和 ISR 程序设计应该遵循如下原则。

- 中断分为FreeRTOS不可屏蔽中断和可屏蔽中断，要根据中 断的重要性和功能为其设置合适的中断优先级。
- **ISR函数的代码应该尽量简短，将比较耗时的功能转移到任务里去实现。**
- **在可屏蔽中断的ISR函数里能调用中断级的FreeRTOS API 函数，绝对不能调用普通的FreeRTOS API函数。在不可屏 蔽中断的ISR函数里，不能调用任何的FreeRTOS API函数。（还得好好参考下表）**

    - 把重要数据放到任务里完成处理，处理完后等待中断发出信号，在可屏蔽中断函数中发送任务处理完后发出的信号完成信号之间的发送，在通过任务接收信号完成对应的动作。

        - ### 一种典型的处理模式

            你提到的处理模式，即在ISR中仅用于快速收发信号或设置标志，然后在任务中完成实际的数据处理，是一种常见且推荐的实践方法。这种方法利用了RTOS的任务调度和同步机制，既能保证及时响应中断，又能避免在中断上下文中进行复杂的处理逻辑。

            - **在ISR中发送信号**：通过调用中断安全的FreeRTOS API（如`xSemaphoreGiveFromISR`），快速标记一个事件发生，或者发送一些简单的数据。
            - **在任务中处理数据**：一个或多个任务等待这些信号，一旦信号到达，任务被唤醒来处理相应的数据或执行必要的动作。这样，复杂或耗时的操作就被移动到了任务中处理，有利于保持系统的响应性和稳定性。

            遵循这样的模式，可以使得系统设计既清晰又高效，同时充分利用了RTOS的任务调度和中断管理能力。


![image-20231206212657932](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312072105314.png)

## 三、任务和中断程序设计示例

使用RTC的周期唤醒中断，在此中断里读取RTC的当前时间 并在OLED上显示，在FreeRTOS中设计一个任务Task_LED1。通 过各种参数设置和稍微修改代码，测试和验证任务与重点的特点

> #### RTC设置		Wake Up Counter 设置为1 是2s唤醒一次

![image-20231208145152645](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312081451732.png)

> #### 任务设置

![](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312081452620.png)

> #### NVIC 管理中断设置（使用CubeMX的一个好处就是可以清楚地看到中断跟任务有没有正确地设置）

![image-20231208145315335](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312081453390.png)

> RTC 中断 ISR

```c
void HAL_RTCEx_WakeUpTimerEventCallback(RTC_HandleTypeDef *hrtc)
{
  RTC_TimeTypeDef sTime;
  RTC_DateTypeDef sDate;
  if(HAL_RTC_GetTime(hrtc,&sTime,RTC_FORMAT_BIN) == HAL_OK)
  {
    HAL_RTC_GetDate(hrtc,&sDate,RTC_FORMAT_BIN);
    /*调试HAl_GetTime()后必须用HAL_RTC_GetDate()解锁数据，才能连续更新日期和时间*/
    OLED_ShowNum(0,0,sTime.Hours,2,16);
    OLED_ShowNum(48,0,sTime.Minutes,2,16);
    OLED_ShowNum(96,0,sTime.Seconds,2,16);
    OLED_ShowString(32,0,":",16);
    OLED_ShowString(80,0,":",16);
  }
  // HAL_Delay(1000);
}
```

> ### FreeRTOS 任务处理函数

```c
void StartDefaultTask(void *argument)
{
  /* USER CODE BEGIN StartDefaultTask */
  /* Infinite loop */
  for(;;)
  {
    LED1_Toggle();
    vTaskDelay(pdMS_TO_TICKS(200));   
  }
  /* USER CODE END StartDefaultTask */
}
```

![rtos4](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312081517254.gif)

项目下载到开发板后，任务函数跟 RTC ISR 都能够按照期望运行，每隔2s在OLED上刷新时间，LED1也是有规律地闪烁。

在这个程序中，RTC唤醒中断的 ISR 处理速度很快，占用 CPU 时间很短，所以不影响任务函数的执行效果。其运行原理可以用下图前半段解释。

![image-20231207191401917](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312081513000.png)

### 各种特性测试

#### 1、中断的 ISR 长时间占用 CPU 对任务的影响

取消前面 RTC ISR 回调函数延迟代码的注释。这样 ISR 就能长达1s，远大于执行一次任务函数的200ms，中断的 ISR 就长时间占用 CPU 导致任务函数不能及时执行。

```ABAP
在中断 ISR 回调函数里，不能将HAL_Delay()替换为vTaskDelay（）,因为 ISR 里不能调用FreeRTOS的普通函数，回调函数是由中断 ISR 调用的
```

将构建完的项目下栽进开发板发现LED1闪烁的频率没以前快了，OLED刷新时间的间隔也变慢了，就算把 RTC 的 中断优先级修改为15也是一样，这正面应验了前面所说的 中断跟任务函数的特性，中断运行时，是轮不到任务函数说话的，除非运行到任务函数时，主动屏蔽中断，但屏蔽的优先级也是有限的。否则就乖乖等待中断结束后抢占CPU使用权，但还是会被中断的到来打断，所以这个函数就是有问题的。原理就不多赘述了。（继续no picture no say）

![image-20231206212657932](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312081527347.png)

![rtos5](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312081531898.gif)

#### 2、在任务中断中屏蔽中断

![image-20231207202138391](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312081534641.png)

在 FreeRTOS 中，我们可以使用上图的函数屏蔽一些可屏蔽中断。在本例中configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY 设置为5所以，中断优先级数字大于5-》15的都可以被屏蔽。

![image-20231208153834643](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312081538688.png)

为测试可屏蔽中断的效果，我们对RTC进行如下修改：

- 将RTC的中断
- 优先级设置为7
- 将RTC唤醒周期设置为1s，方法是将Wake Up Counter 修改为0
- 在RTC唤醒中断的回调函数中，将最后的延迟注释掉。

> ### 任务修改如下

```c
void StartDefaultTask(void *argument)
{
  /* USER CODE BEGIN StartDefaultTask */
  /* Infinite loop */
  for(;;)
  {
    taskDISABLE_INTERRUPTS();
    // taskENTER_CRITICAL();
    LED1_Toggle();
    // vTaskDelay(pdMS_TO_TICKS(200));   
    HAL_Delay(2000);
    // taskEXIT_CRITICAL();
    taskENABLE_INTERRUPTS();
  }
  /* USER CODE END StartDefaultTask */
}
```

这里有一点变化，就是任务在 Hal_Delay() 下是持续运行的，但在vTaskDelay() 下则会请求阻塞把 CPU 执行权交出。

构建项目后，将程序下载到开发板，发现OLED大约2s才刷新一次，而不是1s为周期，这是因为我们在任务函数里设置了中断屏蔽函数，**对中断进行了屏蔽，刚好中断优先级为7，而7又小于5，所以中断没有响应，你说巧不巧。**

下面再对函数进行修改，测试各种情况。

- 情况1：RTC中断优先级为1

    - ```c
        void StartDefaultTask(void *argument)
        {
          /* USER CODE BEGIN StartDefaultTask */
          /* Infinite loop */
          for(;;)
          {
            taskDISABLE_INTERRUPTS();
            // taskENTER_CRITICAL();
            LED1_Toggle();
            // vTaskDelay(pdMS_TO_TICKS(200));   
            HAL_Delay(2000);
            // taskEXIT_CRITICAL();
            taskENABLE_INTERRUPTS();
          }
          /* USER CODE END StartDefaultTask */
        }
        //我们发现OLED可以一秒刷新一次了，但是LED1的到响应的时机却不能确定了，这是因为RTC中断对于此时的 FreeRTOS 来说是不可屏蔽中断，1 小于 5 了，所以任务只能是借机行事，得到CPU执行权的机会并不大。只能但也只能在RTC中断的一个周期内，时间一到就被ISR抢占了，只能说任务太惨了。摆好的特性不用就会这样。有阻塞延迟函数为什么不用呢，更何况还有绝对时间延迟这种好东西
        ```

        

- 情况2：RTC中断优先级为7

    - ```c
        void StartDefaultTask(void *argument)
        {
          /* USER CODE BEGIN StartDefaultTask */
          /* Infinite loop */
          for(;;)
          {
            // taskDISABLE_INTERRUPTS();
            taskENTER_CRITICAL();
            LED1_Toggle();
            // vTaskDelay(pdMS_TO_TICKS(200));   
            HAL_Delay(2000);
            taskEXIT_CRITICAL();
            // taskENABLE_INTERRUPTS();
          }
          /* USER CODE END StartDefaultTask */
        }
        //我们发现效果跟第一次修改样，因为 taskENTER_CRITICAL() 和   taskEXIT_CRITICAL() 他们在底层设计的时候都调用了中断屏蔽函数，所说情况一样
        ```

- 情况3：RTC中断优先级为7，Hal_Delay() 换成 vTaskDelay() 

    - ```c
        void StartDefaultTask(void *argument)
        {
          /* USER CODE BEGIN StartDefaultTask */
          /* Infinite loop */
          for(;;)
          {
            // taskDISABLE_INTERRUPTS();
            taskENTER_CRITICAL();
            LED1_Toggle();
            vTaskDelay(pdMS_TO_TICKS(2000));   		//进入阻塞状态（交出CPU执行权），必然要打开中断进行任务调度
            // HAL_Delay(2000);
            taskEXIT_CRITICAL();
            // taskENABLE_INTERRUPTS();
          }
          /* USER CODE END StartDefaultTask */
        }
        
        ```

        构建项目后，下载到开发板运行测试，发现OLED 上得时间仍然每秒刷新一次，任务函数得临界代码里，延时2000ms对 RTC 的唤醒中断响应没有影响，而使用HAL_Delay() 是有影响的。（我们明明屏蔽了中断，还设置了临界区，为什么这个可屏蔽中断还能响应呢？）

        这是因为执行函数vTaskDelay（）会使当前任务进入阻塞状态，FreeRTOS 要进行任务调度。而任务的切换实在 **PenSV** 的中断里发生的，所以FreeRTOS 必须要打开中断，只要打开中断，RTC 唤醒中断的ISR就能及时执行。（这样临界代码区的作用就形同虚设）

        > **所以在使用屏蔽中断跟临界代码区的函数时，不能调用任务调度函数，如延时函数vTaskDelay（），或申请信号量等进行进程间同步的函数。因为发生任务调度时，就会打开中断，从而失去了中断屏蔽代码段或临界代码断的意义。**
        >
        > 那就直接在屏蔽层完成业务逻辑就可以了，上面的例子只是用了一个延迟函数来模拟一下业务逻辑运行的时间，在处理一些比较重要的数据的时候要求不能被打断，否则会导致系统错误，就可以直接用代码临界区，它底层有中断屏蔽，中断优先级大于等于5的都会被屏蔽掉，所以在设计ISR函数时要注意ISR的优先级。否则会导致数据处理不完整就被中断打断了，进而系统错误。

![image-20231207203350840](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312081612344.png)





















