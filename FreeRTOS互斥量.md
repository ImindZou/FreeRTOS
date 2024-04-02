# FreeRTOS互斥量



[TOC]

## 一、优先级翻转问题

二值信号量也可以用于互斥型资源的访问控制，但是容易出现 **优先级翻转（Priority Inversion）问题。**

![image-20240401182812443](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011828514.png)

![image-20240401182818510](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011828562.png)

![image-20240401182830426](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011828484.png)

## 二、优先级继承

**在二值信号的功能上引入了优先级继承（Priority Inheritance）机制，这就是互斥量（Mutex）。**

![image-20240401183113412](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011831464.png)

**之前分析过这个互斥锁的原理，它的底层机制是调用了Yield来进行任务的调度，就是高低优先级的两个任务是用互斥锁进行绑定的，然后低优先级有高优先级任务运行所需要的资源，在高优先级运行后，它的互斥锁里边的机制会调用Yield来临时把CPU的执行权让步给带有解锁机制的一个低优先级任务，等待低优先级任务执行完成，资源给到高优先级任务后，高优先级任务成功调用，避免了中优先级抢占低优先级的CPU执行权，导致优先级翻转问题的发生。**

![image-20240401184008656](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011840721.png)

![image-20240401184017971](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011840021.png)

![image-20240401184300651](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404011843723.png)

![image-20240401203004193](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404012030333.png)



## 二、API介绍

![image-20240401203920169](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404012039227.png)

![image-20240401203925784](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404012039830.png)



使用互斥锁可以**比较好地**规避优先级翻转的问题，而二值信号量则难以规避。**（比较好~=完全）**

![image-20240402144549284](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404021445378.png)

```c
#if 1
/*^^^^^^^^  recursive mutex semaphore test ^^^^^^^^^*/
void recursive_mutex_test_entry(void)
{
	if(xSemaphoreTakeRecursive(MutexSemaphore,pdMS_TO_TICKS(200) == pdTRUE))
	{
		printf("recursive mutex semaphore take success\r\n");
		xSemaphoreGiveRecursive(MutexSemaphore);
		printf("recursive mutex semaphore give success\r\n");
	}
	else{
		printf("recursive mutex semaphore take failed\r\n");
	}
	HAL_Delay(500);
}
#endif

#if 1	//simulate proity reverse three task
void low_task_entry(void *p)
{
	while(1)
	{
		#if 1	//base one the recursive mutex semaphore
		if (xSemaphoreTakeRecursive(MutexSemaphore,pdMS_TO_TICKS(200)) == pdTRUE){
		taskENTER_CRITICAL();
		printf("low task give the recursive mutex semaphore...\r\n");	
		printf("low task processing some data...\r\n");
		taskEXIT_CRITICAL();
	    
		}
		else{
			printf("low task never give the recursive mutex semaphore...\r\n");
		}
		vTaskDelay(1000);
		#endif
		#if 0	//base one the mutex semaphore
		if(xSemaphoreTake(MutexSemaphore,pdMS_TO_TICKS(200)) == pdTRUE){
		taskENTER_CRITICAL();
		printf("low task give the mutex semaphore...\r\n");	
		printf("low task processing some data...\r\n");
		taskEXIT_CRITICAL();
		HAL_Delay(1000);
		printf("low task give the semaphore...\r\n");
		xSemaphoreGive(MutexSemaphore);
		}	//producter modual
		else{
			printf("low task never take the semaphore...\r\n");
		}
		vTaskDelay(1000);		
		#endif
		#if 0	//base on the binary semaphore
		// xSemaphoreTake(BinarySemaphore,portMAX_DELAY);
		if(xSemaphoreGive(BinarySemaphore) == pdTRUE){
			printf("low task give the semaphore...\r\n");
			HAL_Delay(1000);
		}
		else{
		printf("low task never give the semaphore...\r\n");
		}
		vTaskDelay(1000);
		
		#endif
	}
	
}

void middle_task_entry(void *p)
{
	while(1)
	{
		#if 1	//base one the mutex semaphore
		taskENTER_CRITICAL();
		printf("middle task exclusive the CPU...\r\n");	
		taskEXIT_CRITICAL();

		vTaskDelay(1000);
		// HAL_Delay(1000);
		#endif
		#if 0	//base on the binary semaphore
		printf("middle task exclusive the CPU...\r\n");	

		vTaskDelay(1000);
		#endif
	}
}

void high_task_entry(void *p)
{
	while(1)
	{
		#if 1	//base one the recursive mutex semaphore
		if(xSemaphoreTakeRecursive(MutexSemaphore,pdMS_TO_TICKS(200)) == pdTRUE){
		taskENTER_CRITICAL();
		printf("high task give the recursive mutex semaphore...\r\n");	
		printf("high task processing some data...\r\n");
		taskEXIT_CRITICAL();
		recursive_mutex_test_entry();
		}
		else{
			printf("high task never give the recursive mutex semaphore...\r\n");
		}
		vTaskDelay(1000);
		#endif
		#if 0	//base one the mutex semaphore
		//vTaskPrioritySet(xHandle,9);
		
		if(xSemaphoreTake(MutexSemaphore,pdMS_TO_TICKS(200)) == pdTRUE){	//consumer modual
		taskENTER_CRITICAL();
		printf("high task take the mutex semaphore...\r\n");	
		printf("set the low task pority...\r\n");
		taskEXIT_CRITICAL();
		//vTaskPrioritySet(xHandle,7);
		xSemaphoreGive(MutexSemaphore);
		}
		else{
			printf("high task never take the mutex semaphore...\r\n");
		}
		vTaskDelay(1000);
		#endif
		#if 0	//base on the binary semaphore
		// xSemaphoreGive(BinarySemaphore);
		if(xSemaphoreTake(BinarySemaphore,pdMS_TO_TICKS(200)) == pdTRUE){
		printf("high task take the semaphore...\r\n");	
		// xSemaphoreGive(BinarySemaphore);
		}
		else{
		printf("high task never take the semaphore...\r\n");
		}
		vTaskDelay(1000);
		#endif
	}
}

#endif
```

**更新了个递归互斥锁，同样是成对使用的API：**

![image-20240402151539676](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404021515731.png)

**总结主要看代码逻辑，二值信号量可以不用成对使用，互斥锁在胡哦去信号的时候就需要成对使用。**





































































