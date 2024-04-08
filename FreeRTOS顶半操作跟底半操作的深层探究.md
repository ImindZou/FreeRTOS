# FreeRTOS顶半操作跟底半操作的深层探究

[TOC]

## 问题1：

rtos中顶半操作跟底半操作的区别是什么，为什么说中断属于顶半操作，任务属于底半操作？

```ABAP
在实时操作系统（RTOS）的上下文中，顶半（Top Half）操作和底半（Bottom Half）操作是处理中断和任务的两种机制，它们共同协作以高效地处理系统中的事件和任务。这种划分主要用于管理中断服务和保持系统的实时响应性。
```

## 顶半操作（Top Half）


顶半操作主要指的是中断服务例程（ISR）的处理，这是中断发生时立即执行的部分。顶半处理通常需要快速执行并完成以下任务：

- 快速响应中断信号。
- 保存必要的状态或数据。
- 快速清除中断标志，以便硬件可以继续生成后续中断。
- 触发底半处理（例如，通过设置一个信号量、标志或发送消息）以进一步处理。
- 顶半处理的设计目标是尽量减少在中断上下文中花费的时间，以避免影响系统的实时性和响应其他中断的能力。



## 底半操作（Bottom Half）

底半操作是对中断处理的延续，但它在任务上下文中执行，而不是中断上下文。底半处理通常负责：

- 执行更复杂或耗时的数据处理。
- 完成由顶半触发的操作，如更新系统状态、处理接收到的数据等。
- 通过任务调度，底半操作可以被优先级管理，确保更重要的任务首先得到处理。
- 底半机制使得系统可以在不阻塞中断的情况下，安全地执行复杂或耗时的操作。



## 问题2：

为什么说中断属于顶半操作，任务属于底半操作？

- 中断属于顶半操作：因为中断的处理需要快速响应和处理，以释放中断线以便处理新的中断。中断处理的快速部分（顶半）负责处理最紧急的任务，比如读取数据或者清除中断标志。

- 任务属于底半操作：因为它们处理更复杂的逻辑，可能需要更多的时间，并且通常不需要像顶半那样的即时响应。底半操作使得系统可以在不影响中断响应的前提下，安排和执行这些较为复杂的任务。
- 通过这种顶半和底半的分工，RTOS能够保持良好的实时性，同时处理复杂的任务和事件，确保系统稳定高效地运行。



## 实际问题&解决方案：

问题： 再串口的机制中，串口接收数据的速度是在微妙级别的，而FreeRTOS的任务调度时间确实1000us也就是1ms，所以说如果我们在一个RTOS中设计的顶半操作和底半操作没有之间的连接时间太长就会造成数据的丢包，那我们要如何正确地设计一个能够更好地契合RTOS中顶半操作跟底操作的程序呢？

前两周我测试过，两个任务之间如果调用portYIELD_FROM_ISR（）；这个API则可以不用等待Tick中断的到来，而提前使任务从就绪状态进入运行状态，这个就绪到运行的周期大约60us，具体却决于需要执行业务逻辑的复杂度，双精度浮点型运算也只有200多us。这对于串口而已似乎绰绰有余，相比Tick中断的1000us。

但一般都是60us，因为顶半操作中，我们一般的设计理念都是快进快出，快速地把信号给到底半操作，在底半操作中依赖这个信号量去处理一些比较复杂耗时的业务逻辑。

代码示例：

