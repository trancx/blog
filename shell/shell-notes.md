# Shell 笔记

## 序

最近工作中，脚本的需求越来越高，而自己经常地忘记，所以这里自己总结一下，到时候方便来看，首先是基础的流程控制的语法，这个比较容易记混，然后是 sed grep hexdump cut awk 等利器，实际上每一个都开一课题了，慢慢往上加吧。

## 控制语句，变量以及函数

### 流程控制

```bash
if [ $a == $b ]
then
   echo "a 等于 b"
elif [ $a -gt $b ]
then
   echo "a 大于 b"
elif [ $a -lt $b ]
then
   echo "a 小于 b"
else
   echo "没有符合的条件"
fi
```

值得注意的是，**else** 是没有 **then** 的。

```bash
for str in 'This is a string'
do
    echo $str
done

如下所示，这里的 for 循环与 C 中的相似，但并不完全相同。
通常情况下 shell 变量调用需要加 $,但是 for 的 (()) 中不需要,下面来看一个例子：

for((i=1;i<=5;i++));
do
    echo "这是第 $i 次调用";
done;
```

```bash
while(( $int<=5 ))
do
    echo $int
    let "int++"
done

while true
do
    command
done
```

注意的是，不能有空语句，在流程控制中~

### 特殊参数及运算符

| 参数处理 | 说明 |
| :--- | :--- |
| $\# | 传递到脚本的参数个数 |
| $\* | 以一个单字符串显示所有向脚本传递的参数 |
| $$ | 脚本运行的当前进程ID号 |
| $! | 后台运行的最后一个进程的ID号 |
| $@ | 与$\*相同，但是使用时加引号，并在引号中返回每个参数。 |
| $- | 显示Shell使用的当前选项，与set命令功能相同。 |
| $? | 显示最后命令的退出状态。0表示没有错误，其他任何值表明有错误，调用即更新 |

```bash
function demoFun1(){
    echo "这是我的第一个 shell 函数!"
    return `expr 1 + 1`
}

demoFun1 # 这之后就可以加参数
echo $?
echo $?

output:  2 0
另外参数可以用 $x 表示~
```

