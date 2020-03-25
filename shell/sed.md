---
description: 'sed is not sad, wish u happy~'
---

# sed

## 序

sed \( Stream Editor \)，学习好 Shell 的这些利器，得明白一个特点，那就是他们都是 Line-Oriented  的，每次读出和处理一行，想着你现在写了一个程序，每次从一个缓冲区获取一行，也就是说，直到读到 `'\n'` 为止，接着你开始对你读出来的数据进行处理。

## 基本功能

这里提一个插曲，公司要求 tab 全部为4个空格，但是编码的时候用4个空格，其实非常不灵活，所以我的想法就是之后再使用脚本转换，然后想当然地用了 sed，但是后来发现它是将源文件再生成，于是乎我的 Git 记录变成了删除了全部行，增加了全部行，其实只需要使用 -i 选项即可，后来我使用 vim 自带的 \[range\]%s/pattern/substitution/g  也解决了这个问题~

### How it works

> 1. The next line is read from the input file and places it in the pattern space. If the end of file is found, and if there are additional files to read, the current file is closed, the next file is opened, and the first line of the new file is placed into the pattern space.
> 2. The line count is incremented by one. Opening a new file does not reset this number.
> 3. Each sed command is examined. If there is a restriction placed on the command, and the current line in the pattern space meets that restriction, the command is executed. Some commands, like "n" or "d" cause sed to go to the top of the loop. The "q" command causes sed to stop. Otherwise the next command is examined.
> 4. After all of the commands are examined, the pattern space is output unless sed has the optional "-n" argument.

sed 其实有非常多的功能，下面提一下它的格式

```bash
[addr]X[options]
```

