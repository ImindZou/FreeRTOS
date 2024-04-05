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





























