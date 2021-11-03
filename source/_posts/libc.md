---
title: libc && glibc && glib 杂谈
date: 2020-06-19 15:46:47
categories:
- Theories
- Assembler
tags:
- PWN
- Reverse
---
## Introduction

&emsp;&emsp;这段时间学习PWN题的时候需要使用不同版本的libc库，导致在本地执行EXP脚本的时候总是失败，所以搜集了一些有关linux的C语言环境的资料。

<!-- more -->

## libc && glibc

&emsp;&emsp;Linux平台提供的C标准库包括：

* 一组头文件，定义了很多类型和宏，声明了很多库函数。这些头文件放在哪些目录下取决于不同的编译器，stdarg.h和stddef.h位于/usr/lib/gcc/i486-linux-gnu/4.3.2/include目录下，stdio.h、stdlib.h、time.h、math.h、assert.h位于/usr/include目录下。C99标准定义的头文件有24个。

* 一组库文件，提供了库函数的实现。大多数库函数在libc共享库中，有些库函数在另外的共享库中，例如数学函数在libm中。通常libc共享库是/lib/libc.so.6。

&emsp;&emsp;libc是Linux下的ANSI C函数库；glibc是Linux下的GUN C函数库。 glibc本身是GNU旗下的C标准库，后来逐渐成为了Linux的标准C库，而Linux下原来的标准C库Linux libc逐渐不再被维护。

&emsp;&emsp;Linux下面的标准C库不仅有这一个，如uclibc、klibc，以及上面被提到的Linux libc，但是glibc无疑是用得最多的。glibc在/lib目录下的.so文件为libc.so.6。

### ANSI C

&emsp;&emsp;ANSI C函数库是基本的C语言函数库，包含了C语言最基本的库函数。这个库可以根据头文件划分为15个部分，其中包括：

* <ctype.h>：包含用来测试某个特征字符的函数的函数原型，以及用来转换大小写字母的函数原型；

* <errno.h>：定义用来报告错误条件的宏；

* <float.h>：包含系统的浮点数大小限制；

* <math.h>：包含数学库函数的函数原型；

* <stddef.h>：包含执行某些计算 C 所用的常见的函数定义；

* <stdio.h>：包含标准输入输出库函数的函数原型，以及他们所用的信息；

* <stdlib.h>：包含数字转换到文本，以及文本转换到数字的函数原型，还有内存分配、随机数字以及其他实用函数的函数原型；

* <string.h>：包含字符串处理函数的函数原型；

* <time.h>：包含时间和日期操作的函数原型和类型；

* <stdarg.h>：包含函数原型和宏，用于处理未知数值和类型的函数的参数列表；

* <signal.h>：包含函数原型和宏，用于处理程序执行期间可能出现的各种条件；

* <setjmp.h>：包含可以绕过一般函数调用并返回序列的函数的原型，即非局部跳转；

* <locale.h>：包含函数原型和其他信息，使程序可以针对所运行的地区进行修改。地区的表示方法可以使计算机系统处理不同的数据表达约定，如全世界的日期、时间、美元数和大数字；

* <assert.h>：包含宏和信息，用于进行诊断，帮助程序调试。

&emsp;&emsp;上述库函数在其各种支持C语言的IDE中都是有的。 

### GNU C

&emsp;&emsp;GNU C函数库是一种类似于第三方插件的东西。由于Linux是用C语言写的，所以Linux的一些操作是用C语言实现的，因此，GUN组织开发了一个C语言的库 以便让我们更好的利用C语言开发基于Linux操作系统的程序。 不过现在的不同的Linux的发行版本对这两个函数库有不同的处理方法，有的可能已经集成在同一个库里了。

## glibc && glib

&emsp;&emsp;不是长得像就是双胞胎。glib和glibc基本上没有太大联系，可能唯一的共同点就是，都是C编程需要调用的库而已。 

&emsp;&emsp;glib是Gtk+库和Gnome的基础。glib可以在多个平台下使用，比如Linux、Unix、Windows等。glib为许多标准的、常用的C语言结构提供了相应的替代物。 

&emsp;&emsp;它由基础类型、对核心应用的支持、实用功能、数据类型和对象系统五个部分组成，可以在[gtk网站](http://www.gtk.org)下载其源代码。是一个综合用途的实用的轻量级的C程序库，它提供C语言的常用的数据结构的定义、相关的处理函数，有趣而实用的宏，可移植的封装和一些运行时机能，如事件循环、线程、动态调用、对象系统等的API。GTK+是可移植的，当然glib也是可移植的，你可以在linux下，也可以在windows下使用它。使用gLib2.0（glib的2.0版本）编写的应用程序，在编译时应该在编译命令中加入`pkg-config --cflags --libs glib-2.0`，如：`gcc pkg-config --cflags --libs glib-2.0 hello.c -o hello`
