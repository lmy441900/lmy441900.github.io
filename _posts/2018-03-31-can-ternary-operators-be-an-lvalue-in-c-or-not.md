---
layout: post
title:  "Can ?: Be an Lvalue in C or Not?"
date:   2018-03-31
categories: en
---

**TL;DR: `?:` cannot be an lvalue in C, but it _can_ be an lvalue in C++.**

Okay, I am going to write a loooong piece of bulls\*\*t here.

It starts with a fun story in my final exam of _Structured Programming_ last semester. We learn C programming language in this course; however, we use Visual Studio 2010 for practice (and you know, Visual Studio supports C++, but C). Our final exam was a written one. One of the problem was to write a small piece of code (I can't remember what exactly it required, but it was likely related to linked list), and my answer had an `if-else` which determines which variable to assign:

```c
if (condition) {
  x = expression;
} else {
  y = expression;
}
```

And I was just too paranoid on "DRY" (Don't Repeat Yourself). It looks a little bit redundant: there are 2 `expression`s (they are longer than the word "expression" in my answer). Really, I have no idea why I came up with:

```c++
(condition ? x : y) = expression;
```

I might have seen such code before. I _thought_ I had wrote correctly.

One day my teacher <!-- Dr. Dyce --> had a dinner with me. During the dinner we talked about my project, and she told me that my code was just hard to understand. "It seems that you are trying hard to shorten the length of your code," she said. "I have to tell you that code is written not just for you, but also for others, even it's an examination. _I remember that in the exam you wrote `?:` on the left of `=`. The teacher who read your exam paper gave you zero. Later when I checked back I know that was right. So I went to that teacher, telling her this was correct, but she kept thinking this was wrong. I went to my computer, copied your code to the computer, ran the program, proving that your code is okay, then you finally get the points._"

This interesting fact made me even believe in my beautiful code.

What made me rethink this answer is that a month ago when I was implementing linked list, I wrote like what I had wrote in my final exam, but did not work. GCC kept throwing an error telling me that **lvalue cannot be read-only**. I thought, how come? I wrote this in my final examination, and my teacher even used a computer to prove that this is correct!

Since my habit is always using `gcc -std=c99 -pedantic -Wall -Wextra`, I began thinking about the standard problem. So I changed to `-std=c11`, even `-std=gnu<xx>`, none of them worked. So after several failures I tried to compile my code using `g++`. Surprisingly, it passed!

So it reveals that, ternary operators, also known as conditional operators (`?:`), has behavioral differences between C and C++. From [Wikipedia](https://en.wikipedia.org/wiki/%3F:#C):

> ### C
> The behaviour is _undefined_ if an attempt is made to use the result of the conditional operator as an lvalue.
>
> ...
>
> ### C++
>
> **Unlike in C, the precedence of the ?: operator in C++ is the same as that of the assignment operator (= or OP=)**, and **it can return an lvalue**. This means that expressions like `q ? a : b = c` and **`(q ? a : b) = c`** are both **legal** and are parsed differently, the former being equivalent to `q ? a : (b = c)`.

From [cppreference.com's notes on C's `?:`](http://en.cppreference.com/w/c/language/operator_precedence#Notes):

> The C language standard doesn't specify operator precedence... There is a part of the grammar that cannot be represented by a precedence table: **an assignment-expression is not allowed as the right hand operand of a conditional operator**, so `e = a < d ? a++ : a = d` is an expression that _cannot_ be parsed, and therefore relative precedence of conditional and assignment operators cannot be described easily.
>
> However, many C compilers use non-standard expression grammar where ?: is designated higher precedence than =, which parses that expression as `e = ( ((a < d) ? (a++) : a) = d )`, which then fails to compile due to semantic constraints: **?: is never lvalue and = requires a modifiable lvalue on the left**. This is the table presented on this page.

And from [C++'s](http://en.cppreference.com/w/cpp/language/operator_precedence#Notes):

> ... **?: is never lvalue in C and = requires lvalue on the left. In C++, ?: and = have equal precedence and group right-to-left**, so that `e = a < d ? a++ : a = d` parses as `e = ((a < d) ? (a++) : (a = d))`.

... we know that ternary operators (conditional operators) **can never be an lvalue in C, but it can be in C++.**

So how come my teacher could prove that my answer was correct? **It was because she used Visual Studio to run my code.** Visual Studio treat the source code as C++.

## References

- [https://en.wikipedia.org/wiki/%3F:](https://en.wikipedia.org/wiki/%3F:)
- [http://en.cppreference.com/w/c/language/operator_precedence](http://en.cppreference.com/w/c/language/operator_precedence)
- [http://en.cppreference.com/w/cpp/language/operator_precedence](http://en.cppreference.com/w/cpp/language/operator_precedence)
- On "lvalue" and "rvalue": [https://en.wikipedia.org/wiki/Value_(computer_science)#lrvalue](https://en.wikipedia.org/wiki/Value_(computer_science)#lrvalue)
