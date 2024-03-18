# 0. 重点概览：

1. 用户程序通过操作系统提供的API来使用操作系统的服务
2. kernel是一个操作系统对外提供服务的程序
3. 用户程序通过调用system call来调用kernel，kernel从而可以提供服务，而system call正是一系列API

# 1. System call 功能与使用
## 1.1 概览
![[Pasted image 20240213101741.png]]
## 1.2 详解
- fork(): 创建子进程, 子进程在创建的一刻将具备与父进程完全一致的 memory contents, i.e. instructions & data. 对于父进程而言, 该代码将会返回子进程的 PID. 对于子进程, 该代码返回0. 在调用该代码后, 该代码之后所有代码将会同时有父子两个进程去执行. 因此后续代码中可以通过 PID 来区分父子进程, e.g.: 
	```C
	int pid = fork();
    if(pid > 0){
      printf("parent: child=%d\n", pid);
      pid = wait((int *) 0);
      printf("child %d is done\n", pid);
    } else if(pid == 0){
      printf("child: exiting\n");
      exit(0);
    } else {
      printf("fork error\n");
    }
	```
	- 注意: fork尽管会copy父进程的数据, 但是若父进程读取了文件, 则指向该文件的文件描述符所附带的offset(用于指向当前读到的文件的位置)却是父子进程共享的.
- wait(int \*): 创建了子进程后, 父进程可能需要等待子进程结束运行. 此时可使用wait来阻塞. 该函数在子进程执行完毕后, 会返回子进程PID 若不存在子进程, 则直接返回-1. 通过向wait中传入int指针可以获得子进程的exit status. 如果不想关心子进程的exit status, 直接传入0即可. 例子见上文代码.
- exit(int): 用于关闭当前进程, 无返回. 传入的int参数即为上文wait中所说的exit status. 该int参数将会被传给wait.
- exec(char \* file, char \* argv): 读取file, 并将file替代当前进程的memory(*但 file descriptor table 会被保留*). 在xv6中, 该file是ELF格式.  e.g.:
	- 通常 argv[0]会被忽略. 这是因为它们通常是要调用的程序的名称.
	- 注意到由于 exec 会替换当前 process 的 memory, 因此通常我们需要开一个子进程来代替主进程做 exec 的事情.
```C
    char *argv[3];

    argv[0] = "echo";
    argv[1] = "hello";
    argv[2] = 0;
    exec("/bin/echo", argv);
    printf("exec error\n");
```
	
- open(char\* file, int flags): 打开file, 返回的是打开的file的文件描述符, 以flag对应的方式打开, flag可取以下值:
	```C
	#define O_RDONLY 0x000
	#define O_WRONLY 0x001
	#define O_RDWR 0x002
	#define O_CREATE 0x200
	#define O_TRUNC 0x400
```
	- shell程序需要使用stdin, stdout, stderr, 下面是它使用open来确保打开了三个文件描述符的过程: (*同时需要注意这段代码里我写的注释, 它解释了为什么open返回的fd是0,1,2, 而不是3,4,5, 亦或是0,4,,7*)
	```C
	while((fd = open("console", O_RDWR)) >= 0){
		if(fd >= 3){ // fd只能是0,1,2, 分别是stdIn, stdOut, stdErr. shell不需要再有更多的文件描述符了. 至于说为什么这里肯定是从0开始, 这是因为fd总是使用当前进程能使用的最小整数.(整数范围是从0开始的, 且此处"能使用"指的是该fd是unused.)
			close(fd);
			break;
		}
	}
```
-  read(int fd, char \*buf, int n), write(int fd, char \*buf, int n): 从fd指向的文件中read/write n个字节到buf中. 此处需要注意fd都自带了一个offset, 表示上一次对该文件read/write到了什么位置. 这两个函数分别返回读取/写入的字节数. 下述代码是从stdin中读取, 并且写入到stdout的代码:
	```C
char buf[512];  
int n;  
for (; ; ) {  
    n = read(0, buf, sizeof buf);  
    if (n == 0)  
        break;  
    if (n < 0) {  
        fprintf(2, "read error\n");  
        exit(1);  
    }  
    if (write(1, buf, n) != n) {  
        fprintf(2, "write error\n");  
        exit(1);  
    }  
}
```
	- 在当前没有数据时, read会一直等待数据的输入, 除非所有指向该文件的写端的文件描述符都被关闭了, 此时read才会结束并返回0.
- close(int fd): 释放一个fd, 即该fd这个数字又可以被该进程其他地方所使用. 此处需要理解: 最新分配的fd总是当前进程所能使用的非负整数中最小的那个.
- dup(int fd): 根据传入的文件描述符所指向的文件, 返回一个同样指向该文件的文件描述符. 注意该文件描述符附带的offset与原文件描述符附带的offset是共享的. 
	- 可以发现仅就文件描述符共享offset这一特征来看, 它与fork是一样的效果.
