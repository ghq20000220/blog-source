---
title: 串口与dma收发数据
date: 2026-10-12 11:28:34
tags:
---

# 环境
- linux：5.10.227  
- 串口：pl011  
- dma：pl330  


# 硬件配置信息
- pl011
    1. fifo size:32
    2. tx rx threshold:16

- pl330
单次搬移数据量固定4KB
    - tx
        1. dst_addr_width:1Byte
        2. direction:mem to dev
        3. dst maxburst: fifosize >> 1。(即16B)
    - rx
        1. src_addr_width:1Byte
        2. direction:dev to mem
        3. dst maxburst: fifosize >> 2。(即8B)



# rx驱动逻辑  
串口的接收由中断驱动，在中断中将fifo里的数据读取出，传递给tty。tty缓存最大是640KB
## 中断注册  
- 在pl011_startup中通过pl011_allocate_irq注册pl011_int中断服务函数
    ```text
    pl011_startup
        pl011_allocate_irq
            request_irq(pl011_int)
    ```
- 相关代码
    ```c
    static int pl011_allocate_irq(struct uart_amba_port *uap)
    {
        pl011_write(uap->im, uap, REG_IMSC);

        return request_irq(uap->port.irq, pl011_int, IRQF_SHARED, "uart-pl011", uap);
    }

    static irqreturn_t pl011_int(int irq, void *dev_id)
    {
        struct uart_amba_port *uap = dev_id;
        unsigned long flags;
        unsigned int status, pass_counter = AMBA_ISR_PASS_LIMIT;
        int handled = 0;

        spin_lock_irqsave(&uap->port.lock, flags);
        status = pl011_read(uap, REG_RIS) & uap->im;
        if (status) {
            do {
                check_apply_cts_event_workaround(uap);

                pl011_write(status & ~(UART011_TXIS|UART011_RTIS|
                            UART011_RXIS),
                        uap, REG_ICR);

                if (status & (UART011_RTIS|UART011_RXIS)) {
                    if (pl011_dma_rx_running(uap))
                        pl011_dma_rx_irq(uap);
                    else
                        pl011_rx_chars(uap);
                }
                if (status & (UART011_DSRMIS|UART011_DCDMIS|
                        UART011_CTSMIS|UART011_RIMIS))
                    pl011_modem_status(uap);
                if (status & UART011_TXIS)
                    pl011_tx_chars(uap, true);

                if (pass_counter-- == 0)
                    break;

                status = pl011_read(uap, REG_RIS) & uap->im;
            } while (status != 0);
            handled = 1;
        }

        spin_unlock_irqrestore(&uap->port.lock, flags);

        return IRQ_RETVAL(handled);
    }
    ```

## 中断处理
### 无dma流程  
- 没有dma时，rx会使能两种中断UART011_RTIS（receive timeout）和UART011_RXIS（receive interrupt）
- rx fifo到阈值后即触发UART011_RXIS中断
- 当连续32bit时间没有数据后即触发UART011_RTIS
- 中断触发后cpu将数据从串口硬件fifo中取出

### 有dma流程
- 有dma时，rx仅使能UART011_RTIS（receive timeout）中断
- 在rx fifo到达阈值（16B）时，串口发出信号给dma，dma进行数据搬移，当fifo中的数据低于16B后，信号消失，dma停止搬数
    1. 当dma搬数到达4KB时，触发dma完成中断，此时将dma搬运的数据传递给tty
    2. 当dma搬数不足4KB时，dma中断无法触发，由于dma不会将fifo抽干，连续32bit时间没有数据后即触发UART011_RTIS，进入串口中断，在串口中断中将dma已经搬完的数据给tty，然后cpu将fifo中剩余的数据读出给tty
- 信息点：
    1. fifo阈值与dma的maxburst不同，dma一定不会将fifo抽干。避免当dma中断不触发时，串口中没有数据导致串口超时不触发。致使数据卡在dma fifo中没有向上层tty传递
    2. dma启动后的搬数量在pl011中固定为4KB
    3. 给dma申请了两个buffer，dma将数据搬移到buffer后，仍然需要cpu将数据复制到tty中
    4. dma通过dst address寄存器，获取搬移的数据量

### 风险点（极限状态数据可能在dma fifo中被丢弃）
- pl330不支持pause
- 当串口超时中断触发时，会强制终止dma
- dma会保证读取出数据到dma fifo中，或从dma fifo中将数据写到目的地址。一定会保证读或写全部完成
- 如果dma完成了数据从外设读取到dma内部fifo，没有开始从fifo中写到目的地址。此时终止dma会导致数据丢失
