---
title: Jarvis OJ - Backdoor の Write-Up
date: 2020-08-18 14:44:56
categories:
- WriteUPs
- JarvisOJ
tags:
- PWN
- Stack
---
## Introduction

&emsp;&emsp;根据题目程序存在后门，某个参数可以触发该程序执行后门操作。这个程序是个exe程序，但在win10和win7虚拟机上均无法运行，最后在Win XP上按照提示下载了缺失的MSVCP100D.dll后终于得以成功运行。

<!-- more -->

## Analysis

&emsp;&emsp;一开始在hopper中反编译程序，虽然程序逻辑结构很清楚，但是代码可读性真是一言难尽......还是老老实实打开IDA吧。

```C#
signed int __cdecl wmain(int a1, int a2)
{
  char v3; // [esp+50h] [ebp-2C8h]
  char v4; // [esp+E1h] [ebp-237h]
  char v5; // [esp+E4h] [ebp-234h]
  char Source[4]; // [esp+100h] [ebp-218h]
  char v7; // [esp+104h] [ebp-214h]
  __int16 i; // [esp+108h] [ebp-210h]
  char Dest[2]; // [esp+10Ch] [ebp-20Ch]
  char Dst; // [esp+10Eh] [ebp-20Ah]
  char v11[25]; // [esp+110h] [ebp-208h]
  char v12[483]; // [esp+129h] [ebp-1EFh]
  __int16 v13; // [esp+30Ch] [ebp-Ch]
  LPSTR lpMultiByteStr; // [esp+310h] [ebp-8h]
  int cbMultiByte; // [esp+314h] [ebp-4h]

  cbMultiByte = WideCharToMultiByte(1u, 0, *(LPCWSTR *)(a2 + 4), -1, 0, 0, 0, 0);
  lpMultiByteStr = (LPSTR)sub_4011F0(cbMultiByte);
  WideCharToMultiByte(1u, 0, *(LPCWSTR *)(a2 + 4), -1, lpMultiByteStr, cbMultiByte, 0, 0);
  v13 = *(_WORD *)lpMultiByteStr;
  if ( v13 < 0 )
    return -1;
  v13 ^= 0x6443u;
  strcpy(Dest, "0");
  memset(&Dst, 0, 0x1FEu);
  for ( i = 0; i < v13; ++i )
    Dest[i] = 65;
  *(_DWORD *)Source = 2147108114;
  v7 = 0;
  strcpy(&Dest[v13], Source);
  qmemcpy(&v5, &unk_4021FC, 0x1Au);
  strcpy(&v11[v13], &v5);
  qmemcpy(&v3, &unk_402168, 0x91u);
  v4 = 0;
  strcpy(&v12[v13], &v3);
  sub_401000(Dest);
  return 0;
}
```

&emsp;&emsp;首先看下这个主函数。应该是接收键盘输入的内容，经过转换然后赋给v13，注意v13申明的是int 16，再将v13与0x6443异或。然后向Dest数组里填充v13数值个的ASCII值65也就是’A’。

&emsp;&emsp;再将2147108114即0x7ffa4512这个地址传到了Dest[v13]。而0x7ffa4512这个地址在windows上是一个万能的jmp esp（几乎所有的平台上这个地址都是jmp esp）。所以可以通过这个地址上的jmp esp指令控制程序跳转到某一个函数的地址上去执行设定好的函数。

&emsp;&emsp;再往下走，这个函数最后调用Dest是将其作为sub_401000函数的参数，再去找一下这个函数：

```C#
int __cdecl sub_401000(char *Source)
{
  char Dest[2]; // [esp+4Ch] [ebp-20h]
  int v3; // [esp+4Eh] [ebp-1Eh]
  int v4; // [esp+52h] [ebp-1Ah]
  int v5; // [esp+56h] [ebp-16h]
  int v6; // [esp+5Ah] [ebp-12h]
  int v7; // [esp+5Eh] [ebp-Eh]
  int v8; // [esp+62h] [ebp-Ah]
  int v9; // [esp+66h] [ebp-6h]
  __int16 v10; // [esp+6Ah] [ebp-2h]

  strcpy(Dest, "0");
  v3 = 0;
  v4 = 0;
  v5 = 0;
  v6 = 0;
  v7 = 0;
  v8 = 0;
  v9 = 0;
  v10 = 0;
  strcpy(Dest, Source);
  return 0;
}
```

&emsp;&emsp;果然在这个函数里发现了溢出漏洞，在strcpy(Dest, Source)语句这里。在主函数中Source申明的是char Source[4]，而Dest申明的是char Dest[2]，strcpy函数把一个长字符赋给里一个短字符，典型的溢出漏洞。

&emsp;&emsp;根据Dest的大小，需要0x20的padding加上4个字节的返回地址，一共0x24字节。这样程序执行的时候会将0x7ffa4512覆盖到Dest最后4字节也就是strcpy函数的返回地址，从而控制程序跳转触发后门。

&emsp;&emsp;所以我们需要控制的是v13的值为0x24即可，再注意一下机器的大小端问题以及题目要求的给出参数小写SHA256值，直接给出EXP：

```Python
import hashlib

value = chr(0x24 ^ 0x43)
value += chr(0 ^ 0x64)

print "value: " + value
print "flag: PCTF{" + hashlib.sha256(value).hexdigest() + "}"
```

&emsp;&emsp;执行之后就可以得到出发后门的参数以及flag：

![](/img/Backdoor/Backdoor1.png)

&emsp;&emsp;后门程序在Win XP中复现成功，弹出了计算器：

![](/img/Backdoor/Backdoor2.png)
