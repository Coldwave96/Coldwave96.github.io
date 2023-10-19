---
title: 常用GDB指令
date: 2020-05-01 12:30:35
categories:
- Tools
- Assembler
tags:
- PWN
- Reverse
- Commands
- Updating
---
不断完善中！！！

<!-- more -->

# 选择加载文件
file [file_name]
exec-file [file_name]

# 执行
r(run)

# 反编译main函数
disas [function_name]
disassemble [function_name]

# 设置断点
b(break)
b [function_name] # 对函数下断点
b *addr # 对地址下断点
info b # 查看断点

registers code stack
寄存器取值  代码   栈

# 查看寄存器情况
info [register]

# 单步调试
n(next)

# 步进（到某个函数）
si(step into)

# 查看现在的堆栈情况
bt(backtrace)

# 继续到下一个断点
c(continue)

# 打印地址值
x/wx adress # w代表word，在32位嵌入式系统中，一个字WORD占32bit，即4个字节
例如：
x/10wx 0xffffce90 # 10代表打印10个，w代表32位，可以换成b/h/g，分别对应1，2，8byte，x可以替换为u(unsigned int)/s(string)/d(10进制)/i(指令)

# 设置地址的值
set *addr = value # 设置地址的值

# 列出源码
list

# 打印变量值
print val_name

# 查看所有局部变量的值
info local

# gdb-peda版本中特殊功能
elfsymbol # 可以把程序中的函数及地址列出来（ROP中很有用）
vmmap # 查看进程中的权限
readelf # 查看section
find [string] # 在内存中查找字符串
