# Vscode+Keil+STM32开发环境配置

![image-20231207181149943](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312071811180.png)

**此插件兼容市面上绝大部分单片机的开发流程，上手简单**

![image-20231207180953215](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312071810818.png)

```ini
[env:genericSTM32F103C8]
platform = ststm32
board = genericSTM32F103C8
framework = stm32cube
upload_protocol = stlink
debug_tool = stlink

[platformio]
include_dir=Core/Inc
src_dir=Core/Src
```