- chdir(char \*dir): 进入到dir对应的目录
- mkdir(char \*dir): 略
- mknod(char \*file, int , int): mknod能够创建一个特殊的指向device的文件. 此处传入的两个参数分别是major和minor device number, 通过他们可以唯一确定一个kernel device. 当一个进程后来打开一个设备文件时，内核将读写系统调用重定向到内核设备实现，而不是将它们传递给文件系统。
	```C
	// 上述三种system call的应用:
	mkdir("/dir");
    fd = open("/dir/file", O_CREATE|O_WRONLY);  // 创建文件
    close(fd);
    mknod("/console", 1, 1);
```
- fstat(int fd, struct stat* st): 将fd所指向的文件的metadata放入到结构体st中.
	- 此处需要注意, 既然该文件此时是有fd去指向的, 那么就意味着当前文件是open的. 因为一旦一个文件已经不被当前进程的任何fd所指向, 那么该文件必然是close了.
	- 何谓文件的metadata? 事实上, 文件与文件名是解耦的. 同一份底层文件(称为inode), 可以有多个名称, 这些名称称为link. 而inode中就存储着文件的metadata. metadata定义为一个结构体(在stat.h中), 长这样:
	```C
	#define T_DIR     1   // Directory
    #define T_FILE    2   // File
    #define T_DEVICE  3   // Device

    struct stat {
      int dev;     // File system’s disk device
      uint ino;    // Inode number
      short type;  // Type of file
      short nlink; // Number of links to file
      uint64 size; // Size of file in bytes

};
```
- link(char \*file1, char\* file2): 为file1这个名所代表的文件创建一个新的文件名file2. 其实此处的file1和file2就是底层文件inode的两个link.(inode与link定义见上文). e.g.:
	- 经过下面这端代码后, 该inode的nlink将会是2.
```C
	open("a", O_CREATE|O_WRONLY);
    link("a", "b");
```
- unlink(char* file): 移除对于file所对应的文件名的链接.
	- 所以如果某文件有两个link, 分别为a和b, 那么remove("a")后, 仍然还有一个b可以指向该文件.
	- 一个文件的inode与磁盘空间只有在它没有任何一个link, 且没有fd指向它时, 才会被释放.
## 1.3 System Call应用
1. 输入重定向:
	场景: shell程序希望将文件中的内容读取并输出到stdout. 此处可以看到输入不再是stdin(即键盘), 而是一个文件
	
	思路: 开一个子进程, 使其具备与主进程相同的stdout与stderr, 但其stdin要改为文件. 在此基础上, 通过exec执行cat程序. 因为cat的功能就是从stdin读取并输出到stdout.
	
	提示: fork之后, 子进程与父进程具备相同的file descriptor table. 且exec虽然会替换当前进程的memory, 但会保留file descriptor table.
	
	实现:  例如: cat < input.txt
	```C
	char *argv[2];

    argv[0] = "cat";
    argv[1] = 0;
    if(fork() == 0) {  // 对于子进程才需要执行这里面的代码
      close(0);
      open("input.txt", O_RDONLY);
      exec("cat", argv);

}
```
2. 输出重定向
	场景: shell程序希望将stderr重定向到stdout一样的位置
	
	思路: close(2)来关闭stderr, 而后利用dup(1)复制一个新的文件描述符, 此文件描述符由于此时最小可用非负整数就是刚刚释放的2, 所以会返回2, 但此时的2将会与1具有相同的目的地, 且共享offset.
	
3.  pipe的基本原理
	预备知识: 所谓pipe就是kernel所暴露出的一小段buffer, 进程们可以写入buffer, 另一个进程读取buffer, 从而实现进程间通信. 现有pipe(int p[2])函数提供, 他可以为p的两个元素分别赋值为pipe的两端的文件描述符, p[0]为read端, p[1]为write端.
	
	思路: 例如shell中的wc(word count)功能. 若父进程开出子进程, 将需要wc的数据发送给子进程, 让子进程执行wc函数. 那么父进程要向子进程传递消息, 则可如下实现:
	```C
	int p[2];
    char *argv[2];

    argv[0] = "wc";
    argv[1] = 0;

    pipe(p);
    if(fork() == 0) {
      close(0);
      dup(p[0]);  // 使得stdin(即fd为0)可以重新指向p[0]
      close(p[0]);
      close(p[1]);  // 为何此处要关闭p[1](即写端)? 因为read函数的特性是当前如果没有数据,则read函数会阻塞, 除非新的数据进来让read执行完自己的目标, 或者目标文件所有的写的fd都被关闭. 所以此处如果不关闭, 则read函数将永远看不到EOF, 从而也就无法返回
      exec("/bin/wc", argv);

    } else {
      close(p[0]);
      write(p[1], "hello world\n", 12);
      close(p[1]);

	}
```
	注意: 留意代码中的注释. 结合上文中对read的解释进行理解.
	补充: 其实, 似乎通过一个进程写入临时文件, 另一个进程读取临时文件也可以实现相同的功能, 但pipe相比于临时文件有以下优点:
	1. pipe可以自我clean, 而临时文件还需要小心地去控制删除
	2. pipe可以传递任意长的数据流, 而临时文件还需要磁盘上有足够的空间来存储数据.
	3. pipe允许并行, 但临时文件必须一方写完了, 另一方才能开始.
	4. 如果正在实现进程间通信，pipe的阻塞读写比文件的非阻塞语义更有效率。