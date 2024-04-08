# FreeRTOS移植开源EasyLog流程

[TOC]

首先再搞明白串口log打印前有必要弄明白两个单词意思。

- **asynchronization**	[电] 异步，非同步化
- **synchronization**      [物] 同步；同时性   sɪŋkrənaɪˈzeɪʃ(ə)n/

两个单词比较容易混淆，务必记住，对接下来读代码有一定作用

![image-20240407212539255](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404072125349.png)

## 第一步：官网获取文件

[ImindZou/EasyLogger: An ultra-lightweight(ROM<1.6K, RAM<0.3k), high-performance C/C++ log library. | 一款超轻量级(ROM<1.6K, RAM<0.3k)、高性能的 C/C++ 日志库 (github.com)](https://github.com/ImindZou/EasyLogger)

## 第二步：将需要的文件，（就只有port跟src和inc这三个目录的文件，另外的一个目录是插件，Linux开发适用）添加进工程目录。

![image-20240407213433190](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404072134277.png)

## 第三步：修改文件配置，使其依赖适用于当前开发环境。

![image-20240407215952727](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404072159896.png)



多线程异步模式修改为同步模式。

![image-20240407224805056](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404072248134.png)

![image-20240407224821042](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404072248111.png)

模式配置完美落地，现在只棋差一步，改写log的输出方案。对其进行输出重定向，是操作系统给他上互斥锁，确保线程安全。

![image-20240407224931192](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404072249299.png)



## 最后一步：我们来到elog_port.c文件进行配置端口重定向

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





# EasyLogger 核心功能 API 说明

> ## 结合文档进行演示：

所有核心功能API接口都在[`\easylogger\inc\elog.h`](https://github.com/armink/EasyLogger/blob/master/easylogger/inc/elog.h)中声明。以下内容较多，可以使用 **CTRL+F** 搜索。

## 1、用户使用接口

### 1.1 初始化



初始化的 EasyLogger 的核心功能，初始化后才可以使用下面的API。

```c
ElogErrCode elog_init(void)
```



### 1.2 启动



**注意**：在初始化完成后，必须调用启动方法，日志才会被输出。

```c
void elog_start(void)
```



### 1.3 输出日志



所有日志的级别关系大小如下：

```c
级别 标识 描述
0    [A]  断言(Assert)
1    [E]  错误(Error)
2    [W]  警告(Warn)
3    [I]  信息(Info)
4    [D]  调试(Debug)
5    [V]  详细(Verbose)
```

不同级别的打印颜色各不相同，从上至下级别依次递减。

![image-20240408105720280](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404081057333.png)

#### 1.3.1 输出基本日志



所有级别的日志输出方法如下，每种级别都有两种简化方式，用户可以自行选择。

```c
#define elog_assert(tag, ...) 
#define elog_a(tag, ...) //简化方式1，每次需填写 LOG_TAG
#define log_a(...)       //简化方式2，LOG_TAG 已经在文件顶部定义，使用前无需填写 LOG_TAG

#define elog_error(tag, ...)
#define elog_e(tag, ...)
#define log_e(...)

#define elog_warn(tag, ...)
#define elog_w(tag, ...)
#define log_w(...)

#define elog_info(tag, ...)
#define elog_i(tag, ...)
#define log_i(...)

#define elog_debug(tag, ...)
#define elog_d(tag, ...)
#define log_d(...)

#define elog_verbose(tag, ...)
#define elog_v(tag, ...)
#define log_v(...)
```



| 参数 | 描述                                             |
| ---- | ------------------------------------------------ |
| tag  | 日志标签                                         |
| ...  | 不定参格式，与`printf`入参一致，放入将要输出日志 |

**技巧一** ：对于每个源代码文件，可以在引用 `elog.h` 上方，根据模块的不同功能，定义不同的日志标签，如下所示，这样既可直接使用 `log_x` 这类无需输入标签的简化方式 API 。

```c
//WiFi 协议处理(位于 /wifi/proto.c 源代码文件)
#define LOG_TAG    "wifi.proto"

#include <elog.h>

log_e("我是 wifi.proto 日志");
```

```c
//WiFi 数据打包处理(位于 /wifi/package.c 源代码文件)
#define LOG_TAG    "wifi.package"

#include <elog.h>

log_w("我是 wifi.package 日志");
```



```c
//CAN 命令解析(位于 /can/disp.c 源代码文件)
#define LOG_TAG    "can.disp"

#include <elog.h>

log_w("我是 can.disp 日志");
```

![tutieshi_640x360_4s](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404081042141.gif)

![image-20240408105519885](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404081055975.png)













### 1.7 过滤日志



#### 1.7.1 设置过滤级别



默认过滤级别为5(详细)，用户可以任意设置。在设置高优先级后，低优先级的日志将不会输出。例如：设置当前过滤的优先级为3(警告)，则只会输出优先级别为警告、错误、断言的日志。

```c
void elog_set_filter_lvl(uint8_t level)
```



| 参数  | 描述 |
| ----- | ---- |
| level | 级别 |

#### 1.7.2 设置过滤标签



默认过滤标签为空字符串(`""`)，即不过滤。当前输出日志的标签会与过滤标签做字符串匹配，日志的标签包含过滤标签，则该输出该日志。例如：设置过滤标签为WiFi，则系统中包含WiFi字样标签的（WiFi.Bsp、WiFi.Protocol、Setting.Wifi）日志都会被输出。

> 注：RAW格式日志不支持标签过滤

```c
void elog_set_filter_tag(const char *tag)
```



| 参数 | 描述 |
| ---- | ---- |
| tag  | 标签 |

#### 1.7.3 设置过滤关键词



默认过滤关键词为空字符串("")，即不过滤。检索当前输出日志中是否包含该关键词，包含则允许输出。

> 注：对于配置较低的MCU建议不开启关键词过滤（默认为不过滤），增加关键字过滤将会在很大程度上减低日志的输出效率。实际上当需要实时查看日志时，过滤关键词功能交给上位机做会更轻松，所以后期的跨平台日志助手开发完成后，就无需该功能。

```c
void elog_set_filter_kw(const char *keyword)
```



| 参数    | 描述   |
| ------- | ------ |
| keyword | 关键词 |

#### 1.7.4 设置过滤器



设置过滤器后，只输出符合过滤要求的日志。所有参数设置方法，可以参考上述3个章节。

```c
void elog_set_filter(uint8_t level, const char *tag, const char *keyword)
```



| 参数    | 描述   |
| ------- | ------ |
| level   | 级别   |
| tag     | 标签   |
| keyword | 关键词 |

#### 1.7.3 按模块的级别过滤



这里指的**模块**代表一类具有相同标签属性的日志代码。有些时候需要在运行时动态的修改某一个模块的日志输出级别。

```c
void elog_set_filter_tag_lvl(const char *tag, uint8_t level);
```



| 参数  | 描述           |
| ----- | -------------- |
| tag   | 日志的标签     |
| level | 设定的日志级别 |

参数 level 日志级别可取如下值：

| **级别**               | **名称**         |
| ---------------------- | ---------------- |
| ELOG_LVL_ASSERT        | 断言             |
| ELOG_LVL_ERROR         | 错误             |
| ELOG_LVL_WARNING       | 警告             |
| ELOG_LVL_INFO          | 信息             |
| ELOG_LVL_DEBUG         | 调试             |
| ELOG_LVL_VERBOSE       | 详细             |
| ELOG_FILTER_LVL_SILENT | 静默（停止输出） |
| ELOG_FILTER_LVL_ALL    | 全部             |

函数调用如下所示：

| 功能                           | 函数调用                                                   |
| ------------------------------ | ---------------------------------------------------------- |
| 关闭 `wifi` 模块全部日志       | `elog_set_filter_tag_lvl("wifi", ELOG_FILTER_LVL_SILENT);` |
| 开启 `wifi` 模块全部日志       | `elog_set_filter_tag_lvl("wifi", ELOG_FILTER_LVL_ALL);`    |
| 设置 `wifi` 模块日志级别为警告 | `elog_set_filter_tag_lvl("wifi", ELOG_LVL_WARNING);`       |

静态输出日志。动态好像没效果，不知道咋回事。

![image-20240408114032364](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404081140433.png)





动态过滤



![tutieshi_640x360_25s](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404081200179.gif)



























