1. 核心数据结构: pagetable_t, 它是用于指向 root pagetable page 的指针, 此处的 pagetable 指的可以是 kernel 的或是每个进程各自的 pagetable.
	具体定义为:
```C
typedef uint64 *pagetable_t; // root pagetable的指针, 每个层级的pagetable contains 512 PTEs
typedef uint64 pte_t;  // PTE
```

2. 重要函数 (vm. C):

	-  walk
	```C
		
	// Return the address of the PTE in page table pagetable
	// that corresponds to virtual address va.  If alloc!=0,
	// create any required page-table pages.
	// 说白了就是根据虚拟地址va去返回pagetable中PTE的地址
	
	//
	// The risc-v Sv39 scheme has three levels of page-table
	// pages. A page-table page contains 512 64-bit PTEs.
	// A 64-bit virtual address is split into five fields:
	//   39..63 -- must be zero.
	//   30..38 -- 9 bits of level-2 index.
	//   21..29 -- 9 bits of level-1 index.
	//   12..20 -- 9 bits of level-0 index.
	//    0..11 -- 12 bits of byte offset within the page.
	pte_t *
	walk(pagetable_t pagetable, uint64 va, int alloc)
	{
	  if(va >= MAXVA)
	    panic("walk");
	
	  for(int level = 2; level > 0; level--) {
	    pte_t *pte = &pagetable[PX(level, va)];// 由于pagetable本身就是一个指针,因此pagetable[xxx]表示pagetable指向的地址加上xxx * sizeof(uint64)(因为pagetable是一个指向uint64的指针). 而&号则表示获取该指针指向的地址.
	    if(*pte & PTE_V) {
	      pagetable = (pagetable_t)PTE2PA(*pte);
	    } else {
	      if(!alloc || (pagetable = (pde_t*)kalloc()) == 0)
	        return 0;
	      memset(pagetable, 0, PGSIZE);
	      *pte = PA2PTE(pagetable) | PTE_V;
	    }
	  }
	  return &pagetable[PX(0, va)];
	}
	```
	- mappages
	```C
	// Create PTEs for virtual addresses starting at va that refer to
	// physical addresses starting at pa. va and size might not
	// be page-aligned. Returns 0 on success, -1 if walk() couldn't
	// allocate a needed page-table page.

	// 说白了就是根据虚拟地址va与物理地址pa与size来创建PTE
	int
	mappages(pagetable_t pagetable, uint64 va, uint64 size, uint64 pa, int perm)
	{
	  uint64 a, last;
	  pte_t *pte;
	
	  a = PGROUNDDOWN(va);
	  last = PGROUNDDOWN(va + size - 1);
	  for(;;){
		if((pte = walk(pagetable, a, 1)) == 0)
		  return -1;
		if(*pte & PTE_V)
		  panic("remap");
		*pte = PA2PTE(pa) | perm | PTE_V;
		if(a == last)
		  break;
		a += PGSIZE;
		pa += PGSIZE;
	  }
	  return 0;
	}
	
	```

	- 其他 vm. C 中的函数:
		- Kvm 开头的函数全部用于操纵 kernel 的 pagetable
		- umv 开头的函数全部用于操纵 User 的pagetable
		- 其他函数是 kernel 与 user 都会用到
		- copyout 与 copyin 分别用于拷贝 to 与 from user 的虚拟地址中的数据 (这些数据是系统调用参数)

