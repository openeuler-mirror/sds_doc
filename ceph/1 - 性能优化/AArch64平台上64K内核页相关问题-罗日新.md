# AArch64平台上64K内核页相关问题

## 64K内核页引入
在第一个REHL/Centos使能AArch64平台的版本（7.x）（印象中不保证正确），就默认设置内核页大小为64K，具体原因估计是想获得更好的性能。使用64K内核页，能够较大地减少应用程序需要占用的页表条目数，在相同硬件页表条目数的情况下，能够提升tlb的命中率，提升程序的性能。更大的内核页，带来了更大粒度的内存分配，可能带来了大量的内存浪费，需要权衡。

应用程序基于性能考虑，有时需要申请完整的内核页大小的内存；网络收发包过程中需要完整的内核页；Linux PageCache需要完整的内核页；Linux存储设备中元数据管理需要完整的内核页等等。在需要完整的内核页的地方都可能会出现一些问题，由此可能带来轻则内存浪费，严重时出现内存泄漏，可能引发程序/内核的crash，更致命的存在引发数据的丢失风险。这些问题都需要特别的考虑。

由于X86平台上Linux 4K内核页的大规模使用，有些应用程序在设计之初就只考虑了页大小为4K的情况；有些则可能考虑了其他的页大小的情况，但是存在考虑不周并缺乏实际验证的情况。在将这些软件移植到AArch64平台64K内核页下时，就容易出现一些兼容性问题。下面列举之前实际碰到的一些问题，供参考。

## 内存浪费/内存泄露
应用程序一般使用glibc或者其他库（如tcmalloc）对内存进行管理。这些内存管理程序一般对小块内存分配采取化零为整的方式，以内核页大小为单位从内核申请释放内存，然后内部切分成不同的粒度进行管理，以获取较高的性能和内存利用率。

以gperftools中的tcmalloc为例（google在github上两个内存管理项目TCMalloc和gperftools，区别[参见](https://google.github.io/tcmalloc/gperftools.html)，下文中tcmalloc表示gperftools中的tcmalloc），tcmalloc提供了一个可以配置自身页大小的选项TPS（为了叙述方便下面以KP表示内核页，KPS表示内核页大小，TP表示tcmalloc中的页，TPS表示tcmalloc中配置的页大小），TPS默认为8K，可选配置为4K，8K，16K，32K，64K，128K，256K，一般配置为KPS的整数倍。在tcmalloc中以TPS为单位从内核申请释放内存，理论上讲，更大的TPS能够减少从内核申请内存的开销，有着更好的性能；更大的TPS被切分给更多生命周期不一样的对象，被释放的概率就会越小，带来更大的内存浪费。
### Ceph中tcmalloc内存无法释放
由于tcmalloc出色的性能，良好的管理接口，开源分布式存储软件Ceph中，使用tcmalloc作为内存管理库，一般发行版默认TPS为8K，当KPS为64K时，存在着无法将内存归还给操作系统的问题，内存一直在tcmalloc的freelist中，导致内存一直增长，具体原因是tcmalloc对64K内核页欠考虑，内存释放逻辑有些问题，tcmalloc释放内存给操作系统是以KPS为单位进行的，并且在扫描freelist过程中碰到第一个无法释放的内核页时，就不再对freelist中其他的内存页进行扫描，释放过程就此结束。1个KP包含了8个TP，KP被释放的条件8个TP都是可释放的，8个TP同时满足释放条件概率会比较小，就会发生内存无法释放的问题。修复方案为可以为重新配置TPS为KPS的整数倍或者修改内存释放逻辑，参见[PR1](https://github.com/gperftools/gperftools/pull/1201)
[PR2](https://github.com/gperftools/gperftools/pull/1269)，当前gperftools社区考虑到稳定性和当前维护情况，此问题并没有合入解决。需要自行修改。

### Ceph中页对齐带来的内存浪费
在Ceph中对数据和元数据都进行的内存缓存，缓存有着一定的大小，超出时会进行部分释放，为了追求性能，有些元数据的分配上采用了页对齐的方式申请了内存，但实际上只使用了部分的大小，而预留的部分就浪费了。这些元数据都被放在元数据缓存中，浪费的内存会导致实际缓存的数据减少，影响性能；特别地，如果配置不合理导致浪费空间过大，就会频繁启动内存释放进程，干扰io执行，严重影响性能。参见[PR](https://github.com/ceph/ceph/pull/38701)。

## Superblock数据问题
一般地，存储设备都需要对其superblock写入数据，来存储文件系统元数据等信息，superblock的大小一般为4K，存放在一个KP上，KPS为4K时，superlock数据刚好在KP的0偏移位置，当KPS为64K时，superblock在KP上的偏移位置就不太确定了，此时写入就需要特别的处理；superblock的buffer/direct的写入方式也可能会导致superblock写入不正确，出现superblock数据丢失的问题。
### Bcache superblock写入错误，重启机器后bcache分区无法挂载
在linux内核中使用 __bread将特定的块映射到一个KP中，在KPS为4K时，superblock默认放在偏移为0的位置。当KPS为64K时，superblock就可能
位于页上的非0的偏移的位置，写入superblock的时候，如果还是按0位置写就会出现问题。具体参见[链接1](https://lkml.indiana.edu/hypermail/linux/kernel/1912.0/05134.html) [链接2]( https://listman.redhat.com/archives/dm-devel/2016-July/027545.html)。
该问题在linux 5.6版本上得以修复。

### Ceph上OSD的superblock的写入丢失，导致初始化失败
Ceph中每个OSD对自己管理的存储设备会在设备的起始位置写入4K(设备标签)+4K(superblock)的元数据，它两物理上不一定连续，但都在头64K范围。当KPS为64K时，需要写入的8K的元数据刚好在一个KP上，在写入过程，4K设备标签元数据是采用buffer write的方式写入，4Ksuperblock元数据采用direct write的方式写入。当出现先direct write 写入4K superblock，随后buffer write 写入4K设备标签数据时，buffer write的数据会先写到PageCache，随后由内核flush到设备中，flush的时候时以完整的页写入的，就会导致direct write的4Ksuperblock数据被覆盖，导致superblock丢失，初始化失败。参见[PR](https://github.com/ceph/ceph/pull/48092)。

## BufferIO导致磁盘上写放大，可能影响SSD寿命
一般地，Linux上对文件或者设备默认都时采用BufferIO的形式，通过PageCache对设备上的数据进行缓存，以达到提高性能的目的。通过BufferIO的方式写数据到设备，需要先写到PageCache中，随后由内核flush到设备。当KPS为64K时，如果每次都只改写页中少量字节，但是flush都会刷入64K的数据，相比KPS为4K时，会对设备带来额外的写入操作，对SSD设备的寿命带来不利的影响，开启BufferIO时可能需要一并考虑此问题。

## 总结
采用64K内核页能够带来一定程度的性能提升，但是也对现有的软件的兼容性有一定的影响，在获得性能红利的同时，需要仔细考虑并解决相关问题。
