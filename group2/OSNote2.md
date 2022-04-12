> 4月12日 Operating System Class Note  -- Topic : File System

 [TOC] 

## Review

### VFS

- Windows处理多个文件系统的做法：事实上以$NTFS$为主要系统，将不同的文件系统标上不同的宗卷号。处理文件时显式或隐式地使用该宗卷号会告诉操作系统将请求发送给对应的文件系统。并没有尝试将多个文件系统融为一个整体。

- 而Unix试图让从用户角度看是一个独立完整连续的文件系统层次，而底层杂七杂八的文件系统对用户是不可见的。这个统一的文件系统即为$Virtual\ File\ System$，在**内存**中处理不同文件系统的交互。

- $VFS$抽象出文件系统的共性，从而构造出一个$Layer$，包括Superblock、inode(in Memory)、dentry、file，向上给用户层提供统一的使用接口，向下对不同的文件系统有必须实现的接口和不同的调用组合以实现用户层次的接口。

  [关于上述四个对象的参考](https://bean-li.github.io/vfs-inode-dentry/)

  在$VFS$下，用户调用$open$得到的file descriptor仅仅是一个$int$，作为$file$的索引存在，而任何实现的具体细节都被隐藏。

- 所以任何一个文件系统在Unix中使用前，都必须向操作系统注册对应的$File\ System\ Type$、提供$VFS$要求的各种函数实现。

- $VFS$ 特点

  - 向上统一API
  - 向下兼容多种文件系统（包括远程NFS
  - 提供统一的目录树（*mount*)，同时对底层文件系统不可见

### New (about some optimization of FS)

#### Disk Management：Disk Block 对于 Disk utilization v.s. Disk rate 影响的 tradeoff

- Block 是硬盘操作的**基本单位**：每次会读一个Block的数据进入缓冲区，无论是否是“有效数据”
- 数据传输的时间相比于寻道时间(磁头沿径向移动寻找磁道，沿切向移动确定扇区)可以忽略不计，可以认为不同大小的块读取时间是相同的
- 固态硬盘的读写均有磨损，需要其“主控芯片”来进行**磨损均衡**（将数据不时进行搬运）；而磁盘的读操作磨损很小

#### 管理空闲Block

- 链表：在不同负荷下性能表现平稳（查找操作总在一个卡片当中完成）；占用内存开销保持较小，始终为一个卡片，剩下存入磁盘（占用磁盘开销大）。

  > 磁盘交换策略：将满的卡片分裂，其中一半放入磁盘。避免某些状况下频繁的磁盘内存交换。

- bitmap：在重负荷下性能差劲，且整张表都需要保存在内存当中。

> Supplement ：
>
> reference:stackoverflow
>
> **Linked list**
>
> pros: Small space requirements, as each block can store a pointer to the next free block.
>
> cons: To traverse the list, you need to read each block! Also, it is costly to maintain the list in a "contiguous" manner, in order to avoid fragmentation (think about the cost of updating the list in a smarter way than just appending each new free block at the end).
>
> The linked list scheme can be made slightly more efficient by storing multiple free blocks ID in each block (thus fewer I/Os are required to retrieve some amount of free blocks).
>
> **Bitmap**
>
> pros: Random allocation: checking if a block is free only requires reading the corresponding bit; also, checking for a large continuous section is relatively fast (as you can check a word size of bits from the bitmap in a single read). Fast deletion: you can just flip a bit to "free" a block, without overwriting the data.
>
> cons: Higher memory requirements, as you need one bit per block (~128MB for a 1TB disk with 1KB blocks).

in conclusion:

If the disk is almost full, it might make sense to use a linked list, as it will need less blocks than the bitmap. 

However, most of the time the bitmap will be store in main memory, which will make it more efficient than the linked list

- 注意在使用free list的时候，由于在大部分时间它时存储在磁盘上的，我们需要一些算法来避免IO操作的频发，比如上课说的那样在内存始终保持一个半满的free list 让磁盘里的free list比较满，又或者我们可以采取类似第六点中会提到的（每一个块都单独存储自己的inode），将磁盘中的free block存在相应的block中`storing multiple free blocks ID in each block ` 也可以减少IO操作

#### Disk Quota:

> 对于单一对象所占用的资源限制，避免资源被恶意挤占

- 每个文件有最大限制
- 每个用户有最大限制

#### Robustness

