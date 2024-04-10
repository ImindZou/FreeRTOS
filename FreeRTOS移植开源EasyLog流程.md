# FreeRTOS移植开源EasyLog流程

[TOC]

首先再搞明白串口log打印前有必要弄明白两个单词意思。

- **asynchronization**	[电] 异步，非同步化
- **synchronization**      [物] 同步；同时性   sɪŋkrənaɪˈzeɪʃ(ə)n/

两个单词比较容易混淆，务必记住，对接下来读代码有一定作用

> **说一下对宏定义的一个新的感悟，就在EsaryLogger这个环境而言，宏定义往往都是起到一个牵一发而动全身的作用，修改一个小小的宏会导致开启全局的宏，因为大部分任务的切换都会用到相同的句柄，而句柄又在任务切换之间的某个上下文更改了，这也是导致牵一发而动全身的因素。**

![img](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404101449913.webp)

> 实际开发中需要更加细粒度地去考虑实际上的问题，有时候一些特定的log会特别多，光是等级过滤已是没用，这时候就需要更加细粒度地进行动态过滤，esaylogger也提供了相应的AP**I，对特定标签进行过滤。**

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

## 1.1 初始化



初始化的 EasyLogger 的核心功能，初始化后才可以使用下面的API。

```c
ElogErrCode elog_init(void)
```



## 1.2 启动



**注意**：在初始化完成后，必须调用启动方法，日志才会被输出。

```c
void elog_start(void)
```



## 1.3 输出日志



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

### 1.3.1 输出基本日志



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

elog_set_filter_tag_lvl("wifi.package", ELOG_LVL_ERROR);	
//这里结合我自己的见解进行配置，如果第一第三有效果的话当然是最好的，我是没效果采用了这种方式。这种方法经过实验确实还是比较必要的，因为他要配合标签来进行宏编译时候的初始化，当然不同文件的结果可能不同。


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



**技巧二** ：为了实现按照模块、子模块作用域来限制日志输出级别的功能，可以按照下面的方式，在模块的头文件中定义以下宏定义：

```c
/**
 * Log default configuration for EasyLogger.
 * NOTE: Must defined before including the <elog.h>			记录默认的log配置，记得在<elog.h>之前定义
 */
#if !defined(LOG_TAG)
    #define LOG_TAG                    "xx"
#endif
#undef LOG_LVL
#if defined(XX_LOG_LVL)
    #define LOG_LVL                    XX_LOG_LVL
#endif
```

**XX 是模块名称的缩写**，该段内容务必定义在 `elog.h` 之前，否则失效；这样做的 **好处** 是，如果模块内的源文件没有定义 TAG ，则会自动引用该段内容中的定义的 TAG 。同时可以在 头文件中可以配置 `XX_LOG_LVL` ，这样只会输出比这个优先级高或相等级别的日志。当然 XX_LOG_LVL 这个宏也可以不定义，此时会输出全部级别的日志，定义为 ASSERT 级别，就只剩断言信息了。 此时我们就能够实现 **源文件->子模块->模块->EasyLogger全局** 对于其中任何环节的日志配置及控制。调试时想要查看其中任何环节的日志，或者调整其中的某个环节日志级别，都会非常轻松，极大的提高了调试的灵活性及效率。

> 这让我强烈感觉我自己摸索的方法是巧合，碰巧弄出来了，但也不能这么说，这不是设计上的闭环，可能这说的知识一种策略，我用的是另外一种策略，然后刚好可以达到同样目的地，经过实验不是黑魔法，计算机里也没有黑魔法。有的只是认知差。

### 1.3.2 输出 RAW 格式日志



```c
void elog_raw(const char *format, ...)
```



| 参数   | 描述                       |
| ------ | -------------------------- |
| format | 样式，类似`printf`首个入参 |
| ...    | 不定参                     |



## 1.4 断言



### 1.4.1 使用断言



