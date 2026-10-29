---
title: SocketCAN
date: 2026-10-20 08:56:20
tags:
    - CAN
    - Socket
---

自SocketCAN子系统被并入内核主干，就将CAN驱动纳入了标准的网络子系统框架之下。也就是说CAN和以太网驱动一样，使用的是同一套底层基础设施。

# CAN配置
- 在Networking support中打开CAN bus subsystem support
- 目前启用了部分协议
- 在设备驱动中使能CAST CAN控制器
```TEXT
Networking support
    [*]CAN bus subsystem support
        [*]Raw CAN Protocol (raw access with CAN-ID filtering)
        [*]Broadcast Manager CAN Protocol (with content filtering)
        [*]CAN Gateway/Router (with netlink configuration)
        [*]ISO 15765-2:2016 CAN transport protocol
           CAN Device Drivers
                [*]Platform CAN drivers with Netlink support
                [M]CAST CAN-CTRL (Classical CAN only)
```

# 驱动代码分析
## probe
- 基本的标准结构
    1. 寄存器映射
    2. 分配并初始化 CAN 网络设备
    3. 初始化 CAN 核心层参数
        - 提供位定时能力：priv->can.bittiming_const（可能有data_bittiming_const），内核计算比特率的基础
        - 实现必要回调：设置 priv->can.do_set_mode 回调，这是支持总线错误恢复（Bus-Off restart）所必需的
        - 声明时钟频率：priv->can.clock.freq，位定时计算依赖于时钟频率
        - 设置特性标志：根据硬件能力，设置 priv->can.ctrlmode_supported（如支持回环、CAN FD 等）和 ndev->flags |= IFF_ECHO（启用本地回环）
    4. 硬件与驱动的深度初始化
    5. 注册 CAN 设备
