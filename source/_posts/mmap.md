---
title: Linux内存映射机制
date: 2020-06-22 17:03:22
categories:
- Theories
- Assembler
tags:
- PWN
---
## Introduction

&emsp;&emsp;在做PWN题的时候接触到内存映射，就去网上学习了一下Linux的内存映射机制，在这里做个笔记。不是很全面，仅仅是目前做题需要了解的知识。

<!-- more -->

## MMU

&emsp;&emsp;MMU是内存管理单元，主要功能有两个：

* 负责虚拟地址到物理地址的转换

* 提供硬件机制的内存访问权限检查

&emsp;&emsp;没有启动或者没有MMU时，外设（包括物理内存）等所有部件使用的都是物理地址，CPU通过物理地址来访问外设（包括物理内存）。启动MMU后，CPU核心对外发出虚拟地址给MMU，MMU把虚拟地址转换成物理地址，最后通过物理地址读取实际设备。

&emsp;&emsp;虚拟地址转换成物理地址的方法有：

* 确定的公式转换

* 用表格存储虚拟地址对应的物理地址

&emsp;&emsp;MMU内存映射机制实现不同的进程均可以访问所有的用户空间，同时不同的进程（页目录、页表不一样所以映射到的是不同的物理内存地址）又可以保存自己的私有数据。

## mmap && mprotect

&emsp;&emsp;文件的内存映射可以理解为将文件想象成数据，把这块数据同程序中的一块内存对应起来，当操作这块内存的时候，就实际上在操作这个文件。

&emsp;&emsp;mmap函数将文件映射到进程地址空间，实现直接访问文件内容的功能。

&emsp;&emsp;read/write函数在底层实际上是调用copy_to_user/copy_from_user来实现，而实现过程是通过数据复制完成的。mmap函数因为建立了映射关系，可直接访问数据，所以提高了效率。

```C
#include <sys/mman.h>
void* mmap(void* addr, size_t len, int port, int flag, int filedes, off_t offset）
```

&emsp;&emsp;返回值：成功时返回被映射的内存地址，失败返回MAPP_FAILED。

&emsp;&emsp;参数解释：

* addr：这个参数告诉内核使用addr指定的值来映射指定文件的起始地址，只有极少数情况下不为0。当指定为0的时候，告诉内核返回什么地址内其自身决定。

* len：指定被映射的内存区域的长度。

* port：对应open函数的权限位：

```Code
PROT_READ  映射区可读
PROT_WRITE 映射区可写
PROT_EXEC  映射区可执行
PROT_NONE  映射区不可访问
```

&emsp;&emsp;由于只能映射已经打开的文件，所以这个权限位不能超出open函数指定的权限，比如说在open的时候指定为只读，那就不能在此时指定PORT_WRITE。

* flag：指定映射区的其它一些属性。例如：

```Code
MAP_ FIXED   针对addr属性，如果指定这个位，那么要求系统必需在指定的地址映射，这往往是不可取的。
MAP_ SHARED  此标志说明指定映射区是共享的，意思就是说对内存的操作与对文件的操作是相对应的。
MAP_ PRIVATE 该标志说明映射区是私用的，此时被映射的内存只能被当前里程使用，当进程操作的内存将会产生原文件的一个副本。
```

* filedes：有效的文件描述符。一般是由open()函数返回，其值也可以设置为-1，此时需要指定flags参数中的MAP_ANON,表明进行的是匿名映射。

* offset：被映射对象内容的起点。

&emsp;&emsp;在mmap中有很多选项来控制最后得到的映射区的一些属性，在调用mmap函数之后，仍然可以通过mprotect函数对其中的一些属性进行调整。此外在更新了内存的内容之后，可以通过msync函数将更新的内容同步到磁盘中的文件。

```C
#include <sys/mman.h>
int mprotect(void* addr, size_t len, int port)
```

&emsp;&emsp;返回值：成功返回0，失败返回-1。

&emsp;&emsp;参数解释：

* addr：mmap返回的数值，此时它就是mprotect作用的范围。

* len：指定映射区的长度，它需要与mmap中指定相同。

* port：把指定的属性施加于相应的映射区上。

&emsp;&emsp;需要注意的是内核并不是实时同步映射区与文件的，相反内核很少主动去同步，除非我们调用了函数msync或者关闭映射区（关闭映射区的时候，也不是立即同步的）。