```c
SemaphoreHandle_t uart_rx_sem；

uint8_t uart_rx_byte;
uint8_t uart_rx_buf[100] = {0};
uint8_t uart_rx_cnt = 0;


#if 1
//=======================    USART  =========================
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
	portBASE_TYPE taskWoken = pdFALSE;  // 0
	
	if(huart->Instance == USART1)
	{	
		
		//xSemaphoreGive(uart_rx_sem);
		xSemaphoreGiveFromISR(uart_rx_sem, &taskWoken);
		//printf("release the sem\r\n");
		
		HAL_UART_Receive_IT(&huart1, &uart_rx_byte, 1);
		//ITFlag = true;
		//portYIELD_FROM_ISR(taskWoken);	
		 if (taskWoken == pdTRUE){
		 	portYIELD_FROM_ISR(taskWoken);
		 }
	}
}
#endif

#if 1
void uart_rx_entry(void *p)
{
	while(1)
	{
		//xSemaphoreTakeFromISR(uart_rx_sem,&taskWoken);
		xSemaphoreTake(uart_rx_sem,portMAX_DELAY);
		uart_rx_buf[uart_rx_cnt] = uart_rx_byte;
		//printf("gain the sem\r\n");
		#if 0
		if(ITFlag == true)
		{
		HAL_GPIO_TogglePin(LED2_GPIO_Port, LED2_Pin);
		printf("receive the semphore toggle LED2\r\n");
			vTaskSuspend(xHandle2);
			printf("suspend the task2\r\n");
			HAL_Delay(8000);
			vTaskResume(xHandle2);
			printf("resume the task2\r\n");
		}
		else
		{
			printf("miss the semphore\r\n");
		}
		#endif
		#if 0
		printf("receive:[%02x]\r\n",uart_rx_buf[uart_rx_cnt]);	//ֻ�ܽ���һ���ֽ�
		#endif
		#if 1
		taskENTER_CRITICAL();

		if(uart_rx_buf[uart_rx_cnt] == 0x0D || uart_rx_buf[uart_rx_cnt] == 0x0A)
		{
			uart_rx_buf[uart_rx_cnt] = 0;	
			uart_rx_cnt = 0;				//����ʼ��Ϊ0
			
			 printf("string echo : %s\r\n",uart_rx_buf);
		}
		else
		{
		  uart_rx_cnt ++;
		}
		taskEXIT_CRITICAL();
		#endif
	}
}
#endif

//后面具体都是一些任务句柄就不过多赘述
```

现象：数据基本不会丢包

![image-20240405181648699](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404051816777.png)



这是对于串口来说的，底半操作跟顶半操作的设计哲学，按键的话就不需要这样快速响应了，因为我们人的速度不可能在几十几百微秒内对同一个按键按这么几下，还是得符合是否存在或合理的哲学理念。

这是一个生产者跟消费者模型的一个体现：

中断接收来自外部的数据，每接收一个字节的数据主动进行任务调度，将CPU执行权交到任务手上，然后再继续触发中断，知道遇到回车换行任务将来自中断的数据打包成字符串打印出来。这器件中断扮演的时生产者，任务扮演消费者，两者之间生产跟消费的关系大差不差，可以维持生产消费模型的正常运转。

而如果再顶半操作中没有显式地进行任务的调度，而是依赖系统中本身的Tick进行调度，就好比A一秒内吃了20个小零食而B刚打开背包打算开炫，两者之间差的不是一星半点，量化一下，这差距可想而知虽然不是指数关系，但倍数与单数的差距还是蛮大的。这就是生产跟不上消费的节奏，生产消费模型在此刻无法正常维持运作。

![image-20240405164712713](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404051818492.png)



需要注意的一点：

linux端输出\n,在windows下如何将其转换为\r\n。

不这么做永远都不会有打印出来的那一天，指的用X-Shell的。

![image-20240405183720312](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404051837390.png)



但是这么做任务的运行效率太慢了，任务需要来回不断地切换，非常浪费CPU的资源，有什么好的办法可以改进呢？

在中断中处理号数据，保存到全局变量的缓冲区中，最后处理完毕后释放信号量通知解锁任务，让任务一次性打印出缓冲区的内容，这样就减少了任务频繁切换。不然没接收一个字节都要进行顶半跟底半的调度，如果系统中还有更多的任务呢？那不得频繁打断，甚至数据直接没了。

```c
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
	#if 1
	portBASE_TYPE taskWoken = pdFALSE;
	if (huart->Instance == USART1)
	{
		HAL_UART_Receive_IT(&huart1, &uart_rx_byte, 1);
		uart_rx_buf[uart_rx_cnt] = uart_rx_byte;
		if(uart_rx_buf[uart_rx_cnt] == 0x0D || uart_rx_buf[uart_rx_cnt] == 0x0A)
		{
			uart_rx_buf[uart_rx_cnt] = 0;	
			uart_rx_cnt = 0;				
			//printf("string echo : %s\r\n",uart_rx_buf);
			xSemaphoreGiveFromISR(uart_rx_sem, &taskWoken);
		}
		else uart_rx_cnt ++;
		if(taskWoken == pdTRUE){
		portYIELD_FROM_ISR(taskWoken);
		}
	}	
	#endif
}
#endif


void uart_rx_entry(void *p)
{
	while(1)
	{
		#if 1
		xSemaphoreTake(uart_rx_sem,portMAX_DELAY);
		printf("%s\r\n",uart_rx_buf);
		#endif
	}
}
```

