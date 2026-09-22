---
title: at24-eeprom
date: 2026-09-21 14:01:25
tags:
    - linux
    - at24
    - eeprom
    - pagesize
---
# 环境
- linux：5.10.227  


# 问题现象
- 通过/sys/bus/i2c/devices/0-0050/eeprom设备访问i2c控制器下的eeprom，速率极低
- 在启动时，有打印at24 0-0050: 262144 byte 24c2048 EEPROM, writable, 1 bytes/write


# 问题原因
- 设备树节点中没有指定pagesize，当不指定pagesize时，pagesize跌落为1
- 指定pagesize后，单次写的长度受到at24_io_limit（128B）的限制，不能超at24_io_limit

# dts指定pagesize
```dts
i2c0: i2c@4010b000 {
    compatible = "snps,designware-i2c";
    reg = <0x00 0x4010b000 0x00 0x1000>;
    interrupt-parent = <&gic_400>;
    interrupts = <GIC_SPI 27 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&cru CLK_I2C0_GS1MCLK>, <&cru CLK_I2C0_GS1MCLK>;
    clock-names = "clk", "pclk";
    resets = <&rst I2C0_SOFT_GS1MRESET>;
    reset-names = "i2c0";
    #address-cells = <1>;
    #size-cells = <0>;
    status = "okay";
    clock-frequency = <400000>;

    eeprom@50 {
        compatible = "atmel,24c2048","m24m02";
        reg = <0x50>;
        pagesize = <256>;
        address-width = <16>;
    };
};
```