- 额外添加stb-slots的获取，用于alloc_candev时指定echo_skb_max的个数
```c
static int cast_can_probe(struct platform_device *pdev)
{
	struct net_device *ndev;
	struct cast_can_priv *priv;
	void __iomem *base;
	u32 freq, stb_slots;
	int ret;

	base = devm_platform_ioremap_resource(pdev, 0);     // 获取reg地址
	if (IS_ERR(base))
		return PTR_ERR(base);

	ret = of_property_read_u32(pdev->dev.of_node, "cast,stb-slots",
				   &stb_slots);                         // 获取发送的插槽个数（能够缓存的can帧个数）
	if (ret) {
		dev_err(&pdev->dev, "missing cast,stb-slots property: %d\n", ret);
		return ret;
	}
	if (!stb_slots || stb_slots > CAST_STB_MAX_SLOTS) {
		dev_err(&pdev->dev, "invalid cast,stb-slots value %u\n",
			stb_slots);
		return -EINVAL;
	}

	ndev = alloc_candev(sizeof(*priv), stb_slots);      //分配私有数据，can中间层数据，本地回环缓冲区，完成net_dev和can_priv的初始化
	if (!ndev)
		return -ENOMEM;

	priv = netdev_priv(ndev);
	priv->dev = &pdev->dev;
	priv->reg_base = base;
	priv->stb_slots = stb_slots;
	spin_lock_init(&priv->tx_lock);
	priv->can.bittiming_const = &cast_can_bittiming_const;      //内核计算波特率时的硬件边界
	priv->can.ctrlmode_supported = CAN_CTRLMODE_LOOPBACK;
	priv->can.do_set_mode = cast_can_do_set_mode;               //主要同于总线错误后重启
	priv->can.state = CAN_STATE_STOPPED;

	/* can_clk drives the bit timing; pclk is the optional APB clock. */
	priv->clk_core = devm_clk_get_optional(&pdev->dev, "can_clk");
	if (IS_ERR(priv->clk_core)) {
		ret = PTR_ERR(priv->clk_core);
		goto err_free;
	}

	priv->clk_apb = devm_clk_get_optional(&pdev->dev, "pclk");
	if (IS_ERR(priv->clk_apb)) {
		ret = PTR_ERR(priv->clk_apb);
		goto err_free;
	}

	if (priv->clk_core) {
		priv->can.clock.freq = clk_get_rate(priv->clk_core);
	} else if (!of_property_read_u32(pdev->dev.of_node,
					 "clock-frequency", &freq)) {
		priv->can.clock.freq = freq;
	} else {
		dev_err(&pdev->dev, "no can_clk and no clock-frequency\n");
		ret = -EINVAL;
		goto err_free;
	}

	if (!priv->can.clock.freq) {
		dev_err(&pdev->dev, "can_clk rate is zero\n");
		ret = -EINVAL;
		goto err_free;
	}

	priv->resets = devm_reset_control_array_get_exclusive(&pdev->dev);
	if (IS_ERR(priv->resets)) {
		ret = PTR_ERR(priv->resets);
		if (ret == -ENOENT || ret == -ENODEV) {
			priv->resets = NULL;
		} else {
			goto err_free;
		}
	}

	ndev->irq = platform_get_irq(pdev, 0);
	if (ndev->irq < 0) {
		ret = ndev->irq;
		goto err_free;
	}

	ret = clk_prepare_enable(priv->clk_apb);
	if (ret)
		goto err_free;
	ret = clk_prepare_enable(priv->clk_core);
	if (ret)
		goto err_clk_apb;

	if (priv->resets) {
		ret = reset_control_deassert(priv->resets);
		if (ret)
			goto err_clk;
	}

	/* The device stays configured but clock-gated while the netdev is down. */
	clk_disable_unprepare(priv->clk_core);
	clk_disable_unprepare(priv->clk_apb);

	ndev->flags |= IFF_ECHO;
	ndev->netdev_ops = &cast_can_netdev_ops;
	SET_NETDEV_DEV(ndev, &pdev->dev);
	platform_set_drvdata(pdev, ndev);

	netif_napi_add(ndev, &priv->napi, cast_can_rx_poll, 16);    // 和以太网驱动中用法相同，用于减少中断开销；将中断变成轮询

	ret = register_candev(ndev);
	if (ret) {
		dev_err(&pdev->dev, "failed to register CAN device: %d\n", ret);
		goto err_napi;
	}

	dev_info(&pdev->dev, "CAN 2.0B controller, irq %d, %u Hz\n",
		 ndev->irq, priv->can.clock.freq);
	return 0;

err_napi:
	netif_napi_del(&priv->napi);
	if (priv->resets)
		reset_control_assert(priv->resets);
	goto err_free;
err_clk:
	clk_disable_unprepare(priv->clk_core);
err_clk_apb:
	clk_disable_unprepare(priv->clk_apb);
err_free:
	free_candev(ndev);
	return ret;
}

```

## open
- 时钟使能
- 中断注册
- napi_enable
- chip start：开启收发等的中断
- netif_start_queue：启动网络的发送队列（内核发数据包的几个函数netif_start_queue，netif_wake_queue，netif_stop_queue，netif_queue_stopped）
```c
static int cast_can_open(struct net_device *ndev)
{
	struct cast_can_priv *priv = netdev_priv(ndev);
	int ret;

	ret = clk_prepare_enable(priv->clk_apb);    //时钟使能
	if (ret)
		return ret;
	ret = clk_prepare_enable(priv->clk_core);
	if (ret)
		goto err_apb;

	ret = open_candev(ndev);
	if (ret)
		goto err_clk;

	ret = request_irq(ndev->irq, cast_can_irq, IRQF_SHARED, ndev->name,
			  ndev);    //注册中断
	if (ret) {
		netdev_err(ndev, "failed to request irq %d: %d\n",
			   ndev->irq, ret);
		goto err_candev;
	}

	priv->tx_head = 0;
	priv->tx_tail = 0;

	/*
	 * Enable NAPI before the controller starts raising interrupts,
	 * otherwise a frame received in between could be lost (the flag is
	 * cleared but napi_schedule() is a no-op while NAPI is disabled).
	 */
	napi_enable(&priv->napi);   //先使能NAPI，再使能硬件中断

	ret = cast_can_chip_start(ndev);    //模式配置，中断使能等
	if (ret) {
		netdev_err(ndev, "failed to start controller: %d\n", ret);
		cast_can_enter_reset(priv);
		napi_disable(&priv->napi);
		goto err_irq;
	}

	netif_start_queue(ndev);    //初始化，使能发送路径
	return 0;

err_irq:
	free_irq(ndev->irq, ndev);
err_candev:
	close_candev(ndev);
err_clk:
	clk_disable_unprepare(priv->clk_core);
err_apb:
	clk_disable_unprepare(priv->clk_apb);
	return ret;
}
```

