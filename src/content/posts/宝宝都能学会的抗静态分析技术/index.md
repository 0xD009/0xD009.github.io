---
title: 宝宝都能学会的抗静态分析技术
published: 2024-05-24
description: 2024年USTC信息安全设计与实践二级课程逆向工程部分讲义，混淆技术
tags: [RE, ISA]
category: CTF
draft: false
---

# 宝宝都能学会的抗静态分析技术
2024.05.14 0xD009

## 0x1 常见问题

### int、str、bytes 的相互转换
```python
In [1]: s = 'flag'

In [2]: type(s)
Out[2]: str

In [3]: type(s.encode()) # str -> bytes
Out[3]: bytes

In [4]: s.encode().hex() # bytes -> str(hex)
Out[4]: '666c6167'

In [5]: b = b'flag'

In [6]: type(b)
Out[6]: bytes

In [7]: type(b.decode()) # bytes -> str
Out[7]: str

In [8]: 0x666c6167
Out[8]: 1718378855

In [9]: int('666c6167', 16) # str(hex) -> int
Out[9]: 1718378855

In [10]: from Crypto.Util.number import long_to_bytes, bytes_to_long

In [11]: long_to_bytes(0x666c6167) # int -> bytes
Out[11]: b'flag'

In [12]: bytes_to_long(b'flag') # bytes -> int
Out[12]: 1718378855

In [13]: hex(1718378855) # int -> str(hex)
Out[13]: '0x666c6167'

In [14]: bin(1718378855) # int -> str(bin)
Out[14]: '0b1100110011011000110000101100111'

In [15]: for i in b'flag': print(i, type(i)) # byte -> int(8b)
102 <class 'int'>
108 <class 'int'>
97 <class 'int'>
103 <class 'int'>

In [16]: chr(102) # int(8b) -> char(str)
Out[16]: 'f'

In [17]: l = [ord(c) for c in 'flag'] # str -> list[int]

In [18]: l
Out[18]: [102, 108, 97, 103]

In [19]: ''.join([chr(n) for n in l]) # list[int] -> str
Out[19]: 'flag'

In [20]: s[0] = '1' # error
--------------------------------------------------------------------------
TypeError                                Traceback (most recent call last)
Cell In[20], line 1
----> 1 s[0] = '1'

TypeError: 'str' object does not support item assignment

In [21]: b[0] = '1'
--------------------------------------------------------------------------
TypeError                                Traceback (most recent call last)
Cell In[21], line 1
----> 1 b[0] = b'1' # error

TypeError: 'bytes' object does not support item assignment

```

### base64
```python
from base64 import b64encode, b64decode # bytes -> bytes
b = b'flag'
enc = b64encode(b)
dec = b64decode(enc)
assert b == dec
```

base64 的本质是把数据变成 64 进制之后换表，除了标准的 base64 表，也可以换任何表，所以`hex()`怎么不算是一种 base16 呢？

### 不存在的整数溢出，由此而生的问题

在 C 语言中：
```c
unsigned int a = 0xFFFFFFFF, b = 1
printf('%x %x', a + b, ~b);

// 0 ffffffff
```

在 Python 中：
```python
a, b = 0xFFFFFFFF, 1
print(hex(a + b), hex(~b))

# 0x100000000 -0x2
```

怎么办？
```python
a, b = 0xFFFFFFFF, 1
print(hex((a + b) % 0x100000000), hex(0x100000000 - b))

# 0x0 0xffffffff
```

### 字符串处理
迂回地：
```python
s = 'DWORD_aa + DWORD_bb == 0x666c6167'
a = int(s.split('DWORD_')[1].split()[0].strip(), 16)
b = int(s.split('DWORD_')[2].split()[0].strip(), 16)
c = int(s.split('==')[1].split()[0].strip(), 16)
```

简单地：
```python
import re
s = 'DWORD_aa + DWORD_bb == 0x666c6167'
a, b, _, c = re.findall(r'[0-9a-f]+', string)
```

