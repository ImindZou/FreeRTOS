# Rtos移植问题的解决方法

问题就出在调用xTaskGetIdleTaskHandle();的时候发生的未定义报错事件，文件说明没有定义，但又可以只能补齐，就可能是宏定义那边的问题，直接到对应的freetos.h文件里看看能不能找到相应的宏定义编程，一般都是这么修改的，而且感觉比较麻烦，VubeMX每配置一次可能都要重新去修改一次，比较繁琐。但是能解决问题。

![image-20231206143541079](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312061435122.png)

![image-20231206143515171](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312061435245.png)

![image-20231206143418338](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312061434546.png)

![image-20231206160121833](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312061601919.png)

```ABAP
使用FreeRTOS时，如果编译时遇到函数未定义的错误，要注意查看其源代码里有没有预编译条件。有的函数在头文件里有定义，但源程序里不一定编译，例如，函数xTaskGetIdleTaskHandle()。
```