## ndo_start_xmit
- 由网络协议栈调用，实现真正的数据发送
- 解析skb中的can帧
- 将can帧填入到控制器的特定寄存器
- 根据软件队列剩余空间进行反压
```c
static netdev_tx_t cast_can_start_xmit(struct sk_buff *skb,
				       struct net_device *ndev)
{
	struct cast_can_priv *priv = netdev_priv(ndev);
	struct can_frame *cf = (struct can_frame *)skb->data;
	unsigned long flags;
	unsigned int idx;
	u32 id;
	u8 ctl;

	if (can_dropped_invalid_skb(ndev, skb))     //丢弃非法帧
		return NETDEV_TX_OK;

	spin_lock_irqsave(&priv->tx_lock, flags);
    //检查是否有slots，当slots不足时，返回忙，并停止queue继续发送
	if (unlikely(priv->tx_head - priv->tx_tail >= priv->stb_slots)) {
		spin_unlock_irqrestore(&priv->tx_lock, flags);
		netif_stop_queue(ndev);
		return NETDEV_TX_BUSY;
	}
    // 获取skb占用的位置
	idx = priv->tx_head % priv->stb_slots;

	ctl = cf->can_dlc & CCAN_CTL_DLC_MASK;
	if (cf->can_id & CAN_EFF_FLAG)
		ctl |= CCAN_CTL_IDE;
	if (cf->can_id & CAN_RTR_FLAG)
		ctl |= CCAN_CTL_RTR;

	id = cf->can_id & ((cf->can_id & CAN_EFF_FLAG) ?
			   CAN_EFF_MASK : CAN_SFF_MASK);

	/*
	 * Select the STB and keep TBSEL set until TSNEXT has been written.
	 * The core points the TBUF window at the next free STB slot.
	 */
	cast_can_write_ctl(priv, priv->cfg, CCAN_TCMD_TBSEL, 0, 0);     //使用stb slots，而非ptb

	cast_can_write32(priv, CCAN_TBUF_ID, id);
	if (!(ctl & CCAN_CTL_RTR))
		cast_can_write_payload(priv, cf->data, cf->can_dlc);        //写数据部分
	cast_can_write32(priv, CCAN_TBUF_CTL, ctl);                     //写控制部分

	can_put_echo_skb(skb, ndev, idx);

	/*
	 * Mark this slot as filled and request transmission of all frames in
	 * the STB.  TSALL is written on every fill: if the previous batch has
	 * just become empty (and TSALL was auto-cleared) this starts a new
	 * batch; if TSALL is still active the redundant set is harmless and
	 * the new frame joins the running batch.
	 */
	cast_can_write_ctl(priv, priv->cfg,
			   CCAN_TCMD_TBSEL | CCAN_TCMD_TSALL,
			   CCAN_TCTRL_TSNEXT, 0);       //选择stb slots，发送所有slots，硬件选中下一个可用slot

	priv->tx_head++;

	if (priv->tx_head - priv->tx_tail >= priv->stb_slots)
		netif_stop_queue(ndev);

	spin_unlock_irqrestore(&priv->tx_lock, flags);

	return NETDEV_TX_OK;
}
```

## TX DONE
- 由中断调用，当所有slots中的数据都发送完毕，触发
- 批量回收在途帧
- 唤醒队列

