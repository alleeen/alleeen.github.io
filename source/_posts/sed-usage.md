---
layout: post
title:  "Sed 命令详解"
date:   2016-09-17 19:27:03
comments: true
ads: true
categories: 软件技术
tags: [Linux,Shell,sed]
---

sed是stream editor的简称，也就是流编辑器。它一次处理一行内容，处理时，把当前处理的行存储在临时缓冲区中，称为“模式空间”（pattern space），接着用sed命令处理缓冲区中的内容，处理完成后，把缓冲区的内容送往屏幕。接着处理下一行，这样不断重复，直到文件末尾。文件内容并没有改变，除非你使用重定向存储输出。

## 使用语法
```shell
sed [option] 'command' input_file
```

常用的option有如下几种：
+ -n 使用安静(silent)模式。默认条件下，所有来自stdin的内容一般都会被列出到屏幕上。但如果加上-n参数后，则只有在脚本中使用`p`，被匹配的行才会被列出来，比如：`sed -n '/<HTML>/p'`（仅显示<HTML>这一行）；
+ -e 用于执行多个编辑命令，如：`sed -e '1,3s/my/your/g' -e '3,$s/This/That/g' my.txt`；
+ -f 从 script-file 中读取 sed 编辑命令，可以将多个编辑命令写在文件中，使用`sed -f script-file ...`读取；
+ -r 让sed命令支持扩展的正则表达式(默认是基础正则表达式)；
+ -i 直接修改读取的文件内容，默认下，sed 不会直接修改文件，当提供`-i`选项时 sed 会直接修改文件内容。

<!--more-->

常用的命令有以下几种：

+ `a \`： 在匹配的行下新起一行，追加字符串，`a \`的后面跟上字符串(多行字符串可以用`\n`分隔)；
+ `c \`： 取代/替换字符串，`c \`后面跟上字符串s(多行字符串可以用`\n`分隔)，会将当前选中的行替换成字符串s；
+ `d`： delete即删除，该命令会将当前选中的行删除；
+ `i \`： insert即插入字符串，`i \`后面跟上字符串s(多行字符串可以用\n分隔)，则会在当前选中的行的前面都插入字符串s；
+ `p`： print即打印，该命令会打印当前选择的行到屏幕上，通常同`-n`一起使用，打印选中的行；
+ `s`： 替换，通常s命令的用法是这样的：s/old/new/g，将old字符串替换成new字符串

## 多个匹配
有时我们需要一次进行多次匹配，可参考下面的示例：（第一个模式把第一行到第三行的my替换成your，第二个则把第3行以后的This替换成了That）

```
$ sed '1,3s/my/your/g; 3,$s/This/That/g' my.txt
This is your cat, your cat's name is betty
This is your dog, your dog's name is frank
That is your fish, your fish's name is george
That is my goat, my goat's name is adam
```
上面的命令等价于：（注：下面使用的是sed的-e命令行参数）

```
sed -e '1,3s/my/your/g' -e '3,$s/This/That/g' my.txt
```

我们可以使用&来当做被匹配的变量，然后可以在基本左右加点东西。如下所示：

```
$ sed 's/my/[&]/g' my.txt
This is [my] cat, [my] cat's name is betty
This is [my] dog, [my] dog's name is frank
This is [my] fish, [my] fish's name is george
This is [my] goat, [my] goat's name is adam
```

## 命令示例

假设有一个本地文件test.txt，文件内容如下：

```shell
$ cat test.txt

