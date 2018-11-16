---
title:  "Mbed Bootloader配置笔记"
categories: Mbed
---

此笔记记录 Mbed OS 在 STM32F405RG 下配置 Bootloader 的过程。 Mbed OS 是 Arm 公司推出的专门面向 Arm 微处理器以及物联网应用的开源实时操作系统。

对于物联网应用而言，由于其设备联网的特性，可以方便的进行远程升级。一般而言，服务端会记录各个设备的版本信息，当有固件更新时，系统会发消息通知设备端，然后由用户确认升级或者设备自动选择合适时机升级。通常一个简单的升级流程为：设备接收到升级消息后，根据消息内容通过http、ftp等方式下载到本地文件系统，校验通过后按预设名称存储，然后重启。重启后先进入bootloader，bootloader查询文件系统中有预设名称的文件后就会使用 IAP（ In Application Programming）功能，读取新固件并且覆盖旧应用区的内容，删除文件系统中的固件文件，然后跳转到应用区执行。


## Bootloader

关于Mbed Bootloader的内容请参考：https://os.mbed.com/docs/latest/tutorials/bootloader.html

在此借用一下官方的图示：


```
|-------------------|   APPLICATION_ADDR + APPLICATION_SIZE == End of ROM
|                   |
...
|                   |
|    Application    |
|  (main program )  |
|                   |
+-------------------+   APPLICATION_ADDR == BOOTLOADER_ADDR + BOOTLOADER_SIZE
|                   |
|    Bootloader     |
|(my_bootloader.bin)|
|                   |
+-------------------+   BOOTLOADER_ADDR == Start of ROM
```

这次需要实现的 Bootloader 功能与官方范例（ https://github.com/ARMmbed/mbed-os-example-bootloader ）类似，区别在于从spi flash的littlefs上加载固件。由于系统初次上电时可能spi flash可能处于空的状态，因此需要加入文件系统格式化相关检测。由于需要使用 spi flash，在 targets.json 对应配置里需要加上：

```
"device_has_add": ["FLASH"]
"components": ["SPIF"]
```

在 mbed_app.json 中加上:

```
"target.restrict_size": "0x20000"
```

其作用在于限制Bootloader的大小，当Bootloader太大时会链接失败，当Bootloader太小时会自动补全到固定大小。

## Application
           
应用程序mbed_config.json的配置：

```
"target.bootloader_img": "bootloader/my_bootloader.bin"

```

Mbed 编译的时候会编译出两个bin文件，一个包含 Bootloader，用于出厂刷写，一个不包含 Bootloader，用于发布到后台下载升级。

由于 Mbed 里并不是所有芯片都加入了 Bootloader 支持，如果你发现 Bootloader 无法跳转到应用程序，可以检查一下芯片的 ld 配置文件，譬如GCC：

```
/* Linker script for STM32F405 */
#if !defined(MBED_APP_START)
  #define MBED_APP_START 0x08000000
#endif

#if !defined(MBED_APP_SIZE)
  #define MBED_APP_SIZE 1024k
#endif

/* Linker script to configure memory regions. */
MEMORY
{ 
  FLASH (rx) : ORIGIN = MBED_APP_START, LENGTH = MBED_APP_SIZE
  CCM (rwx) : ORIGIN = 0x10000000, LENGTH = 64K
  RAM (rwx) : ORIGIN = 0x20000188, LENGTH = 128K - 0x188
}

```

Mbed cli 会根据项目中对bootloader_img等配置定义MBED_APP_START宏。

将 system_clock.c 中 VTOR 的配置删除，因为 mbed_start_application 调用中已经进行适当的配置（参考 https://github.com/ARMmbed/mbed-os/commit/d2cf3413486f8c69a7fe81412c82e18bbf534a1e#diff-ea6486a2a085323e73374a757e011799）

```

/**
/*!< Uncomment the following line if you need to relocate your vector Table in
     Internal SRAM. */
/* #define VECT_TAB_SRAM */
#define VECT_TAB_OFFSET  0x00 /*!< Vector Table base offset field.
                                   This value must be a multiple of 0x200. */
```

```

    /* Configure the Vector Table location add offset address ------------------*/
#ifdef VECT_TAB_SRAM
    SCB->VTOR = SRAM_BASE | VECT_TAB_OFFSET; /* Vector Table Relocation in Internal SRAM */
#else
    SCB->VTOR = FLASH_BASE | VECT_TAB_OFFSET; /* Vector Table Relocation in Internal FLASH */
#endif

```
                                  