```c
static void cast_can_tx_done(struct net_device *ndev)
{
	struct cast_can_priv *priv = netdev_priv(ndev);
	struct net_device_stats *stats = &ndev->stats;
	unsigned long flags;
	unsigned int idx, len;
	bool wake = false;
	u8 tsstat;

	spin_lock_irqsave(&priv->tx_lock, flags);

	tsstat = cast_can_read8(priv, CCAN_TCTRL) & CCAN_TCTRL_TSSTAT_MASK;
	if (tsstat)
		goto out;

	wake = priv->tx_tail != priv->tx_head;

    // 状态修改，计数
	while (priv->tx_tail != priv->tx_head) {
		idx = priv->tx_tail % priv->stb_slots;
		len = can_get_echo_skb(ndev, idx);
		stats->tx_bytes += len;
		stats->tx_packets++;
		priv->tx_tail++;
	}

out:
	spin_unlock_irqrestore(&priv->tx_lock, flags);

	if (wake)
		netif_wake_queue(ndev);     //唤醒发送队列
}
```

## RX
- 依赖于NAPI进行
- 轮询进行数据接收
- cast_can_rx_one中进行实际的buffer读取，组帧
- 接收后统一将数据sbk向上层传递，和以太网驱动类似

```c
static int cast_can_rx_poll(struct napi_struct *napi, int quota)
{
	struct net_device *ndev = napi->dev;
	struct cast_can_priv *priv = netdev_priv(ndev);
	int work = 0;

	while (work < quota) {
		if (!(cast_can_read8(priv, CCAN_RCTRL) &
		      CCAN_RCTRL_RSTAT_MASK))
			break;  //接收buff空，退出循环
		work += cast_can_rx_one(ndev);
	}

	if (work < quota) {
		napi_complete_done(napi, work);
		priv->rtie |= CCAN_RTIE_RIE;
		cast_can_write_irq(priv, 0, 0);

		/* Close the race between the empty check and RIE enable. */
		if ((cast_can_read8(priv, CCAN_RCTRL) &
		     CCAN_RCTRL_RSTAT_MASK) &&
		    napi_schedule_prep(napi)) {
			priv->rtie &= ~CCAN_RTIE_RIE;
			cast_can_write_irq(priv, 0, 0);
			__napi_schedule(napi);
		}
	}

	return work;
}

static int cast_can_rx_one(struct net_device *ndev)
{
	struct cast_can_priv *priv = netdev_priv(ndev);
	struct net_device_stats *stats = &ndev->stats;
	struct sk_buff *skb;
	struct can_frame *cf;
	u32 id;
	u8 ctl, len;

	ctl = cast_can_read8(priv, CCAN_RBUF_CTL);
	id = cast_can_read32(priv, CCAN_RBUF_ID);
	len = cast_can_dlc2len(ctl & CCAN_CTL_DLC_MASK);

	/*
	 * Classical CAN only.  A CC-only core never sets FDF, but if the
	 * core was synthesised with CAN FD support, ignore FD frames rather
	 * than misinterpreting them as Classical CAN.
	 */
	if (ctl & CCAN_CTL_FDF) {
		stats->rx_dropped++;
		cast_can_release_rx(priv);
		return 1;
	}

	skb = alloc_can_skb(ndev, &cf);
	if (!skb) {
		stats->rx_dropped++;
		/* Release the slot so the FIFO does not stall. */
		cast_can_release_rx(priv);
		return 1;
	}

	cf->can_id = id & ((ctl & CCAN_CTL_IDE) ? CAN_EFF_MASK : CAN_SFF_MASK);
	if (ctl & CCAN_CTL_IDE)
		cf->can_id |= CAN_EFF_FLAG;
	if (ctl & CCAN_CTL_RTR)
		cf->can_id |= CAN_RTR_FLAG;

	cf->can_dlc = len;
	if (!(ctl & CCAN_CTL_RTR))
		cast_can_read_payload(priv, cf->data, len);

	/* Reading RREL moves the RBUF window to the next oldest slot. */
	cast_can_release_rx(priv);

	stats->rx_bytes += len;
	stats->rx_packets++;
	netif_receive_skb(skb);

	return 1;
}
```

# ip link set
- 使用ip link set可以配置can的波特率，以及开关can
- ip link set依赖于buildroot中的iproute2

```
ip link set can0 down
ip link set can0 type can bitrate 500000 loopback on    //自回环开启
ip link set can0 up
```