this is first line
this is second line
this is third line
this is fourth line
this fifth line
happy everyday
end
```

本节将使用该文件详细演示每一个命令的用法。

### a命令
```shell
$ sed '1a \add one' test.txt
this is first line
add one
this is second line
this is third line
this is fourth line
this is fifth line
happy everyday
end
```

本例命令部分中的1表示第一行，同样的第二行写成2，第一行到第三行写成`1,3`，用`$`表示最后一行，比如`2,$`表示第二行到最后一行中间所有的行(包含第二行和最后一行)。
本例的作用是在第一行之后增加字符串”add one”，从输出可以看到具体效果。需要注意的是在 Mac OS X 系统上，`a \`后的追加文本需要另起一行写，如：

```
$ sed '1a \
>add one' test.txt
```

```shell
$ sed '1,$a \add one' test.txt
this is first line
add one
this is second line
add one
this is third line
add one
this is fourth line
add one
this is fifth line
add one
happy everyday
add one
end
add one
```

本例表示在第一行和最后一行所有的行后面都加上”add one”字符串，从输出可以看到效果。

```shell
$ sed '/first/a \add one' test.txt
this is first line
add one
this is second line
this is third line
this is fourth line
this is fifth line
happy everyday
end
```

本例表示在包含”first”字符串的行的后面加上字符串”add one”，从输出可以看到第一行包含first，所以第一行之后增加了”add one”

```
$ sed '/^ha.*day$/a \add one' test.txt
this is first line
this is second line
this is third line
this is fourth line
this is fifth line
happy everyday
add one
end
```

本例使用正则表达式匹配行，`^ha.*day$`表示以ha开头，以day结尾的行，则可以匹配到文件的”happy everyday”这样，所以在该行后面增加了”add one”字符串。

### i命令

i命令使用方法和a命令一样的，只不过是在匹配的行的前面插入字符串，所以直接将上面a命令的示例的a替换成i即可，在此就不啰嗦了。

### c命令

```shell
$ sed '$c \add one' test.txt
this is first line
this is second line
this is third line
this is fourth line
this is     fifth line
happy everyday
add one
```

本例表示将最后一行替换成字符串”add one”，从输出可以看到效果。同`a`命令一样在 Mac OS X 系统上，`c \`后文本需要另起一行写，如：

```
$ sed '$c \
>add one' test.txt
```

```shell
$ sed '4,$c \add one' test.txt
this is first line
this is second line
this is third line
add one
```

本例将第四行到最后一行的内容替换成字符串”add one”。

```shell
$ sed '/^ha.*day$/c \replace line' test.txt
this is first line
this is second line
this is third line
this is fourth line
this is fifth line
replace line
end
```

本例将以ha开头，以day结尾的行替换成”replace line”。

### d命令

```shell
$ sed '/^ha.*day$/d' test.txt
this is first line
this is second line
this is third line
this is fourth line
this is fifth line
end
```

本例删除以`ha`开头，以`day`结尾的行。

```shell
$ sed '4,$d' test.txt
this is first line
this is second line
this is third line
```

本例删除第四行到最后一行中的内容。

### p命令

```shell
$ sed -n '4,$p' test.txt
this is fourth line
this is fifth line
happy everyday
end
```

本例在屏幕上打印第四行到最后一行的内容，p命令一般和-n选项一起使用。

```shell
$ sed -n '/^ha.*day$/p' test.txt
happy everyday
```

本例打印以`ha`开始，以`day`结尾的行。

### s命令

实际运用中s命令式最常使用到的。

```shell
$ sed 's/line/text/g' test.txt
this is first text
this is second text
this is third text
this is fourth text
this is fifth text
happy everyday
end
```

本例将文件中的所有line替换成text，最后的`g`是global的意思，也就是全局替换，如果不加g，则只会替换本行的第一个line。

```shell
$ sed '/^ha.*day$/s/happy/very happy/g' test.txt
this is first line
this is second line
this is third line
this is fourth line
this is fifth line
very happy everyday
end
```

本例首先匹配以ha开始，以day结尾的行，本例中匹配到的行是”happy everyday”这样，然后再将该行中的happy替换成very happy。

```shell
$ sed 's/\(.*\)line$/\1/g' test.txt
this is first
this is second
this is third
this is fourth
this is fifth
happy everyday
end
```

这个例子有点复杂，先分解一下。首先s命令的模式是s/old/new/g这样的，所以本例的old部分即`\(.*\)line$`，sed命令中使用`\(\)`包裹的内容表示正则表达式的第n部分，序号从1开始计算，本例中只有一个`\(\)`所以`\(.*\)`表示正则表达式的第一部分，这部分匹配任意字符串，所以`\(.*\)line$`匹配的就是以line结尾的任何行。然后将匹配到的行替换成正则表达式的第一部分（本例中相当于删除line部分），使用`\1`表示匹配到的第一部分，同样`\2`表示第二部分，`\3`表示第三部分，可以依次这样引用。比如下面的例子：

```shell
$ sed 's/\(.*\)is\(.*\)line/\1\2/g' test.txt
this  first
this  second
this  third
this  fourth
this  fifth
happy everyday
end
```

正则表达式中is两边的部分可以用`\1`和`\2`表示，该例子的作用其实就是删除中间部分的is。

## 一些关于 sed 的基础知识
前面通过实例说完了 sed 的运用，下面来说一些和 sed 相关的基础知识

### Pattern Space
什么是Pattern Space，Pattern space相当于车间sed把流内容在这里处理，你可以将pattern space看成是一个流水线，所有的动作都是在“流水线”上执行的。不理解？没关系，我们来看看 sed 的伪代码：
```
foreach line in file {
    //放入把行Pattern_Space
    Pattern_Space <= line;

    // 对每个pattern space执行sed命令
    Pattern_Space <= EXEC(sed_cmd, Pattern_Space);

    // 如果没有指定 -n 则输出处理后的Pattern_Space
    if (sed option hasn't "-n")  {
       print Pattern_Space
    }
}
```

![sed 执行流程图](/assets/images/sed-usage/pattern-space.png)

### Hold Space
什么是Hold Space？Hold space相当于仓库，加工的半成品在这里临时储存。由于各种各样的原因，比如用户希望在某个条件下脚本中的某个命令被执行，或者希望模式空间得到保留以便下一次的处理，都有可能使得sed在处理文件的时候不按照正常的流程来进行。这个时候，sed设置了一些高级命令来满足用户的要求。

+ g：[address[,address]]g 将hold space中的内容拷贝到pattern space中，原来pattern space里的内容清除
+ G：[address[,address]]G 将hold space中的内容append到pattern space后
+ h：[address[,address]]h 将pattern space中的内容拷贝到hold space中，原来的hold space里的内容被清除
+ H：[address[,address]]H 将pattern space中的内容append到hold space后
+ x： 交换pattern space和hold space的内容

那么这些命令怎么用呢，我们来看些例子，示例文件如下：
```
$ cat t.txt
one
two
three
```

如果我需要使用 sed 完成文件倒序输出要怎么做呢？你可以这样写：

```
sed '1!G;h;$!d' t.txt
```

其中的 ‘1!G;h;$!d’ 可拆解为三个命令
+ `1!G` —— 只有第一行不执行G命令，将hold space中的内容append回到pattern space
+ `h` —— 第一行都执行h命令，将pattern space中的内容拷贝到hold space中
+ `$!d` —— 除了最后一行不执行d命令，其它行都执行d命令，删除当前行

![执行序列](/assets/images/sed-usage/sed_demo.jpg)
