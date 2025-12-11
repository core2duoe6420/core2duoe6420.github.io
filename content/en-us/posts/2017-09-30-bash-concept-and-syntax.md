---
title: "BASH Concepts and Common Syntax"
date: 2017-09-30
slug: bash-concept-and-syntax
tags:
- Shell
---

{{<alert warning>}}
This article is translated from Chinese to English by ChatGPT. There might be errors.
{{</alert>}}

This article is a brief summary of the [Bash Reference Manual](https://www.gnu.org/software/bash/manual/bash.html).

<!--more-->

{{< toc >}}

# Basic Concepts

- `control operator`: used to control methods (to separate commands): `newline` `||` `&&` `&` `;` `;;` `;&` `;;&` `|` `|&` `(` `)`
- `metacharacter`: used to separate words (`word`): `|` `&` `;` `(` `)` `<` `>`
- The process Bash follows to execute a command is as follows:
	1. Read commands from a file or terminal;
	2. Split the input into words (`word`) and operators (`operator`); quoting rules (`Quoting`) are applied in this step. Words are separated by `metacharacter`. Alias expansion is performed at this step;
	3. Parse tokens (`token`) into simple (`simple`) and compound (`compound`) commands;
	4. Perform various expansions (`expansion`) (see below for expansion order), and put the expanded words into commands and arguments;
	5. Perform necessary redirections (`redirection`), and remove redirection operators from the argument list;
	6. Execute the command;
	7. Optionally wait for the command to complete and obtain the exit status.
- Quoting rules:
	- `\` escapes the next character; `\newline` is treated as line joining. Unless inside single quotes `'`, the result of the escape will **not** contain a newline (that is, `\n`);
	- `'` quotes all characters appearing inside the quotes;
	- `"` quotes all characters except `$` <code>&#96;</code> `\` `!`. `$` and <code>&#96;</code> retain their full expansion meaning. `\` only has escaping semantics when followed by `$` <code>&#96;</code> `"` `\` `newline`; otherwise it is output as a backslash. If history expansion is enabled, `!` will expand; if `!` is escaped by `\`, it will not expand, and the `\` will not be removed (it still outputs as `\!`);
	- ANSI-C quoting, in the form `$'\n'`, is interpreted using ANSI C escape semantics, for example:
		- `\nnn`: octal escape (up to 3 digits);
		- `\xHH`: hexadecimal escape (up to 2 digits);
		- `\uHHHH`: Unicode escape (up to 4 digits);
		- `\UHHHHHHHH`: Unicode escape (up to 8 digits);
		- `\cx`: `control-x` escape.
- `|` and `|&` form a pipeline. If `!` is added in front of the pipeline, the final exit code is logically negated, for example `! ls` (the space cannot be omitted, otherwise it is parsed as history expansion). `|&` redirects both `stdout` and `stderr`;
- `;` `&` `&&` `||` form a list, and the end of the list is marked by `;` `&` `newline`, among which `&&` `||` have the highest precedence, followed by `;` `&`. Commands marked with `&` run in the background. If job control is not enabled, the process’s `stdin` will be redirected to `/dev/null`;
- `( list )` creates a subshell and then executes the commands; `{ list; }` executes the commands in the current shell. Note that the space and the `;` cannot be omitted;
- `coproc` creates a coprocess;
- `parallel` is similar to `xargs`, but executes commands in parallel;


# Basic Syntax

In the syntax, most `;` can be replaced by `newline`.

## Loops

```bash
until test-commands; do      # Loop until test-commands returns 0
	consequent-commands;
done # exitcode is the exit code of the last command executed; if no commands are executed, it is 0
```
```bash
while test-commands; do      # Loop while test-commands does not return 0
	consequent-commands;
done # exitcode is the exit code of the last command executed; if no commands are executed, it is 0
```
```bash
for name [ [in [words …] ] ; ] do  # If the in clause is omitted, it is equivalent to in "$@"
	commands $name; 
done # exitcode is the exit code of the last command executed; if no commands are executed, it is 0
```
```bash
for (( expr1 ; expr2 ; expr3 )) ; do  # This is Bash-specific syntax
	commands; 
done # exitcode is the exit code of the last command executed; if any expr is invalid, it returns false
```

## Conditionals

```bash
if test-commands; then
	consequent-commands;
elif more-test-commands; then
	more-consequents;
else
	alternate-consequents;
fi # exitcode is the exit code of the last command executed; if no condition is true, returns 0
```
```bash
case word in 
	pattern1 | pattern2) command-list1 ;;  # If it matches and ends with ;;, return immediately
	pattern3 | pattern4) command-list2 ;&  # If it matches and ends with ;&, continue executing the next command-list
	pattern5 | pattern6) command-list3 ;;& # If it matches and ends with ;;&, continue matching the next item, and execute when matched
	*) command-list ;;
esac # exitcode is the exit code of the command-list executed; if there is no match, returns 0
```
```bash
select name [in words …]; do # Construct a menu for the user to choose from
	commands $name $REPLY; # $REPLY holds the selected index
	break;                 # break exits the select environment; otherwise select is executed repeatedly
done
```
```bash
(( expression ))  # Bash arithmetic expression; if the expression is non-zero, exitcode is 0, otherwise exitcode is 1
[[ expression ]]  # Bash conditional expression
```

### Bash Conditional Expressions (Bash Extensions)

Unless otherwise specified, files are followed through symbolic links and operations are performed on the target file rather than on the symbolic link itself.

|Syntax|Meaning|
|-|-|
|`-a file`|Return true if the file exists|
|`-b file`|Return true if the file exists and is a block device|
|`-c file`|Return true if the file exists and is a character device|
|`-d file`|Return true if the file exists and is a directory|
|`-e file`|Return true if the file exists|
|`-f file`|Return true if the file exists and is a regular file|
|`-g file`|Return true if the file exists and the set-group-id bit is set|
|`-h file`|Return true if the file exists and is a symbolic link|
|`-k file`|Return true if the file exists and the sticky bit is set|
|`-p file`|Return true if the file exists and is a named pipe (`FIFO`)|
|`-r file`|Return true if the file exists and is readable|
|`-s file`|Return true if the file exists and has a size greater than 0|
|`-t fd`|Return true if the file descriptor is open and refers to a terminal|
|`-u file`|Return true if the file exists and the set-user-id bit is set|
|`-w file`|Return true if the file exists and is writable|
|`-x file`|Return true if the file exists and is executable|
|`-G file`|Return true if the file exists and is owned by the current `egid`|
|`-L file`|Return true if the file exists and is a symbolic link|
|`-N file`|Return true if the file exists and has been modified since it was last read|
|`-O file`|Return true if the file exists and is owned by the current `euid`|
|`-S file`|Return true if the file exists and is a `socket`|
|`file1 -ef file2`|Return true if `file1` and `file2` refer to the same device and inode number|
|`file1 -nt file2`|Return true if `file1` is newer than `file2` (based on `mtime`), or if `file1` exists and `file2` does not|
|`file1 -ot file2`|Return true if `file1` is older than `file2` (based on `mtime`), or if `file1` does not exist and `file2` does|
|`-o optname`|Return true if the shell option `optname` is enabled|
|`-v varname`|Return true if the variable exists|
|`-R varname`|Return true if the variable exists and is a name reference|
|`-z string`|Return true if the string length is 0|
|`-n string` `string`|Return true if the string length is not 0|
|`string1 == string2` `string1 = string2`|Return true if the strings are equal. When used within `[[`, pattern matching is also supported|
|`string1 != string2`|Return true if the strings are not equal|
|`string1 < string2`|Return true if `string1` sorts before `string2` in lexicographical order|
|`string1 > string2`|Return true if `string1` sorts after `string2` in lexicographical order|
|`arg1 OP arg2`|`OP` can be `-eq` `-ne` `-lt` `-le` `-gt` `-ge`, meaning respectively that numeric `arg1` is equal to, not equal to, less than, less than or equal to, greater than, greater than or equal to `arg2`|

## Functions

Use `local` inside functions to define local variables.

```bash
name () compound-command [ redirections ]
```
```bash
# The command must end with ; & or newline
function name [()] compound-command [ redirections ]
```

## Special Parameters

|Parameter|Meaning|
|-|-|
|`$*`|All positional parameters from the first to the last. `$*` expands to `$1 $2 ...`, and `"$*"` expands to `$1c$2c...`, where `c` is `$IFS$`|
|`$@`|All positional parameters from the first to the last. `$@` expands to `$1 $2 ...`, and `"$@"` expands to `"$1" "$2" ...`. If there are no parameters, `$@` is removed|
|`$#`|The number of positional parameters|
|`$?`|The exit code of the most recently executed **foreground pipeline**|
|`$-`|The current shell option flags (set via `set`)|
|`$$`|The `pid` of the current shell; in a subshell, `$$` expands to the `pid` of the invoking shell|
|`$!`|The `pid` of the most recently started background job|
|`$0`|The name used to invoke the program (questionable)|
|`$_`|The last argument of the previous command executed (questionable)|

## Expansion

Shell expansions are performed in the following order:
1. Brace expansion;
2. Tilde expansion, parameter and variable expansion, arithmetic expansion, command substitution, process substitution, from left to right;
3. Word splitting;
4. Filename expansion (wildcards);
5. Quote removal.

Expansion syntaxes:

|<div style="width:150px">Expansion type</div>|Syntax|
|----|---|
|Brace expansion|`a{b,c,d}e` `{x..y[..incr]}`. Note that there must be no spaces before or after `,`. `x` and `y` can be numbers or single characters|
|Tilde expansion|`~` expands to `$HOME`, `~+` to `$PWD`, `~-` to `$OLDPWD`, `~N ~+N ~-N` expand to <code>&#96;dirs +/-N&#96;</code>|
|Parameter expansion|`${parameter}`|
|Arithmetic expansion|`$(( expression ))`|
|Command substitution|`$(command)` <code>&#96;command&#96;</code>. With the `$()` syntax, all characters retain their original meaning and none are special; with backticks, `$` <code>&#96;</code> `\` are special and can be escaped by `\`|
|Process substitution|`<(list)` `>(list)`, note that there must be no space between the angle bracket and the parentheses|
|Filename expansion|`*` `?` `[...]`|

Parameter expansion forms:
- `${parameter:-word}`: If `parameter` is unset, expand to `word`; otherwise expand to `parameter`;
- `${parameter:=word}`: If `parameter` is unset, assign the value obtained by expanding `word` to `parameter`, and then expand it;
- `${parameter:?word}`: If `parameter` is unset, output the expanded `word` to `stderr`; if the shell is non-interactive, exit;
- `${parameter:+word}`: If `parameter` is unset, substitute nothing; otherwise substitute the expansion of `word`;
- `${parameter:offset}` `${parameter:offset:length}`: Take a substring of `parameter`. When `offset` and `length` are negative, they count from the end of the string; when `offset` is negative, there must be a space between `:` and `-`;
- `${!prefix*}` `${!prefix@}`: Expand the names of variables whose names begin with `prefix` and join them using `$IFS`;
- `${!name[@]}` `${!name[*]}`: If `name` is an array (or associative array), expand to its indices (or keys). If not, then if `name` is defined, expand to 0; otherwise to `null`. If `@` is used, each key is expanded as a separate `"`-quoted word;
- `${#parameter}`: Expand to the length of the array;
- `${parameter#word}`: Expand after removing the shortest match of `word` from the beginning of `parameter`;
- `${parameter##word}`: Expand after removing the longest match of `word` from the beginning of `parameter`;
- `${parameter%word}`: Expand after removing the shortest match of `word` from the end of `parameter`;
- `${parameter%%word}`: Expand after removing the longest match of `word` from the end of `parameter`;
- `${parameter/pattern/string}`: Expand after replacing the part of `parameter` that matches `pattern` with `string`. `pattern` can begin with:
	- `/`: replace all matches; by default only the first match is replaced;
	- `#`: the match must start at the beginning;
	- `%`: the match must include the end.
- `${parameter^pattern}` `${parameter^^pattern}` `${parameter,pattern}` `${parameter,,pattern}`: Find parts of `parameter` matching `pattern` and change their case. `^` `^^` convert lowercase to uppercase; `,` `,,` convert uppercase to lowercase. `^` `,` convert only once; `^^` `,,` convert all matches. If `pattern` is omitted, it is equivalent to matching the entire content.

When filename expansion is performed and the `extglob` option is enabled (via `shopt`), the following patterns are also supported:
- `?(pattern-list)` matches zero or one occurrence
- `*(pattern-list)` matches zero or more occurrences
- `+(pattern-list)` matches one or more occurrences
- `@(pattern-list)` matches exactly one occurrence
- `!(pattern-list)` matches any string except those that match pattern-list

## Redirection

Bash uses some special files in redirections. If the operating system supports these files, Bash uses the OS-provided files; otherwise Bash simulates them:

|File|Meaning|
|-|-|
|`/dev/fd/fd`|Duplicate `fd`|
|`/dev/stdin`|Standard input|
|`/dev/stdout`|Standard output|
|`/dev/stderr`|Standard error|
|`/dev/tcp/host/port`|Redirect to `tcp://host:port`|
|`/dev/udp/host/port`|Redirect to `udp://host:port`|

### Redirection Syntax

|<div style="width:250px">Syntax</div>|Meaning|
|-|-|
|`[n]<word`|Redirect input|
|`[n]>[\|]word`|Redirect output; if `\|` is present, `noclobber` setting is ignored|
|`[n]>>word`|Append output|
|`&>word` `>&word` `>word 2>&1`|Redirect both `stdout` and `stderr`|
|`&>>word` `>>word 2>&1`|Append both `stdout` and `stderr`|
|<code>[n]\<\<[-]word<br/>&nbsp;&nbsp;&nbsp;&nbsp;here-document<br/>&nbsp;delimiter</code>|`here document`. If `word` is quoted (enclosed in `'`), `delimiter` is the de-quoted `word`, and no expansion is performed in the here-document. If `word` is unquoted, parameter and variable expansion, command substitution, and arithmetic expansion are performed inside the here-document; `\` must be used to escape `\` `$` <code>&#96;</code>. If `<<-` is used, leading tabs in the here-document are removed.|
|`[n]<<< word`|`here string`. `word` undergoes brace expansion, tilde expansion, parameter and variable expansion, command substitution, arithmetic expansion and quote removal; filename expansion and word splitting are not performed. A newline is appended to the result.|
|`[n]<&word` `[n]>&word`|Duplicate a file descriptor. If the descriptor specified by `word` is not open, an error is reported. If `word` is `-`, file descriptor `n` is closed.|
|`[n]<&digit-` `[n]>&digit-`|Move a file descriptor; that is, duplicate and then close `digit`.|
|`[n]<>word`|Open `word` as file descriptor `n` for both reading and writing. If `n` is omitted, it defaults to 0. If the file does not exist, it is created.|

## Arrays and Associative Arrays

Defining arrays:
- `name[subscript]=value`
- `declare -a name`
- `name=(value1 value2 ... )`

Defining associative arrays:
- `declare -A name`

Referencing elements of arrays or associative arrays:
- `${name[subscript]}`

# Shell Builtins

- Use `help cmd` to obtain help for a shell builtin;
- Use `type cmd` to check whether `cmd` is a shell builtin;
- `trap`: set a signal handler. Example:
```bash
trap "echo SIGINT" 2    # Set a handler for SIGINT
trap - 2                # Clear the handler
```

# Line Editing

Bash uses `readline` for line editing. `readline` attempts to read the configuration file specified by the shell variable `INPUTRC`; if the variable does not exist, it reads `~/.inputrc`; if that file does not exist, it reads `/etc/inputrc`.

Use `bind -P` to list all current key bindings and their functions, where
- `\C-x` means `ctrl-x`;
- `\M-x` means `Meta-x`, usually the `alt` key;
- `\C-\M-x` means press `ctrl` and `alt` simultaneously;
- `\ex` means `Esc-x`.

Use `bind -V` to list the current settings. For example, `bind "set completion-disabled on"` disables automatic completion.