- Backup: 整体数据破坏的恢复；增量式备份

- Crash(整体未受影响，最近的几个操作执行错误)的恢复：$Journaling\ File\ Systems$

  > 一个操作往往并不能保证是原子性的，在其中间终止会导致文件系统级别的问题出现。提高鲁棒性的思路是：通过先将要做的事情(必需性质：**idempotent**)记录下来，再去做，若发现有pending中的任务则优先去做，从而达到一种伪原子性。这就是$Journaling\ File\ Systems$。

> 补充两个转储的算法
>
> **物理转储：**物理转储是从磁盘的第0块开始，将全部的磁盘块按序输出到磁带上，直到最后一块复制完毕。此程序很简单，可以确保万无一失，这是其他任何实用程序所不能比的。物理转储的主要优点是简单、极为快速（基本上是以磁盘的速度行）。
> **主要缺点：**既不能跳过选定的目录，也无法增量转储，还不能满足恢复个人文件的请求。
>
> **逻辑转储:**
>
> unix上广为使用
>
> 从一个或几个指定的目录开始，递归地转储其自给定基准日期后有所更改的全部文件和目录。所以，在逻辑转储中，转储磁带上会有一连串精心标识的目录和文件，这样就很容易满足恢复特定文件或目录的请求。
>
> ![304_9f0_8b9.png](https://s2.loli.net/2022/04/12/D2FJ4wHGolKeVsi.jpg)
>
> 1. 从起始目录开始(本例中为根目录)检查所有目录项，对每个修改过的文件在位图中标记，并递归标记所有目录(包括未被修改的)。
>    ![305_b2b_1e4.png](https://s2.loli.net/2022/04/12/F9UDqeCJnXYjuLw.png)再次递归遍历目录树，并去掉目录书中所有不含被修改过的文件或目录的目录上的标记(这一阶段结束后所有被标记的文件和目录就是最后必须被转储的)。
>    ![306_d09_ba3.png](https://s2.loli.net/2022/04/12/Y5szJnZxMCuVipf.png)
> 3. 以节点号为序，扫描i节点，并转储步骤2中所有标记的目录，并为这些目录添加目录属性前缀。
>    ![307_704_ae4.png](https://s2.loli.net/2022/04/12/oCk2NaZM1f8GFOy.png)
> 4. 转储步骤2中所有标记的文件，并为这些文件添加文件属性前缀。
>    ![308_dde_fb9.png](https://s2.loli.net/2022/04/12/w6es4OVRd3EpoUN.png)

#### Disk Consistency

- 未能保证文件系统“完全正确”
- 原因：Crash的发生/黑客的攻击
- $fsck$命令进行开机检查Consistency

具体有两种

> 1. 块的一致性
>
>    程序构造两张表，每张表为每一个块设立一个计数器，初始为0，第一个表中的计数器跟踪该块在文件中出现的次数，第二个表中的计数器跟踪该块在空闲表中出现的次数。
>
>    接着检验程序使用原始设备读取全部i结点，从0开始。由i节点开始，可以建立相应文件中采用的全部块的块号表。每当读到一个块时，该块在第一个表中计数器加1。然后，程序检查空闲表或空闲位图，每当一个块未使用时，在第二张表中加1。
>
>    检查完成后可能出现结果如下：
>
>    1. 一致，即每一块要么在第一个表计数器为1，要么在第二个表计数器为1。
>    2. 如果一个磁盘块在任何一张表中都是0，那么就称该块块丢失。解决方法是，检验程序将该块加入到空闲表中。
>    3. 一个块在空闲表中出现了两次，即一个块在第二张表中数字为2。解决方法是，重新建立空闲表。
>    4. 两个或多个文件中出现同一个数据块，即一个块在第一张表中数字为2。解决方法是，先分配一空闲块，将这个重复块的内容复制到空闲块中，然后把它插入到其中一个文件中。
>
> 2. 文件一致性
>
>    检验程序检查目录系统，同样创建一张计数表，但每个文件而不是一个块对应一个计数器。程序从根目录开始检验，沿着目录树递归，检查文件系统中的每个目录。对每个目录中的每个文件，将文件使用计数器加1。遇到硬连接，同样加一，但是符号连接不计数。
>
>    检验完成后，会得到一张由i节点号索引的表，说明每个文件被多少个目录包含。然后，检验程序将这些数字与存储在文件i节点中的连接数目比较。
>
>    最终可能得到的结果如下：
>
>    1. i节点连接数与计数器相等，则说明文件系统一致
>    2. i节点连接数大于计数器个数，说明即使所有该文件都被删除，计数仍是非0，i节点不会被删除。解决方法是，将i节点的值更新为计数器的值。
>    3. i节点连接数小于计数器个数。可能导致一个目录指向错误i节点。解决方法同样是更新i节点值

#### Reducing Disk Arm Motion

- 关于磁盘结构的复习

  > <img src="https://s2.loli.net/2022/04/12/UpjZAGtiawgqJLh.png" alt="image-20220412133824907.png" style="zoom:33%;" />
  >
  > 每个盘面有两面，称为表面($surface$)，每个表面是由一组称为磁道($track$)的同心圆组成，每个磁道被化成一组扇区($sector$),每个扇区包含相同数量的数据位(通常是512 byte),磁盘制造商通常用柱面（$cylinder$）来描述多个盘片驱动的构造，柱面是所有盘片表面上到主轴中心距离相等的磁道的集合，这整个装置叫做 旋转磁盘($rotation  $ $disk$),与固态硬盘($SSD$) 区分，后者没有移动部分

- 第一种可以提升文件系统性能的技术是用 $bitmap$ 或者 $free$ $list$ 来记录空闲块：这种技术的目的是把所有可能顺序访问的块放在一起，最好是同一个柱面，从而减少磁盘臂移动的次数。

- 具体实现： 可以把 $bitmap$ 整个存在内存里面 或者 用 $block $ $clustering$ 技术来来优化磁盘上的$free$ $list$  ,`The trick is to keep track of disk storage not in blocks, but in groups of consecutive blocks ` .

- 第二个性能瓶颈是，读文件需要两次磁盘访问 ，分别需要读$i-node$ 和文件真实存储的地址，由此引出了第二种可以提升文件系统性能的技术

- 关于磁盘上inode

  > 文件储存在硬盘上，硬盘的最小存储单位叫做"扇区"（Sector）。每个扇区储存512字节（相当于0.5KB）。
  >
  > 操作系统读取硬盘的时候，不会一个个扇区地读取，这样效率太低，而是一次性连续读取多个扇区，即一次性读取一个"块"（block）。这种由多个扇区组成的"块"，是文件存取的最小单位。"块"的大小，最常见的是4KB，即连续八个 sector组成一个 block。
  >
  > 文件数据都储存在"块"中，那么很显然，我们还必须找到一个地方储存文件的元信息，比如文件的创建者、文件的创建日期、文件的大小等等。这种储存文件元信息的区域就叫做inode，中文译名为"索引节点"。
  >
  > 每一个文件都有对应的inode，里面包含了与该文件有关的一些信息。

  > inode也会消耗硬盘空间，所以硬盘格式化的时候，操作系统自动将**硬盘分成两个区域**。一个是**数据区**，存放文件数据；另一个是**inode区**（inode table），存放inode所包含的信息。
  >
  > 每个inode节点的大小，一般是128字节或256字节。inode节点的总数，在格式化时就给定，一般是每1KB或每2KB就设置一个inode。假定在一块1GB的硬盘中，每个inode节点的大小为128字节，每1KB就设置一个inode，那么inode table的大小就会达到128MB，占整块硬盘的**12.8%。**

- 通常情况下的做法<img src="https://s2.loli.net/2022/04/12/lBqRnMwZAdQCbX8.png" alt="image-20220412162822504.png" style="zoom:25%;" />

​		全部$inode$ 存储在靠近磁盘开始的地方，$inode$ 和它指向的块的距离是柱面数的一半，寻道时间比较长

- 改进方法<img src="https://s2.loli.net/2022/04/12/mKAU2fvg3pVbnuR.png" alt="image-20220412163112898.png" style="zoom:25%;" />

  

在磁盘的中部存储$inode$ 或者向上图一样，将磁盘分成多个柱面组，每个柱面组有自己的$inode$ ,$free $ $list$ 和数据，这样的情况下，创建好文件后，我们任选一个柱面的$inode$ 去看这个柱面是否还存在空闲的块，如果没有的话就去相邻的柱面寻找空闲块。

**但是以上的讨论都是基于有磁盘臂的磁盘的情况，当我们使用SSD时不会有这样的问题**



reference:MOS

edited by :yichuan wang yilong zhao yijia hong
