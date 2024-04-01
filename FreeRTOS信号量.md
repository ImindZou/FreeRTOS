# FreeRTOS信号量

[TOC]



## 一、信号量和互斥量概述

信号量（Semaphore）和互斥量（Mutex）都基于消息队列的基本数据结构，但是信号量和互斥量又有一些区别。

信号量没有**优先级继承机制**，使用二值信号量时容易出现优先级翻转问题，**而互斥量可以减缓优先级翻转的问题**。

**二值信号量适用于进程间同步**，计数信号量适用于对多个共享资源的访问控制，互斥量适用于对一个资源的互斥访问控制。

![image-20240401154335846](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011543919.png)

### 二值信号量

- 二值信号量（Binary Semaphore）就是只有一个项的队列。**二值信号量就像是一个标志，适用于进程间同步的通信。**
- ![image-20240401154507940](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011545989.png)
- ADC中断读取ADC转换结果后写入数据缓冲区，并且释放（Give）二值信号量。
- 数据处理任务总是获取（Take）二值信号量。无效时，任务在阻塞状态等待；有效后，任务立即进入就绪状态参与任务调度，就可以读取缓冲区的数据并进行处理。

### 计数信号量

- 计数型信号量（Counting Semaphore）就是有固定长度的队列，每个项都是一个标志。**计数型信号量通常用于多个共享资源的访问控制。**
- ![image-20240401155243995](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011552058.png)
- 一个计数型信号量被创建时设置了初值4，这个值只是个计数值。
- 有一个客人进店时就获取（Take）信号量，计数信号量值减1。当计数信号量的值变为0时，再有客人要进店时就得等待。
- 如果有一个客人用餐结束了离开就是释放（Give）信号量，计数信号量的值加1，表示可用资源数量增加了1个。

### 互斥量

- 使用二值信号量可能会出现**优先级翻转的问题**。**互斥量引入了优先级继承机制，**可以减缓优先级翻转问题。
- ![image-20240401155258245](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011552295.png)
- 两个任务互斥性地访问串口，即在任务A访问串口时，其他任务不能访问串口。
- 互斥量相当于管理串口的一把钥匙。一个任务可以获取（Take）互斥量，获取互斥量后将独占对串口的访问，访问后要释放（Give）互斥量。
- 一个任务获得互斥量后，对自由进行访问时，其他想要获取互斥量的进程只能等待。

![image-20240401155517241](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011555294.png)

- **进程间同步**：一个进程只负责释放信号量，另一个进程只负责获取信号量

- **共享资源互斥访问**：一个任务对互斥量既有获取操作，也有释放操作。

    - 注意：互斥量不能再ISR函数中使用，因为互斥量具有任务的优先级继承机制，而ISR不是任务。另外，ISR函数中不能设置等待时间，而获取信号量时经常时需要等待的。

    - 
        您提到的问题确实是嵌入式系统和实时操作系统（RTOS）中的一个重要考虑点。在中断服务例程（ISR）中使用互斥量（Mutex）确实是不推荐的，主要因为两个原因：

        1. **优先级继承**：互斥量常常用来避免优先级翻转问题，通过优先级继承机制。但是，中断服务例程（ISR）通常不具有任务优先级的概念，它们执行于硬件中断的上下文中，而不是任务或线程的上下文。
        2. **阻塞操作**：在ISR中进行阻塞操作（如等待互斥量）是不合适的，因为这可能导致系统响应时间不可预测，影响系统实时性。ISR应该尽可能快地执行完毕，避免任何形式的等待。

        > ### 解决方案

        针对在ISR中访问共享资源的问题，有几种解决方案：

        1. **直接操作共享资源，使用中断屏蔽**：
            - 在访问共享资源之前禁用中断，访问完成后再恢复中断。
            - 这种方法适用于访问非常快且简单的资源，但要注意可能影响系统的实时性，因为它会暂时禁止所有中断。
        2. **使用二值信号量**：
            - 对于需要从ISR与任务交互的场景，可以使用二值信号量（或者称为“信号量”）。二值信号量可以从ISR中安全地释放（给出），允许一个任务等待（取得）该信号量。
            - 重要的是，从ISR释放信号量时不应该等待。
        3. **使用消息队列**：
            - ISR可以向消息队列发送消息，而任务可以等待这些消息。这种方法既避免了在ISR中等待，也提供了一种有效的机制来传递更复杂的数据或事件给任务。
        4. **使用软件定时器或者延时服务**：
            - 如果ISR的处理需要稍后进行且不急于立即完成，可以考虑使用软件定时器或者将事件排队到一个后台服务中处理。这样，ISR只负责触发定时器或发送事件，实际的处理逻辑在任务中完成。

        每种方法都有其适用场景，选择哪一种取决于具体需求、系统的实时性要求和资源限制。在设计系统时，应仔细考虑这些因素，以确保既满足实时性要求，又能保持系统的稳定性和可靠性。

### 递归互斥量

- **递归互斥量（Recursive Mutex）是一种特殊的互斥量，可以用于需要递归调用的函数中。**
- 一个任务在获得了互斥量之后就不能再获得互斥量，而一个任务获得递归互斥量之后，可以再次获得此递归互斥量，当然每次必须与一次释放配对使用。
- 递归互斥量同样不能再ISR函数中使用。

## 二、API介绍

![image-20240401161954565](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011619632.png)

![image-20240401162002708](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011620760.png)

![image-20240401162157428](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011621492.png)

