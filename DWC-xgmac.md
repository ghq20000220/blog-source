---
title: DWC xgmac
date: 2026-08-12 16:48:52
tags:
    - 以太网
    - xgmac
hide: true
---

# dw xgmac使用记录

## 典型结构介绍
![xgmac结构](./DWC-xgmac/xgmac%E7%BB%93%E6%9E%84%E7%A4%BA%E6%84%8F%E5%9B%BE.JPG)

从典型结构图可以看出，xgmac接口部分由MAC、PCS和PMA组成
- PMA层：利用SerDes技术实现，实现数据的串并转换
- PCS层：实现编解码，扰码/解扰。对MAC层的数据进行信道编码等
- MAC层：由XGMAC-AXI，XGMAC-MTL，XGMAC-CORE组成

## uboot mac初始化
初始化做的事情，dma,mtl,mac
dma init:
1. 复位，等复位完成
2. 配置dma burst
3. 创建描述符
4. 配置描述符位置和长度等
5. 配置dma中断
6. 启动dma
mtl init：
