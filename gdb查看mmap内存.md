---
title: gdb查看mmap内存
date: 2026-09-16 15:30:46
tags:
    - gdb
    - mmap
---

# 环境
- linux：5.10.227  
- CPU：arm cortex-a53  
- IDE：ARM Development Studio 2022.1

# mmap内存无法访问
- [gdb](https://stackoverflow.com/questions/654393/examining-mmaped-addresses-using-gdb)无法访问mmap后的地址，案例
- 在应用中使用/dev/mem设备，通过mmap进行地址映射
- 映射后的地址cpu读写正常
- 在终端中通过devmem命令可以访问
- 在IDE Memory窗口中无法访问

# gdb与ptrace
- 在通过IDE Memory窗口访问mmap内存时，是通过gdb命令访问
- gdb的调试机制（ptrace）限制了映射后的物理内存的访问
- 在内核中已经预留了访问的接口，需要打开配置CONFIG_HAVE_IOREMAP_PROT
- 打开后即可通过generic_access_phys接口访问
- generic_access_phys接口，在mmap_mem函数中注册到vm_ops

    ```c
    static const struct vm_operations_struct mmap_mem_ops = {
    #ifdef CONFIG_HAVE_IOREMAP_PROT
        .access = generic_access_phys
    #endif
    };

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
    ```
- ptrace调用流程
```text
COMPAT_SYSCALL_DEFINE4(ptrace, compat_long_t, request, compat_long_t, pid,
		       compat_long_t, addr, compat_long_t, data)

ptrace
    compat_arch_ptrace(arm64)
        compat_ptrace_request
            ptrace_access_vm
                __access_remote_vm  //主要关注此函数
                    vma->vm_ops->access（generic_access_phys）

```
```c
int __access_remote_vm(struct task_struct *tsk, struct mm_struct *mm,
		unsigned long addr, void *buf, int len, unsigned int gup_flags)
{
	struct vm_area_struct *vma;
	void *old_buf = buf;
	int write = gup_flags & FOLL_WRITE;

	if (mmap_read_lock_killable(mm))
		return 0;

	/* ignore errors, just check how much was successfully transferred */
	while (len) {
		int bytes, ret, offset;
		void *maddr;
		struct page *page = NULL;

		ret = get_user_pages_remote(mm, addr, 1,
				gup_flags, &page, &vma, NULL);      //检查VM_IO 和 VM_PFNMAP标志，remap_pfn_range映射的内存会被标记有该标志（/dev/mem通过该方式做mmap）
		if (ret <= 0) {
#ifndef CONFIG_HAVE_IOREMAP_PROT
			break;
#else
			/*
			 * Check if this is a VM_IO | VM_PFNMAP VMA, which
			 * we can access using slightly different code.
			 */
			vma = find_vma(mm, addr);
			if (!vma || vma->vm_start > addr)
				break;
			if (vma->vm_ops && vma->vm_ops->access)
				ret = vma->vm_ops->access(vma, addr, buf,
							  len, write);  //调用generic_access_phys访问内存，因VM_IO 和 VM_PFNMAP标志/dev/mem，mmap映射的内存在gdb访问时走此通道
			if (ret <= 0)
				break;
			bytes = ret;
#endif
		} else {
			bytes = len;
			offset = addr & (PAGE_SIZE-1);
			if (bytes > PAGE_SIZE-offset)
				bytes = PAGE_SIZE-offset;

			maddr = kmap(page);
			if (write) {
				copy_to_user_page(vma, page, addr,
						  maddr + offset, buf, bytes);
				set_page_dirty_lock(page);
			} else {
				copy_from_user_page(vma, page, addr,
						    buf, maddr + offset, bytes);
			}
			kunmap(page);
			put_page(page);
		}
		len -= bytes;
		buf += bytes;
		addr += bytes;
	}
	mmap_read_unlock(mm);

	return buf - old_buf;
}
```

# bug: mmap reserved-memory device_type:memory
- 节点使用device_type:memory属性，结果是该节点会被memblock管理，如果节点没有使用no-map;则进行pfn_valid检查时，一定为真
- 只有当pfn_valid为真时，在/dev/mem设备的 mmap中才能将内存属性配置为PROT_NORMAL_NC，否则为PROT_DEVICE_nGnRE
- 当pfn_valid为真时，一定无法使用__ioremap，因为__ioremap需要pfn_valid为假，才会进行映射
- mmap的generic_access_phys是用于在用户态进行mmap内存查看所需的函数，该函数一样会对pfn_valid进行检查，只有当pfn_valid为假时才进行映射等操作
- 可以发现，当使用memory属性时，就必然与gdb查看mmap内存相冲突

# patch
- 开启HAVE_IOREMAP_PROT选项后需要修改部分代码，否则编译报错
- arch/arm64/include/asm/io.h 中加入 ioremap_prot
```c
#define ioremap(addr, size)		__ioremap((addr), (size), __pgprot(PROT_DEVICE_nGnRE))
#define ioremap_wc(addr, size)		__ioremap((addr), (size), __pgprot(PROT_NORMAL_NC))

+ #define ioremap_prot(addr, size, port)	__ioremap((addr), (size), __pgprot(port))
```
- arch/arm64/include/asm/pgtable.h 加入pte_pgprot
```c
#define pgprot_nx(prot) \
	__pgprot_modify(prot, PTE_MAYBE_GP, PTE_PXN)

+ static inline pgprot_t pte_pgprot(pte_t pte)
+ {
+ 	unsigned long pfn = pte_pfn(pte);
+ 
+ 	return __pgprot(pte_val(pfn_pte(pfn, __pgprot(0))) ^ pte_val(pte));
+ }
```
- 在arch/arm64/Kconfig中新增HAVE_IOREMAP_PROT
```
	select THREAD_INFO_IN_TASK
	+ select HAVE_IOREMAP_PROT
```
- 移除arch/arm64/mm/hugetlbpage.c中的pte_pgprot
```c
/*
 * Select all bits except the pfn
 */
- static inline pgprot_t pte_pgprot(pte_t pte)
- {
- 	unsigned long pfn = pte_pfn(pte);
- 
- 	return __pgprot(pte_val(pfn_pte(pfn, __pgprot(0))) ^ pte_val(pte));
- }
```