{% hint style="info" %}
最开始接触的时候往往觉得 sed 指令非常复杂，其实不然，它一共就是三部分，第一部分，叫 address restriction, 第二部分就是 command, 然后是 参数，理解了这一点，才能理解那些一长串的指令。
{% endhint %}

 If `[addr]` is specified, the command X will be executed only on the matched lines. `[addr]`can be a single line number, a regular expression, or a range of lines \(see [sed addresses](https://www.gnu.org/software/sed/manual/sed.html#sed-addresses)\). Additional `[options]` are used for some `sed` commands.

对于提到的 addr 有以下的俩种

1. 行数
2. Pattern - /regexp/

其实最经常用到的是如下几种

```bash
sed  /pattern/ command [ additional args ]
sed  [beginNUM, endNum]s/PATTERN/sub/[flags]
```

### commands: d  -n p I a/i/c -e

```bash
sed 'addr' d 
例如 这其实实现了 head 的功能，只留下了前10行，并打印了出来~
sed '11,$ d' < file 
```

还可以用空行来指范围

```bash
sed '1,/^$/ d' < file 
```

删除 \# 开头的空行，都类似了，即删除 addr 所指代的行

```bash
sed '/^#/ d'

Removing comments and blank lines takes two commands. 
The first removes every character from the "#" to the 
end of the line, and the second deletes all blank lines:

sed -e 's/#.*//' -e '/^$/ d'
```

下面来实现，grep 的功能，  -n 即 -noprint

```bash
Sed can act like grep by combining the print operator 
to function on all lines that match a regular expression:

sed -n '/match/ p' 

whichis the same as:

grep match
```

当然，grep 的递归查找还是很方便的~  `-n` 代表只会打印匹配了的那一行

![](../.gitbook/assets/image%20%28161%29.png)

```bash
#!/bin/sh
sed '
/WORD/ {
i\
Add this line before
a\
Add this line after
c\
Change the line to this one
}'

for WORD in file, will be placed a line before/after/or changed
```

-e 的原理就是，例子上面也有，不多解释

> Copy the input line into the pattern space.  
>
>
> Apply the first   
> sed command on the pattern space, if the address restriction is true.  
>
>
> Repeat with the next sed expression, again  
> operating on the pattern space.  
>
>
> When the last operation is performed, write out the pattern space  
> and read in the next line from the input file.

{% hint style="info" %}
注意例子均来自 ref\[1\]，这里主要做一个笔记，然后加上点自己的理解
{% endhint %}

##  substitute command 

s 这是 sed 最出名的指令了，其实其中复杂还是在于正则表达式的使用，真正使用起来的方式非常的简单。

```bash
sed "[addr]s/PATTERN/CHANGED/[flags]" 

当然，前面还可以加 addr 
```

参考链接有一节，专门提到了这个 [Addresses and Ranges of Text](http://www.grymoire.com/Unix/Sed.html#toc-uh-25)

```bash
# Many UNIX utilities like vi and more use a slash 
# to search for a regular expression. Sed uses the same
# convention, provided you terminate the expression with 
# a slash. To delete the first number on all lines that
# start with a "#," use:

sed '/^#/ s/[0-9][0-9]*/
```

### Regular Expression

#### &

& 用来指代匹配的 pattern，跟 grep 是一样的，对于 find 指令，则是 { }

```bash
echo "123 abc" | sed 's/[0-9][0-9]*/& &/'
123 123 abc
```

#### -r

```bash
echo "123 abc" | sed -r 's/[0-9]+/& &/'
123 123 abc
```

-r 是 extended 的正则模式，支持一些集合，总之，发现不支持了，就得思考是不是因为不支持的原因，对于 grep 是 -E

![](../.gitbook/assets/image%20%28173%29.png)

#### \1 \2  to keep part of the pattern

这一点和 grep 也是一样的

```bash
sed -r 's/([a-z]+) ([a-z]+)/\2 \1/' # Using GNU sed
```

####  flags or command

对于 `s` 指令的格式是这样的

`‘s/regexp/replacement/flags’`

下面对于 flag 的说明，来自 ref\[2\]

> `g`
>
> Apply the replacement to _all_ matches to the regexp, not just the first.`number`
>
> Only replace the numberth match of the regexp.
>
> interaction in `s` command Note: the POSIX standard does not specify what should happen when you mix the `g` and number modifiers, and currently there is no widely agreed upon meaning across `sed` implementations. For GNU `sed`, the interaction is defined to be: ignore matches before the numberth, and then match and replace all matches from the numberth on.`p`
>
> If the substitution was made, then print the new pattern space.
>
> Note: when both the `p` and `e` options are specified, the relative ordering of the two produces very different results. In general, `ep` \(evaluate then print\) is what you want, but operating the other way round can be useful for debugging. For this reason, the current version of GNU `sed`interprets specially the presence of `p` options both before and after `e`, printing the pattern space before and after evaluation, while in general flags for the `s` command show their effect just once. This behavior, although documented, might change in future versions.`w filename`
>
> If the substitution was made, then write out the result to the named file. As a GNU `sed` extension, two special values of filename are supported: /dev/stderr, which writes the result to the standard error, and /dev/stdout, which writes to the standard output.[3](https://www.gnu.org/software/sed/manual/sed.html#FOOT3)`e`
>
> This command allows one to pipe input from a shell command into pattern space. If a substitution was made, the command that is found in pattern space is executed and pattern space is replaced with its output. A trailing newline is suppressed; results are undefined if the command to be executed contains a NUL character. This is a GNU `sed` extension.`Ii`
>
> The `I` modifier to regular-expression matching is a GNU extension which makes `sed` match regexp in a case-insensitive manner.`Mm`
>
> The `M` modifier to regular-expression matching is a GNU `sed` extension which directs GNU `sed` to match the regular expression in multi-linemode. The modifier causes `^` and `$` to match respectively \(in addition to the normal behavior\) the empty string after a newline, and the empty string before a newline. There are special character sequences \(``\``` and `\'`\) which always match the beginning or the end of the buffer. In addition, the period character does not match a new-line character in multi-line mode.

值得注意的是，p 同时也是一条 command，g 也是，只是用的不多，下面提一些例子，来自 ref\[1\]

```bash
# GNU has added another pattern flags - /I

# This flag makes the pattern match case insensitive. 
# This will match abc, aBc, ABC, AbC, etc.:

sed -n '/abc/I p' <old >new

# Note that a space after the '/I' and the 'p' (print) command 
# emphasizes that the 'p' is not a modifier of the pattern matching 
# process, , but a command to execute after the pattern matching.
```

```bash
# By default, sed prints every line. If it makes a substitution,
# the new text is printed instead of the old one. If you use an
# optional argument to sed, "sed -n," it will not, by default,
# print any new lines. I'll cover this and other options later. 
# When the "-n" option is used, the "p" flag will cause the modified 
# line to be printed. Here is one way to duplicate the function 
# of grep with sed:

sed -n 's/pattern/&/p' <file
```

## Cheat Sheet

![](../.gitbook/assets/image%20%28153%29.png)

{% hint style="info" %}
ref\[3\] 建议 sed 用双引号，而不是单引号
{% endhint %}

![](../.gitbook/assets/image%20%2889%29.png)

### 转义

`/ /` 内的是原生字符串，照道理我们应该只需要  `\\` 即可，但是这里却非常奇怪，跟 `Java` 写这条语句一样了，需要 `4`个 `\`

```bash
$ echo "path\to\windows\file" | sed "s/\\\\/\//g"
/path/to/windows/file
```

## References

[Sed](http://www.grymoire.com/Unix/Sed.html#uh-15b)

[Sed Manual](https://www.gnu.org/software/sed/manual/sed.html#sed-addresses)

[Advanced Bash-Scripting Guide](https://www.tldp.org/LDP/abs/html/index.html)

