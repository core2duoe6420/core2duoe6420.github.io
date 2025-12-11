---
title: "BASH概念和常见语法"
date: 2017-09-30
slug: bash-concept-and-syntax
tags:
- Shell
---

本文是[Bash用户手册](https://www.gnu.org/software/bash/manual/bash.html)的一个简要总结。

<!--more-->

{{< toc >}}

# 基础概念

- `control operator`：用于控制方法（分割命令）：`newline` `||` `&&` `&` `;` `;;` `;&` `;;&` `|` `|&` `(` `)`
- `metacharacter`：用于分割单词（`word`）：`|` `&` `;` `(` `)` `<` `>`
- Shell执行命令的过程如下：
	1. 从文件或终端读入命令；
	2. 将输入分割成单词（`word`）和操作符（`operator`），这个步骤会应用转义规则（`Qouting`）。单词通过 `metacharacter`划分。同义展开（`alias expansion`）在这一步执行；
	3. 将词（`token`）解析为简单（`simple`）和复合（`compound`）命令；
	4. 执行各种展开（`expansion`)（展开顺序见下），将展开的词放入命令和参数中；
	5. 执行必要的重定向（`redirection`），从参数列表中去掉重定向操作符；
	6. 执行命令；
	7. 选择性的等待命令完成，获取退出状态。
- 转义规则：
	- `\`转义下一个单词，`\newline`被认为行拼接，除非在`'`转义中，否则转义结果中**不会**出现换行（即`\n`）；
	- `'`转义出现在引号内的所有字符；
	- `"`转义转义除`$` <code>&#96;</code> `\` `!`外所有字符。`$` <code>&#96;</code>保持全有展开含义，`\`仅当后跟`$` <code>&#96;</code> `"` `\` `newline`这些字符时才有转义语义，否则仍然输出反斜杠。若启用了历史展开，则`!`会展开，若`!`被`\`转义，则不会展开，并且`\`不会移除（仍然输出`\!`）；
	- ANSI-C转义，形如`$'\n'`通过ANSI C语义转义，如：
		- `\nnn`：8进制转义（最多3位）；
		- `\xHH`：16进制转义（最多2位）；
		- `\uHHHH`：Unicode转义（最多4位）；
		- `\UHHHHHHHH`：Unicode转义（最多8位）；
		- `\cx`：`control-x`转义。
- `|`和`|&`组成管道（`pipeline`），若在管道前加入`!`，则最终的exitcode被取反，如`! ls`（空格不能省略，否则被解析为历史展开），`|&`同时重定向`stdout`和`stderr`；
- `;` `&` `&&` `||`组成列表（`list`），并由`;` `&` `newline`表示结束，其中`&&` `||`优先级最高，`;` `&`其次。由`&`表示的命令在后台运行，若没有启用任务控制（`job control`），则进程的`stdin`会被重定向到`/dev/null`；
- `( list )`会创建一个subshell，然后执行命令，`{ list; }`会在当前shell执行命令，注意空格和`;`不能省略；
- `coproc`会创建一个coprocess；
- `parallel`与`xargs`类似，但是会并行执行命令；


# 基础语法

语法中的`;`大多可以被`newline`替换。

## 循环

```bash
until test-commands; do      # 循环直到test-commands为0
	consequent-commands;
done # exitcode是最后执行的命令的exitcode，如果没有任何语句执行则为0
```
```bash
while test-commands; do     # 循环直到test-commands不为0
	consequent-commands;
done # exitcode是最后执行的命令的exitcode，如果没有任何语句执行则为0
```
```bash
for name [ [in [words …] ] ; ] do  # in语句省略时，相当于in "$@"
	commands $name; 
done # exitcode是最后执行的命令的exitcode，如果没有任何语句执行则为0
```
```bash
for (( expr1 ; expr2 ; expr3 )) ; do  # 应该是bash特有语法
	commands; 
done # exitcode是最后执行的命令的exitcode，如果任何expr非法，返回false
```

## 条件