3.  流程:
	1. main 调用 kvminit (内部是 kvmmap)来创建 kernel 的 pagetable. 此时页表映射还没生效, 所以虚拟地址就是物理地址.
	```C
	/*
	 * the kernel's page table.
	 */
	pagetable_t kernel_pagetable;

	/*
	 * create a direct-map page table for the kernel.
	 */
	void
	kvminit()
	{
	  // 创建一个page用来hold root kernel pagetable
	  kernel_pagetable = (pagetable_t) kalloc();  
	  memset(kernel_pagetable, 0, PGSIZE);
	
	  // uart registers
	  kvmmap(UART0, UART0, PGSIZE, PTE_R | PTE_W);
	
	  // virtio mmio disk interface
	  kvmmap(VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);
	
	  // CLINT
	  kvmmap(CLINT, CLINT, 0x10000, PTE_R | PTE_W);
	
	  // PLIC
	  kvmmap(PLIC, PLIC, 0x400000, PTE_R | PTE_W);
	
	  // map kernel text executable and read-only.
	  kvmmap(KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);
	
	  // map kernel data and the physical RAM we'll make use of.
	  kvmmap((uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);
	
	  // map the trampoline for trap entry/exit to
	  // the highest virtual address in the kernel.
	  kvmmap(TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);
	}

	// add a mapping to the kernel page table.
	// only used when booting.
	// does not flush TLB or enable paging.
	void
	kvmmap(uint64 va, uint64 pa, uint64 sz, int perm)
	{
	  if(mappages(kernel_pagetable, va, sz, pa, perm) != 0)
	    panic("kvmmap");
	}

	// Allocate one 4096-byte page of physical memory.
	// Returns a pointer that the kernel can use.
	// Returns 0 if the memory cannot be allocated.
	void *
	kalloc(void)
	{
	  struct run *r;
	
	  acquire(&kmem.lock);
	  r = kmem.freelist;
	  if(r)
	    kmem.freelist = r->next;
	  release(&kmem.lock);
	
	  if(r)
	    memset((char*)r, 5, PGSIZE); // fill with junk
	  return (void*)r;
	}
		```
	2. 为每个进程分配 kernel stack. 使用 KSTACK 产生的 kernel stack 的虚拟地址将会被映射到虚拟地址上 (使用 kvmmap), 在映射时还会注意留出 invalid 的 guard page (由 KSTACK 保证)
	```C
	// map kernel stacks beneath the trampoline,
	// each surrounded by invalid guard pages.
	# define KSTACK(p) (TRAMPOLINE - ((p)+1)* 2*PGSIZE)

	// initialize the proc table at boot time.
	void
	procinit(void)
	{
		struct proc *p;
		
		initlock(&pid_lock, "nextpid");
		for(p = proc; p < &proc[NPROC]; p++) {  // NPROC是进程的最大数量(64个)
			initlock(&p->lock, "proc");
			
			// Allocate a page for the process's kernel stack.
			// Map it high in memory, followed by an invalid
			// guard page.
			char *pa = kalloc();
			if(pa == 0)
			panic("kalloc");
			uint64 va = KSTACK((int) (p - proc));
			kvmmap(va, (uint64)pa, PGSIZE, PTE_R | PTE_W);
			p->kstack = va;
			p->traceMask=0;
		}
		kvminithart();
	}

	// add a mapping to the kernel page table.
	// only used when booting.
	// does not flush TLB or enable paging.
	void
	kvmmap(uint64 va, uint64 pa, uint64 sz, int perm)
	{
	// 对 a range of 虚拟地址, 在pagetable中install它们到物理地址的映射
	// 以一个page的大小的间隔对所有范围内的虚拟地址一一映射
	  if(mappages(kernel_pagetable, va, sz, pa, perm) != 0)
	    panic("kvmmap");
	}

	// Switch h/w page table register to the kernel's page table,
	// and enable paging.
	void
	kvminithart()
	{
	  w_satp(MAKE_SATP(kernel_pagetable));
	  sfence_vma();  // 用于flush当前CPU的TLB(关于TLB是什么见下文)
	}
	```
	- main 调用 Kvminithart 来 install kernel page table. 它将 root pagetable page 的物理地址写入寄存器 satp 中 (具体代码见上文)
	- 每一个 CPU 都会把 PTE 缓存在 TLB (tranlation look-aside buffer)中. 而当 xv 6 修改 pagetable 时, 它都会让 CPU 无效化相应的缓存的 TLB 的 entry. (见上文的 sfence_vma). Sfence_vma 还被用于: Trampoline. S 中在回到 user space 前, 它会被调用, 从而把页表切换到 User pagetable
	```C
	.globl userret
	userret:
		# userret(TRAPFRAME, pagetable)
		# switch from kernel to user.
		# usertrapret() calls here.
		# a0: TRAPFRAME, in user page table.
		# a1: user page table, for satp.
	
		# switch to the user page table.
		csrw satp, a1
		sfence.vma zero, zero
	```
