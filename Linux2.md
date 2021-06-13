

# 文件系统

主要需要弄清楚一下几个问题：

- 第一点，文件系统要有严格的组织形式，使得文件能够以块为单位进行存储
- 第二点，文件系统中要有索引区，用来方便查找一个文件分成的多个块都存放在了什么位置
- 第三点，如果文件系统中有的文件是热点文件，近期经常被读取和写入，文件系统应该有缓存层
- 第四点，文件应该用文件夹的形式组织起来，方便管理和查询
- 第五点，Linux 内核要在自己的内存里面维护一套数据结构，来保存哪些文件被哪些进程打开和使用



## 硬盘文件系统

### inode 与块的存储

硬盘分成相同大小的单元，我们称为**块**（Block）。一块的大小是扇区大小的整数倍**，默认是 4K**。在格式化的时候，这个值是可以设定的。一大块硬盘被分成了一个个小的块，用来存放文件的数据部分。



**inode**

但是这也带来一个新的问题，那就是文件的数据存放得太散，找起来就比较困难。我们是不是可以像图书馆那样，也设立一个索引区域，用来维护“某个文件分成几块、每一块在哪里“等等这些**基本信息**?另外，文件还有**元数据**部分，例如名字、权限等，这就需要一个结构**inode**来存放。

inode 的“i”是 index 的意思，其实就是“索引”，类似图书馆的索引区域。既然如此，我们**每个文件都会对应一个 inode**；一个文件夹就是一个文件，也对应一个 inode。

```c
struct ext4_inode {
	__le16	i_mode;		/* File mode */
	__le16	i_uid;		/* Low 16 bits of Owner Uid */
	__le32	i_size_lo;	/* Size in bytes */
	__le32	i_atime;	/* Access time */
	__le32	i_ctime;	/* Inode Change time */
	__le32	i_mtime;	/* Modification time */
	__le32	i_dtime;	/* Deletion Time */
	__le16	i_gid;		/* Low 16 bits of Group Id */
	__le16	i_links_count;	/* Links count */
	__le32	i_blocks_lo;	/* Blocks count */
	__le32	i_flags;	/* File flags */
......
	__le32	i_block[EXT4_N_BLOCKS];/* Pointers to blocks */
	__le32	i_generation;	/* File version (for NFS) */
	__le32	i_file_acl_lo;	/* File ACL */
	__le32	i_size_high;
......
};
```

- inode 里面有文件的读写权限 i_mode，属于哪个用户 i_uid，哪个组 i_gid，大小是多少 i_size_io，占用多少个块 i_blocks_io。咱们讲 ls 命令行的时候，列出来的权限、用户、大小这些信息，就是从这里面取出来的。

- 另外，这里面还有几个与文件相关的时间。i_atime 是 access time，是最近一次访问文件的时间；i_ctime 是 change time，是最近一次更改 inode 的时间；i_mtime 是 modify time，是最近一次更改文件的时间。

  这里需要注意区分几个地方

  - 首先，访问了，不代表修改了，也可能只是打开看看，就会改变 access time。
  - 其次，修改 inode，有可能修改的是用户和权限，没有修改数据部分，就会改变 change time。
  - 只有数据也修改了，才改变 modify time。



我们刚才说的“某个文件分成几块、每一块在哪里”，这些在 inode 里面，应该保存在 i_block 里面。

具体如何保存的呢？EXT4_N_BLOCKS 一共有 15 项。在 ext2 和 ext3 中，其中前 12 项直接保存了块的位置，也就是说，我们可以通过 i_block[0-11]，直接得到保存文件内容的块。

<img src="截图/Linux/inode块结构.jpeg" alt="下载" style="zoom:25%;" />

但是，如果一个文件比较大，12 块放不下。当我们用到 i_block[12] 的时候，就不能直接放数据块的位置了，要不然 i_block 很快就会用完了。我们可以让 i_block[12] 指向一个块，这个块里面不放数据块，而是放数据块的位置，这个块我们称为**间接块**。也就是说，我们在 i_block[12] 里面放间接块的位置，通过 i_block[12] 找到间接块后，间接块里面放数据块的位置，通过间接块可以找到数据块。

