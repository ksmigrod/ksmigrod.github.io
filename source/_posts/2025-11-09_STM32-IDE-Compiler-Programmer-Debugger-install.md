---
title: STM32 IDE/Compiler/Programmer/Debugger install
date: 2025-11-09 08:54:32
tags: STM32
---

## Why not STM32CubeIDE

The course I follow uses the default setup for STM32 development, that is STM32CubeIDE.
I on the other hand have background in Java and hate Eclipse, base IDE that STM32CubeIDE is derived from, with passion.
Therefore I want to use Jetbrains CLion IDE.

## My setup

I will be using the following stack:

* Ubuntu Linux 24.04 LTS (at least until I'll get new laptop).
* Jetbrains CLion (initially 2025.2.4, but I intend to upgrade aggresively)
* STM32CubeMX (Initially 6.15.0)
* STM32CubeCLT (Initially 1.19.0)

STM32CubeCLT added required rules to udev configuration (for non-root user access to ST-LINK), and it added environment variables through `/etc/profile.d/cubeclt-bin-path_1.19.0.sh`. To activate this changes, a logout/login is required.

## First project

I've tried to build a Blinking LED project to check if integration of IDE, compiler, debuger works. 

### STM32CubeMX

To this end I've used STM32CubeMX to select NUCLEO-F411RE board, and generate project. I've followed guidelines presented in [CLion's documentation](https://www.jetbrains.com/help/clion/embedded-stm32.html), that is CMake type project, but with separate .c/.h files.

### CMake problem

Right after import, CMake does not recognize, that this project requires cross compilation for ARM Cortex-M architecture.
When trying to build, the following error message is presented:

```
====================[ Build | blink | Debug ]===================================
/home/ksm/.local/share/JetBrains/Toolbox/apps/clion/bin/cmake/linux/x64/bin/cmake --build /home/ksm/CLionProjects/STM32-2025/blink/cmake-build-debug --target blink -j 10
[1/16] Building ASM object CMakeFiles/blink.dir/startup_stm32f411xe.s.o
FAILED: CMakeFiles/blink.dir/startup_stm32f411xe.s.o 
/usr/bin/cc -DDEBUG -DSTM32F411xE -DUSE_HAL_DRIVER -I/home/ksm/CLionProjects/STM32-2025/blink/cmake/stm32cubemx/../../Core/Inc -I/home/ksm/CLionProjects/STM32-2025/blink/cmake/stm32cubemx/../../Drivers/STM32F4xx_HAL_Driver/Inc -I/home/ksm/CLionProjects/STM32-2025/blink/cmake/stm32cubemx/../../Drivers/STM32F4xx_HAL_Driver/Inc/Legacy -I/home/ksm/CLionProjects/STM32-2025/blink/cmake/stm32cubemx/../../Drivers/CMSIS/Device/ST/STM32F4xx/Include -I/home/ksm/CLionProjects/STM32-2025/blink/cmake/stm32cubemx/../../Drivers/CMSIS/Include -g -MD -MT CMakeFiles/blink.dir/startup_stm32f411xe.s.o -MF CMakeFiles/blink.dir/startup_stm32f411xe.s.o.d -o CMakeFiles/blink.dir/startup_stm32f411xe.s.o -c /home/ksm/CLionProjects/STM32-2025/blink/startup_stm32f411xe.s
/home/ksm/CLionProjects/STM32-2025/blink/startup_stm32f411xe.s: Assembler messages:
/home/ksm/CLionProjects/STM32-2025/blink/startup_stm32f411xe.s:27: Error: unknown pseudo-op: `.syntax'
/home/ksm/CLionProjects/STM32-2025/blink/startup_stm32f411xe.s:28: Error: unknown pseudo-op: `.cpu'
```

it ends with:

```
/home/ksm/CLionProjects/STM32-2025/blink/cmake/stm32cubemx/../../Drivers/STM32F4xx_HAL_Driver/Inc/stm32f4xx_hal_dma.h:562:110: note: in definition of macro ‘__HAL_DMA_CLEAR_FLAG’
  562 |  ((uint32_t)((__HANDLE__)->Instance) > (uint32_t)DMA1_Stream3)? (DMA1->HIFCR = (__FLAG__)) : (DMA1->LIFCR = (__FLAG__)))
      |                                                                                                              ^~~~~~~~
/home/ksm/CLionProjects/STM32-2025/blink/Drivers/STM32F4xx_HAL_Driver/Src/stm32f4xx_hal_dma_ex.c:200:33: note: in expansion of macro ‘__HAL_DMA_GET_FE_FLAG_INDEX’
  200 |     __HAL_DMA_CLEAR_FLAG (hdma, __HAL_DMA_GET_FE_FLAG_INDEX(hdma));
      |                                 ^~~~~~~~~~~~~~~~~~~~~~~~~~~
ninja: build stopped: subcommand failed.
```

One can fix it by changing _Settings_ (Ctrl-Alt-S), _Build, Execution, Deployment_ > _CMake_ and adding in the text box titled _CMake options:_ the following incantation:

```
--toolchain $CMakeProjectDir$/cmake/gcc-arm-none-eabi.cmake
```

As far as I can understand, recent version of STM32CubeMX allows a choice between GCC and CLang compilers, but CLion did not catch up.

### Source code

All this project does is a modification of `Core/Src/main.c` file aroung line 93-105:

```c
/* USER CODE END 2 */

/* Infinite loop */
/* USER CODE BEGIN WHILE */
while (1)
{
    HAL_GPIO_TogglePin(LD2_GPIO_Port, LD2_Pin);
    /* USER CODE END WHILE */
    
    /* USER CODE BEGIN 3 */
    HAL_Delay(100);
}
/* USER CODE END 3 */
```

it builds fine, but there is a debug problem.

### ST-LINK in CLion

Debugger console in CLion shows the following message:

```
GNU gdb (GDB; JetBrains IDE bundle; build 209) 15.2
Copyright (C) 2024 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-pc-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word".
monitor reset hardware
"monitor" command not supported by this target.
```

I've found the cause of this problem by running ST-LINK_gdbserver from console:

```
$ ST-LINK_gdbserver -p 61234 \
   -cp /opt/st/stm32cubeclt_1.19.0/STM32CubeProgrammer/bin \
   --halt  --apid 0 -d --frequency 8000


STMicroelectronics ST-LINK GDB server. Version 7.11.0
Copyright (c) 2025, STMicroelectronics. All rights reserved.

Starting server with the following options:
        Persistent Mode            : Disabled
        Logging Level              : 1
        Listen Port Number         : 61234
        Status Refresh Delay       : 15s
        Verbose Mode               : Disabled
        SWD Debug                  : Enabled


Error in initializing ST-LINK device.
Reason: ST-LINK firmware upgrade required. Please upgrade the ST-LINK firmware using the upgrade tool.
```

It turns out, that ST-LINK firmware on my devboard is too old, and needs to be upgraded:

```
cd /opt/st/stm32cubectl_1.19.0
./STLinkUpgrade.sh
```

A few moments later the following communicate announced success:

```
Firmware version detected: V2J38M27
..........................Upgrade is successful.
Version read: 2.46.33
```

With this hurdle out of the way, I was able to run and debug my program in CLion.