运算符比较复杂，需要的时候参考[菜鸟](https://www.runoob.com/linux/linux-shell-basic-operators.html)。

![](../.gitbook/assets/image%20%285%29.png)

![](../.gitbook/assets/image%20%2851%29.png)

注意的是， Shell 也支持 && \|\| 

* ```text
  if [ -e myfile ] || [ -L myfile ]; then …
  ```

![](../.gitbook/assets/image%20%2858%29.png)

![](../.gitbook/assets/image%20%2846%29.png)

还有一些文件操作函数，嗯，用的不多，一般就是 -f

![](../.gitbook/assets/image%20%2862%29.png)

### 输入输出重定向

说明的是在脚本中的重定向，

![](../.gitbook/assets/image%20%28119%29.png)

```bash
 #pipe
 cat /sites/linuxpig.com.txt | while read LINE
 do
     echo $LINE
 done

 #===========================================================#
while read LINE
 do
     echo $LINE
 done < /xxxx.txt
 
 #===========================================================#
 exec 3<&0 #先将文件描述符0复制到文件描述符3，也就是给文件描述符0做个备份
 exec 0</sites/linuxpig.com.txt  #读文件到文件描述符0
 while read LINE # read 指定是从 stdin 读
 do
     echo $LINE
 done
 exec 0<&3 #将文件描述符3复制给文件描述符0（恢复0从键盘读入）
```

## 字符串拼接，变量索引

关联数组（Discrete Array）这里不多说了，下面提一下字符串的自带拆分功能，这个比较好用。

```bash
1、第一种方法

${varible##*string}  从左向右截取最后一个string后的字符串
${varible#*string}  从左向右截取第一个string后的字符串
${varible%%string*}  从右向左截取最后一个string后的字符串
${varible%string*}  从右向左截取第一个string后的字符串

“*” 不能省略

例子：
$ MYVAR=foodforthought.jpg
$ echo ${MYVAR##*fo}
rthought.jpg
$ echo ${MYVAR#*fo}
odforthought.jpg

2、第二种方法：${varible:n1:n2}:截取变量varible从n1到n2之间的字符串。

可以根据特定字符偏移和长度，使用另一种形式的变量扩展，来选择特定子字符串。试着在 bash 中输入以下行：
$ EXCLAIM=cowabunga
$ echo ${EXCLAIM:0:3}
cow
$ echo ${EXCLAIM:3:7}
abunga
```

当然， cut sed tr 那些同样可以实现，这里都自带的功能~

### Bash operators \[\[ vs \[ vs \( vs \(\(

```bash
if [[condition]]

if [condition]

if ((condition))

if (condition)
```

此处是引用 自 [Exchange Stack](https://unix.stackexchange.com/questions/306111/what-is-the-difference-between-the-bash-operators-vs-vs-vs)  

* `(…)` parentheses indicate a [subshell](http://www.gnu.org/software/bash/manual/bash.html#Command-Grouping). What's inside them isn't an expression like in many other languages. It's a list of commands \(just like outside parentheses\). These commands are executed in a separate subprocess, so any redirection, assignment, etc. performed inside the parentheses has no effect outside the parentheses.
  * With a leading dollar sign, `$(…)` is a [command substitution](http://www.gnu.org/software/bash/manual/bash.html#Command-Substitution): there is a command inside the parentheses, and the output from the command is used as part of the command line \(after extra expansions unless the substitution is between double quotes, but that's [another story](https://unix.stackexchange.com/questions/131766/why-does-my-shell-script-choke-on-whitespace-or-other-special-characters)\).
* `{ … }` braces are like parentheses in that they group commands, but they only influence parsing, not grouping. The program `x=2; { x=4; }; echo $x` prints 4, whereas `x=2; (x=4); echo $x` prints 2. \(Also braces being _keywords_ need to be delimited and found in command position \(hence the space after `{` and the `;` before `}`\) whereas parentheses don't. That's just a syntax quirk.\)
  * With a leading dollar sign, `${VAR}` is a [parameter expansion](http://www.gnu.org/software/bash/manual/bash.html#Shell-Parameter-Expansion), expanding to the value of a variable, with possible extra transformations. The `ksh93` shell also supports `${ cmd;}` as form of command substitution that doesn't spawn a subshell.
* `((…))` double parentheses surround an [arithmetic instruction](http://www.gnu.org/software/bash/manual/bash.html#Shell-Arithmetic), that is, a computation on integers, with a syntax resembling other programming languages. This syntax is mostly used for assignments and in conditionals. This only exists in ksh/bash/zsh, not in plain sh.
  * The same syntax is used in arithmetic expressions `$((…))`, which expand to the integer value of the expression.
* `[ … ]` single brackets surround [conditional expressions](http://www.gnu.org/software/bash/manual/bash.html#index-_005b_005b). Conditional expressions are mostly built on [operators](http://www.gnu.org/software/bash/manual/bash.html#Bash-Conditional-Expressions) such as `-n "$variable"` to test if a variable is empty and `-e "$file"` to test if a file exists. Note that you need a space around each operator \(e.g. `[ "$x" = "$y" ]`, not ~~`[ "$x"="$y" ]`~~\), and a space or a character like `;` both inside and outside the brackets \(e.g. `[ -n "$foo" ]`, not `[-n "$foo"]`\).
* `[[ … ]]` double brackets are an alternate form of conditional expressions in ksh/bash/zsh with a few additional features, for example you can write `[[ -L $file && -f $file ]]` to test if a file is a symbolic link to a regular file whereas single brackets require `[ -L "$file" ] && [ -f "$file" ]`. See [Why does parameter expansion with spaces without quotes works inside double brackets \[\[ but not single brackets \[?](https://unix.stackexchange.com/questions/32210/using-single-or-double-bracket-bash/32227#32227) for more on this topic.

In the shell, _every_ command is a conditional command: every command has a return status which is either 0 indicating success or an integer between 1 and 255 \(and potentially more in some shells\) indicating failure. The `[ … ]` command \(or `[[ … ]]` syntax form\) is a particular command which can also be spelled `test …` and succeeds when a file exists, or when a string is non-empty, or when a number is smaller than another, etc. The `((…))` syntax form succeeds when a number is nonzero. Here are a few examples of conditionals in a shell script:

* Test if `myfile` contains the string `hello`:

  ```text
  if grep -q hello myfile; then …
  ```

* If `mydir` is a directory, change to it and do stuff:

  ```text
  if cd mydir; then
    echo "Creating mydir/myfile"
    echo 'some content' >myfile
  else
    echo >&2 "Fatal error. This script requires mydir to exist."
  fi
  ```

* Test if there is a file called `myfile` in the current directory:

  ```text
  if [ -e myfile ]; then …
  ```

* The same, but also including dangling symbolic links:

  ```text
  if [ -e myfile ] || [ -L myfile ]; then …
  ```

* Test if the value of `x` \(which is assumed to be numeric\) is at least 2, portably:

  ```text
  if [ "$x" -ge 2 ]; then …
  ```

* Test if the value of `x` \(which is assumed to be numeric\) is at least 2, in bash/ksh/zsh:

  ```text
  if ((x >= 2)); then …
  ```

==================提到的网站有一个这样的问题=====================

I'm confused with using single or double brackets. Look at this code:

```text
dir="/home/mazimi/VirtualBox VMs"

if [[ -d ${dir} ]]; then
    echo "yep"
fi
```

It works perfectly although the string contains a space. But when I change it to single bracket:

```text
dir="/home/mazimi/VirtualBox VMs"

if [ -d ${dir} ]; then
    echo "yep"
fi
```

It says:

```text
./script.sh: line 5: [: /home/mazimi/VirtualBox: binary operator expected
```

When I change it to:

```text
dir="/home/mazimi/VirtualBox VMs"

if [ -d "${dir}" ]; then
    echo "yep"
fi
```

It works fine. Can someone explain what is happening? When should I assign double quotes around variables like `"${var}"` to prevent problems caused by spaces?

下面是解答

The single bracket `[` is actually an alias for the `test` command, it's _not_ syntax.

One of the downsides \(of many\) of the single bracket is that if one or more of the operands it is trying to evaluate return an empty string, it will complain that it was expecting two operands \(binary\). This is why you see people do `[ x$foo = x$blah ]`, the `x` guarantees that the operand will never evaluate to an empty string.

The double bracket `[[ ]]`, on the other hand, _is syntax_ and is much more capable than `[ ]`. As you found out, it does not have the single operand issue and it also allows for more C-like syntax with `>, <, >=, <=, !=, ==, &&, ||` operators.

My recommendation is the following: If your interpreter is `#!/bin/bash`, then _always_ use `[[ ]]`

It is important to note that `[[ ]]` is not supported by all POSIX shells, however many shells do support it such as `zsh` and `ksh` in addition to `bash`

当然，`man bash` 才是王道~

上面解释的比较的关键的一点就在于，$\(\) 其实是把 sub shell 的 output 作为了一个变量

### Always use double quotes around variable substitutions and command substitutions: `"$foo"`, `"$(foo)"`

引用自 [Exchange Stack](https://unix.stackexchange.com/questions/131766/why-does-my-shell-script-choke-on-whitespace-or-other-special-characters)

#### Why do I need to write `"$foo"`? What happens without the quotes?

`$foo` does not mean “take the value of the variable `foo`”. It means something much more complex:

* First, take the value of the variable.
* Field splitting: treat that value as a whitespace-separated list of fields, and build the resulting list. For example, if the variable contains `foo * bar ​` then the result of this step is the 3-element list `foo`, `*`, `bar`.
* Filename generation: treat each field as a glob, i.e. as a wildcard pattern, and replace it by the list of file names that match this pattern. If the pattern doesn't match any files, it is left unmodified. In our example, this results in the list containing `foo`, following by the list of files in the current directory, and finally `bar`. If the current directory is empty, the result is `foo`, `*`, `bar`.

具体生成 image 在笔记中，`shell/`

![](../.gitbook/assets/image%20%2832%29.png)

对于数组，还有特性，这俩图均参考于[这儿](https://www.tldp.org/LDP/abs/html/index.html)

```bash
string=abcABC123ABCabc
echo ${string[@]}               # abcABC123ABCabc
echo ${string[*]}               # abcABC123ABCabc 
echo ${string[0]}               # abcABC123ABCabc
echo ${string[1]}               # No output!
                                # Why?
echo ${#string[@]}              # 1
                                # One element in the array.
                                # The string itself.

# Thank you, Michael Zick, for pointing this out.
```

