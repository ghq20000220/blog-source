---
title: UX900 linux
date: 2026-08-14 15:40:36
tags:
    - linux
    - nfs
categories:
    - 未解决
---
在nuclei UX900 cpu上运行linux时遇到问题 未解决。在此记录

## 环境

1. board：UX900 4core + uart + eth
2. ubuntu：版本20.04 + NFS（作为server用于挂载网络文件系统） + FTP（用于传输linux内核）

## 现象

- PASS: 单核 + 本地文件系统
- PASS: 单核 + NFS文件系统
- PASS: 多核 + 本地文件系统
- FAIL: 多核 + NFS文件系统

1. 内核版本为[dev_nuclei_5.10_v3](https://gitee.com/Nuclei-Software/nuclei-linux-sdk/tree/dev_nuclei_5.10_v3/)时，表现为NFS挂载后卡死，且没有任何打印信息
2. 内核版本为[dev_nuclei_6.18_v3](https://gitee.com/Nuclei-Software/nuclei-linux-sdk/tree/dev_nuclei_6.18_v3/)时，表现为NFS挂载后出现kernel panic，有概率系统会启动，可以输入用户密码进入系统，大概率在多次kernel panic后挂掉

## 怀疑方向

1. 多核cache一致性
2. 以太网稳定性，误码？
