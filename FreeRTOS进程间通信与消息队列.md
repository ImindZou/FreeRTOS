# FreeRTOS进程间通信与消息队列

[TOC]





## 一、进程间通信

任务与任务之间，或任务与ISR之间有时需要进行通讯同步，这称为进程间通信（IPC，Inter-process communication）

![image-20240331172836247](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202403311728343.png)

```c
//ISR与任务间的通讯示例

void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
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
	#if 1	//count semaphore test
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
}
^^^^^^^^^^^^^^^^^ 我是分割线 ^^^^^^^^^^^^^^^^^^^^^^6
void key1_press_entry(void *p)
{
	BaseType_t KeysRet = pdFALSE;	//PV return value
	while(1)
	{
		#if 0
		xSemaphoreTake(key1_sem, portMAX_DELAY);
		printf("gain the key1 sem\r\n");
		vTaskDelay(100);
		#endif
		#if 1
		KeysRet = (pdTRUE == xSemaphoreTake(key1s_sem, pdFALSE));	// 0
		if (pdTRUE == KeysRet)
		printf("gain the key1 sem,remaing count:%d part\r\n",uxSemaphoreGetCount(key1s_sem));
		else
		printf("the key1 sem is already full\r\n");

		vTaskDelay(1000);
		#endif
	}
}
```

传统编程中会使用标志变量，不断查询标志变量的状态；FreeRTOS提供了完善的IPC计数，包括队列、信号量、互斥量等，与C++的多线程同步类似。

![image-20240331173217692](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202403311732748.png)

图中的一个互斥量用来解决优先级反转问题已经实验过了，它的底层调用了YIELD，执行完逻辑后主动让出CPU

```c
#if 1	//simulate proity reverse three task
void low_task_entry(void *p)
{
	while(1)
	{
		xSemaphoreGive(MutexSemaphore);	//producter modual
		//taskENTER_CRITICAL();
		printf("low task give semaphore...\r\n");	
		//taskEXIT_CRITICAL();
		//vTaskDelay(1000);		
	}
}

void middle_task_entry(void *p)
{
	while(1)
	{
		printf("middle task exclusive the CPU...\r\n");	

		//vTaskDelay(1000);
	}
}

void high_task_entry(void *p)
{
	while(1)
	{
		//vTaskPrioritySet(xHandle,9);
		
		xSemaphoreTake(MutexSemaphore,portMAX_DELAY);	//consumer modual
		//taskENTER_CRITICAL();
		printf("high task wait for the semaphore..\r\n");
		printf("set pority...\r\n");
		//taskEXIT_CRITICAL();
		//vTaskPrioritySet(xHandle,7);
		
		//vTaskDelay(1000);
	}
}

#endif
```

![image-20240313193946742](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202403311735074.png)



1. **队列（Queue）**
    - 队列就是一个缓冲区，用于进程间传递少量的数据，所以也称消息队列。
2. **信号量（Semaphore）**
    - 分为二值信号量（Binary Semaphore）和计数信号量（Counting Semaphore）。二值信号量用于进程间同步，计数信号量一般用于共享资源的管理。（**上面第一例就是二值信号量跟计数信号量的两个代码例程）**
3. **互斥量（Mutex）**
    - 分为互斥量（Mutex）和递归互斥量（Recursive Mutex）。**互斥量具有优先级继承机制，可以减轻优先级反转问题**。（第二例就是优先级反转问题的解决方案，主要是内部机制）
4. **事件组（Event Group）**
    - 事件组适用于多个事件触发一个或多个任务的运行，可以实现事件的广播，还可以实现多个任务的同步运行。
5. **流缓冲区（Stream Buffer）和消息缓冲区（Message Buffer）**
    - 是FreeRTOS V10版本新增的给你，是一种优化进程间通信机制，专门应用于只有一个写入者（writer）和一个读取者（reader）的场景，还可以用于多核CPU的两个内核之间高效传输数据。
6. **任务通知（Task Notification）**
    - **使用任务通知不需要创建任何中间变量**，可以直接从任务向任务，或ISR向任务发送通知，传递一个通知值。
    - 任务通知可以模拟二值信号量、计数信号量、或长度为1的消息队列，使用任务通知效率更高，消耗内存更少。





## 二、队列的特点和基本操作

![image-20240331174810794](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202403311748864.png)

### 1、队列的创建和存储

队列创建时被分配固定个数的存储单元，每个存储单元存储固定大小的数据，进程间传递的数据就保存在队列存储单元里。

函数xQueueCreate（）以动态分配内存的方式创建队列，队列需要用的存储空间由FreeRTOS从堆空间自动分配。

![image-20240331175410364](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202403311754423.png)

