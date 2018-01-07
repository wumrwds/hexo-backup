---
title: Unicode 和 UTF编码（未完）
toc: true
date: 2017-12-18 23:30:21
tags:
- Unicode
categories:
- Encoding
---

今天在使用Apache Commons中的 RandomStringUtils组件时，观察到它的源码中有这样一段代码（ch是char类型，buffer是一个char数组）：

``` java
if(ch >= 56320 && ch <= 57343) {
  if(count == 0) {
    count++;
  } else {
    // low surrogate, insert high surrogate after putting it in
    buffer[count] = ch;
    count--;
    buffer[count] = (char) (55296 + random.nextInt(128));
  }
} else if(ch >= 55296 && ch <= 56191) {
  if(count == 0) {
    count++;
  } else {
    // high surrogate, insert low surrogate before putting it in
    buffer[count] = (char) (56320 + random.nextInt(128));
    count--;
    buffer[count] = ch;
  }
} else if(ch >= 56192 && ch <= 56319) {
  // private high surrogate, no effing clue, so skip it
  count++;
} else {
  buffer[count] = ch;
}
```

这段代码出现在判断ch是letter或number两者中的之一之后，ch为正常字母或数字时将直接进入最后的else子句。

但是前面这么一大串判断是在做什么？‘56320’， ‘57343’， ... ， ‘56319’ 这些数字代表什么意思？注释里提到的`low surrogate` 和`high surrogate`又是什么？

<!-- more -->

<br/>

## Java 与 UTF - 16

通过查阅资料， 大致可以推断出上面这段代码大致与Unicode编码相关。并且在查找资料的过程中，发现了以下这段关于Java内部编码的说明。

> Java uses the UTF-16 encoding scheme to store strings.

Java使用UTF-16 编码存储字符。

这点回看Java String类的源代码，我们也不难发现一些细节来印证上面说明。

String类底层存储字符的结构是一个char类型数组：

``` java
private final char value[];
```

Java中的char类型是一种基本类型，其占用字节大小为两字节。

Java作为一门一直十分注重通用性的语言，自然不会选择非Unicode类的不是太流行或表述能力有限的编码格式，而在Unicode标准中又只有UTF-16是符合两字节的大小（UTF-8: 1-4字节， UTF-32: 4字节）。并且，使用这类双字节的编码模式，在计算字符串长度、执行索引操作时速度会比较快。因此

