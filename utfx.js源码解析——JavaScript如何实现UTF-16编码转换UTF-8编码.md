# 概述

本文通过对[utfx.js](https://github.com/dcodeIO/utfx)这个库的代码进行分析，带大家深入了解UTF8和UTF16这两种编码方式，并且介绍两种编码方式之间互相转换的方法。

本文的主要内容为：

- 编码规范介绍
- utfx.js API简单介绍
- UTF-16编码转换为UTF-8编码
- UTF-8编码字符串长度计算
- 实验性功能：window.TextEncoder

如果有读者想要了解该库相关的转换使用场景，可以阅读我之前的博客[WebSocket系列之JavaScript字符串如何与二进制数据间进行互相转换](https://juejin.im/post/5abdc38ef265da2375070008)。

# 编码规范

本章主要是介绍UTF-8和UTF-16两种编码规范。在网上看到很多有关这两种编码方式的介绍和说明，但是感觉说的都太过学术化，下面我们通过更加简单的方法来了解下这两种编码方式。

## UTF-8



## UTF-16



# utfx.js API简介

在进行具体的代码详解之前，我们先来了解下我们需要介绍的库——utfx.js。我们只有了解了这个库的使用方法，我们才能够更好的理解源码。

utfx.js代码不多，一共只有八个API接口，分别为：

- encodeUTF8：将UTF-8编码的字符串code码转换为二进制bytes。
- decodeUTF8：将UTF-8编码的二进制bytes解码城字符串code码。
- UTF16toUTF8：将UTF-16的字符转换为UTF-8的code码。
- UTF8toUTF16：将UTF-8的code码转换为UTF-16的字符。
- encodeUTF16toUTF8：将UTF-16编码的字符转换为UTF-8编码的bytes。
- decodeUTF8toUTF16：将UTF-8编码的bytes转换为UTF-16编码的字符。
- calculateCodePoint：计算UTF-8编码下的字符长度。
- calculateUTF8：计算需要用来存储UTF-8编码code码的bytes的长度。
- calculateUTF16asUTF8：计算UTF-16编码的字符在转换成UTF-8后需要的存储长度。

看完上面的接口，大家可能会奇怪为什么每次我们都是只操作UTF-8的code码和二进制bytes，而UTF-16则一直是字符形式存在？

因为在JavaScript中，string类型数据是以UTF-16的编码存在的，即JavaScript中的字符串是UTF-16编码，所以我们日常使用中，遇到的都是UTF-16的字符串。

# UTF-16编码转换为UTF-8编码



# UTF-8编码字符串长度计算



# 总结

