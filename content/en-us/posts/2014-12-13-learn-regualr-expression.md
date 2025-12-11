---
title:  "Summary of Learning Regular Expressions"
date: 2014-12-13
slug: regular-expression
---

{{<alert warning>}}
This article is translated from Chinese to English by ChatGPT. There might be errors.
{{</alert>}}

Most of my previous understanding of regular expressions came from scattered bits of knowledge. Often, when I needed to use some regular expression and forgot how to write it, I would look it up on this [page](http://deerchao.net/tutorials/regex/regex.htm). Last time, when helping dhl solve a problem of extracting strings of the form `http://xxx.jpg` from HTML, after a series of frustrations, I decided to properly study regular expressions. I bought the book “Mastering Regular Expressions” during the Double 11 sale. I’ve read five chapters so far, hence this short summary.

<!--more-->

## Regular Expression Modes
This part is placed first because in many cases different modes will affect the matching results, and this is also something beginners in regular expressions tend to overlook. The modes and features supported by most engines are shown in the table below.

|Mode name|Function|Mode modifier|
|--|--|--|
|Case-insensitive mode|Case-insensitive, including characters in Unicode|`i`|
|Free-spacing (comment) mode|Ignores all whitespace characters outside character classes|`x`|
|Single-line mode (dotall mode)|`.` matches newline characters|`s`|
|Multi-line text mode (enhanced line-anchor mode)|`^` `$` match the start and end positions of embedded text lines in a string|`m`|

You can usually change the mode of a regular expression by setting a parameter. In engines that support mode modifiers, you can use `(?modifier)` to turn on a mode and `(?-modifier)` to turn off a mode, where `modifier` corresponds to the mode modifiers in the table above. The scope of a mode is within a single pair of parentheses, or between `(?modifier)` and `(?-modifier)`.

## Grouping
Content enclosed in `()` is generally called a **group**. Ordinary groups are captured by the regex engine, and you can later use metacharacters like `\1` `\2` to refer to previously captured content; they can also be used in replacements. This is called a **backreference**, and it is very common. The capture order is arranged according to the order of the left parentheses, so when there are nested parentheses, just count the left parentheses. In addition, there are many grouping constructs that are only used for grouping and are not captured (i.e., they do not produce backreference numbers). See the table below:

|Name|Syntax|Capturing|
|--|--|--|
|Named capturing|`(?<name>)`|Yes|
|Non-capturing group|`(?:...)`|No|
|Atomic group|`(?>...)`|No|
|Lookaround|`(?=...)` `(?!...)` `(?<=...)` `(?<!...)`|No|
|Mode modifier|`(?modifier)` `(?-modifier)`|No|
|Conditional|`(?if then \| else)` |No|

## Zero-Width Assertions
Zero-Width Assertions do not match actual text, but rather positions in the text. Because they do not match text, they are called “zero-width (or zero-length)”. The “zero-width assertions” in deerchao’s “Regular Expressions in 30 Minutes” tutorial are actually referring to lookahead; broadly speaking, anchors like `^` `$` are also zero-width assertions, because they do not match text either, only positions. The table below summarizes some anchors and zero-width assertions:

|Syntax|Description|
|--|--|
|`^`|Matches the start of the text. If multi-line text mode is enabled, it also matches the position after each newline character.|
|`\A`|Matches the start of the input text regardless of any mode.|
|`$`|Matches the end of the target string, or the position before the newline at the end of the entire string. If multi-line text mode is enabled, it matches the position before any newline character in the string.|
|`\Z`|Equivalent to `$` when multi-line text mode is disabled.|
|`\z`|Matches only the end of the string, regardless of any newline characters.|
|`\G`|The end position of the previous match (most engines), or the start position of the match (a few engines). This concept is a bit complex and I haven’t fully understood it yet.|
|`\b`|Word boundary: matches a position where exactly one of the two sides is a letter, digit, or underscore.|
|`\B`|Non-word boundary: matches a position where both sides are either letters, digits, or underscores, or both are not.|
|`(?=...)`|Positive lookahead. Asserts that the text to the right of this position matches the subexpression.|
|`(?!...)`|Negative lookahead. Asserts that the text to the right of this position does not match the subexpression.|
|`(?<...)`|Positive lookbehind. Asserts that the text to the left of this position matches the subexpression.|
|`(?<!...)`|Negative lookbehind. Asserts that the text to the left of this position does not match the subexpression.|

## Quantifiers and Alternation
The most basic quantifiers are shown in the table below; these are almost universally known.

|Syntax|Meaning|
|--|--|
|`*`|Can be repeated infinitely many times, or not appear at all|
|`+`|Can be repeated infinitely many times, but must appear at least once|
|`?`|May not appear, or may appear only once|
|`{m, n}`|Can appear from n to m times|

The key here is that quantifiers also have different modes, generally divided into the three categories below. The first two are widely supported; support for the third is weaker.

|Name|Syntax|Meaning|
|--|--|--|
|Greedy quantifiers|`*` `+` `?` `{m, n}`|Match as much as possible|
|Lazy quantifiers|`*?` `+?` `??` `{m, n}?`|Match as little as possible|
|Possessive quantifiers|`*+` `++` `?+` `{m, n}+`|Similar to greedy quantifiers, but once matched they will not backtrack|

The key to understanding these modes lies in understanding how regex engines perform matching. An expression-driven NFA engine, when encountering a quantifier or an alternation (`|`), records the current position, and then chooses the next matching path depending on the quantifier. Once matching fails, it backtracks to the last recorded optional position and chooses another path. For alternation, “another path” is the next alternative in the alternation structure; for greedy quantifiers, it first chooses to match as much as possible and, when matching fails, the “other path” is to match less; for lazy quantifiers, it first chooses not to match, and when matching fails, the “other path” is to try matching; for possessive quantifiers, their behavior is the same as greedy quantifiers, except that they discard the “other path”, so once matching fails, they cannot backtrack.

As for using quantifiers, theory is important, but experience matters even more. You only get a feel for them after using them a lot. After reading the book, I feel there are many traps involving quantifiers, especially with things like `.*`, which must be used with caution.

## Character Representation
Character representation is mainly where Unicode causes trouble. Relatively speaking, Chinese has fewer pitfalls, so if you’re not doing international programming work, you might never run into them. The main issue is that in some languages one character is composed of one base character plus any number of combining characters, while `.` only matches a single Unicode code point. I don’t know much about this, so I won’t go into detail. Some common character shorthands are listed in the table below:

|Syntax|Meaning|
|--|--|
|`\d`|Digit, equivalent to `[0-9]`. If Unicode is supported, it also matches Unicode digits (such as Roman numerals).|
|`\D`|Non-digit, equivalent to `[^\d]`.|
|`\w`|**In general** equivalent to `[a-zA-Z0-9_]`.|
|`\W`|Non-word character, equivalent to `[^\w]`.|
|`\s`|Whitespace character, equivalent to `[ \f\n\r\t\v]`. If Unicode is supported, it also includes Unicode whitespace characters.|
|`\S`|Non-whitespace character, equivalent to `[^\s]`.|
|`\p{Prop}`|A character that satisfies a certain Unicode property. There are many properties, which are not listed here.|
|`\P{Prop}`|A character that does not satisfy a certain Unicode property.|
|`\xnum` `\x{num}` <br>`\unum` `\Unum`|Hexadecimal escape or Unicode escape; usage varies across engines.|

I’ll stop here for now and add more content in the future if needed.