```bash
if test-commands; then
	consequent-commands;
elif more-test-commands; then
	more-consequents;
else
	alternate-consequents;
fi # exitcode是最后执行的命令的exitcode，如没有任何条件为真返回0
```
```bash
case word in 
	pattern1 | pattern2) command-list1 ;;  # ;;结尾若匹配直接返回
	pattern3 | pattern4) command-list2 ;&  # ;&结尾若匹配继续执行下一个command-list
	pattern5 | pattern6) command-list3 ;;& # ;;&结尾若匹配则继续匹配下一项，匹配成功则执行
	*) command-list ;;
esac # exitcode是执行的command-list的exitcode，若没有任何匹配则返回0
```
```bash
select name [in words …]; do # 构造一个menu供用户选择
	commands $name $REPLY; # $REPLY表示序号
	break;                 # break退出select环境，否则会循环执行select
done
```
```bash
(( expression ))  # bash算术表达式，若表达式非0，exitcode为0，否则exitcode为1
[[ expression ]]  # bash条件表达式
```

### Bash条件表达式（Bash扩展）

若没有特殊指定，则文件会跟踪符号连接，在目标文件上操作，而不是符号链接本身。

|语法|含义|
|-|-|
|`-a file`|若文件存在，返回真|
|`-b file`|若文件存在并且是块设备，返回真|
|`-c file`|若文件存在并且是字符设备，返回真|
|`-d file`|若文件存在并且是目录，返回真|
|`-e file`|若文件存在，返回真|
|`-f file`|若文件存在并且是普通文件，返回真|
|`-g file`|若文件存在并且`set-group-id`被设置，返回真|
|`-h file`|若文件存在并且是符号链接，返回真|
|`-k file`|若文件存在并且`sticky`被设置，返回真|
|`-p file`|若文件存在并且是命名管道（`FIFO`），返回真|
|`-r file`|若文件存在并且可读，返回真|
|`-s file`|若文件存在并且大小大于0，返回真|
|`-t fd`|若文件描述符打开并且指向终端，返回真|
|`-u file`|若文件存在并且`set-user-id`被设置，返回真|
|`-w file`|若文件存在并且可写，返回真|
|`-x file`|若文件存在并且可执行，返回真|
|`-G file`|若文件存在并且被当前`egid`拥有，返回真|
|`-L file`|若文件存在并且是符号链接，返回真|
|`-N file`|若文件存在并且自它上次被读取以来被修改过，返回真|
|`-O file`|若文件存在并且被当前`euid`拥有，返回真|
|`-S file`|若文件存在并且是`socket`，返回真|
|`file1 -ef file2`|若`file1`和`file2`指向相同的设备和`inode`号，返回真|
|`file1 -nt file2`|若`file1`比`file2`新（根据`mtime`）或者`file1`存在而`file2`不存在，返回真|
|`file1 -ot file2`|若`file1`比`file2`老（根据`mtime`）或者`file1`不存在而`file2`存在，返回真|
|`-o optname`|若shell选项`optname`打开，返回真|
|`-v varname`|若变量存在，返回真|
|`-R varname`|若变量存在，并且是一个引用（`name reference`），返回真|
|`-z string`| 若字符串长度为0，返回真|
|`-n string` `string`|若字符串长度不为0，返回真|
|`string1 == string2` `string1 = string2`| 若字符串相等，返回真。若在`[[`中使用，还可以进行模式匹配|
|`string1 != string2`| 若字符串不等，返回真|
|`string1 < string2`|若`string1`在字典序上排序比`string2`靠前，返回真|
|`string1 > string2`|若`string1`在字典序上排序比`string2`靠后，返回真|
|`arg1 OP arg2`|`OP`可以是`-eq` `-ne` `-lt` `-le` `-gt` -`ge`。分别表示数字`arg1`比`arg2`相等、不等、小于、小于等于、大于、大于等于|

## 函数

在函数中使用`local`语句定义局部变量

```bash
name () compound-command [ redirections ]
```
```bash
# 命令必须以; & newline结尾
function name [()] compound-command [ redirections ]
```

## 特殊参数

