---
mermaid: true
hide: true
---

# 核心数据结构

start_kernel
    setup_arch
        setup_machine_fdt
            early_init_dt_scan
                early_init_dt_scan_nodes
                    early_init_dt_scan_memory
                        early_init_dt_add_memory_arch
                            memblock_add


mmap_mem
    phys_mem_access_prot
        pfn_valid
            memblock_is_map_memory
                memblock_search // 没有memory属性，就不在其中


```c
pgprot_t phys_mem_access_prot(struct file *file, unsigned long pfn,
			      unsigned long size, pgprot_t vma_prot)
{
	if (!pfn_valid(pfn))
		return pgprot_noncached(vma_prot);
	else if (file->f_flags & O_SYNC)
		return pgprot_writecombine(vma_prot);
	return vma_prot;
}
EXPORT_SYMBOL(phys_mem_access_prot);

bool __init_memblock memblock_is_map_memory(phys_addr_t addr)
{
	int i = memblock_search(&memblock.memory, addr);

	if (i == -1)
		return false;
	return !memblock_is_nomap(&memblock.memory.regions[i]);
}
```