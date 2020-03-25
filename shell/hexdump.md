# hexdump

## 序

hexdump 的技巧关键在于实现它的 format 功能，其实就是 line-oriented 的一个 print 函数 

## Format String

### -e 选项

```bash
echo "Hello, world!~" \
    | hexdump  -e  '"%06_ax " 8/1 "%02x_L[green:0x65@1-2,!red:0xAA@1-2] ""\t"' \
                -e '8/1 "%c" "\n" '
```

其中下划线开始的 format  代表的是

```text
Conversion strings
       The hexdump utility also supports the following additional conversion
       strings.

       _a[dox]
              Display the input offset, cumulative across input files, of
              the next byte to be displayed.  The appended characters d, o,
              and x specify the display base as decimal, octal or
              hexadecimal respectively.

       _A[dox]
              Identical to the _a conversion string except that it is only
              performed once, when all of the input data has been processed.
              
       _c     Output characters in the default character set.  Non-printing
              characters are displayed in three-character, zero-padded
              octal, except for those representable by standard escape
              notation (see above), which are displayed as two-character
              strings.

       _p     Output characters in the default character set.  Non-printing
              characters are displayed as a single '.'.

       _u     Output US ASCII characters, with the exception that control
              characters are displayed using the following, lower-case,
              names.  Characters greater than 0xff, hexadecimal, are
              displayed as hexadecimal strings.

                 000 nul   001 soh   002 stx   003 etx   004 eot   005 enq
                 006 ack   007 bel   008 bs    009 ht    00A lf    00B vt
                 00C ff    00D cr    00E so    00F si    010 dle   011 dc1
                 012 dc2   013 dc3   014 dc4   015 nak   016 syn   017 etb
                 018 can   019 em    01A sub   01B esc   01C fs    01D gs
                 01E rs    01F us    0FF del
```

hexdump 的格式和 printf 相似，这就相当于我们能直接控制整个文件的输出格式，是非常方便的格式，除此之外，还支持高亮，在数据多的时候相当于多了一个检索的功能，是一款非常牛的工具。

上面的格式也很清楚，每一行解析一次所有的 -e ，跟 sed 是非常类似的，然后格式跟 printf 类似。

#### counter

区别在于增加了一个 counter，这是内置的计数器，用来输出地址，其实准确度的说是 offset，下划线开头就是其内置的，上面的manual说的比较清楚了。值得注意的是，format 必须得用双引号。

#### x/y

这个不需要双引号，x 代表的是 y 的个数，可以理解为 blcok counter，而 y 则是代表一个 block 内有几个bytes，老生常谈的参数了，正是这俩个参数可以自定义每一行的格式，使得 hexdump 成为了一个强大的工具。

#### format specifier

![](../.gitbook/assets/image%20%2886%29.png)

除了几个特别的其他的都是类似，当然还有一些对齐，还有补0，都可以支持，不详细说了。

#### color

在 `format specifier` 后可以跟颜色标签

![](../.gitbook/assets/image%20%2890%29.png)

最开始的例子已经非常清晰了！~ 

## 末

总结一句，这一条指令，就是一个强大的 printf，足以匹敌所有的二进制编辑软件，然后支持的颜色，可以让打印的时候突出某些数据~

