---
title: can调试记录
date: 2026-07-13 08:52:04
tags:
---

# 主要问题和解决思路
    1. 模拟源can，互连收发测试正常
    2. soc can0外部自回环正常
    3. soc收不到模拟源的数据（确认soc会操作can总线进行发送）
    4. 模拟源收不到soc数据（确认模拟源会操作can总线进行发送）
    5. 进行soc can0、1互连测试，确认状态。0发1收状态下，收发正常
    5. 怀疑总线实际速率不对，怎么便于测试？
    7. 模拟源can实测速率一个bit 4.099us
    8. soc can实测速率一个bit 5.032us，与实际速率不匹配
    9. 翻阅手册，查得F_PRESC_SJW_SEG参数配置不对，分频系数和seg1,seg2,swj配置错误
    10. 修改F_PRESC_SJW_SEG配置，再测soc can一个bit 4.011us，基本匹配

# 其它问题
    1. soc can发数据，总线上没有波形，此时通过can寄存器可发现，出现bit错误的标志;
    后续硬件确认为器件损坏，修复后可发送数据，can总线有波形；

![can bit err](./can%E8%B0%83%E8%AF%95%E8%AE%B0%E5%BD%95/can_bit_error.JPG)

    2. soc和模拟源互联，soc发送时，由于速率不同，导致模拟源没有ack回应时。can寄存器有ack err

![can ack err](./can%E8%B0%83%E8%AF%95%E8%AE%B0%E5%BD%95/can_ack_err.JPG)


# 速率配置
    1. 一个Bit Time被分为多个TQ，TQ作为最小的控制单元，决定seg1和seg2的事件，Bit Time = t(seg1)+t(seg2)
![bit time](./can%E8%B0%83%E8%AF%95%E8%AE%B0%E5%BD%95/normal_bit_time.JPG)
    2. t(seg1)等对应的TQ范围限制
![tq 限制](./can%E8%B0%83%E8%AF%95%E8%AE%B0%E5%BD%95/tq%E9%99%90%E5%88%B6.JPG)
    3. 注意t(seg1)=(S_Seg_1+2)*TQ， **S_Seg_1** 是要写入寄存器中的值
