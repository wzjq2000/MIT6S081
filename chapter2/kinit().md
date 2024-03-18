以下为 kinit ()的代码:
```C
extern char end[]; // first address after kernel.
                   // defined by kernel.ld.
                   
void
kinit()
{
  initlock(&kmem.lock, "kmem");
  freerange(end, (void*)PHYSTOP);
}
```
可以看出其完成了两步:
1. 初始化了 lock, 初步猜测这是为了避免多进程互相干扰?? TODO
2. 从 end 到 PHYSTOP 都清空了内存以供后续使用. 此处为重点:
	1. end 是 kernel 之后的第一个地址.
	2. PHYSTOP 的定义则在 kernel/memlayout. h 中. 具体为:
		```C
		// the kernel expects there to be RAM
		// for use by the kernel and user pages
		// from physical address 0x80000000 to PHYSTOP.
		#define KERNBASE 0x80000000L
		#define PHYSTOP (KERNBASE + 128*1024*1024)
		```
		此处可以看出, kernel 预期 RAM 中从 0 x 80000000 L 到 PHYSTOP (总共 128GB )都是可供 kernel 与 user page 使用的.
			那么从 0x 0 到 0 x 80000000 L 都是啥呢? -> 供 IO 设备使用