### 1、二值信号量

接下来有的函数会进行深入剖析：

![image-20240401162450040](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011624107.png)

主要逻辑还是在这个if上面，大概就是判断队列是否为空，为空了创建一个新的，不为空在判断是是否有内存空间，有的话新PC指针插入数据，没有的话返回报错帧。然后更新队列。

![image-20240401163600237](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011636338.png)



![image-20240401164303989](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011643056.png)

![image-20240401164358123](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011643201.png)

![image-20240401164604473](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011646527.png)

![image-20240401170057000](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011700083.png)

![image-20240401170231734](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011702802.png)



![image-20240401171225027](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011712086.png)

![image-20240401171240832](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011712923.png)



![image-20240401172325160](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011723263.png)

后面逻辑基本都是一样，都是基于队列的。

![image-20240401172339757](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011723807.png)

```c
//信号量的创建最好都基于一个健壮性比较高的模板

//   key1_sem = xSemaphoreCreateBinary();
//   key2_sem = xSemaphoreCreateBinary();	
  	key1s_sem = xSemaphoreCreateCounting(5,0);
	if (NULL != key1s_sem){
		printf("create key1s_sem success\r\n");
	}
	/*Create the queue*/
	QueueHandle = xQueueCreate(QUEUE_LEN,QUEUE_SIZE);
	if (NULL != QueueHandle){
		printf("create the queue success\r\n");
	}

void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
	#if 1	//binary semaphore test
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
	#if 0	//count semaphore test
	portBASE_TYPE taskWoken = pdFALSE; 
	if (GPIO_Pin == KEY1_Pin)
	{
		xSemaphoreGiveFromISR(key1s_sem, &taskWoken);	//key1 release the sem
		printf("Key1 Pressed and release the sem\r\n");
		if(taskWoken == pdTRUE){
			portYIELD_FROM_ISR(taskWoken);
		}
	}
	if (GPIO_Pin == KEY2_Pin)
	{
		xSemaphoreTakeFromISR(key1s_sem, &taskWoken);	//key2 wait for the sem
		printf("Key2 Pressed and wait for the sem\r\n");
		if(taskWoken == pdTRUE){
			portYIELD_FROM_ISR(taskWoken);
		}
	}
	
	#endif
	#if 0		//send message base on message list
		BaseType_t xReturn = pdPASS;	//a create return info 
		uint32_t send_data1 = 1;
		uint32_t send_data2 = 2;
			if(GPIO_Pin == KEY1_Pin){
				//key1 pressed
				printf("send message send_data1\r\n");
				xReturn = xQueueSendFromISR(QueueHandle,	//queue handle
									&send_data1,	//send message content
									0);				//block time
				
				if (pdPASS == xReturn)
					printf("message1 send success\r\n");
				else
					printf("send message1 failed\n");
			}
			if(GPIO_Pin == KEY2_Pin){
				//key2 pressed
				printf("send message send_data2\r\n");   //queue handle
				xReturn = xQueueSendFromISR(QueueHandle,   //send message content
									&send_data2,    //block time
									0 );
				if (pdPASS == xReturn)
					printf("message2 send success\r\n");
				else
					printf("send message2 failed\r\n");
		}
	#endif
}



void key1_press_entry(void *p)
{
	
	
	#if 1		 // 	simple semaphore trigger
	while(1)
	{
		xSemaphoreTake(key1_sem, portMAX_DELAY);
		printf("gain the key1 sem\r\n");
		vTaskDelay(100);
	}
	#endif
	#if 0  	//  muilt semaphore trigger
	BaseType_t KeysRet = pdFALSE;	//PV return value
	while(1)
	{
		KeysRet = (pdTRUE == xSemaphoreTake(key1s_sem, pdFALSE));	// 0
		if (pdTRUE == KeysRet)
		printf("gain the key1 sem,remaing count:%d part\r\n",uxSemaphoreGetCount(key1s_sem));
		else
		printf("the key1 sem is already full\r\n");

		vTaskDelay(1000);
	}
	#endif
	#if 0		//receive the message list info
	BaseType_t xReturn = pdTRUE;	//define a create info return's value,default is pdTRUE
	uint32_t r_queue;
	while(1)
	{
		xReturn = xQueueReceive(QueueHandle,
								&r_queue,
								portMAX_DELAY);
		if(pdTRUE == xReturn)
		{
			printf("received %d\r\n",r_queue);
		}
		else
		{
			printf("receive error : 0x%lx\r\n",xReturn);
		}
	}
	#endif
}

void key2_press_entry(void *p)
{
	
	#if 1
	while(1)
	{
		xSemaphoreTake(key2_sem, portMAX_DELAY);
		printf("gain the key2 sem\r\n");
		vTaskDelay(100);
	}
		#endif
		#if 1		// queue gather send message
	

	#endif
	
}
```



### 2、计数信号量

代码示例在上面有

![image-20240401173218729](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011732798.png)

![image-20240401173311164](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011733225.png)

![image-20240401173318413](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011733471.png)

![image-20240401173325423](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011733478.png)

代码的效果：

![image-20240401174057397](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011740477.png)

![image-20240401174819960](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011748013.png)

![image-20240401174804131](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011748237.png)

获取信号量的实现原理：

**获取当前消息队列中阻塞信号的个数。放到一个叫uxReturn的变量并返回就可以了，他是一个输出类型的函数。**

![image-20240401175213420](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011752506.png)

![image-20240401175723880](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011757995.png)



























































