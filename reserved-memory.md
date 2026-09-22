---
title: reserved-memory
date: 2026-09-15 16:32:30
tags:
    - reserved-memory
    - device_type = "memory"
    - linux
    - mmap
---

# 环境
- linux：5.10.227  
- CPU：arm cortex-a53  

# reserved-memory与device_type = "memory"

```dts
reserved-memory {
        #address-cells = <2>;
        #size-cells = <2>;
        ranges;
        rmem: rmem_0x800000000 {
                device_type = "memory";	//memory must be accessible 
                reg = 	<0x00000000 0x01000000 0x00000000 0x100000>;
        };

};
```
- device_type：设备树中device_type是memory属性的节点，都会被early_init_dt_scan_memory处理。通过memblock_add加入memblock.memory中
```c
int __init early_init_dt_scan_memory(unsigned long node, const char *uname,
				     int depth, void *data)
{
	const char *type = of_get_flat_dt_prop(node, "device_type", NULL);
	const __be32 *reg, *endp;
	int l;
	bool hotpluggable;

	/* We are scanning "memory" nodes only */
	if (type == NULL || strcmp(type, "memory") != 0)
		return 0;

	reg = of_get_flat_dt_prop(node, "linux,usable-memory", &l);
	if (reg == NULL)
		reg = of_get_flat_dt_prop(node, "reg", &l);
	if (reg == NULL)
		return 0;

	endp = reg + (l / sizeof(__be32));
	hotpluggable = of_get_flat_dt_prop(node, "hotpluggable", NULL);

	pr_debug("memory scan node %s, reg size %d,\n", uname, l);

	while ((endp - reg) >= (dt_root_addr_cells + dt_root_size_cells)) {
		u64 base, size;

		base = dt_mem_next_cell(dt_root_addr_cells, &reg);
		size = dt_mem_next_cell(dt_root_size_cells, &reg);

		if (size == 0)
			continue;
		pr_debug(" - %llx ,  %llx\n", (unsigned long long)base,
		    (unsigned long long)size);

		early_init_dt_add_memory_arch(base, size);

		if (!hotpluggable)
			continue;

		if (early_init_dt_mark_hotplug_memory_arch(base, size))
			pr_warn("failed to mark hotplug range 0x%llx - 0x%llx\n",
				base, base + size);
	}

	return 0;
}

void __init __weak early_init_dt_add_memory_arch(u64 base, u64 size)
{
	const u64 phys_offset = MIN_MEMBLOCK_ADDR;

	if (size < PAGE_SIZE - (base & ~PAGE_MASK)) {
		pr_warn("Ignoring memory block 0x%llx - 0x%llx\n",
			base, base + size);
		return;
	}

	if (!PAGE_ALIGNED(base)) {
		size -= PAGE_SIZE - (base & ~PAGE_MASK);
		base = PAGE_ALIGN(base);
	}
	size &= PAGE_MASK;

	if (base > MAX_MEMBLOCK_ADDR) {
		pr_warn("Ignoring memory block 0x%llx - 0x%llx\n",
			base, base + size);
		return;
	}

	if (base + size - 1 > MAX_MEMBLOCK_ADDR) {
		pr_warn("Ignoring memory range 0x%llx - 0x%llx\n",
			((u64)MAX_MEMBLOCK_ADDR) + 1, base + size);
		size = MAX_MEMBLOCK_ADDR - base + 1;
	}

	if (base + size < phys_offset) {
		pr_warn("Ignoring memory block 0x%llx - 0x%llx\n",
			base, base + size);
		return;
	}
	if (base < phys_offset) {
		pr_warn("Ignoring memory range 0x%llx - 0x%llx\n",
			base, phys_offset);
		size -= phys_offset - base;
		base = phys_offset;
	}
	memblock_add(base, size);
}

```

# 使用device_type = "memory"的原因
- 在linux应用中，使用了/dev/mem设备，映射后访问保留内存。在使用memset清零时，触发异常。原因是内存属性是device，与memset中的指令冲突
- 使用memory属性后，phys_mem_access_prot中pfn_valid会返回真，不会将属性设置为pgprot_noncached（device属性）
- 在mmap参数中使用O_SYNC，会将内存属性设置为普通内存（MT_NORMAL_NC）


