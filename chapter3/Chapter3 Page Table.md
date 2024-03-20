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
  - 细节如图：![image-20240320124405207](C:\Users\andy2000\AppData\Roaming\Typora\typora-user-images\image-20240320124405207.png)
  - 图中可见，最后一个层级的PPN（physical page number）与物理地址的前44位完全一致。
  - 每个层级的pagetable中的PTE(pagetable entry)的数量都是2^9(=512)个. 这对应于前文所说的9-9-9.
  - 