如果文件再大一些，i_block[13] 会指向一个块，我们可以用二次间接块。二次间接块里面存放了间接块的位置，间接块里面存放了数据块的位置，数据块里面存放的是真正的数据。如果文件再大一些，i_block[14] 会指向三次间接块。原理和上面都是一样的，就像一层套一层的俄罗斯套娃，一层一层打开，才能拿到最中心的数据块。

这里面有一个非常显著的问题，对于大文件来讲，我们要多次读取硬盘才能找到相应的块，这样访问速度就会比较慢。为了解决这个问题，ext4 做了一定的改变。它引入了一个新的概念，叫作**Extents**。

我们来解释一下 Extents。比方说，一个文件大小为 128M，如果使用 4k 大小的块进行存储，需要 32k 个块。如果按照 ext2 或者 ext3 那样散着放，数量太大了。但是 Extents 可以用于存放连续的块，也就是说，我们可以把 128M 放在一个 Extents 里面。这样的话，对大文件的读写性能提高了，文件碎片也减少了。

<img src="截图/Linux/inode_block_extend.jpeg" alt="下载" style="zoom: 33%;" />

Exents 其实会保存成一棵树。树有一个个的节点，有叶子节点，也有分支节点。每个节点都有一个头，ext4_extent_header 可以用来描述某个节点。

```c
struct ext4_extent_header {
	__le16	eh_magic;	/* probably will support different formats */
	__le16	eh_entries;	/* number of valid entries */
	__le16	eh_max;		/* capacity of store in entries */
	__le16	eh_depth;	/* has tree real underlying blocks? */
	__le32	eh_generation;	/* generation of the tree */
};
```

- eh_entries 表示这个节点里面有多少项。

  这里的项分两种

  - 如果是叶子节点，这一项会直接指向硬盘上的连续块的地址，我们称为数据节点 ext4_extent；
  - 如果是分支节点，这一项会指向下一层的分支节点或者叶子节点，我们称为索引节点 ext4_extent_idx。

  这两种类型的项的大小都是 12 个 byte。

```c
/*
 * This is the extent on-disk structure.
 * It's used at the bottom of the tree.
 */
struct ext4_extent {
	__le32	ee_block;	/* first logical block extent covers */
	__le16	ee_len;		/* number of blocks covered by extent */
	__le16	ee_start_hi;	/* high 16 bits of physical block */
	__le32	ee_start_lo;	/* low 32 bits of physical block */
};
/*
 * This is index on-disk structure.
 * It's used at all the levels except the bottom.
 */
struct ext4_extent_idx {
	__le32	ei_block;	/* index covers logical blocks from 'block' */
	__le32	ei_leaf_lo;	/* pointer to the physical block of the next *
				 * level. leaf or next index could be there */
	__le16	ei_leaf_hi;	/* high 16 bits of physical block */
	__u16	ei_unused;
};
```

- 如果文件不大，inode 里面的 i_block 中，可以放得下一个 ext4_extent_header 和 4 项 ext4_extent。所以这个时候，eh_depth 为 0，也即 inode 里面的就是叶子节点，树高度为 0。

- 如果文件比较大，4 个 extent 放不下，就要分裂成为一棵树，eh_depth>0 的节点就是索引节点，**其中根节点深度最大**，在 inode 中。最底层 eh_depth=0 的是叶子节点。

除了根节点，其他的节点都保存在一个块 4k 里面，4k 扣除 ext4_extent_header 的 12 个 byte，剩下的能够放 340 项，每个 extent 最大能表示 128MB 的数据，340 个 extent 会使你的表示的文件达到 42.5GB。这已经非常大了，如果再大，我们可以增加树的深度。





### inode 位图和块位图















# 常用命令

// todo 文章后面的留下的问题中还有大量的命令需要积累。



## TOP

