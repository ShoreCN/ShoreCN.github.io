---
title: macOS上修复word/excel等office文件乱码问题
date: 2019-03-19 11:14:39
categories:
- 工具
---

## 原因
编码问题。因为在Windows系统上面，通常使用的是GBK字符编码方式，但是在macOS的系统上使用的是UTF-8的编码方式。所以常常会遇到从Windows系统上编辑创建的word/excel等office文件，在macOS或iOS上面打开显示为乱码。

## 解决办法
既然明白了问题产生原因是字符编码方式的差异，那么就容易处理了，我们只需要将文件的字符编码方式转换为系统支持的即可。
macOS或Linux系统下的iconv命令的作用就是进行文件编码方式转换。

> iconv的作用是在多种国际编码格式之间进行文本内码的转换。  

```
Usage: iconv [OPTION...] [-f ENCODING] [-t ENCODING] [INPUTFILE...]
or:    iconv -l

Converts text from one encoding to another encoding.

Options controlling the input and output format:
  -f ENCODING, --from-code=ENCODING
                              the encoding of the input
  -t ENCODING, --to-code=ENCODING
                              the encoding of the output

Options controlling conversion problems:
  -c                          discard unconvertible characters
  --unicode-subst=FORMATSTRING
                              substitution for unconvertible Unicode characters
  --byte-subst=FORMATSTRING   substitution for unconvertible bytes
  --widechar-subst=FORMATSTRING
                              substitution for unconvertible wide characters

Options controlling error output:
  -s, --silent                suppress error messages about conversion problems

Informative output:
  -l, --list                  list the supported encodings
  --help                      display this help and exit
  --version                   output version information and exit
```

通过iconv尝试进行文件编码方式转换，执行命令`iconv -s -c -f GBK -t UTF8 input.file > output.file`  
参数解释：  
-f 输入文件的编码方式  
-t 输出文件的编码方式  
input.file 输入文件名  
output.file 输出文件名  
通过执行这条命令，我们打开新生成的输出文件，可以看到文件内容已经正常显示了。  

> iconv命令支持很多文件编码方式，具体的可以通过命令`iconv -l`进行查询。  


