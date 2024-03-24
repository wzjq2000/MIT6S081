# 0 . 前言
1. 无论是 kernel 或是 user 都使用的是虚拟地址.
2. RISC-V 使用 page table hardware 来做虚拟地址到物理地址的映射.

# 1. Paging Hardware

- RISC-V基于64位机器，因此其虚拟地址或是物理地址都用64位来表示。但RISC-V：
  - 对于虚拟地址而言：只采用了其后39位。前25位闲置。
  - 对于物理地址而言：只采用了后56位
- 虚拟地址的后39位可分为：9-9-9-12
  - 这里的三个9分别用于在不同层级的page table上进行寻址
  - 这里的12表示偏移量，与物理地址的末12位完全一样。并且可以注意到由于偏移量只有12位，故最多可以从0偏移到2^12 -1, 因此同一个PPN可以管理总共2^12(=4096)个字节,即4KB, 这正是一个page的大小
  - 细节如图：![[Pasted image 20240324151932.png]]
  - 图中可见，最后一个层级的PPN（physical page number）与物理地址的前44位完全一致。
  - 每个层级的pagetable中的PTE(pagetable entry)的数量都是2^9(=512)个. 这对应于前文所说的9-9-9.

# 2. Kernel Virtual Address Space
任意一个进程, 都具备 1 个一个用于用户虚拟地址空间 (见 #用户的虚拟地址空间  )的页表. 

而对于 kernel 来说, 它全局只有一个页表, 用于 kernel 虚拟地址空间 (见 [[kernel的虚拟地址空间]]). kernel 的虚拟地址空间如下:
![Pasted image 20240317112342.png](app://d16c9988c2976e9be965135c43431680fcaf/Users/andy/SelfLearn/ObsidianVaultsForComputerScience/OS/6S081_OperatingSystem/Pasted%20image%2020240317112342.png?1710645822755) 
可以看到:
- kernel 预期的可用的 RAM (同时被 kernel 与用户程序使用)范围是: 从 KERNBASE 到 PHYSTOP
- kernel 自身 (KERNBASE)是进行了直接映射 (即物理地址=虚拟地址)
- 但 kernel 中仍有一些地址是真正虚拟的地址, 并且由于这些地址所指向的对象本身处于 kernel 中, 从而它们本就具备了直接映射. 因此这些地址相当于被映射了两次 (一次直接映射, 一次通过页表转化来映射), 这些地址是:
	- trampoline page: 它位于 kernel 虚拟地址的顶端. 此外还需注意用户地址空间也有该映射.
	- kenel stack pages: 它位于 Trampoline page 的下方, 且多个 kernel stack pages 之间用 Guard Page 来分隔. (所谓的 Guard Page 是 PTE_V 没有 set 的 page). 当有数据进入 guard page, kernel 就会 panic.
- 其他相对不重要的细节:
	- MAXVA=2^38-1, 这是因为虚拟地址只采用了 64 位中的后 39 位, 而 xv6 又只采用了其中的 38 位 (源码中的解释: MAXVA is actually one bit less than the max allowed by Sv 39, to avoid having to sign-extend virtual addresses that have the high bit set.)
	- 0x80000000 以下是各类设备. 它们也是直接映射.
	- 关于 PHYSTOP 为什么是 0x86400000->因为程序中有定义: PHYSTOP=KERNBASE+128\*1024\*1024. 可以看到这个范围正好是 2^27, 正好与上文的 9-9-9-12 对应. -> 推测原因: 由于上文的 9-9-9 表明了三层级的页表总共可表示 2^27 个 PTE, 而 kernel 所能看到的虚拟地址空间正好也是 2^27 个字节, 则这里的每个字节是否可以看作一个 page 的起始地址? (因为一个 page 正好是 2^12 ). 从而这也就解释了为什么 kernel 的操作粒度是一个 page.