---
title: JFlash OpenFlashLoader
date: 2026-08-14 15:10:36
tags: 
    - jlink
    - jflsh
---
J-Flash是SEGGER推出的通过J-Link对Flash进行烧录的工具，内置多款主流芯片和存储器。且支持自定义芯片和烧录算法，提供了高灵活性

## 使用说明

目标：通过自定义设备和烧录算法，实现对片外Flash的编程

### J-Flash基本原理

![硬件结构](./JFlash-OpenFlashLoader/连接结构示意图.JPG)

主机端运行J-Flash，通过J-Link作为硬件桥梁，控制目标板上的cpu通过spi，qspi等方式对外部flash进行编程

- PC上通过J-Flash的图形界面下达烧录指令
- J-Flash会通过J-Link，将flash烧录算法下载到Board上的RAM中
- J-Flash通过J-Link控制CPU，将PC端传输过来的数据，逐步写入到flash中

### 添加自定义设备

在运行segger jlink，jflash等软件时，一般都会需要选择Target Device。
![设备选择](./JFlash-OpenFlashLoader/target_device.JPG)

实现参考[J-Link_Device_Support_Kit](https://kb.segger.com/J-Link_Device_Support_Kit)

1. 文件等存放路径 C:\Users\<USER>\AppData\Roaming\SEGGER\JLinkDevices\
当C:\Users\<USER>\AppData\Roaming\SEGGER目录下没有JLinkDevices目录时建立目录即可

2. 在JLinkDevices下建立Devices.xml文件，示例如下
```
<Database>
  <Device>
    <ChipInfo Vendor="SEGGER" Name="SEGGER_Device0" Core="JLINK_CORE_CORTEX_M0"/>
  </Device>
  <Device>
    <ChipInfo Vendor="SEGGER" Name="SEGGER_Device1" Core="JLINK_CORE_CORTEX_M4"/>
  </Device>
</Database>
```

3. 当需要J-Link script file时，在xml中指定(JLinkScriptFile="SEGGER/Example.jlinkscript")，示例如下
```
<Database>
  <Device>
    <ChipInfo Vendor="SEGGER" Name="SEGGER_Device0" Core="JLINK_CORE_CORTEX_M0" JLinkScriptFile="SEGGER/Example.jlinkscript"/>
  </Device>
</Database>
```

4. 指定flash信息和对应的烧录算法，示例如下
```
<Database>
  <Device>
    <ChipInfo Vendor="SEGGER" Name="SEGGER_Device0" WorkRAMAddr="0x20000000" WorkRAMSize="0x8000" Core="JLINK_CORE_CORTEX_M4" />
    <FlashBankInfo Name="Internal code flash" BaseAddr="0x08000000" AlwaysPresent="1" >
      <LoaderInfo Name="Default" MaxSize="0x80000" Loader="Flashloader_Device0_InternalCodeFlash.elf" LoaderType="FLASH_ALGO_TYPE_OPEN" />
    </FlashBankInfo>
  </Device>
</Database>
```

5. 示例
```
<Database>
  <Device>
    <ChipInfo Vendor="Heimda" Name="Heimda" WorkRAMAddr="0x2000000" WorkRAMSize="0x100000" Core="JLINK_CORE_CORTEX_A53" JLinkScriptFile="A53.jlinkscript" />
    <FlashBankInfo Name="AT25DQ041" BaseAddr="0x00000000" AlwaysPresent="1" >
      <LoaderInfo Name="SPI0 FLASH" MaxSize="0x100000" Loader="Heimda_Test.elf" LoaderType="FLASH_ALGO_TYPE_OPEN" />
    </FlashBankInfo>

    <FlashBankInfo Name="AT25DQ041" BaseAddr="0x00000000" AlwaysPresent="1" >
      <LoaderInfo Name="SPI1 FLASH" MaxSize="0x100000" Loader="Heimda_Test.elf" LoaderType="FLASH_ALGO_TYPE_OPEN" />
    </FlashBankInfo>
  </Device>
</Database>
```
![temp](./JFlash-OpenFlashLoader/temp.JPG)

### Open Flash Loader 

[flash下载算法实现参考](https://kb.segger.com/SEGGER_Flash_Loader)
我们通过Open Flash Loader示例进行修改，实现自定义flash下载算法

[工程示例](https://github.com/404Zen/SFL/tree/master)

1. 需要实现指定的函数功能
2. 需要按照指定的代码布局进行链接
3. 注意变量的类型，在变量类型不对时可能出现奇怪的问题
4. 尽量提高代码执行速度，比如使用o2进行编译（在使用o0编译的elf时，大量读取（1MB）会超时，o2编译的不会）

# FlashOS.h
```c
#ifndef SEGGER_FLASH_OS_H
#define SEGGER_FLASH_OS_H
#include <stdint.h>

#define ONCHIP     (1)             // On-chip Flash Memory

#define MAX_NUM_SECTORS (512)      // Max. number of sectors, must not be modified.
#define ALGO_VERSION    (0x0101)   // Algo version, must not be modified.

struct SECTOR_INFO  {
	uint32_t SectorSize;       // Sector Size in bytes
	uint32_t SectorStartAddr;  // Start address of the sector area (relative to the "BaseAddr" of the flash)
};

struct FlashDevice  {
  uint16_t AlgoVer;
  uint8_t  Name[128];
  uint16_t Type;                    // Flash device type. Currently ignored. Set to 1 to get max. compatibility.
  uint32_t BaseAddr;                // Flash base address. It is recommended to always use the real address of the flash here, even if the flash is also available at other addresses (via an alias / remap), depending on the current settings of the device.
  uint32_t TotalSize;               // Total flash device size in bytes.
  uint32_t PageSize;                // This field describes in what chunks J-Link feeds the flash loader.
  uint32_t Reserved;                // 	Set this element to 0
  uint8_t  ErasedVal;               // Most flashes have an erased value of 0xFF (set this element to 0xFF in such cases).
  uint32_t TimeoutProg;             // Timeout in milliseconds (ms) to program one chunk of <PageSize>.
  uint32_t TimeoutErase;            // Timeout in milliseconds (ms) to erase one sector.
  struct SECTOR_INFO SectorInfo[MAX_NUM_SECTORS];       // 	This element is actually a list of different sector sizes present on target flash.Having a flash with uniform sectors will result in only SectorInfo[0] being used for sectorization information.
};

#endif
```

# flash_loader.h
```c
#ifndef SEG_FL_H
#define SEG_FL_H
#include "FlashOS.h"

#define PrgCode  __attribute__ ((section ("PrgCode"), __used__))
#define DevDescr __attribute__ ((section ("DevDscr"), __used__))

int SEGGER_FL_Prepare(uint32_t PreparePara0, uint32_t PreparePara1, uint32_t PreparePara2);
int SEGGER_FL_Restore(uint32_t RestorePara0, uint32_t RestorePara1, uint32_t RestorePara2);
int SEGGER_FL_Program(uint32_t DestAddr, uint32_t NumBytes, uint8_t *pSrcBuff);
int SEGGER_FL_Erase(uint32_t SectorAddr, uint32_t SectorIndex, uint32_t NumSectors);
int SEGGER_FL_EraseChip(void);
int SEGGER_FL_Read(uint32_t Addr, uint32_t NumBytes, uint8_t *pDestBuff);

#endif
```

# flash_loader.c
```c
#include "flash_loader.h"
#include "at25dq041.h"
#include "soc_address_map.h"
#include "soc_perpheral_struct.h"

#define DEBUG_PRINTF(fmt, args...)
#define DEBUG_DISPLAY(fmt, args...)
#define DEBUG(fmt, args...)


struct FlashDevice const FlashDevice DevDescr =  {
  ALGO_VERSION,              // Algo version
  "AT25DQ041", // Flash device name
  0,                    // Flash device type
  0x0,                // Flash base address
  0x100000,                  // Total flash device size in Bytes
  256,                       // Page Size (number of bytes that will be passed to ProgramPage(). May be multiple of min alignment in order to reduce overhead for calling ProgramPage multiple times
  0,                         // Reserved, should be 0
  0xFF,                      // Flash erased value
  20000,                       // Program page timeout in ms
  20000,                      // Erase sector timeout in ms
  //
  // Flash sector layout definition
  //
  {
      {0x00001000, 0x0},   //
      {0xFFFFFFFF, 0xFFFFFFFF},    // Indicates the end of the flash sector layout. Must be present.
  }
};



int PrgCode SEGGER_FL_Prepare(uint32_t PreparePara0, uint32_t PreparePara1, uint32_t PreparePara2){
    int ret_value = 0;

    at25dq041_default_initial(SPI0_REGDEF);
    at25dq041_global_unprotect(SPI0_REGDEF);
    
    return ret_value;
};


int PrgCode SEGGER_FL_Restore(uint32_t RestorePara0, uint32_t RestorePara1, uint32_t RestorePara2){

    return 0;
};


int PrgCode SEGGER_FL_Program(uint32_t DestAddr, uint32_t NumBytes, uint8_t *pSrcBuff){
    uint32_t addr=0;


    if(DestAddr >= FlashDevice.BaseAddr){
        addr = DestAddr - FlashDevice.BaseAddr;
    }else{
        addr = DestAddr;
    }
    
    at25dq041_program(SPI0_REGDEF, addr, pSrcBuff, NumBytes);

    return 0;
};

int PrgCode SEGGER_FL_Erase(uint32_t SectorAddr, uint32_t SectorIndex, uint32_t NumSectors){
    uint32_t addr=0;

    DEBUG("=>Erase call addr:0x%0X  index:%d sectors:%d bytes",SectorAddr,SectorIndex,NumSectors);

    if(SectorAddr >= FlashDevice.BaseAddr){
        addr = SectorAddr - FlashDevice.BaseAddr;
    }

    for(uint32_t i=SectorIndex; i<(SectorIndex+NumSectors); i++){
        at25dq041_block_erase(SPI0_REGDEF, addr+(i*FlashDevice.SectorInfo->SectorSize), BLOCKSIZE_4K);
    }
    return 0;

};

int PrgCode SEGGER_FL_EraseChip(void){

    DEBUG("=>Erase chip call");

    at25dq041_chip_erase(SPI0_REGDEF);
    return 0;

};

int PrgCode SEGGER_FL_Read(uint32_t Addr, uint32_t NumBytes, uint8_t *pDestBuff){
    uint32_t addr=0;

    DEBUG_PRINTF("SEGGER_FL_Read\r\n");
    DEBUG_DISPLAY(Addr);
    DEBUG_PRINTF("\r\n");
    DEBUG_DISPLAY(NumBytes);
    DEBUG_PRINTF("\r\n");

    if(Addr >= FlashDevice.BaseAddr){
        addr = Addr - FlashDevice.BaseAddr;
    }

    at25dq041_read_bytes(SPI0_REGDEF, addr, pDestBuff, NumBytes);

    return NumBytes;
}
```

# lscript.ld
```ld

/* Define stack and heap size in the system */
_STACK_SIZE = DEFINED(_STACK_SIZE) ? _STACK_SIZE : 0x2000;
_HEAP_SIZE = DEFINED(_HEAP_SIZE) ? _HEAP_SIZE : 0x2000;

/*if used printf_ra, 256Byte is not enough*/
_EL0_STACK_SIZE = 1024;	
_EL1_STACK_SIZE = 1024;
_EL2_STACK_SIZE = 1024;

/* Define Memories in the system */
MEMORY
{
   iram_m_address : ORIGIN = 0x0002000000, LENGTH = 0x0000100000
}

/* Specify the default entry point to the program */

ENTRY(_boot)

/* Define the sections, and where they are mapped in memory */

SECTIONS
{
/* 必须放在开头 */
PrgCode :
{
   . = ALIGN(4);
   KEEP(*(PrgCode));
   KEEP(*(PrgCode*));
   . = ALIGN(4);
} > iram_m_address

/* 放在text，rodata后，data前 */
/* Marks the end of the code + rodata region (functions, const data, ...) and the start of the data region (static + global variables) */
PrgData :
{
    . = ALIGN(4);
    KEEP(*(PrgData));
    KEEP(*(PrgData*));
    . = ALIGN(4);
}


/* 放在最后 */
/* Marks the location of the <FlashDevice> structure variable and also the end of the loader. Must(!!!) be the very last section */
DevDscr :
{
    . = ALIGN(4);
    KEEP(*(DevDscr));
    KEEP(*(DevDscr*));
    . = ALIGN(4);
} > iram_m_address

}


```