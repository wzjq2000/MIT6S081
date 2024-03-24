## 1. 操作系统三大功能: Isolation, Multiplexing, Interaction

- 如何实现?
	- 阻止用户进程直接访问硬件资源, 而是将硬件资源以 service 的形式提供出去, 如 open, read, write 等
- 这种实现方式的好处:
	- 用户程序所看到的是 file system (以 path name 的形式展示), 而不是直接接触 disk, 所以编写起来更加方便
	- 操作系统可以统筹安排 disk 分配, 更好地利用 disk 空间
	- 操作系统可以让多个进程分享 CPU (如果是进程直接操纵 CPU, 可能会出现占着 CPU 不放的情况), 从而做到了multiplexing
	- 实现了 Isolation, 避免一个进程的失败影响其他进程
	- 进程对于 IO 操作只需要关心 file descriptor 即可, 无需考虑底层细节, 方便了进程间的Interaction
- 那么如何该实现的底层原理是什么?
	- 见 2

# 2. 操作系统三大 Mode: Machine, Supervisor, User

1. 为了阻止用户进程直接访问硬件资源, ==CPU 提供了硬件层级的支持==, 将执行代码的权利分成了三个层级:
	1. Machine Mode: 该 mode 下指令具备完全的优先级. CPU 在这种模式下启动. Xv 6 只会执行几行 Machine Mode 下的代码, 然后便会进入 Supervisor Mode.
	2. Supervisor Mode: 该 mode 下的指令具备优先级. 例如读写寄存器, 启停 interrupts 等. 用户进程如果试图执行这个 mode 下的代码, 则 CPU 会切换到 Supervisor Mode 来关闭该应用.
	3. User Mode
2. 一个进程既可以进入 Supervisor Mode, 也可以进入 User Mode
	- ==SuperVisor Mode 时可以执行优先的代码, 此时称为该进程在 kernel space 中运行==
	- ==User Mode 中运行时, 称为该进程在 user space 中运行== 
