---
title: "Using LLVM LIT Out-Of-Tree"
categories: tech
tags:
    - Compiler
    - LLVM
    - Testing
---
[LIT](https://llvm.org/docs/CommandGuide/lit.html) is an end-to-end testing infrastructure used in [LLVM Project](https://github.com/llvm/llvm-project). It’s powerful, flexible, and most important of all, it’s modular: The core of lit is simply a python module that you can fetch from [PyPi](https://pypi.org/). Making many people want to use it in their own projects, but pushed back by its non-trivial setups. In this article, I’m first gonna show you when and why should you consider using lit for your tests. Then I’ll demonstrate how to integrate it into your project without a **~20GB** LLVM build sitting in your disk, or even putting your test into the LLVM tree.

## When and Why should I use LIT
To give you a flavor of how lit works, here is a simple motivated example:

{% highlight cpp %}
Put better example here
{% endhighlight %}

This code was extracted from [one of clang’s lexer regression tests](https://github.com/llvm/llvm-project/blob/master/clang/test/Lexer/header.cpp). The entire snippet is nothing but normal C/C++ code, surrounded by some weird-written comments. Despite their special notations, those comments are describing how to run this test case and how to validate the output.

The `RUN` directives in the first and second lines are the commands to process this source file, note that result of the second line is piped into another program, `FileCheck`, that will check the output against the `CHECK` directives in the comments of the very same source file. As you might have guess, `CHECK` directives are a bunch of validation rules written in a regex-alike syntax.

The example above has already shown some advantages using lit over other end-to-end testing infrastructures:
1. Test cases and runners are integrated together. Not only does this eliminate the need to write extra code to pull test cases (if they’re organized in files) into your test runner, it’s also easier to read individual test.
2. The command after `RUN` directive is not tied to any specific (LLVM) tool: You can write whatever shell commands you want. For example: `//RUN: echo "Hello World!" | grep -e "orld"`
3. LIT provides some handy macros. For example, %s will be the current file path, and %t will be a temporary file lit generated for you, such that you can piped result through different RUN directives:
```
// RUN: echo "/etc/passwd" > %t
// RUN: cat %t | grep -e "root"
```

A full list of LIT macros can be found [here](https://llvm.org/docs/CommandGuide/lit.html#substitutions).

Of course, all of these design philosophies only work for tests that have a similar input and output formats as tools in LLVM. So I would recommend you to use lit in your project if:
1. Your test targets are executables that have textual outputs
2. The executables have command line interface.
3. (Bonus) You’re using CMake / Autoconf, or build systems that have boilerplate mechanisms (we will cover this later).

In the next two sections I’m going to show you how to use lit in your out-of-tree project. First, in the **Short Story** section, I’m showing you how to get started with lit in less than 1 minute, but coming with lots of problems. The whole purpose of that section is demonstrating lit as a standalone tool even outside the LLVM tree. The following **Long(er) Story** section will try to fix those shortcomings and present you a more realistic and (hopefully) more useful use case.

---
## The Short Story
First, let’s grab lit from PyPi repository:
```
pip install --user lit
```
Unfortunately the current version of lit does not contain an entry point in its main module that can be invoked with `python -m`, so we need to call the main function manually with a simple wrapper script, `my-lit.py` :
{% highlight python %}
#!/usr/bin/env python
from lit.main import main

if __name__ == '__main__':
    main()
{% endhighlight %}

Here is the motivated example we’re going to use: compile a simple hello world C program and validate its execution output:

{% highlight c %}
// RUN: cc %s -o %t
// RUN: %t | grep -e "hello world"
#include <stdio.h>

int main() {
  puts("hello world");
  return 0;
}
{% endhighlight %}

According to the `RUN` directives in line 1 and 2, the test runner will compile this source code into an executable (with a temporary name), run it, and validate its stdout output with `grep`.

Note that lit will fail if the commands finish with exit code 1. So if the `grep` command doesn’t find any match, which will result in exit code 1, it would be considered as a failure.

We also need some simple configurations. Lit’s config files are plain python script with some pre-populated variables.

{% highlight python%}
import lit.formats

config.name = "My Example"
config.test_format = lit.formats.ShTest(True)

config.suffixes = ['.c']
{% endhighlight %}