更多关于Java characters encoding 的内容可以查阅：[Supplementary Characters in the Java Platform](http://www.oracle.com/technetwork/articles/javase/supplementary-142654.html)

其中详细阐述了Java平台从开始的UCS-2转至UTF - 16编码（see **Background**）的原因；Java平台关于字符编码最后选择的方案(see paragraph **Design Approach for Supplementary Characters in the Java Platform**); 运行时选择UTF - 16 而不选择 UTF - 8编码的原因（see paragraph **Modified UTF-8**； **注意**：class 文件编码仍是使用UTF - 8编码的，由于节省空间）。

<br/>

## Unicode 和 UTF - X

简单来说，Unicode可以理解为是一种字符集。而UTF-8、UTF-16、UTF-32则可以理解为是一种将Unicode映射至二进制的编码方案(character encoding scheme)。

下面进行详细介绍。

### Unicode

Unicode是一项业界标准，它对世界上大部分的文字系统进行了整理和编码。

**Unicode可以理解为是一种字符集，即一个字符到一个表示该字符的数字的映射关系的集合。**

Unicode与ASCII (American Standard Code for Information Interchange)类似，都是一种字符集。

拿我们熟知的ASCII 码来举一个例子。比如，在ASCII中，英文字母`a`对应97；而类似的在Unicode中，比如中文字符`日`对应 十六进制下65E5 (即 0x65E5)。这就是一种映射关系。

<br/>

Unicode的编码空间从U+0000到U+10FFFF，共有1,112,064个码位（code point）可用来映射字符. Unicode的编码空间可以划分为17个平面（plane），每个平面包含216（65,536）个码位。具体如下所示：

|   Basic   | Supplementary | Supplementary | Supplementary | Supplementary | Supplementary |
| :-------: | :-----------: | :-----------: | :-----------: | :-----------: | :-----------: |
|  Plane 0  |    Plane 1    |    Plane 2    |  Planes 3–13  |   Plane 14    | Planes 15–16  |
| 0000–FFFF |  10000–1FFFF  |  20000–2FFFF  |  30000–DFFFF  |  E0000–EFFFF  | F0000–10FFFF  |
|  基本多文种平面  |    多文种补充平面    |   表意文字补充平面    |     *未分配*     |   特别用途补充平面    |    私人补充平面     |
|    BMP    |      SMP      |      SIP      |       —       |      SSP      |   SPUA-A/B    |

17个平面的码位可表示为从U+xx0000到U+xxFFFF，其中xx表示十六进制值从$00_{16}$到$10_{16}$，共计17个平面。第一个平面称为**基本多语言平面**（Basic Multilingual Plane, **BMP**），或称第零平面（Plane 0）。其他平面称为**辅助平面**（Supplementary Planes）。基本多语言平面内，从U+D800到U+DFFF之间的码位区块是永久保留不映射到Unicode字符。

上文提到的中文字符`日`就属于Plane 1 多文种补充平面，而`a`则是属于Plane 0基本多语言平面。

<br/>

### UTF - 8

**UTF（Unicode Transformation Format）则是一种编码方案，即它是一套将Unicode码这一数字对应到具体在计算机中进行传输存储的二进制字节的映射方案。**

举个例子，下面将说明如何使用UTF-8编码去表示中文字符`日`：

UTF-8是一套以 8位(bit) 为最小编码单元的编码方案，它将会把Unicode码制下的字符对应的编码数值按照下表的规则，映射成成相应的二进制字节。

| Numberof bytes | Bits forcode point | Firstcode point | Lastcode point |   Byte 1   |   Byte 2   |   Byte 3   |   Byte 4   |
| :------------: | :----------------: | :-------------: | :------------: | :--------: | :--------: | :--------: | :--------: |
|       1        |         7          |     U+0000      |     U+007F     | `0xxxxxxx` |            |            |            |
|       2        |         11         |     U+0080      |     U+07FF     | `110xxxxx` | `10xxxxxx` |            |            |
|       3        |         16         |     U+0800      |     U+FFFF     | `1110xxxx` | `10xxxxxx` | `10xxxxxx` |            |
|       4        |         21         |     U+10000     |    U+10FFFF    | `11110xxx` | `10xxxxxx` | `10xxxxxx` | `10xxxxxx` |

`日`对应的Unicode为U+65E5, 属于多文种辅助平面内，查表可发现`日`应当对应第三行的编码规则。

将 0x65E5 转换为二进制，可得 0110 0101 1110 0101，再填入第三行的三字节模板，可得 11100110 10010111 10100101。上面的这三字节的二进制数即是Unicode字符集中 U+65E5 在UTF - 8编码下对应的二进制字节。

<br/>

### UTF - 16

同样的，UTF-16也是一套以 16位(bit) 为最小编码单元的编码方案，但其规则与UTF-8有所不同。

具体需要分平面进行介绍。

#### 基本平面的码位（从U+0000至U+D7FF以及从U+E000至U+FFFF的码位）

UTF-16对基本平面的码位直接映射。

比如字符`a`对应十进制下97（十六进制下0x61，二进制下0110 0001），即U+61。因而其UTF-16编码下的二进制字节为 00000000 01100001 。

呃，等等，基本平面的Unicode码（U+0000 - U+FFFF）好像看起来就有两字节了，UTF-16不是总共就只有两字节大小么，看起来并装不下U+0000到U+10FFFF这1,112,064个码位啊？那么对于辅助平面内的字符，UTF-16又该怎么表述呢？

<br/>

#### 辅助平面的码位（从U+10000到U+10FFFF的码位）

其实在上节的标题已经透露了答案：在基本平面内，有一段码段是专门分配给UTF-16使用的。

我们来看一下截止至2017年6月的基本多语言平面的编码区段分配的情况（U+0000至U+FFFF）:

![Unicode Plane 0](/images/Unicide/unicode-plane-0.png)

从图中可以发现其中的有一段区段U+D800 - U+DFFF的码位是来专门用来对辅助平面的字符的码位进行编码的，也就是说此区段内的Unicode码是不对应任何具体字符的。

辅助平面中的码位，在UTF-16中被编码为一对16比特长的码元（即32位,4字节），称作*代理对*（surrogate pair），具体方法是：

1. 码位减去0x10000,得到范围为20比特长的一个值。
2. 将20比特分为高低位两部分，每部分10比特。
3. 高位的10比特的值（值的范围为 0 到 0x3FF）被加上0xD800得到第一个码元或称作高位代理（high surrogate），值的范围是0xD800 到 0xDBFF. 由于高位代理比低位代理的值要小，所以为了避免混淆使用，Unicode标准现在称高位代理为**前导代理**（lead surrogates）。
4. 低位的10比特的值（值的范围也是 0 到0x3FF）被加上0xDC00得到第二个码元或称作低位代理（low surrogate），现在值的范围是0xDC00 到 0xDFFF.由于低位代理比高位代理的值要大，所以为了避免混淆使用，Unicode标准现在称低位代理为**后尾代理**（trail surrogates）。

<br/>

那么如何理解这种编码方式呢？

首先，我们来计算一下剩余的辅助平面内还有多少个码位需要表示： 0x10FFFF - 0x10000 = 0xFFFFF

显然还需要0xFFFFF（即$2^{20}$）个码位需要表示。也就是说如果我们以上节基本平面那样的思路去表示，至少需要20 Bits才行。

如果我们将其拆分为两部分，则每部分表示$2^{10}$（即0x400）

因此我们只需要在基本平面划出 0x400 * 2 = 0x800 个码位，然后使用两个UTF-16字符就可以表示出所有辅助平面的码位了。下面是实际基本平面中UTF-16高低位代理的分布：

| D800-DBFF | UTF-16的高半区 | High-half zone of UTF-16 |
| :-------: | :--------: | :----------------------: |
| DC00-DFFF | UTF-16的低半区 | Low-half zone of UTF-16  |

高低半区正好都分别是400位。

<br/>

例如emoji字符[😂](https://apps.timwhitlock.info/emoji/tables/unicode#emoji-modal)的Unicode码位为U+1F602:

- 0x1F602减去0x10000,结果为0x0F602,二进制为1111 0110 0000 0010。
- 分区它的上10位值和下10位值（使用二进制）:0000111101 and 1000000010, 转换为16进制 0x003D and 0x0202
- 添加0xD800到上值，以形成高位：0xD800 + 0x003D = 0xD83D。
- 添加0xDC00到下值，以形成低位：0xDC00 + 0x0202 = 0xDE02。

因而emoji字符[😂](https://apps.timwhitlock.info/emoji/tables/unicode#emoji-modal)的UTF-16表示为 0xD83D   0xDE02。