![image-20240405202406727](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404052024823.png)



通过这个实验，有种豁然开朗的感觉，感觉看这种东西的一个维度有上了一个层次，前面的实验跟这个实验对比起来确实可以说非常鸡肋。不过前面只是实验，也仅仅只是实验，后面要站在更高维度去设计这些东西。

对于前面的总结：频繁地去调用底半任务是有代价的，代价就是怎加了CPU无意义的工作流程。真正的高手都是顶半处理数据，底半直接用的。但普通的串口中断数据处理起来确实还是有点慢，而且占用CPU，而STM32刚好就有相关外设可以为CPU分担压力，DMA！！！再HAL中DMA的调用是自动开中断的。

> ## 总之RTOS任务设计的原则：
>
> - 提高任务的运行效率。
> - **做批量的处理不做少量的处理。**（如何提高效率？）



# RTOS中使用DMA方式接收串口数据

> **图文中的理解：**
>
> **DMA规定接收字节大小，达到后触发DMA传输完成中断，把数据从顶半拷贝到全局的一个缓冲区，再释放一个信号量，底半任务接收信号到后直接完成相应的任务。如果数据长度不够触发DMA传输完成中断的长度，再来个UART_IDLE中断，空闲中断，顾名思义串口在接收到数据后一段时间内没有新的数据到来自动触发的一个中断，出发后将数据送到全局缓冲区，然后释放信号，底半任务接收到后理解处理任务。两个中断确保了数据参数的稳定性。**



![image-20240405204421350](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404052044512.png)

> #### **问答：uart的idle中断是怎么个事？  IDLE是UART的专属，不是USART的！！！**

```apl
UART的空闲（Idle）中断是一个特定于串行通信的中断事件，它发生在UART接口检测到数据线空闲一定时间后。在很多微控制器和通信设备中，UART接口在接收到一系列数据后，如果在一定时间内没有新的数据到来，就会认为数据传输已经结束，此时产生一个空闲中断。

空闲中断特别适用于处理不定长数据包或在没有特定结束符的情况下接收数据。在很多实际应用场景中，发送方可能会发送不定长度的数据包，接收方难以预先知道每个数据包的具体长度。利用空闲中断，接收方可以在数据传输暂停时得到一个通知，进而处理已接收到的数据，而不必等待固定长度的数据或特定的数据包结束符。

> ### 工作原理

1. **数据接收**：UART接口持续接收来自发送方的数据。
2. **空闲检测**：在接收完一串数据后，UART硬件会开始监测线路状态。如果在预定的时间内（这个时间由UART的具体实现和配置决定）没有新的数据到达，接口就被认为是空闲的。
3. **中断触发**：一旦检测到空闲状态，UART硬件生成一个中断信号，通知CPU数据接收可能已经完成。
4. **数据处理**：中断服务程序（ISR）随后被执行，可以处理或者读取UART接口收到的数据。

> ### 应用场景

- **命令解析**：在命令-响应式通信协议中，发送方发送一个命令后可能会停止发送数据，等待响应。接收方可以利用空闲中断知道一条命令的结束，进行解析并给出响应。
- **流数据处理**：处理来自传感器或其他设备的流数据，这些数据的发送可能有间断性。空闲中断可以帮助确定数据流的边界。

> ### 注意事项

- **配置和使用**：正确配置UART空闲中断需要阅读和理解微控制器的具体手册，包括如何使能中断和如何设置空闲时间阈值。
- **实时性要求**：虽然空闲中断是一个有用的特性，但在高实时性要求的应用中，开发者需要仔细考虑处理中断的时间和方式，确保系统的响应时间符合要求。
```

接下来的处理会有两个顶半操作，一个是UART的IDLE另外一个是DMA的RX