|参数|含义|
|-|-|
|`$*`|从第一个到最后一个参数，`$*`会展开为`$1 $2 ...`，`"$*"`会展开`$1c$2c...`，其中`c`是`$IFS$`|
|`$@`|从第一个到最后一个参数，`$@`会展开为`$1 $2 ...`，`"$@"`会展开为`"$1" "$2" ...`，若没有参数则`$@`会被移除|
|`$#`|参数数量|
|`$?`|最近执行的**前台管道**的exitcode|
|`$-`|当前shell的选项标志（通过`set`设置）|
|`$$`|当前shell的`pid`，在subshell中，`$$`展开为调用shell的`pid`|
|`$!`|最近放入后台运行的job的`pid`|
|`$0`|程序运行的名字（有疑点）|
|`$_`|上一个执行的命令的最后一个参数（有疑点）|

## 展开

Shell展开通过以下顺序进行：
1. 括号展开（`brace expansion`）；
2. 波浪号展开（`tilde expansion`），参数和变量展开（`parameter and variable expansion`），算数展开（`arithmetic expansion`），命令替换（`command substitution`），进程替换（`process substitution`），展开从左到右进行；
3. 单词分割（`word splitting`）；
4. 文件名展开（`filename expansion`，即wildcard）；
5. 转义移除（`quote removal`）。

各种展开语法：

|<div style="width:150px">展开类型</div>|语法|
|----|---|
|括号展开|`a{b,c,d}e` `{x..y[..incr]}` 注意`,`前后不能有空格，`x` `y`可以是数字或者单个字符|
|波浪号展开|`~`展开为`$HOME`，`~+`展开为`$PWD`，`~-`展开为`$OLDPWD`，`~N ~+N ~-N`展开为<code>&#96;dirs +/-N&#96;</code>|
|参数展开|`${parameter}`|
|算数展开|`$(( expression ))`|
|命令替换|`$(command)` <code>&#96;command&#96;</code> 使用`$()`语法时，所有字符保持原有语义，没有任何特殊字符，使用反引号语法时，`$` <code>&#96;</code> `\`有特殊语义，可以被`\`转义|
|进程替换|`<(list)` `>(list)`，注意尖括号与圆括号之间不能有空格|
|文件名展开|`*` `?` `[...]`|

参数展开的格式：
- `${parameter:-word}`：若`paramter`没有设置，则展开`word`，否则展开`parameter`；
- `${parameter:=word}`：若`paramter`没有设置，则将`word`展开后的值赋给`parameter`，然后展开；
- `${parameter:?word}`：若`parameter`没有设置，则将`word`展开后输出到`stderr`，若shell不是交互式的则退出；
- `${parameter:+word}`：若`parameter`没有设置，不替换任何值，否则替换为`word`的展开；
- `${parameter:offset}` `${parameter:offset:length}`：取`parameter`的子字符串。`offset`和`length`为负数时，表示从字符串末端向前数的第n个字符，`offset`为负数时，`:`和`-`之间必须有空格；
- `${!prefix*}` `${!prefix@}`：将变量名以`prefix`开头的变量展开，并通过`$IFS`连结；
- `${!name[@]}` `${!name[*]}`：若`name`是数组（或字典），则展开为下标（或键值），如果不是，若`name`已定义，则展开为0，否则为`null`，如果使用`@`，则每个键值被展开为单独的`"`转义的词；
- `${#parameter}`：展开为数组长度；
- `${parameter#word}`：将`parameter`从左边开始最短匹配`word`的部分删除后展开；
- `${parameter##word}`：将`parameter`从左边开始最长匹配`word`的部分删除后展开；
- `${parameter%word}`：将`parameter`从右边开始最短匹配`word`的部分删除后展开；
- `${parameter%%word}`：将`parameter`从右边开始最长匹配`word`的部分删除后展开；
- `${parameter/pattern/string}`：将`parameter`匹配`pattern`的部分替换为`string`后展开。`pattern`可以以以下字符开始：
	- `/`：表示所有的匹配都要替换，一般情况下只替换一次；
	- `#`：匹配必须从开始位置匹配；
	- `%`：必须匹配结束位置。