EasyLogger自带的断言，可以直接用户软件，在断言表达式不成立后会输出断言信息并保持`while(1)`，或者执行断言钩子方法，钩子方法的设定参考 [`elog_assert_set_hook`](https://github.com/armink/EasyLogger/blob/master/docs/zh/api/kernel.md#114-设置断言钩子方法)。

```c
#define ELOG_ASSERT(EXPR)
#define assert(EXPR)   //简化形式
```



| 参数 | 描述   |
| ---- | ------ |
| EXPR | 表达式 |

### 1.4.2 设置断言钩子方法



默认断言钩子方法为空，设置断言钩子方法后。当断言`ELOG_ASSERT(EXPR)`中的条件不满足时，会自动执行断言钩子方法。断言钩子方法定义及使用可以参考上一章节的例子。

```c
void elog_assert_set_hook(void (*hook)(const char* expr, const char* func, size_t line))
```

> 这是一个函数指针，需要用户自己定义，下面1.5有一个写入Flash用到断言的示例，这里没有用到Flash也没有配置开启Flash的宏，就不去实验了。

| 参数 | 描述         |
| ---- | ------------ |
| hook | 断言钩子方法 |

## 1.5 日志输出控制



### 1.5.1 使能/失能日志输出



```c
void elog_set_output_enabled(bool enabled)
```

> 这个可以单独设置日志的输出状态

| 参数    | 描述                    |
| ------- | ----------------------- |
| enabled | true: 使能，false: 失能 |

### 1.5.2 获取日志使能状态



```c
bool elog_get_output_enabled(void)
```



### 1.5.3 使能/失能日志输出锁



默认为使能状态，当系统或MCU进入异常后，需要输出异常日志时，就必须失能日志输出锁，来保证异常日志能够被正常输出。

```c
void elog_output_lock_enabled(bool enabled)
```

> 实验日志：
>
> ```c
> //Enable/disable  logger lock that determines all the area loggers output!!!	
> elog_set_output_enabled(true);
> //If i make it false then the system logger is disable,if not the system logger is print nomally
> ```
>
> ![image-20240409191559712](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404091915912.png)

| 参数    | 描述                    |
| ------- | ----------------------- |
| enabled | true: 使能，false: 失能 |

例子：

```c
/* EasyLogger断言钩子方法 */
static void elog_user_assert_hook(const char* ex, const char* func, size_t line) {
    /* 失能异步输出方式（异步输出模块自带方法） */
    elog_async_enabled(false);
    /* 失能日志输出锁 */
    elog_output_lock_enabled(false);
    /* 失能 EasyLogger 的 Flash 插件自带同步锁（Flash 插件自带方法） */
    elog_flash_lock_enabled(false);
    /* 输出断言信息 */
    elog_a("elog", "(%s) has assert failed at %s:%ld.\n", ex, func, line);
    /* 将缓冲区中所有日志保存至 Flash （Flash 插件自带方法） */
    elog_flash_flush();
    while(1);
}
```



## 1.6 日志格式及样式



### 1.6.1 设置日志格式



每种级别可对应一种日志输出格式，日志的输出内容位置顺序固定，只可定义开启或关闭某子内容。可设置的日志子内容包括：级别、标签、时间、进程信息、线程信息、文件路径、行号、方法名。

> 注：默认为 RAW格式

```c
void elog_set_fmt(uint8_t level, size_t set)
```

> 实验日志：
>
> 这里之前从未打开过标签的开关，在这里抱着试试看的心态，是能了一下标签开关，没想到标签就在这里看到了，看来这些 **级别、标签、时间、进程信息、线程信息、文件路径、行号、方法名。**都是需要自己手动开启的。
>
> 另外的经验就是：标签开关当在一个文件内打开后，就会影响到所有有配置标签的文件，都会一并打印出来，这些对于自己调试的时候效果非常好，对于出产品则需要关闭全局的标签号，甚至从配置文件中关闭再作全局检查是否关闭标签输出。这样保证产品不会被抄，确保封闭性。
>
> ![image-20240409192514957](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404091925020.png)
>
> ```c
> #if 1 //##1.6 elog format(layout) and style
> 	elog_set_fmt(ELOG_LVL_INFO,ELOG_FMT_TAG|ELOG_FMT_FUNC|ELOG_FMT_TIME);
> 	elog_set_filter_tag_lvl("ET",ELOG_LVL_INFO);
> 	log_a("elogtask1_test ET...\r\n");	
> 	log_e("elogtask1_test ET...\r\n");	
> 	log_w("elogtask1_test ET...\r\n");	
> 	log_i("elogtask1_test ...\r\n");	
> 	log_d("elogtask1_test ...\r\n");	
> 	log_v("elogtask1_test ...\r\n");
> 	#endif
> ```

| 参数  | 描述     |
| ----- | -------- |
| level | 级别     |
| set   | 格式集合 |

例子：

```c
/* 断言：输出所有内容 */
elog_set_fmt(ELOG_LVL_ASSERT, ELOG_FMT_ALL);
/* 错误：输出级别、标签和时间 */
elog_set_fmt(ELOG_LVL_ERROR, ELOG_FMT_LVL | ELOG_FMT_TAG | ELOG_FMT_TIME);
/* 警告：输出级别、标签和时间 */
elog_set_fmt(ELOG_LVL_WARN, ELOG_FMT_LVL | ELOG_FMT_TAG | ELOG_FMT_TIME);
/* 信息：输出级别、标签和时间 */
elog_set_fmt(ELOG_LVL_INFO, ELOG_FMT_LVL | ELOG_FMT_TAG | ELOG_FMT_TIME);
/* 调试：输出除了方法名之外的所有内容 */
elog_set_fmt(ELOG_LVL_DEBUG, ELOG_FMT_ALL & ~ELOG_FMT_FUNC);
/* 详细：输出除了方法名之外的所有内容 */
elog_set_fmt(ELOG_LVL_VERBOSE, ELOG_FMT_ALL & ~ELOG_FMT_FUNC);
```

> 实验日志：对于输出的参数我们该如何选择？
>
> ![image-20240409194051143](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404091940231.png)

### 1.6.2 使能/失能日志颜色



日志颜色功能是将各个级别日志按照颜色进行区分，默认颜色功能是关闭的。日志的颜色修改方法详见《EasyLogger 移植说明》中的 `设置参数` 章节。

```c
void elog_set_text_color_enabled(bool enabled)
```



| 参数    | 描述                    |
| ------- | ----------------------- |
| enabled | true: 使能，false: 失能 |

#### 1.6.3 查找日志级别



在日志中查找该日志的级别。查找成功则返回其日志级别，查找失败则返回 -1 。

> **注意** ：使用此功能时，请务必保证所有级别的设定的日志格式里均已开启输出日志级别功能，否则会断言错误。

```c
int8_t elog_find_lvl(const char *log)
```

> 实验日志：// failed!!! Not at all  用不了一点啊，断言了，CPU直接让给优先级最高的任务了。
>
> 老老实实每个任务进行必要打印就可以了，太多太杂太乱，难以控制。就直接断言了。

| 参数 | 描述               |
| ---- | ------------------ |
| log  | 待查找的日志缓冲区 |

### 1.6.4 查找日志标签



在日志中查找该日志的标签。查找成功则返回其日志标签及标签长度，查找失败则返回 NULL 。

> **注意** ：使用此功能时，首先请务必保证该级别对应的设定日志格式中，已开启输出日志标签功能，否则会断言错误。其次需保证设定日志标签中 **不能包含空格** ，否则会查询失败。

```c
const char *elog_find_tag(const char *log, uint8_t lvl, size_t *tag_len)
```



| 参数    | 描述               |
| ------- | ------------------ |
| log     | 待查找的日志缓冲区 |
| lvl     | 待查找日志的级别   |
| tag_len | 查找到的标签长度   |





## 1.7 过滤日志



### 1.7.1 设置过滤级别



默认过滤级别为5(详细)，用户可以任意设置。在设置高优先级后，低优先级的日志将不会输出。例如：设置当前过滤的优先级为3(警告)，则只会输出优先级别为警告、错误、断言的日志。

```c
void elog_set_filter_lvl(uint8_t level)
```

> 实验日志：这个函数它的原型决定了它比较适合用在ZNS中，用于动态设置过滤等级
>
> ```c
> elog_set_filter_lvl(ELOG_LVL_ASSERT);
> log_a("hello world %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
> log_e("hello world %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
> log_w("hello world %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
> log_i("hello world %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
> log_d("hello world %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
> log_v("hello world %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
> ```
>
> ![image-20240410142958978](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404101429085.png)

## 动态过滤：

> > #### 在配置文件采用默认的过滤策略，配合ZNS，自定义一个输入类型的API，传参值要从char 类型转换为 int 类型，给到elog_set_filter_lvl(level)中去，只要调用ZNS就可以动态地开启查看不同等级的log了。
>
> ![image-20240409162207617](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404091622675.png)
>
> ![tutieshi_640x360_25s](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404081200179.gif)

| 参数  | 描述 |
| ----- | ---- |
| level | 级别 |

### 1.7.2 设置过滤标签



默认过滤标签为空字符串(`""`)，即不过滤。当前输出日志的标签会与过滤标签做字符串匹配，日志的标签包含过滤标签，则该输出该日志。例如：设置过滤标签为WiFi，则系统中包含WiFi字样标签的（WiFi.Bsp、WiFi.Protocol、Setting.Wifi）日志都会被输出。

> 注：RAW格式日志不支持标签过滤
>
> ```c
> 
> ```
>
> 

```c
void elog_set_filter_tag(const char *tag)
```



| 参数 | 描述 |
| ---- | ---- |
| tag  | 标签 |

### 1.7.3 设置过滤关键词



默认过滤关键词为空字符串("")，即不过滤。检索当前输出日志中是否包含该关键词，包含则允许输出。

> 注：对于配置较低的MCU建议不开启关键词过滤（默认为不过滤），增加关键字过滤将会在很大程度上减低日志的输出效率。实际上当需要实时查看日志时，过滤关键词功能交给上位机做会更轻松，所以后期的跨平台日志助手开发完成后，就无需该功能。

```c
void elog_set_filter_kw(const char *keyword)
```

> ## 演示视频：

<video src="L:/OBS_Vedio/2024-04-10%2015-18-54.mkv"></video>

> 关键字过滤演示视频。（作用是输出符合过滤要求的日志）

| 参数    | 描述   |
| ------- | ------ |
| keyword | 关键词 |

### 1.7.4 设置过滤器



设置过滤器后，只输出符合过滤要求的日志。所有参数设置方法，可以参考上述3个章节。

```c
void elog_set_filter(uint8_t level, const char *tag, const char *keyword)
```



| 参数    | 描述   |
| ------- | ------ |
| level   | 级别   |
| tag     | 标签   |
| keyword | 关键词 |

### 1.7.3 按模块的级别过滤



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

## 单片机上模拟Linux的正则表达式

> 实验日志：灵感来源于前面几个Shell交互的表达式，刚好esaylogger有开放四个参数的API，综合起来正好可以过滤等级，标签，关键字，这可以使过滤工作一步到位。
>
> ![image-20240410155917156](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404101559249.png)
>
> ![fedd748229fbb5ff5118b8f6f94a909](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404101559160.png)

## 静态过滤

> #### 静态输出日志，指定全局输出固定级别的log，当然如果不是最高级的还是可以配合ZNS使用动态设置的。

![image-20240408114032364](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404081140433.png)



> **对一些bug的优化：**
>
> 优化了指令参数读取的顺序，按照指令输入顺序即可的到理想的信息。
>
> ![image-20240410163346048](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404101633126.png)





## 模块化标签过滤的用法：

> 在不同**任务模块文件**中可以自主设置需要打印的标签，先有标签后有过滤水准。两者缺一不可。如右图一样就是正确设置的结果。<u>（这里一定是分模块文件的，而不是指具体某个任务，因为#define 的作用域是当前整个文件）</u>

![image-20240409161803063](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404091618191.png)

> #### 最新版本消息，就是即使同一个文件的任务设置不同的标签过滤等级，也可以生效，这可能于FreeRTOS是操作系统有关系，它的内存是动态创建的，每个任务都有一个独立的任务栈空间，或者堆空间，是一个相对独立的内存，所以elog对任务单独过滤是没问题的，即使在同一个文件中有多个任务，设置了标签后也可以过滤不同的等级。

![image-20240409163954713](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404091639768.png)

```c
//同一个文件中两个任务不同等级标签的过滤，RTOS的优势太大了，EasyLogger不仅轻量级，而且支持还挺多，后期用插件玩玩看。
void watch_test1_entry(void *p)
{
	#if 0
	while (1)
	{
		printf("watch test1 task run\r\n");
		vTaskDelay(200);
	}
	#endif
	#if 1
	while (1)
	{
		elog_set_filter_tag_lvl("main.c",ELOG_LVL_INFO);
		#if 0
		elog_a("HELLO WORLD %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		elog_d("HELLO WORLD %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		elog_e("HELLO WORLD %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		elog_i("HELLO WORLD %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		elog_w("HELLO WORLD %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		#endif
		#if 1
		log_a("HELLO WORLD %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		log_e("HELLO WORLD %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		log_w("HELLO WORLD %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		log_i("HELLO WORLD %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		log_d("HELLO WORLD %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		log_v("HELLO WORLD %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		#endif
		#if 0
		log_i("我是标签实验1! \r\n");
		log_a("我是标签实验1! \r\n");
		#endif
		vTaskDelay(pdMS_TO_TICKS(1000));
	}

#endif
}
void watch_test2_entry(void *p)
{
	#if 0
	while (1)
	{
		color_1++;
		if(color_1 > 10)
		{
			color_1 = 0;
			printf("\033[31m ERROR! \033[35m\r\n");
		}
		printf("watch test2 task run...\r\n");
		vTaskDelay(200);
	}
	#endif
	#if 1
	while (1)
	{
		elog_set_filter_tag_lvl("main.c",ELOG_LVL_ERROR);
		#if 0
		elog_a("hello world %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		elog_d("hello world %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		elog_e("hello world %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		elog_i("hello world %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		elog_w("hello world %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		#endif
		#if 1
		log_a("hello world %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		log_e("hello world %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		log_w("hello world %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		log_i("hello world %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		log_d("hello world %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		log_v("hello world %s\t %d\t %s\t \r\n", __FILE__, __LINE__, __TIME__);
		
		#endif
		#if 0
		log_i("我是标签实验2! \r\n");
		// elog_i("main.c","我是标签实验2! \r\n");
		#endif
		vTaskDelay(pdMS_TO_TICKS(1000));
	}

    #endif
}

```

## 1.8 缓冲输出模式



### 1.8.1 使能/失能缓冲输出模式



```c
void elog_buf_enabled(bool enabled)
```



| 参数    | 描述                    |
| ------- | ----------------------- |
| enabled | true: 使能，false: 失能 |

### 1.8.2 将缓冲区中的日志全部输出



在缓冲输出模式下，执行此方法可以将缓冲区中的日志全部输出干净。

```c
void elog_flush(void)
```



## 1.9 异步输出模式



### 1.9.1 使能/失能异步输出模式



```c
void elog_async_enabled(bool enabled)
```



| 参数    | 描述                    |
| ------- | ----------------------- |
| enabled | true: 使能，false: 失能 |

### 1.9.2 在异步输出模式下获取日志



在异步输出模式下，如果用户没有启动 pthread 库，此时需要启用额外线程来实现日志的异步输出功能。使用此方法即可获取到异步输出缓冲区中的指定长度的日志。如果设定日志长度小于日志缓冲区中已存在日志长度，将只会返回已存在日志长度。

```c
size_t elog_async_get_log(char *log, size_t size)
```



| 参数 | 描述             |
| ---- | ---------------- |
| log  | 取出的日志内容   |
| size | 待取出的日志大小 |

### 1.9.3 在异步输出模式下获取行日志（以换行符结尾）



异步模式下获取行日志与 1.9.2 中的直接获取日志功能类似，只不过这里所获取到的日志内容，必须为 **行日志** （以换行符结尾）格式，为后续的日志按行分析功能提供便利。如果设定日志长度小于日志缓冲区中已存在日志长度，将只会返回日志缓冲区中行日志的长度。如果缓冲区中不存在行日志，将不能保证返回的日志格式是行日志。

```c
size_t elog_async_get_line_log(char *log, size_t size)
```



| 参数 | 描述               |
| ---- | ------------------ |
| log  | 取出的行日志内容   |
| size | 待取出的行日志大小 |



## 纠正：

### **刚开始学：只有特定标签有标志**

![image-20240410164358227](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404101643303.png)

### **看了官方文档教学：每个标签都可以输出特定的格式**

![image-20240410164408901](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404101644962.png)

![image-20240410164905575](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404101649730.png)

![image-20240410165148877](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404101651974.png)

> ## 分文件任务设置前：
>
> ![image-20240410165222457](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404101652520.png)
>
> ## 设置后：
>
> 这才是合理的设置方法，设置之后我们可以通过Shell动态地进行过滤信息，而不会mian开启全局，需要查看子任务的信息，而子任务把信息主动屏蔽了，这样就查看不了了，这是因为任务都是一个独立的任务栈的原因，他们的属性设置可以在自己的任务栈中生效，全局管不了。
>
> ![image-20240410165322527](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202404101653595.png)
>
> ## 可以了全局都验证过了，只要指令符合需求，都是可以进行跳转的，只要不是死机就行，如果遇到没有输出，但没有死机，尝试再次输入正确指令进行过滤，这是可行的。





## 2、EasyLogger初始化通用方法

```c
/* close printf buffer */
//setbuf(stdout, NULL);			//通用设置方法，这里的FreeRTOS环境是同步通信，且关闭缓冲区的，所以没必要对缓冲区进行设置。
/* initialize EasyLogger */
elog_init();
/* set EasyLogger log format */
elog_set_fmt(ELOG_LVL_ASSERT, ELOG_FMT_ALL);
elog_set_fmt(ELOG_LVL_ERROR, ELOG_FMT_LVL | ELOG_FMT_TAG | ELOG_FMT_TIME);
elog_set_fmt(ELOG_LVL_WARN, ELOG_FMT_LVL | ELOG_FMT_TAG | ELOG_FMT_TIME);
elog_set_fmt(ELOG_LVL_INFO, ELOG_FMT_LVL | ELOG_FMT_TAG | ELOG_FMT_TIME);
elog_set_fmt(ELOG_LVL_DEBUG, ELOG_FMT_ALL & ~ELOG_FMT_FUNC);
elog_set_fmt(ELOG_LVL_VERBOSE, ELOG_FMT_ALL & ~ELOG_FMT_FUNC);
/* start EasyLogger */
elog_start();
```



# 完结