-  uxQueueLength表示队列的长度，也就是存储单元的个数

-  uxItemSize是每个存储单元的字节数

- ucQueueType表示创建的对象的类型，有以下几种常数取值：

    - ```ABAP
        #define queueQUEUE_TYPE_BASE ( ( uint8_t ) 0U ) //队列
        #define queueQUEUE_TYPE_SET ( ( uint8_t ) 0U ) //队列集合
        #define queueQUEUE_TYPE_MUTEX ( ( uint8_t ) 1U ) //互斥量
        #define queueQUEUE_TYPE_COUNTING_SEMAPHORE ( ( uint8_t ) 2U ) //计数信号量
        #define queueQUEUE_TYPE_BINARY_SEMAPHORE ( ( uint8_t ) 3U ) //二值信号量
        #define queueQUEUE_TYPE_RECURSIVE_MUTEX ( ( uint8_t ) 4U ) //迭代互斥量
        ```

函数xQueueGenericCreate（）的返回值是QueueHandle_t 类型，是所创建队列的句柄。

```c
//调用函数xQueueCreate()的示例如下：
Queue_KeysHandle = xQueueCreate (5, sizeof(uint16_t));
//这行代码创建了一个具有5个存储单元的队列，每个单元占用sizeof(uint16_t)个字节，也就是2个字节。
```

![image-20240331180152025](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202403311801080.png)

```ABAP
数据项比较大（比如数组），复制数据会占用较大空间，怎么办？

传递数据的指针，通过指针再去读取原始数据
```



### 2、向队列写入数据

一个任务或ISR写入数据称为发送消息。队列是一个共享的存储区域，可以被多个进程写入，也可以被多个进程读取。

![image-20240331180559298](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202403311805355.png)

这些都是宏函数，他们跟上面一样，用宏修饰过的。

FIFO 和 LIFO 是两种不同的数据管理策略，用于控制数据的存储和检索顺序。它们的英文全称分别是：

- **FIFO**: First In, First Out（先进先出）
- **LIFO**: Last In, First Out（后进先出）

> ### FIFO（先进先出）

在 FIFO 策略中，最先被添加到队列中的数据项将是第一个被移除的。这种方式就像排队一样，先来的人先得到服务，然后离开。FIFO 是队列数据结构的典型行为模式。

> ### LIFO（后进先出）

与 FIFO 相对，LIFO 策略中最后被添加进去的数据项将是第一个被移除的。这种方式类似于堆叠盘子，你总是从顶部添加或移除盘子。LIFO 是栈数据结构的典型行为模式。

> ### 在 FreeRTOS 中的应用

在 FreeRTOS 中，队列（Queue）通常用于任务间的通信。`xQueueSendToBack()` 和 `xQueueSendToFront()` 函数允许你根据需要选择使用 FIFO 或 LIFO 策略向队列添加数据：

- 使用 `xQueueSendToBack()` 函数向队列后端发送数据时，遵循 FIFO 策略。这意味着数据将按照发送的顺序被接收，最先发送的数据最先被处理。
- 使用 `xQueueSendToFront()` 函数向队列前端发送数据时，采用 LIFO 策略。这使得最后发送的数据可以被优先处理，覆盖了队列的默认行为。

选择使用哪种方式取决于你的具体需求和场景。FIFO 是更常见的选择，因为它保证了公平性和顺序性，而 LIFO 在某些特定情况下，如需要快速访问最近添加的数据时，可能会更有用。

![image-20240331180941445](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202403311809503.png)

这两个函数在队列未满时能正常向队列写入数据，函数返回值为pdTRUE；如果队列已满，这两个函数不能再向队列写入数据，函数返回值为errQUEUE_FULL

![image-20240331181150109](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202403311811172.png)

![image-20240331181316493](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202403311813545.png)

### 3、从队列读取数据

可以在任务或ISR里读取队列的数据，称为**接收消息**。总是从队列头读取数据，读出后删除这个单元的数据，后面的数据前移。

![image-20240331181540385](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202403311815450.png)

### 4、队列操作相关函数

![image-20240331181733092](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202403311817141.png)

![image-20240331181753063](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202403311817118.png)

![image-20240331182016471](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202403311820526.png)

![image-20240331182022096](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202403311820151.png)

![image-20240331191116551](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202403311911663.png)

![image-20240331191227792](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202403311912970.png)

![image-20240331194834323](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202403311948532.png)

## 示例：

```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
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
	#if 1		//send message base on message list
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
	
	
	#if 0		 // 	simple semaphore trigger
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
	#if 1		//receive the message list info
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
```

效果：

![image-20240331215944503](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202403312159610.png)



