> [linux top命令查看内存及多核CPU的使用讲述](https://www.cnblogs.com/dragonsuc/p/5512797.html)

Linux top命令用于实时显示 process 的动态。

**参数说明**：

- d : 改变显示的更新速度，或是在交谈式指令列( interactive command)按 s
- q : 没有任何延迟的显示速度，如果使用者是有 superuser 的权限，则 top 将会以最高的优先序执行
- c : 切换显示模式，共有两种模式，一是只显示执行档的名称，另一种是显示完整的路径与名称
- S : 累积模式，会将己完成或消失的子行程 ( dead child process ) 的 CPU time 累积起来
- s : 安全模式，将交谈式指令取消, 避免潜在的危机
- i : 不显示任何闲置 (idle) 或无用 (zombie) 的行程
- n : 更新的次数，完成后将会退出 top
- b : 批次档模式，搭配 "n" 参数一起使用，可以用来将 top 的结果输出到档案内
- -p: 监控特定的PID，你可以用-p选项监控指定的PID。PID的值为0将被作为top命令自身的PID。



例如：

`top -c`在查看命令（最后一栏）那一栏可以看到完成的执行命令路径。

`top -Hp 3033` 显示一个进程的线程运行信息列表。按下P,进程按照Cpu使用率排序





## PS

process status命令用于显示当前进程的状态，类似于 windows 的任务管理器。

**参数说明**：

- ps 的参数非常多, 在此仅列出几个常用的参数并大略介绍含义

- -A 列出所有的进程

- ps -e 此参数的效果和指定"A"参数相同

- -w 显示加宽可以显示较多的资讯

- -au 显示较详细的资讯

- -aux 显示所有包含其他使用者的行程

- -f 带有完整的命令行路径

- au(x) 输出格式 :

  ```
  USER PID %CPU %MEM VSZ RSS TTY STAT START TIME COMMAND
  ```

  - USER: 行程拥有者
  - PID: pid
  - %CPU: 占用的 CPU 使用率
  - %MEM: 占用的记忆体使用率
  - VSZ: 占用的虚拟记忆体大小
  - RSS: 占用的记忆体大小
  - TTY: 终端的次要装置号码 (minor device number of tty)
  - STAT: 该行程的状态:
    - D:  无法中断的休眠状态 (通常 IO 的进程)
    - R: 正在执行中
    - S: 静止状态
    - T: 暂停执行
    - Z: 不存在但暂时无法消除
    - W: 没有足够的记忆体分页可分配
    - <: 高优先序的行程
    - N: 低优先序的行程
    - L: 有记忆体分页分配并锁在记忆体内 (实时系统或捱A I/O)
  - START: 行程开始时间
  - TIME: 执行的时间
  - COMMAND:所执行的指令

  

最常用的`ps -ef|grep keyword`：显示所有命令，连带命令行



## 端口相关

> [Linux 查看端口占用情况](https://www.runoob.com/w3cnote/linux-check-port-usage.html)

### netstat

Linux netstat 命令用于显示网络状态。

**参数说明**：

- -a或--all   显示所有连线中的Socket。
- -A<网络类型>或--<网络类型>   列出该网络类型连线中的相关地址。
- -c或--continuous   持续列出网络状态。
- -C或--cache   显示路由器配置的快取信息。
- -e或--extend   显示网络其他相关信息。
- -F或--fib   显示路由缓存。
- -g或--groups   显示多重广播功能群组组员名单。
- -h或--help   在线帮助。
- -i或--interfaces   显示网络界面信息表单。
- -l或--listening   显示监控中的服务器的Socket。
- -M或--masquerade   显示伪装的网络连线。
- -n或--numeric   直接使用IP地址，而不通过域名服务器。
- -N或--netlink或--symbolic   显示网络硬件外围设备的符号连接名称。
- -o或--timers   显示计时器。
- -p或--programs   显示正在使用Socket的程序识别码和程序名称。
- -r或--route   显示Routing Table。
- -s或--statistics   显示网络工作信息统计表。
- -t或--tcp   显示TCP传输协议的连线状况。
- -u或--udp   显示UDP传输协议的连线状况。
- -v或--verbose   显示指令执行过程。
- -V或--version   显示版本信息。
- -w或--raw   显示RAW传输协议的连线状况。
- -x或--unix   此参数的效果和指定"-A unix"参数相同。
- --ip或--inet   此参数的效果和指定"-A inet"参数相同。



`netstat -tunlp|grep port`：其实可以展示tcp的源和目标ip:port，然后grep去查找



### lsof

lsof(list open files)是一个列出当前系统打开文件的工具。

**参数说明**：

-a 列出打开文件存在的进程

-c<进程名> 列出指定进程所打开的文件

-g 列出GID号进程详情

-d<文件号> 列出占用该文件号的进程

+d<目录> 列出目录下被打开的文件

+D<目录> 递归列出目录下被打开的文件

-n<目录> 列出使用NFS的文件

-i<条件> 列出符合条件的进程。（4、6、协议、:端口、 @ip ）

-p<进程号> 列出指定进程号所打开的文件

-u 列出UID号进程详情

-h 显示帮助信息

-v 显示版本信息



`lsof -i:端口号`：lsof  查看端口占用

```sh
# lsof -i:8000
COMMAND   PID USER   FD   TYPE   DEVICE SIZE/OFF NODE NAME
nodejs  26993 root   10u  IPv4 37999514      0t0  TCP *:8000 (LISTEN)
```

lsof输出各列信息的意义如下：

COMMAND：进程的名称 PID：进程标识符

USER：进程所有者

FD：文件描述符，应用程序通过文件描述符识别该文件。如cwd、txt等 TYPE：文件类型，如DIR、REG等

DEVICE：指定磁盘的名称

SIZE：文件的大小

NODE：索引节点（文件在磁盘上的标识）

NAME：打开文件的确切名称



## ip相关

![img](截图/Linux/nettools vs iproute2.png)

## df

disk free 命令用于显示目前在 Linux 系统上的文件系统磁盘使用情况统计。

**参数说明**：

- 文件-a, --all 包含所有的具有 0 Blocks 的文件系统
- 文件--block-size={SIZE} 使用 {SIZE} 大小的 Blocks
- 文件-h, --human-readable 使用人类可读的格式(预设值是不加这个选项的...)
- 文件-H, --si 很像 -h, 但是用 1000 为单位而不是用 1024
- 文件-i, --inodes 列出 inode 资讯，不列出已使用 block
- 文件-k, --kilobytes 就像是 --block-size=1024
- 文件-l, --local 限制列出的文件结构
- 文件-m, --megabytes 就像 --block-size=1048576
- 文件--no-sync 取得资讯前不 sync (预设值)
- 文件-P, --portability 使用 POSIX 输出格式
- 文件--sync 在取得资讯前 sync
- 文件-t, --type=TYPE 限制列出文件系统的 TYPE
- 文件-T, --print-type 显示文件系统的形式
- 文件-x, --exclude-type=TYPE 限制列出文件系统不要显示 TYPE
- 文件-v (忽略)
- 文件--help 显示这个帮手并且离开
- 文件--version 输出版本资讯并且离开

`df -h`



## free

free指令会显示内存的使用情况，包括实体内存，虚拟的交换文件内存，共享内存区段，以及系统核心使用的缓冲区等。

**参数说明**：

- -b 　以Byte为单位显示内存使用情况。

- -k 　以KB为单位显示内存使用情况。

- -m 　以MB为单位显示内存使用情况。

- -h 　以合适的单位显示内存使用情况，最大为三位数，自动计算对应的单位值。单位有：

  ```
  B = bytes
  K = kilos
  M = megas
  G = gigas
  T = teras
  ```

- -o 　不显示缓冲区调节列。

- -s<间隔秒数> 　持续观察内存使用状况。

- -t 　显示内存总和列。

- -V 　显示版本信息。

`free -h`

