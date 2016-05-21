##磁盘文件复制

###描述ucore操作系统中“磁盘文件复制”从请求到完成的整个执行过程，并分析I/O过程的时间开销。



磁盘文件复制在用户态需要完成以下操作：

打开源文件
读取源文件
打开目标文件
写入目标文件

即进行两次open和一次read和一次write

流程：

用户态： 打开：open() -> sys_open() -> syscall(SYS_OPEN, ……) 读写：read/write()-> sys_read/write() -> syscall(SYS_READ/WRITE, ……)

上面的过程主要还是做一个从用户态到内核态的切换(利用中断处理例程)

内核态： 打开：sys_open()->sysfile_open()->file_open()； 读写：sys_read/write() -> sysfile_read/write()-> file_read/write()；

抽象层： 打开：file_open() -> vfs_open()； 读写：file_read/write() ->　vfs_read/write()；

SFS层： 打开：sfs_opendir()->sfs_openfile()->sfs_load_inode()； 读写：sfs_load_inode->sfs_io()->sfs_r/wbuf->sfs_rwblock_nolock; 

分三种情况进行读写： 读写起始位置所在的block 读写起始位置到终止位置中间的block 读写终止位置所在block 注意到后面就没有open的事情了。open只是做一个磁盘映射

I/O层： dop_io();disk0_io() :检验参数 -> disk0_read_blks_nolock() 调用驱动函数，连续读取若干个block

驱动层： ide_read_secs() :读取扇区；


仔细观察ide_read_secs()的实现，可以发现这里是采用CPU轮询的方式来进行 磁盘的读写，因此主要的I/O时间开销都在这一部分操作。尝试采用计时的方法来考察这部分的时间开销，但发现ucore内部实现的计时函数只有基于时钟中断的计数，粒度太粗，无法精细的记录这部分时间开销。