# 源码部分
```c
static int at24_probe(struct i2c_client *client)
{
	struct regmap_config regmap_config = { };
	struct nvmem_config nvmem_config = { };
	u32 byte_len, page_size, flags, addrw;
	const struct at24_chip_data *cdata;
	struct device *dev = &client->dev;
	bool i2c_fn_i2c, i2c_fn_block;
	unsigned int i, num_addresses;
	struct at24_data *at24;
	struct regmap *regmap;
	bool writable;
	u8 test_byte;
	int err;

	i2c_fn_i2c = i2c_check_functionality(client->adapter, I2C_FUNC_I2C);
	i2c_fn_block = i2c_check_functionality(client->adapter,
					       I2C_FUNC_SMBUS_WRITE_I2C_BLOCK);

	cdata = at24_get_chip_data(dev);
	if (IS_ERR(cdata))
		return PTR_ERR(cdata);

	err = device_property_read_u32(dev, "pagesize", &page_size);
	if (err)
		/*
		 * This is slow, but we can't know all eeproms, so we better
		 * play safe. Specifying custom eeprom-types via device tree
		 * or properties is recommended anyhow.
		 */
		page_size = 1;      //当没有pagesize时，pagesize跌落为1

	flags = cdata->flags;
	if (device_property_present(dev, "read-only"))
		flags |= AT24_FLAG_READONLY;
	if (device_property_present(dev, "no-read-rollover"))
		flags |= AT24_FLAG_NO_RDROL;

	err = device_property_read_u32(dev, "address-width", &addrw);
	if (!err) {
		switch (addrw) {
		case 8:
			if (flags & AT24_FLAG_ADDR16)
				dev_warn(dev,
					 "Override address width to be 8, while default is 16\n");
			flags &= ~AT24_FLAG_ADDR16;
			break;
		case 16:
			flags |= AT24_FLAG_ADDR16;
			break;
		default:
			dev_warn(dev, "Bad \"address-width\" property: %u\n",
				 addrw);
		}
	}

	err = device_property_read_u32(dev, "size", &byte_len);
	if (err)
		byte_len = cdata->byte_len;

	if (!i2c_fn_i2c && !i2c_fn_block)
		page_size = 1;

	if (!page_size) {
		dev_err(dev, "page_size must not be 0!\n");
		return -EINVAL;
	}

	if (!is_power_of_2(page_size))
		dev_warn(dev, "page_size looks suspicious (no power of 2)!\n");

	err = device_property_read_u32(dev, "num-addresses", &num_addresses);
	if (err) {
		if (flags & AT24_FLAG_TAKE8ADDR)
			num_addresses = 8;
		else
			num_addresses =	DIV_ROUND_UP(byte_len,
				(flags & AT24_FLAG_ADDR16) ? 65536 : 256);
	}

	if ((flags & AT24_FLAG_SERIAL) && (flags & AT24_FLAG_MAC)) {
		dev_err(dev,
			"invalid device data - cannot have both AT24_FLAG_SERIAL & AT24_FLAG_MAC.");
		return -EINVAL;
	}

	regmap_config.val_bits = 8;
	regmap_config.reg_bits = (flags & AT24_FLAG_ADDR16) ? 16 : 8;
	regmap_config.disable_locking = true;

	regmap = devm_regmap_init_i2c(client, &regmap_config);
	if (IS_ERR(regmap))
		return PTR_ERR(regmap);

	at24 = devm_kzalloc(dev, struct_size(at24, client, num_addresses),
			    GFP_KERNEL);
	if (!at24)
		return -ENOMEM;

	mutex_init(&at24->lock);
	at24->byte_len = byte_len;
	at24->page_size = page_size;
	at24->flags = flags;
	at24->read_post = cdata->read_post;
	at24->num_addresses = num_addresses;
	at24->offset_adj = at24_get_offset_adj(flags, byte_len);
	at24->client[0].client = client;
	at24->client[0].regmap = regmap;

	at24->vcc_reg = devm_regulator_get(dev, "vcc");
	if (IS_ERR(at24->vcc_reg))
		return PTR_ERR(at24->vcc_reg);

	writable = !(flags & AT24_FLAG_READONLY);
	if (writable) {
		at24->write_max = min_t(unsigned int,
					page_size, at24_io_limit);      //单次写长度，是page_size和at24_io_limit中的最小值
		if (!i2c_fn_i2c && at24->write_max > I2C_SMBUS_BLOCK_MAX)
			at24->write_max = I2C_SMBUS_BLOCK_MAX;
	}

	/* use dummy devices for multiple-address chips */
	for (i = 1; i < num_addresses; i++) {
		err = at24_make_dummy_client(at24, i, &regmap_config);
		if (err)
			return err;
	}

	/*
	 * We initialize nvmem_config.id to NVMEM_DEVID_AUTO even if the
	 * label property is set as some platform can have multiple eeproms
	 * with same label and we can not register each of those with same
	 * label. Failing to register those eeproms trigger cascade failure
	 * on such platform.
	 */
	nvmem_config.id = NVMEM_DEVID_AUTO;

	if (device_property_present(dev, "label")) {
		err = device_property_read_string(dev, "label",
						  &nvmem_config.name);
		if (err)
			return err;
	} else {
		nvmem_config.name = dev_name(dev);
	}

	nvmem_config.type = NVMEM_TYPE_EEPROM;
	nvmem_config.dev = dev;
	nvmem_config.read_only = !writable;
	nvmem_config.root_only = !(flags & AT24_FLAG_IRUGO);
	nvmem_config.owner = THIS_MODULE;
	nvmem_config.compat = true;
	nvmem_config.base_dev = dev;
	nvmem_config.reg_read = at24_read;
	nvmem_config.reg_write = at24_write;
	nvmem_config.priv = at24;
	nvmem_config.stride = 1;
	nvmem_config.word_size = 1;
	nvmem_config.size = byte_len;

	i2c_set_clientdata(client, at24);

	err = regulator_enable(at24->vcc_reg);
	if (err) {
		dev_err(dev, "Failed to enable vcc regulator\n");
		return err;
	}

	/* enable runtime pm */
	pm_runtime_set_active(dev);
	pm_runtime_enable(dev);

	/*
	 * Perform a one-byte test read to verify that the
	 * chip is functional.
	 */
	err = at24_read(at24, 0, &test_byte, 1);
	if (err) {
		pm_runtime_disable(dev);
		if (!pm_runtime_status_suspended(dev))
			regulator_disable(at24->vcc_reg);
		return -ENODEV;
	}

	at24->nvmem = devm_nvmem_register(dev, &nvmem_config);
	if (IS_ERR(at24->nvmem)) {
		pm_runtime_disable(dev);
		if (!pm_runtime_status_suspended(dev))
			regulator_disable(at24->vcc_reg);
		return dev_err_probe(dev, PTR_ERR(at24->nvmem),
				     "failed to register nvmem\n");
	}

	/* If this a SPD EEPROM, probe for DDR3 thermal sensor */
	if (cdata == &at24_data_spd)
		at24_probe_temp_sensor(client);

	pm_runtime_idle(dev);

	if (writable)
		dev_info(dev, "%u byte %s EEPROM, writable, %u bytes/write\n",
			 byte_len, client->name, at24->write_max);
	else
		dev_info(dev, "%u byte %s EEPROM, read-only\n",
			 byte_len, client->name);

	return 0;
}
```