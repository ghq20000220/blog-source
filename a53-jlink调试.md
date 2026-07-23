---
title: a53-jlink调试
date: 2026-07-16 17:10:52
tags:
---

在win环境下实现jlink调试ARM Cortex-A53

# 环境
1. jlink 软硬件环境
    - 硬件版本：V11 Base。注意pin1的状态（可通过内部跳线帽修改）
    - 驱动版本：[JLink_Windows_V810_x86_64.exe](https://www.segger.com/downloads/jlink/JLink_Windows_V810_x86_64.exe)

2. jlink script files
    - 配置CORESIGHT_SetCoreBaseAddr和CORESIGHT_SetCTICoreBaseAddr
    - 当存在多核心时，各核心的地址不同（由硬件决定），当前仅实现单核的debug
    ```
    int ConfigTargetSettings(void) {
        JLINK_ExecCommand("JLINK_CPU = CORTEX_A53;");
        JLINK_ExecCommand("CORESIGHT_SetCoreBaseAddr = 0x80010000");
        JLINK_ExecCommand("CORESIGHT_SetCTICoreBaseAddr = 0x80020000");
        return 0;
    }
    ```

3. 硬件调试环境
    - 通过JTAG连接目标板

# 调试
1. 启动JLinkGDBServer，启动时配置
    - Target device: Cortex-A53
    - Target Interface: JTAG
    - Speed: 推荐先使用固定的低速率进行测试
    - Script file: 指定为上面的脚本，配置CoreSight相关的信息
    - GDB port: 可指定端口号

![jlink gdbserver](./a53-jlink%E8%B0%83%E8%AF%95/jlink_gdbserver_cfg.JPG)
```
-select USB=0 -device Cortex-A53 -endian little -if JTAG -speed auto -ir -LocalhostOnly -nologtofile -scriptfile "A53.JLinkScript" -port 2331 -SWOPort 2332 -TelnetPort 2333
```

2. gdb调试
    - 启动gdb
    - 连接至JLinkGDBServer对应的GDB port
    - load可执行文件

# 图形化调试
1. 可以使用CLion实现图形化调试


# 多核调试
1. 使用openocd替换JLinkGDBServer
    - 转换jlink驱动
    - 建立openocd.cfg，可以参考raspi的相关调试配置