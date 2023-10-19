---
title: 古典密码对照
date: 2019-11-07 00:31:18
categories: 
- Tools
- Cryptography
tags: 
- Encrypt
---
最近在刷CTF密码学以及MISC的时候遇到一些古典的加密方式，下面是总结的一些古典加密方法对照表。

<!-- more -->

## QWE

<center>a b c d e f g h i j k l m n o p q r s t u v w x y z</center>

<center>q w e r t y u i o p a s d f g h j k l z x c v b n m</center>

## atbash（反字母）

<center>a b c d e f g h i j k l m n o p q r s t u v w x y z</center>

<center>z y x w v u t s r q p o n m l k j i h g f e d c b a</center>

## PC键盘

<center>a b c d e f g h i j k l m n o p q r s t u v w x y z</center>

<center>12 53 33 32 31 42 52 62 81 72 82 92 73 63 91 01 11 41 22 51 71 43 21 23 61 13</center>

## 5*5矩阵

<center>a b c d e f g h i j k l m n o p q r s t u v w x y z</center>

<center>11 12 13 14 15 21 22 23 24 24 25 31 32 33 34 35 41 42 43 44 45 51 52 53 54 55</center>
注：此类存在变式，不是唯一对照规则。

## 摩斯

<center>A *- B -*** C -*-* D -** E * F **-* G --* H **** I ** J *--- K -*- L *-** M -- N -* O --- P *--* Q --*- R *-* S *** T - U **- V ***- W *-- X -**- Y -*-- Z --** 0 ----- 1 *---- 2 **--- 3 ***-- 4 ****- 5 ***** 6 -**** 7 --*** 8 ---** 9 ----*</center>

## MNB

<center>a b c d e f g h i j k l m n o p q r s t u v w x y z</center>

<center>m n b v c x z l k j h g f d s a p o i u y t r e w q</center>
注：属QWE同类的对应。扩展：qaz--abc等竖向对应。

## 三进制

<center>a b c d e f g h i j k l m n o p q r s t u v w x y z</center>

<center>001 002 010 011 012 020 021 022 100 101 102 110 111 112 120 121 122 200 201 202 210 211 212 220 221 222</center>

## 键盘V字

<center>a b c d e f g h i j k l m n o p q r s t u v w x y z</center>

<center>13 58 36 35 34 46 57 68 89 79 80 9- 70 69 90 0- 12 45 24 56 78 47 23 25 67 14</center>

## 三分密码

<center>a b c d e f g h i j k l m n o p q r s t u v w x y z</center>

<center>111 121 131 112 122 132 113 123 133 211 221 231 212 222 232 213 223 233 311 321 331 312 322 332 313 323</center>
三分密码理解为把字母排成：（仿九宫格）

|   | 1 | 2 | 3 |
|:-----:|:-----:|:-----:|:-----:|
| 1 | abc | def | ghi |
| 2 | jkl | mon | opq |
| 3 | stu | vwx | yz  |

第一位行数，第三位列数，中间该格的第几个字母。

## ADFGVX

### 原始对应

<center>a b c d e f g h i j k l m n o p q r s t u v w x y z 0 1 2 3 4 5 6 7 8 9</center>

<center>DV FF FG AG XD XV GV DX VG GA FD AV GX AX DG AD XX VV VD DD GD VF GG VA XF FX XG DA VX AF DF FV GF FA AA XA</center>

### 按顺序

<center>a b c d e f g h i j k l m n o p q r s t u v w x y z 0 1 2 3 4 5 6 7 8 9</center>

<center>AA AD AF AG AV AX DA DD DF DG DV DX FA FD FF FG FV FX GA GD GF GG GV GX VA VD VF VG VV VX XA XD XF XG XV XX</center>

为了帮助理解，下面放出一个字典表实例：

|  | A | D | F | G | X |
|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|
| A | b | t | a | l | p |
| D | d | h | o | z | k |
| F | q | f | v | s | n |
| G | g | j | c | u | x |
| X | m | r | e | w | y |

## 键盘移位

### 右移

<center>q w e r t y u i o p a s d f g h j k l z x c v b n m</center>

<center>w e r t y u i o p a s d f g h j k l z x c v b n m q</center>

## 左移

<center>q w e r t y u i o p a s d f g h j k l z x c v b n m</center>

<center>m q w e r t y u i o p a s d f g h j k l z x c v b n</center>

## 特殊符号

<center>q w e r t y u i o p a s d f g h j k l z x c v b n m</center>

<center>§ № ☆ ★ ○ ● ◎ ◇ ◆ □ ■ △ ▲ ※ → ← ↑ ↓ 〓 ＃ ＆ ＠ ＼ ＾ ＿ ￣</center>

## dvorak键盘

<center>a b c d e f g h i j k l m n o p q r s t u v w x y z</center>

<center>p y f g c r l a o e u i d h t n s q j k x s m w v z</center>

## 费娜姆密码（即密码管）

<center>A 1000001 B 1000010 C 1000011 D 1000100 E 1000101 F 1000110 G 1000111 H 1001000 I 1001001 J 1001010 K 1001011 L 1001100 M 1001101 N 1001110 O 1001111 P 1010000 Q 1010001 R 1010010 S 1010011 T 1010100 U 1010101 V 1010110 W 1010111 X 1011000 Y 1011001 Z 1011010</center>

## 培根密码

<center>A aaaaa B aaaab C aaaba D aaabb E aabaa F aabab G aabba H aabbb I abaaa J abaab K ababa L ababb M abbaa N abbab O abbba P abbbb Q baaaa R baaab S baaba T baabb U babaa V babab W babba X babbb Y bbaaa Z bbaab</center>

注1：培根密码有两种对应，这是较为常用的一种。
注2：替代成a=1,b=0或a=0,b=1，作为1和0的培根也是可以的。

## 台湾拼音

<center>ㄅ->b ㄉ->d ㄓ->zh ㄚ->a ㄞ->ai ㄦ->er ㄆ->p ㄊ->t ㄍ->g ㄐ->j ㄔ->ch ㄗ->z 一->i ㄛ->o ㄟ->ei ㄣ->en ㄇ->m ㄋ->n ㄎ->k ㄑ->q ㄕ->sh ㄘ->c ㄨ->u ㄜ->e ㄠ->ao ㄤ->ang ㄈ->f ㄌ->l ㄏ->h ㄒ->x ㄖ->r ㄙ->s ㄩ->..(就是拼音输入法v) u ㄝ->e(和ㄜ->e不同的是,ㄜ用在ye,te,de声母后面,而ㄝ,只能跟在yue,tie之类韵母后面) ㄡ->ou ㄥ->eng ㄢ->an</center>

## 北约音标

<center>A Alpha B Bravo C Charlie D Delta E Echo F Foxtrot G Golf H Hotel I India J Juliet K Kilo L Lima M Mike N November O Oscar P Papa Q Quebec R Romeo S Sierra T Tango U Uniform V Victor W Whiskey X X-ray Y Yankee Z Zulu 0 Zero 1 Wun 2 Two 3 Three（或Tree） 4 Four 5 Five 6 Six 7 Seven 8 Eight 9 Niner</center>

## 标准电阻值

<center>黑 棕 红 橙 黄 绿 蓝 紫 灰 白</center>

<center>0 1 2 3 4 5 6 7 8 9</center>

## 中文电码（汉字代码）

这种方式可以叫做编码方式，具体对应规则网上有很多，需要时可自行百度。