```c
#define pgprot_noncached(prot) \
	__pgprot_modify(prot, PTE_ATTRINDX_MASK, PTE_ATTRINDX(MT_DEVICE_nGnRnE) | PTE_PXN | PTE_UXN)
#define pgprot_writecombine(prot) \
	__pgprot_modify(prot, PTE_ATTRINDX_MASK, PTE_ATTRINDX(MT_NORMAL_NC) | PTE_PXN | PTE_UXN)

static int mmap_mem(struct file *file, struct vm_area_struct *vma)
{
	size_t size = vma->vm_end - vma->vm_start;
	phys_addr_t offset = (phys_addr_t)vma->vm_pgoff << PAGE_SHIFT;

	/* Does it even fit in phys_addr_t? */
	if (offset >> PAGE_SHIFT != vma->vm_pgoff)
		return -EINVAL;

	/* It's illegal to wrap around the end of the physical address space. */
	if (offset + (phys_addr_t)size - 1 < offset)
		return -EINVAL;

	if (!valid_mmap_phys_addr_range(vma->vm_pgoff, size))
		return -EINVAL;

	if (!private_mapping_ok(vma))
		return -ENOSYS;

	if (!range_is_allowed(vma->vm_pgoff, size))
		return -EPERM;

	if (!phys_mem_access_prot_allowed(file, vma->vm_pgoff, size,
						&vma->vm_page_prot))
		return -EINVAL;

	vma->vm_page_prot = phys_mem_access_prot(file, vma->vm_pgoff,
						 size,
						 vma->vm_page_prot);

	vma->vm_ops = &mmap_mem_ops;

	/* Remap-pfn-range will mark the range VM_IO */
	if (remap_pfn_range(vma,
			    vma->vm_start,
			    vma->vm_pgoff,
			    size,
			    vma->vm_page_prot)) {
		return -EAGAIN;
	}
	return 0;
}

pgprot_t phys_mem_access_prot(struct file *file, unsigned long pfn,
			      unsigned long size, pgprot_t vma_prot)
{
	if (!pfn_valid(pfn))
		return pgprot_noncached(vma_prot);
	else if (file->f_flags & O_SYNC)
		return pgprot_writecombine(vma_prot);
	return vma_prot;
}

int pfn_valid(unsigned long pfn)
{
	phys_addr_t addr = pfn << PAGE_SHIFT;

	if ((addr >> PAGE_SHIFT) != pfn)
		return 0;

#ifdef CONFIG_SPARSEMEM
	if (pfn_to_section_nr(pfn) >= NR_MEM_SECTIONS)
		return 0;

	if (!valid_section(__pfn_to_section(pfn)))
		return 0;

	/*
	 * ZONE_DEVICE memory does not have the memblock entries.
	 * memblock_is_map_memory() check for ZONE_DEVICE based
	 * addresses will always fail. Even the normal hotplugged
	 * memory will never have MEMBLOCK_NOMAP flag set in their
	 * memblock entries. Skip memblock search for all non early
	 * memory sections covering all of hotplug memory including
	 * both normal and ZONE_DEVICE based.
	 */
	if (!early_section(__pfn_to_section(pfn)))
		return pfn_section_valid(__pfn_to_section(pfn), pfn);
#endif
	return memblock_is_map_memory(addr);
}
EXPORT_SYMBOL(pfn_valid);

bool __init_memblock memblock_is_map_memory(phys_addr_t addr)
{
	int i = memblock_search(&memblock.memory, addr);        //查找是否被memblock.memory管理

	if (i == -1)
		return false;
	return !memblock_is_nomap(&memblock.memory.regions[i]); //如果使用了no-map，一样会被设置为device属性
}

static inline bool memblock_is_nomap(struct memblock_region *m)
{
	return m->flags & MEMBLOCK_NOMAP;
}
```

# device_type = "memory"与ioremap_wc
- 使用memory属性后，将不能使用ioremap_wc在内核中进行映射，同样是因为pfn_valid的检查
```c
#define ioremap(addr, size)		__ioremap((addr), (size), __pgprot(PROT_DEVICE_nGnRE))
#define ioremap_wc(addr, size)		__ioremap((addr), (size), __pgprot(PROT_NORMAL_NC))

void __iomem *__ioremap(phys_addr_t phys_addr, size_t size, pgprot_t prot)
{
	return __ioremap_caller(phys_addr, size, prot,
				__builtin_return_address(0));
}
EXPORT_SYMBOL(__ioremap);


static void __iomem *__ioremap_caller(phys_addr_t phys_addr, size_t size,
				      pgprot_t prot, void *caller)
{
	unsigned long last_addr;
	unsigned long offset = phys_addr & ~PAGE_MASK;
	int err;
	unsigned long addr;
	struct vm_struct *area;

	/*
	 * Page align the mapping address and size, taking account of any
	 * offset.
	 */
	phys_addr &= PAGE_MASK;
	size = PAGE_ALIGN(size + offset);

	/*
	 * Don't allow wraparound, zero size or outside PHYS_MASK.
	 */
	last_addr = phys_addr + size - 1;
	if (!size || last_addr < phys_addr || (last_addr & ~PHYS_MASK))
		return NULL;

	/*
	 * Don't allow RAM to be mapped.
	 */
	if (WARN_ON(pfn_valid(__phys_to_pfn(phys_addr))))       //在此处进行了检查
		return NULL;

	area = get_vm_area_caller(size, VM_IOREMAP, caller);
	if (!area)
		return NULL;
	addr = (unsigned long)area->addr;
	area->phys_addr = phys_addr;

	err = ioremap_page_range(addr, addr + size, phys_addr, prot);
	if (err) {
		vunmap((void *)addr);
		return NULL;
	}

	return (void __iomem *)(offset + addr);
}
```


# device_type = "memory"不应当使用
- device_type：设备树中device_type是memory属性的节点，都会被early_init_dt_scan_memory处理。通过memblock_add加入memblock.memory中
- 在没有no-map时，pfn_valid一定返回真，导致无法进行ioremap映射
- 考虑使用自定义字符设备，在设备树中添加属性，自由控制内存属性