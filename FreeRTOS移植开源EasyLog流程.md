# FreeRTOS移植开源EasyLog流程

[TOC]

首先再搞明白串口log打印前有必要弄明白两个单词意思。

- **asynchronization**	[电] 异步，非同步化
- **synchronization**      [物] 同步；同时性   sɪŋkrənaɪˈzeɪʃ(ə)n/

两个单词比较容易混淆，务必记住，对接下来读代码有一定作用

![image-20240407212539255](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404072125349.png)

第一步：官网获取文件

[ImindZou/EasyLogger: An ultra-lightweight(ROM<1.6K, RAM<0.3k), high-performance C/C++ log library. | 一款超轻量级(ROM<1.6K, RAM<0.3k)、高性能的 C/C++ 日志库 (github.com)](https://github.com/ImindZou/EasyLogger)

第二步：将需要的文件，（就只有port跟src和inc这三个目录的文件，另外的一个目录是插件，Linux开发适用）添加进工程目录。

![image-20240407213433190](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404072134277.png)

第三步：修改文件配置，使其依赖适用于当前开发环境。

![image-20240407215952727](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404072159896.png)



多线程异步模式修改为同步模式。

![image-20240407224805056](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404072248134.png)

![image-20240407224821042](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404072248111.png)

模式配置完美落地，现在只棋差一步，改写log的输出方案。对其进行输出重定向，是操作系统给他上互斥锁，确保线程安全。

![image-20240407224931192](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404072249299.png)



最后一步：我们来到elog_port.c文件进行配置端口重定向

- 需求：1.elog通过串口打印输出	2.elog需要确保串口打印时候的线程安全
    - 基于以上两点我们可以利用串口重定向跟互斥锁的原理来实现，这对于有操作系统而言轻轻松松。

**Port中的API：**

```c
#include <elog.h>
#include "main.h"
#include "cmsis_os.h"					//锁的来源

extern SemaphoreHandle_t elog_mutex;	//互斥锁句柄，需要在elog每次初始化都创建互斥锁，确保每次调用都是基于线程安全的状态

/**
 * EasyLogger port initialize		端口初始化
 *
 * @return result
 */
ElogErrCode elog_port_init(void) {
    ElogErrCode result = ELOG_NO_ERR;

    /* add your code here */		//不是必须
    
    return result;
}

/**
 * EasyLogger port deinitialize		//复位初始化
 *
 */
void elog_port_deinit(void) {

    /* add your code here */

}

/**
 * output log port interface		//端口重定向
 *
 * @param log output of log
 * @param size log size
 */
void elog_port_output(const char *log, size_t size) {
    
    /* add your code here */
    int i = 0;
	
	for(i = 0; i < size; i++)
	{
		while((USART1->SR & 0x40) == 0);
		USART1->DR = log[i];
	}
}

/**
 * output lock					//输出锁定
 */
void elog_port_output_lock(void) {
    
    /* add your code here */
    xSemaphoreTake(elog_mutex, portMAX_DELAY);	//获取锁
}

/**
 * output unlock				//输出解锁
 */
void elog_port_output_unlock(void) {
    
    /* add your code here */
    xSemaphoreGive(elog_mutex);		//释放锁
}

/**
 * get current time interface		//获取时间
 *
 * @return current time
 */
const char *elog_port_get_time(void) {
    
    /* add your code here */		//后期备注
    
}

/**
 * get current process name interface	//获取当前进程的接口
 *
 * @return current process name
 */
const char *elog_port_get_p_info(void)
    
    /* add your code here */			//后期备注
    
}

/**
 * get current thread name interface	//获取线程接口名称
 *
 * @return current thread name
 */
const char *elog_port_get_t_info(void) {
    
    /* add your code here */			//后期备注
    
}
```

来到elog的初始化接口，再每次elog初始化的时候都对互斥锁进行创建。

```c
SemaphoreHandle_t elog_mutex;		//创建互斥锁句柄为全局变量

ElogErrCode elog_init(void){
//////省略
	{	//于代码块中创建一个互斥锁。
		elog_mutex = xSemaphoreCreateMutex();
		if(elog_mutex != false)
			printf("elog_mutex create success...%s\t %d\t\r\n",__FILE__,__LINE__);
	}
    
 //////省略
}
```

大功告成，要使用elog打印日志，不仅要对elog进行初始化，还要对elog进行开启；

![image-20240407231110054](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404072311181.png)



![image-20240407231541243](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404072315300.png)

```c
//例程代码
void watch_test1_entry(void *p)
{
	while (1)
	{
		//printf("watch test1 task run\r\n");
		log_a("watch_test1_task running... %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		log_d("watch_test1_task running... %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		log_e("watch_test1_task running... %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		log_i("watch_test1_task running... %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		log_w("watch_test1_task running... %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		vTaskDelay(200);
		vTaskDelay(200);
	}
}
void watch_test2_entry(void *p)
{
	while (1)
	{
		//printf("watch test2 task run\r\n");
		log_a("watch_test2_task running... %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		log_d("watch_test2_task running... %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		log_e("watch_test2_task running... %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		log_i("watch_test2_task running... %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		log_w("watch_test2_task running... %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);		
		vTaskDelay(200);
	}
}
void watch_test3_entry(void *p)
{
	while (1)
	{
		//printf("watch test3 task run\r\n");
		log_a("watch_test3_task running... %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		log_d("watch_test3_task running... %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		log_e("watch_test3_task running... %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		log_i("watch_test3_task running... %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		log_w("watch_test3_task running... %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);		
		vTaskDelay(200);
	}
}
```

![image-20240407231925116](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404072319187.png)

后期我们进行过滤处理的深入了解。这还不是elog的真正魅力。

















































