3. 三大 Mode 如何互相进入? -> 使用 CPU 提供的特殊指令, 此处以 RISC-V 为例:
	1. user mode -> supervisor mode: ecall
		- mode 转换的入口 (即 entry point)由 kenel 来控制
		- 当用户代码需要调用 system call 时, 内部就会使用 ecall (见 usys.S). ecall 将会升高 hardware privilege level 并把 program counter 改变为 kernel 所控制的 entry point. entry point 的代码将切换到 kernel stack (这是什么? 见 #kernel_stack ) 并执行 kernel instruction 来执行 system call
	2. supervisor mode -> user mode: sret
		- 当 system call 执行结束, 进程会回到 user stack, 然后通过 sret 指令回到 user space, 这将会 lower hardware privilege 并恢复执行 user instruction
	3. machine mode -> supervisor mode: mret, `mret` 是 RISC-V 指令集中的一条指令，用于从机器模式（Machine Mode）的异常或中断处理程序返回到之前的特权级别。RISC-V 架构定义了多种特权级别，包括用户模式（User Mode, U-Mode）、监管者模式（Supervisor Mode, S-Mode）、机器模式（Machine Mode, M-Mode）等。


## 3. Kernel 的两大结构: Monolithic 与 Micro Kernel

- 分为两类结构的原因:
	- 操作系统的多大部分需要在 supervisor mode 下执行?
- 两类结构的区分标准:
	- monolithic: 整个操作系统都运行在 supervisor mode 下
		- 缺点: 操作系统各个部分的接口复杂, 使得操作系统开发者容易写bug
		- 优点: 无需考虑哪些指令需要放在 User mode; 各个操作系统的部分交互更加方便
		- 由于 xv6 是 monolithic, 因此其所有 kernel interface 都与操作系统的 interface 对应
	- micro: 操作系统基本功能运行在 supervisor mode 下, 其他运行在 user mode 下
	


## 4. Kernel 提供的函数所在的文件描述
![[Pasted image 20240224182058.png]]

## 5. 实现进程的机制
1. supervisor/user mode flag
2. 线程的时间分片
3. 地址空间
	- 每个进程都会看到一个从零开始的地址空间, 为他一人所用
	- Xv 6 使用了硬件提供的页表功能来将虚拟地址映射到真实的内存地址
	- 每个进程各自都有一个页表, 具体页表构造见: #TODO
	- 每个进程所能看到的虚拟地址空间 (注意此处的图是 #用户的虚拟地址空间 , 并非 kernel 的虚拟地址空间)长这样: (与之形成对比的是 kernel 的虚拟地址空间: [[kernel的虚拟地址空间]]) ![[Pasted image 20240224182724.png]]
	- 地址由低到高分别是:
		- instruction
		- global variable
		- user stack
		- heap (用于 malloc)
		- trapframe
		- trampoline
	- 由于 RISC-V 上的指针大小为 64 位, 而硬件只使用低 39 位来寻址, 而 xv6 又只使用其中的 38 位, 故 MAXVA = 2^38-1=0 x 3 fffffffff

## 6. 进程
1. 进程结构体 (见 proc. h 的 86 行) 定义:
```C
// Per-process state
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state: 当前进程的状态,分为: the process is allocated, ready to run, waiting for IO, exiting
  struct proc *parent;         // Parent process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table,其format为RISC-V的硬件所规定
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
};
```

2. 线程的状态, (如局部变量, 方法的返回地址)存放在线程栈上.
3. 每个进程有两种栈:
	1. user stack
	2. kernel stack (即上文代码块的 kstack) #kernel_stack
		- 当进程执行 user instruction, 其 kernel stack 为空.
		- 当进程进入 kernel 执行 (调用了 system call 或者 interrupt), 则 user stack 中仍然存储数据 (但不再被调用), 而 kernel stack 则被启用
		- kernel stack 从 user code 中是分离的, 从而 user stack 崩溃不会影响 kernel stack

## 7. xv 6 的启动流程和第一个进程
1. 当 RISC-V 启动时, 它将会自我初始化并运行一个 boot loader (该 loader 存储在一个只读内存上)
2. boot loader 加载 xv6 的 kernel 进内存, 物理地址为 0x 8000000 (具体细节见 [[kinit()]])
	- 为何使用这个地址而不是 0x 0? 因为 0x 0:0x 8000000 要给 IO 设备使用
3. 进入 machine mode, CPU 执行 xv6 的以下代码 (\_entry), 此时 paging hardware 尚未生效, 故物理地址与虚拟地址完全一样:
```assembly
_entry:
	# set up a stack for C.
        # stack0 is declared in start.c,
        # with a 4096-byte stack per CPU.
        # sp = stack0 + (hartid * 4096)
        la sp, stack0  # 将标签 `stack0` 的地址加载到栈指针 `sp` 中
        li a0, 1024*4  # 将立即数 `4096`（即 `1024*4`，代表 4KB）加载到寄存器 `a0` 中
		csrr a1, mhartid  # 从 CSR（控制和状态寄存器）中读取当前硬件线程的ID（`mhartid`），并将其存入寄存器 `a1` 中。
        addi a1, a1, 1  # 将寄存器 `a1` 的值加 `1`，这是为了计算当前硬件线程的栈空间的起始地址。这样做是因为 `stack0` 已经是为第一个硬件线程（Hart 0）预留的，所以为了得到当前线程的栈起始地址，需要将线程ID加 `1`
        mul a0, a0, a1  # 将寄存器 `a0`（每个线程的栈空间大小）与寄存器 `a1`（经过加 `1` 处理的线程ID）相乘，结果存回 `a0`。这个操作计算出了从 `stack0` 开始，当前线程应有的栈偏移量
        add sp, sp, a0  # 将寄存器 `sp`（包含 `stack0` 的地址）与寄存器 `a0`（栈偏移量）相加，更新 `sp` 的值。这样，每个硬件线程的栈指针 `sp` 都指向了一个唯一的栈空间，避免了不同线程之间的栈空间冲突
	# jump to start() in start.c
        call start
```

此处为后续执行 C 代码开辟了一个 stack, 大小为 4096byte. 后续的 start.c 中声明了 initial stack (变量名为 stack0) 的空间.

```C
// entry.S needs one stack per CPU.
__attribute__ ((aligned (16))) char stack0[4096 * NCPU];
```

在开辟了 stack 后, \_entry 加载了 stack pointer register sp 到地址 stack0+4096 (kernel/start.c:11). 这是在栈顶, 这是因为 RISC-V 的 stack grows down.
这样一来,kernel 就有 stack 了, 从而后续可以执行 C 代码 (即 start. c) (kernel/start.c:21).
4.  Start 函数在 machine mode 下执行了一些配置的操作, 随后切换到了 supervisor mode. 但此处并不是立刻使用 mret 这个命令返回的. 而是做了以下动作:
	1. 将寄存器 mstatus 的值从之前的 privilege mode 改为了 supervisor mode. 而 mret 的作用正是从 machine mode 回到之前的 privilege mode. 因此这一步是后续调用 mret 的基础.
	2. 修改了寄存器 mepc 中的值, 将其改为 main 函数的地址, 从而函数的返回地址变成了main
	3. 在页表寄存器 satp 中写入 0, 从而关闭了 supervisor mode 中的虚拟地址转换功能
	4. 将所有中断和异常委托给 supervisor mode
5. 进入 supervisor mode 之前, start 函数对 clock chip 进行编排, 从而产生 timer interrupts.
6. start 函数调用 mret来 "return"到 supervisor mode. 这会让程序计数器转到 main 函数上. (kernel/main.c:11).
7. 在 main 方法对多个 devices 和子系统初始化后 (这其中包含了 kinit 函数来创建 physical page allocator , 具体细节见 [[kinit()]] ), 它会调用 userinit (kernel/proc.c:212) 来创建第一个进程
```C
// a user program that calls exec("/init")
// od -t xC initcode
uchar initcode[] = {
  0x17, 0x05, 0x00, 0x00, 0x13, 0x05, 0x45, 0x02,
  0x97, 0x05, 0x00, 0x00, 0x93, 0x85, 0x35, 0x02,
  0x93, 0x08, 0x70, 0x00, 0x73, 0x00, 0x00, 0x00,
  0x93, 0x08, 0x20, 0x00, 0x73, 0x00, 0x00, 0x00,
  0xef, 0xf0, 0x9f, 0xff, 0x2f, 0x69, 0x6e, 0x69,
  0x74, 0x00, 0x00, 0x24, 0x00, 0x00, 0x00, 0x00,
  0x00, 0x00, 0x00, 0x00
};
// Set up first user process.
void
userinit(void)
{
  struct proc *p;

  p = allocproc();
  initproc = p;
  
  // allocate one user page and copy init's instructions
  // and data into it.
  uvminit(p->pagetable, initcode, sizeof(initcode));
  p->sz = PGSIZE;

  // prepare for the very first "return" from kernel to user.
  p->trapframe->epc = 0;      // user program counter
  p->trapframe->sp = PGSIZE;  // user stack pointer

  safestrcpy(p->name, "initcode", sizeof(p->name));
  p->cwd = namei("/");

  p->state = RUNNABLE;

  release(&p->lock);
}
	```
8.  而 userinit 则会执行 initcode. S, 这会调用 exec 系统调用 (从上面的代码段中可以看出采用的是 exec("/init")) 从而重新进入 kernel. exec 会替换当前进程的 memory 与寄存器. kernel 运行结束后, 就会回到 user space 执行 user/init.c: 15 来开启控制台. 系统从而启动了.