- `${parameter^pattern}` `${parameter^^pattern}` `${parameter,pattern}` `${parameter,,pattern}`：查找`parameter`中匹配`pattern`的部分，并转换大小写，其中`^` `^^`将小写转换为大写，`,` `,,`将大写转换为小写，`^` `,`仅转换一次，`^^` `,,`转换全部。若`pattern`省略，则相当于匹配全部内容。

文件名展开时，若`extglob`选项打开（通过`shopt`命令），则还支持以下命令语法：
- `?(pattern-list)` 匹配0个或1个
- `*(pattern-list)` 匹配0个或多个
- `+(pattern-list)` 匹配1个或多个
- `@(pattern-list)` 匹配1个
- `!(pattern-list)` 匹配除pattern外的任何字符串

## 重定向

Bash使用一些在重定向中的特殊文件，如果操作系统支持这些文件，Bash会使用操作系统提供的文件，如果不支持，则Bash会模拟相应的文件：

|文件|含义|
|-|-|
|`/dev/fd/fd`|复制`fd`|
|`/dev/stdin`|标准输入|
|`/dev/stdout`|标准输出|
|`/dev/stderr`|标准错误|
|`/dev/tcp/host/port`|重定向到`tcp://host:port`|
|`/dev/udp/host/port`|重定向到`udp://host:port`|

### 重定向语法

|<div style="width:250px">语法</div>|含义|
|-|-|
|`[n]<word`|重定向输入|
|`[n]>[\|]word`|重定向输出，如果有`\|`，则无视`noclobber`设置|
|`[n]>>word`|追加输出|
|`&>word` `>&word` `>word 2>&1`|同时重定向`stdout`和`stderr`|
|`&>>word` `>>word 2>&1`|同时追加`stdout`和`stderr`|
|<code>[n]\<\<[-]word<br/>&nbsp;&nbsp;&nbsp;&nbsp;here-document<br/>&nbsp;delimiter</code>| `here document`，如果`word`被转义（被`'`括起），`delimiter`是转义后的`word`，并且不会展开`here-document`中的任何内容，如果`word`没有被转义，则`here-document`会进行参数和变量展开、命令替换和算数展开，`\`必须用于转义`\` `$` <code>&#96;</code>。如果使用`<<-`，则`here-document`中的前导`tab`会被移除。|
|`[n]<<< word`|`here string`，`word`会经过括号展开、波浪号展开、参数和变量展开、命令替换、算数展开和转义移除，文件名展开和单词分割不会进行，结果中会追加一个新行。|
|`[n]<&word` `[n]>&word`| 复制文件描述符，如果`word`指定的描述符没有打开，则报错，如果`word`是`-`，则文件描述符`n`被关闭。|
|`[n]<&digit-` `[n]>&digit-`| 移动文件描述符，即复制后关闭`digit`。|
|`[n]<>word`|打开`word`为文件描述符`n`用于读写，若`n`省略默认为0，若文件不存在，则会创建。|

## 数组和字典

定义数组：
- `name[subscript]=value`
- `declare -a name`
- `name=(value1 value2 ... )`

定义字典：
- `declare -A name`

引用数组或字典中的值：
- `${name[subscript]}`

# Shell builtin

- 使用`help cmd`获取Shell builtin的帮助；
- 使用`type cmd`可以查询`cmd`是否是Shell builtin；
- `trap`：设置信号处理程序。例子：
```bash
trap "echo SIGINT" 2    # 为SIGINT设置处理程序
trap - 2                # 清除处理程序
```

# Line Editing

Bash使用`readline`控制行编辑。`readline`会尝试读取shell变量`INPUTRC`中指定的配置文件，若变量不存在，读取`~/.inputrc`，若该文件不存在，读取`/etc/inputrc`。

使用`bind -P`可以列出所有当前绑定的按键和操作，其中
- `\C-x`表示`ctrl-x`；
- `\M-x`表示`Meta-x`，通常`Meta`键是`alt`键；
- `\C-\M-x`表示同时按下`ctrl`和`alt`
- `\ex`表示`Esc-x`。

使用`bind -V`列出当前设置。例如，使用`bind "set completion-disabled on"`可以禁用自动补全。