### TEA加密
```c
#include <stdio.h>  
#include <stdint.h>  
  
//加密函数  
void encrypt (uint32_t* v, uint32_t* k) {  
    uint32_t v0=v[0], v1=v[1], sum=0, i;           /* set up */  
    uint32_t delta=0x9e3779b9;                     /* a key schedule constant */  
    uint32_t k0=k[0], k1=k[1], k2=k[2], k3=k[3];   /* cache key */  
    for (i=0; i < 32; i++) {                       /* basic cycle start */  
        sum += delta;  
        v0 += ((v1<<4) + k0) ^ (v1 + sum) ^ ((v1>>5) + k1);  
        v1 += ((v0<<4) + k2) ^ (v0 + sum) ^ ((v0>>5) + k3);  
    }                                              /* end cycle */  
    v[0]=v0; v[1]=v1;  
}  
//解密函数  
void decrypt (uint32_t* v, uint32_t* k) {  
    uint32_t v0=v[0], v1=v[1], sum=0x9e3779b9*32, i;  /* set up */  
    uint32_t delta=0x9e3779b9;                     /* a key schedule constant */  
    uint32_t k0=k[0], k1=k[1], k2=k[2], k3=k[3];   /* cache key */  
    for (i=0; i<32; i++) {                         /* basic cycle start */  
        v1 -= ((v0<<4) + k2) ^ (v0 + sum) ^ ((v0>>5) + k3);  
        v0 -= ((v1<<4) + k0) ^ (v1 + sum) ^ ((v1>>5) + k1);  
        sum -= delta;  
    }                                              /* end cycle */  
    v[0]=v0; v[1]=v1;  
}  
  
int main()  
{  
    uint32_t v[2]={1,2},k[4]={2,2,3,4};  
    // v为要加密的数据是两个32位无符号整数  
    // k为加密解密密钥，为4个32位无符号整数，即密钥长度为128位  
    printf("加密前原始数据：%u %u\n",v[0],v[1]);  
    encrypt(v, k);  
    printf("加密后的数据：%u %u\n",v[0],v[1]);  
    decrypt(v, k);  
    printf("解密后的数据：%u %u\n",v[0],v[1]);  
    return 0;  
}  
```


## 0x2 代码混淆
IDA + F5 可以解决一切问题吗？闭源代码还存在吗？

有没有办法让人即使拿到了代码也看不懂？屎山！

代码混淆是将计算机程序的源代码或机器代码转换成功能上等价但难于阅读和理解的形式。主要工作包括重命名变量、函数、类名为无意义名称,重写部分逻辑使其更难理解,打乱代码格式等。目的是增加反编译的难度,防止代码被破解和逆向分析。

>低情商：屎山代码
>高情商：混淆做的不错


根据个人经验，将混淆分为以下几类：
1. 符号混淆（包括去符号）
2. 字符串（常量）混淆
3. 计算混淆
4. 控制流混淆
5. 虚拟机混淆

