---
title: 汇编指令长度
date: 2020-05-01 12:19:01
categories:
- Tools
- Assembler
tags:
- PWN
- Reverse
- Updating
---
&emsp;&emsp;不断完善中！！！

<!-- more -->

## 没有操作数的指令

* 指令长度为1个字节，如：pop rax

## 操作数只涉及寄存器的的指令

* 指令长度为2个字节，如：mov bx,ax

## 操作数涉及内存地址的指令

* 指令长度为3个字节，如：mov ax,ds:[bx+si+idata]

## 操作数涉及立即数的指令

* 指令长度为：寄存器类型+1

* 8位寄存器，寄存器类型=1，如：mov al,8；指令长度为2个字节

* 16位寄存器，寄存器类型=2，如：mov ax,8；指令长度为3个字节

## 跳转指令

### 段内跳转

* 指令长度为2个字节或3个字节，jmp指令本身占1个字节

* 段内短转移，8位位移量占一个字节，加上jmp指令一个字节，整条指令占2个字节，如：jmp short opr

* 段内近转移，16位位移量占两个字节，加上jmp指令一个字节，整条指令占3个字节
如：jmp near ptr opr

### 断间跳转

* 指令长度为5个字节，如：jmp dword ptr table[bx][di]，或 jmp far ptr opr，或 jmp dword ptr opr

## inc指令

* 占用一个字节

## push指令

* 占用一个字节

## segment声明

* 占用两个字节，如codesg segment

## int 21h

* 占用两个字节