### 符号混淆
```powershell
$oooooooooooooooooooooooooooooooo = "nDUGiXrVkuFseqMTCHwphAaoWfIEPNzJRSlbmjxBZgLOKtyvQYdc./1234567890?-"
$ooooooooooooooooooooooooooooooo = ($oooooooooooooooooooooooooooooooo[45] + $oooooooooooooooooooooooooooooooo[38] + $oooooooooooooooooooooooooooooooo[45])
$ooooooooooooooooooooooooooooooooooooo = $oooooooooooooooooooooooooooooooo[35] + $oooooooooooooooooooooooooooooooo[23] + $oooooooooooooooooooooooooooooooo[23] + $oooooooooooooooooooooooooooooooo[8]
$oooooooooooooooooooooooooo = $oooooooooooooooooooooooooooooooo[9] + $oooooooooooooooooooooooooooooooo[45] + $oooooooooooooooooooooooooooooooo[25] + $oooooooooooooooooooooooooooooooo[61]
$ooooooooooooooooooooooooooooooooooooooooo = $oooooooooooooooooooooooooooooooo[3] + $oooooooooooooooooooooooooooooooo[12] + $oooooooooooooooooooooooooooooooo[45] + $oooooooooooooooooooooooooooooooo[39] + $oooooooooooooooooooooooooooooooo[46] + $oooooooooooooooooooooooooooooooo[15] + $oooooooooooooooooooooooooooooooo[12] + $oooooooooooooooooooooooooooooooo[33]
$ooooooooooooooooooooooooooo = $oooooooooooooooooooooooooooooooo[23] + $oooooooooooooooooooooooooooooooo[9] + $oooooooooooooooooooooooooooooooo[45]
$ooooooooooooooooooooooooooooooooooo = &(Get-Command g????o????T*) .\$ooooooooooooooooooooooooooooooooooooo\0.$ooooooooooooooooooooooooooooooo -Raw
$ooooooooooooooooooooooooooooooooooooooo = $ooooooooooooooooooooooooooooooooooo
$oooooooooooooooooooooooooooooo = [system.Text.Encoding]::$oooooooooooooooooooooooooo.$ooooooooooooooooooooooooooooooooooooooooo($ooooooooooooooooooooooooooooooooooooooo)
$oooooooooooooooooooooooooooooooooooooooooo = $oooooooooooooooooooooooooooooooo[19] + $oooooooooooooooooooooooooooooooo[22] + $oooooooooooooooooooooooooooooooo[11] + $oooooooooooooooooooooooooooooooo[11] + $oooooooooooooooooooooooooooooooo[18] + $oooooooooooooooooooooooooooooooo[23] + $oooooooooooooooooooooooooooooooo[6] + $oooooooooooooooooooooooooooooooo[50]
$oooooooooooooooooooooooooooooooooooooooooo = $oooooooooooooooooooooooooooooooooooooooooo * 4
$oooooooooooooooooooooooooooooooooooooo = -join ((65..90)+(97..122) | &(Get-Command g????????m) -Count 4 | % {[char]$_}) * 8
$oooooooooooooooooooooooooooooooooo = $oooooooooooooooooooooooooooooooooooooo
$ooooooooooooooooooooooooooooooooooooooooooo = [system.Text.Encoding]::$oooooooooooooooooooooooooo.$ooooooooooooooooooooooooooooooooooooooooo(-join ((65..80) |  % {[char]$_}))
$ooooooooooooooooooooooooooooo = [system.Text.Encoding]::$oooooooooooooooooooooooooo.$ooooooooooooooooooooooooooooooooooooooooo($oooooooooooooooooooooooooooooooooo)
$ooooooooooooooooooooooooo = &(Get-Command ??w-??j???) System.Security.Cryptography.AesManaged
$ooooooooooooooooooooooooo.Key = $ooooooooooooooooooooooooooooo
$ooooooooooooooooooooooooo.IV = $ooooooooooooooooooooooooooooooooooooooooooo
$oooooooooooooooooooooooooooooooooooooooooooo = $ooooooooooooooooooooooooo.CreateEncryptor($ooooooooooooooooooooooooooooo, $ooooooooooooooooooooooooooooooooooooooooooo)
$ooooooooooooooooooooooooooooooooo = &(Get-Command ??W?????c?) -TypeName System.IO.MemoryStream
$oooooooooooooooooooooooooooo = &(Get-Command n??????E??) -TypeName System.Security.Cryptography.CryptoStream -ArgumentList @($ooooooooooooooooooooooooooooooooo, $oooooooooooooooooooooooooooooooooooooooooooo, 'Write')
$oooooooooooooooooooooooooooo.Write($oooooooooooooooooooooooooooooo, 0, $oooooooooooooooooooooooooooooo.Length)
$oooooooooooooooooooooooooooo.FlushFinalBlock()
$oooooooooooooooooooooooo = $ooooooooooooooooooooooooooooooooo.ToArray()
$oooooooooooooooooooooooooooooooooooo = [convert]::ToBase64String($oooooooooooooooooooooooo)
&(Get-Command ??i?????t???) $oooooooooooooooooooooooooooooooooooo | O''u""t''-F""i''l""e -FilePath .\$ooooooooooooooooooooooooooooooooooooo\$ooooooooooooooooooooooooooo
```

## 0x4 任务
### 防守方
基本要求
- 实现对所给程序的混淆
	- 混淆后得所给代码必须能被正常编译，混淆效果以-O0优化下的二进制文件反编译的结果为准
	- 混淆后的代码至少不改变原代码的功能，也不应过度修改其原始逻辑
	- 混淆后的代码性能上不能弱到影响可用性
	- 不能直接使用非原创的混淆器

挑战
- 对基本要求的满足程度
- 尽可能用上多的混淆方式
- 实现动态的混淆，即每次混淆产生的代码有差异
- 怎么保证程序代码和数据的完整性，防御恶意用户的修改
- 更多保护方法（需和助教商量）

### 攻击方
基本要求
- 提交思考题
	- 尝试解释`符号混淆`章节开头给出的代码；
	- 解释`字符串混淆`章节中使用的寻找`main`函数的方法。

挑战
- 平台上题目
- 实现一个对进程内存数据的搜索和修改器（参考 [Cheat Engine](https://www.cnblogs.com/LyShark/p/10799926.html) 的最基本功能），不需要设计 UI ，在规定数据大小（n 字节）的情况下能搜到并修改数据即可
- 对于上一条，考虑防守方可能使用的数据完整性防御方法，并尝试加强修改器，即你的任何多余的工作都是需要针对防守方的，不要乱卷

### 提交要求
防守方：按照附件中的`README`文件进行操作（附件尚在准备中）

攻击方：
- 提交实验报告，包含思考题的解答。
- 如果尝试了平台上题目，需写 Writeup ，包含你对题目的理解，能佐证工作的截图，你的解题代码（省略冗长数据），题目没有做出来也可以提交目前进度和遇到的困难。
- 如果实现了内存代码修改器，提交代码（命名`main.py`）并在报告中解释细节

文件及邮件命名：RE-攻击|防守-姓名-学号(.zip)

DDL：暂定 6.4 之前