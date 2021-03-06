+++
date = "2014-04-21T19:51:00-04:00"
draft = false
title = "Syntax extensions and regular expressions for Rust"
author = "Andrew Gallant"
url = "rust-regex-syntax-extensions"
+++

**WARNING:** <em>2018-04-12</em>: The code snippets for this post are no longer
available. This is just as well anyway, since they all depended on an unstable
internal compiler interface, which hasn't existed for years.

A few weeks ago, I set out to add regular expressions to the
[Rust](http://www.rust-lang.org/)
distribution with an implementation and feature set heavily inspired by
[Russ Cox's RE2](http://swtch.com/~rsc/regexp/).
It was just recently added to the
[Rust distribution](http://static.rust-lang.org/doc/master/regex/index.html).

This regex crate includes a syntax extension that compiles a regular expression
to native Rust code *when a Rust program is compiled*. This can be thought of
as "ahead of time" compilation or
something similar to [compile time function
execution](http://en.wikipedia.org/wiki/Compile_time_function_execution).
These special natively compiled regexes have the *same exact* API as regular
expressions compiled at runtime.

In this article, I will outline my implementation strategy---including code
samples on how to write a Rust syntax extension---and describe how I was able
to achieve an identical API between regexes compiled at compile time and
regexes compiled at runtime.

<!--more-->


### Brief notes on format

All code samples in this post compile with `rustc` from commit `8bc286`
(compiled on `2014-06-30, 21:16:32+0`).
Code samples with an `// Output:` comment are tested before uploaded.

Most code samples *should* correspond to a complete program that is
compilable. This makes samples a bit longer than they need to be, but I think
it's important to include complete working examples for a language still in
development and not (yet) widely known.

Note that Rust is still under heavy development. I will do my best to keep this
article updated with any breaking changes.


### How a regex matcher works

There are many different ways to implement a regular expression engine, so I
will just briefly outline how mine works, which should closely correspond to
RE2.

First, a regular expression is parsed and converted into an abstract syntax
tree. For example, the regex `a|b` might be represented as
`Alternate(Literal(a), Literal(b))`, which signifies "match either `a` or `b`."
Second, the regex's abstract syntax is converted to a sequence of
instructions. This sequence of instructions can then be evaluated by a
[virtual machine](http://swtch.com/~rsc/regexp/regexp2.html)
which will determine if a regex matches some text (and where it matches).

For the regex `a|b`, the sequence of instructions looks something like:

* 1: Split(2, 4)
* 2: Char(a)
* 3: Jump(5)
* 4: Char(b)
* 5: Match

In this case, `Split` means "jump to two different instructions
simultaneously." In particular, if *either* branch executes the `Match`
instruction, then the entire regex will match. If both `Char` instructions
fail, then it's impossible to reach the `Match` instruction.


### Clarifying the divide: native regexes vs. dynamic regexes

The word "compiled" is heavily overloaded, so it's worth making sure that we
get our terms straight. For example, in Python, you can *compile* a regular
expression with `re.compile("...")`, which is done at runtime, and converts the
pattern given into a data structure that is fed into a matching algorithm.
Often, it is good practice to compile a regular expression outside of loops
so that it doesn't have to go through the overhead of compilation each time the
expression is used to search text. (Assuming you don't want to rely on Python's
caching.)

This is emphatically *not* what I mean by compiling regexes "ahead of time."
What I mean is the more literal translation: a regular expression is converted
to native Rust code when you compile your Rust program. That is, there is
(virtually) zero cost of compiling a regex during runtime. Perhaps more
importantly, because it's compiled to native Rust code, your regex will always
run faster. (I promise evidence of this claim with benchmarks toward the end of
this article.)

Let's clarify with an example. First, we can try compiling a regular expression
at runtime:

<code
  data-gist-id="11161035"
  data-gist-file="intro.rs"
  data-gist-hide-footer="true"
  data-gist-hide-line-numbers="true"
 ></code>

Notice that we call `Regex::new` to create a regex, and the call may fail if
the expression given isn't a valid regex. I call this a *dynamic* regex
because the regular expression compilation happens at runtime. This example is
analogous to Python's `re.compile`.

When a regular expression is compiled *dynamically*, it is converted to a
sequence of instructions that are
[executed by a virtual machine](http://swtch.com/~rsc/regexp/regexp2.html).
This virtual machine can execute **any** valid regular expression expressed as
a sequence of instructions.

In contrast, I've called regexes that are compiled to Rust code when your Rust
program compiles *native*. A native regex is indistinguishable from a dynamic
regex, except that it must be created with a `regex!` macro:

<code
  data-gist-id="11161035"
  data-gist-file="intro-aot.rs"
  data-gist-highlight-line="1,3,6"
  data-gist-hide-footer="true"
  data-gist-hide-line-numbers="true"
 ></code>

The highlighted lines indicate a change from the previous code sample.

(Note about Rust: Syntax extensions require a special compiler
directive `#[phase(plugin)]` to work. The `phase` directive must be enabled
explicitly: `#![feature(phase)]`.)

Notice that we no longer have to check if `regex!` returns an error because
all errors are converted to *compile time* errors. For example, the following
program will fail to compile:

<code
  data-gist-id="11161035"
  data-gist-file="unbalanced-paren.rs"
  data-gist-hide-footer="true"
  data-gist-hide-line-numbers="true"
 ></code>

The error given by `rustc`: `Regex syntax error near position 3: Unclosed
parenthesis.`

So what exactly does `regex!` expand to if not a sequence of instructions?
Well, remember the virtual machine I mentioned earlier that can execute a
sequence of instructions? That's precisely what `regex!` gets replaced with,
except it's *specialized* to the particular sequence of instructions
corresponding to the regular expression given. This specialization allows us to
perform optimizations such as removing heap allocation and embedding the
instructions directly instead of relying on a generalized algorithm.

If you want an example, you can take a look at how
[this
program](https://gist.githubusercontent.com/BurntSushi/11161035/raw/unexpanded.rs)
expands the `regex!` macro into
[this
program](https://gist.githubusercontent.com/BurntSushi/11161035/raw/expanded.rs).
(It's not important to understand the expanded code, but it is notable that it
expands to a lot of code! If you have a lot of `regex!` calls in your program,
then you might see your binary size increase. But compiling with `-O`
optimization will help with that.)

Native regexes are pretty good. They provide extra safety (cannot compile a
program with an invalid regex) and extra performance. They do come with a
downside though: code bloat. If your program has a lot of `regex!` calls
(hundreds) and is compiled unoptimized, then you'll notice a bigger binary.
However, optimization can help shrink them back down.


### Related work

Note that I am not the first to do this.
[D has ctRegex](http://dlang.org/phobos/std_regex.html#.ctRegex) which claims
to do something similar.
There is also
[xpressive](http://www.boost.org/doc/libs/1_41_0/doc/html/xpressive/user_s_guide.html)
in Boost, but native regexes must be written with template syntax.
Finally, there is also
[CL-PPCRE](http://weitz.de/cl-ppcre/) for Common Lisp which also claims to have
native regexes.

[Nimrod](http://nimrod-lang.org/) seems like it is capable of producing native
regexes, but I don't think it has been done yet.

A related project is
[Ragel](http://www.complang.org/ragel/),
which can compile state machines to native code in a variety of languages.
In principle, Ragel is also producing native regexes, but my approach is
automated by the Rust compiler and provides an API identical to that of
dynamic regexes.

(Please alert me if I've missed anything.)


### Native regexes are implemented with macros

Rust has
[hygienic macros](http://en.wikipedia.org/wiki/Hygienic_macro),
but they can be written in one of two ways.
The first way is to use `macro_rules!` to conveniently specify some source
transformation with quasiquoting.
For example, here is how the `try!` macro is defined, which provides an easy
way to return early in a function that returns a value with type `Result<T,
E>`:

<code
  data-gist-id="11161035"
  data-gist-file="try.rs"
  data-gist-hide-footer="true"
  data-gist-hide-line-numbers="true"
 ></code>

Notice the use of quasiquoting with `$e`. The expression `$e` is spliced into
the `match` expression *at compile time*. The quasiquoting in this case
distinguishes between something that parameterizes the macro at compile time
and the actual source code being written.

In general, if a macro can be written with `macro_rules!`, then it *should* be
written with `macro_rules!` because they are simple to write. However, they
are not powerful enough to implement native regexes because they are restricted
to source code transformations. To implement native regexes, we need to
actually compile a regular expression during the compilation of a Rust program.

Luckily, the second way to write a macro is via a syntax extension (also known
as a "procedural macro"). This is done by a compile time hook that lets you
execute arbitrary Rust code, rewrite the abstract syntax to whatever you want
and opens up access to the Rust compiler's parser. In short, you get a lot of
power. And it's enough to implement native regexes. The obvious drawback is
that they are more difficult to implement.

Note that *all* macros are invoked with the `name!` syntax, regardless of
whether they are defined with `macro_rules!` or as a syntax extension.


### Setting up a syntax extension

Syntax extensions only became available to user programs a few months ago.
Before that, they were only available to the Rust compiler.
Thus, there is little documentation, so I've learned what I know by example and
by examining the
[syntax](http://static.rust-lang.org/doc/master/syntax/index.html)
crate, which defines Rust's abstract syntax, provides a parser and the syntax
extension functionality itself. (Note that syntax extensions are currently an
experimental feature of Rust.)

Since there is a fair bit of boiler plate required, let's take a look at
implementing a trivial macro that returns the factorial of `5`:

<code
  data-gist-id="11161035"
  data-gist-file="lib-factorial_no_quote.rs"
  data-gist-hide-footer="true"
  data-gist-hide-line-numbers="true"
 ></code>

(N.B. There are some comments in the code above that explain some details.)

The magic happens with the `macro_registrar` function, which must be exported
and labelled with the special `#[macro_registrar]` attribute. This function is
where the compiler lets you register syntax extensions, which have a name and
provide a value of type `SyntaxExtension` which indicates the sort of expansion
that you intend to do. In this case, we're using `NormalTT`, which is a
function-like macro, but there
[are
others](http://static.rust-lang.org/doc/master/syntax/ext/base/enum.SyntaxExtension.html),
including the `macro_rules!` form.

Here, we register a single macro: `factorial`, which is *expanded* by the
`expand` function. The type of `expand` is specified by the `NormalTT`
expansion, which gives us an
[extension
context](http://static.rust-lang.org/doc/master/syntax/ext/base/struct.ExtCtxt.html)
(e.g., the parsing state),
a
[span](http://static.rust-lang.org/doc/master/syntax/codemap/struct.Span.html)
(e.g., a region of the actual code, used for error reporting)
and a
[token
tree](http://static.rust-lang.org/doc/master/syntax/ast/enum.TokenTree.html).
But most importantly, the *return value* is a
[`Box<MacResult>`](http://static.rust-lang.org/doc/master/syntax/ext/base/trait.MacResult.html)
("macro result"), where `MacResult` is a trait.
Types that satisfy the `MacResult` trait can be automatically spliced into the
AST of a Rust program.
In this case, we're just creating an expression, so we can use the
[MacExpr](http://static.rust-lang.org/doc/master/syntax/ext/base/struct.MacExpr.html)
helper type to construct a value that has type `Box<MacResult>`. Finally, we
see that `MacExpr::new` has type `Gc<Expr> -> Box<MacResult>`. Ah ha! So if we
can build an expression, then we can build a `Box<MacResult>`, and therefore be
done with our macro expansion.

The body of the `expand` function shows how to manually create an expression
with a single integer literal. Even though the expression is very simple, I
wrote two helper functions `uint_literal` and `dummy_expr` to encapsulate some
of the more mundane details. If we have to go through this much work for just a
single literal, you might imagine that it gets very tedious pretty quickly.

But remember the quasiquoting that was available to use when defining macros
with `macro_rules!`? They are available to us here too, courtesy of the
[`syntax::ext::quote`](http://static.rust-lang.org/doc/master/syntax/ext/quote/index.html)
module. Here's the same syntax extension implemented using quasiquoting:

<code
  data-gist-id="11161035"
  data-gist-file="lib-factorial.rs"
  data-gist-hide-footer="true"
  data-gist-hide-line-numbers="true"
 ></code>

The `quote_expr!` macro does all the heavy lifting for us. It splices the
value of `answer` into an expression for us, handling all the nitty-gritty
details automatically. (We could also write, say, `$answer + 1` which would
produce an expression with an addition operation. The `$answer` part is
computed at compile time and the addition operation would be done at
runtime---if it isn't optimized away.) Fundamentally, the
quasiquoting works the same here as it does with `macro_rules!`.

(There is some
[middle
ground](http://static.rust-lang.org/doc/master/syntax/ext/build/trait.AstBuilder.html)
between building the AST manually and using
quasiquoting, but I won't cover it here.)

And finally, we can actually *use* it with:

<code
  data-gist-id="11161035"
  data-gist-file="factorial.rs"
  data-gist-hide-footer="true"
  data-gist-hide-line-numbers="true"
 ></code>

The comments in the code explain some of the extra compiler directives we have
to write in order to import a syntax extension. (Note that this is no different
than importing the `log` crate from the Rust distribution, except that the
`log` crate requires `#[phase(syntax, link)]` because it provides more stuff
than just a syntax extension.)

### Accessing the parser

The above example demonstrates how to write a really simple---but pretty
useless---syntax extension. To write a useful syntax extension, we probably
need to define a macro that can accept arguments. With `macro_rules!`, this is
really easy because there's some convenient syntax similar to defining a
regular function. But in the land of syntax extensions, we must deal with
Rust's parser directly.

An obvious extension to our `factorial!` macro is to let it take an integer
argument so that we can compute any factorial. But how do we get the arguments
given to a macro defined as a syntax extension?

Well, if you look back at the last example, then you'll notice that the
`expand` function has a few parameters that we didn't use. One of which is a
sequence of token trees with type `&[ast::TokenTree]`. Luckily, a sequence of
`ast::TokenTree` values can be used to build a parser with
`new_parser_from_tts` defined in the
[syntax::parse](http://static.rust-lang.org/doc/master/syntax/parse/index.html)
module. With a parser, we can ask it for expressions---and once we have
expressions, we can translate those to real Rust values. In this case, that's
just going to be an integer.
(You can see the
[parser API
here](http://static.rust-lang.org/doc/master/syntax/parse/parser/struct.Parser.html),
and although it is largely undocumented, most of the names are reasonably
descriptive.)

Let's take a look at the code. The principal changes from the last example are
highlighted:

<code
  data-gist-id="11161035"
  data-gist-file="lib-factorial_arg.rs"
  data-gist-hide-footer="true"
  data-gist-hide-line-numbers="true"
  data-gist-highlight-line="999999999,20-25,35-65"
 ></code>

The first few changes are fairly trivial. This time, instead of hard-coding the
factorial of `5`, we parse an unsigned integer literal and pass that to the
`factorial` function. If a single unsigned integer literal could not be parsed,
then we return a
"[dummy
expression](http://static.rust-lang.org/doc/master/syntax/ext/base/struct.DummyResult.html#method.expr)".
The actual error is written to the parser state in `parse`. The code is
structured this way to allow the compiler to continue even if an error is
found.

The `parse` function shows how to create a parser from a sequence of token
trees. A single expression is parsed, and we use pattern matching to make sure
it corresponds to an unsigned integer literal.
(To figure out which value constructors to use in your patterns, you'll want to
dig into the
[ast::Expr](http://static.rust-lang.org/doc/master/syntax/ast/struct.Expr.html)
documentation, specifically the `Expr_` type.)

After we find a literal, we make sure it's the last thing in the sequence of
token trees and then return the integer as a real Rust value with type `u64`.

If anything unexpected occurs, we log an error with the Rust compiler via the
`ExtCtxt` and return `None`. As a bonus, we use the `syntax::print::pprust`
module to pretty-print Rust expressions to make our error messages nicer.

Finally, the macro can be used the same way as last time, except we call
`factorial!` with an integer argument:

<code
  data-gist-id="11161035"
  data-gist-file="factorial_arg.rs"
  data-gist-hide-footer="true"
  data-gist-hide-line-numbers="true"
  data-gist-highlight-line="5"
 ></code>

This concludes the tutorial aspect of this article. Now we'll get back to
regexes.

### Creating the `regex!` syntax extension

Instead of going through the
[full code
generator](https://github.com/mozilla/rust/blob/master/src/libregex_macros/lib.rs),
which would require more explanation about how the virtual machine works,
I'll explain the representation of native regexes and how some of the
optimizations work.

But first, we must figure out how the `regex!` macro will be used. If possible,
it should reuse the existing
[API for the Regex
type](http://static.rust-lang.org/doc/master/regex/struct.Regex.html).
The benefit of this is that native regexes won't introduce any additional
complexity on top of dynamic regexes other than how they are constructed. The
`regex!` macro should also probably return an expression, so that we can write
things like `regex!("abc").is_match("xyz")`.

Perhaps we can construct a value with type `Regex`. If we could do that, then
we'd be done---because such values already support all the convenient matching,
splitting and replacing functions. So let's take a look at the representation
of a dynamic `Regex`:

<code
  data-gist-id="11161035"
  data-gist-file="rep-regex-dynamic.rs"
  data-gist-hide-footer="true"
  data-gist-hide-line-numbers="true"
  data-gist-line="2-1000"
 ></code>

The representation shown here is very simple: it's just a sequence of
instructions. The problem is that this representation is for *dynamic*
regexes. Remember that dynamic regexes work by being compiled to a sequence of
instructions that is then executed by a general virtual machine. The VM can be
thought of as a Rust function with type
`fn(insts: &[Inst], search: &str) -> Captures`, where `Captures` is just the
location of matches in the `search` text.

Native regexes on the other hand should be native Rust code with *precisely the
same type* as above with one omission: the sequence of instructions. Why?
Because the sequence of instructions is encoded directly into the function by
the `regex!` syntax extension. Therefore, the type of a native regex VM would
have to be `fn(search: &str) -> Captures`.

So we're left with an interesting set of constraints. On the one hand, we have
a VM that can execute a sequence of instructions on search text, and on the
other, we have a specialized VM that just executes on search text. Since a
regex must be *either* dynamic or native, we can represent this state of
affairs with a sum type:

<code
  data-gist-id="11161035"
  data-gist-file="rep-regex-both.rs"
  data-gist-hide-footer="true"
  data-gist-hide-line-numbers="true"
  data-gist-line="2-1000"
 ></code>

And now we can easily write a function that executes a `Regex` on search text,
regardless of whether its dynamic or native:

<code
  data-gist-id="11161035"
  data-gist-file="rep-regex-exec.rs"
  data-gist-hide-footer="true"
  data-gist-hide-line-numbers="true"
  data-gist-line="12-1000"
 ></code>

These details of the representation are hidden from well behaving clients, and
therefore, dynamic and native regexes are indistinguishable.


### Why did you just say "well behaving clients"?

Shouldn't Rust's module system prevent clients from accessing parts of the
representation that aren't exported?

Herein lay the dirty little secret of the `regex_macros` crate: it *must*
be able to access the internal representation of a `Regex` (along with other
things, like the set of all instructions). But if you look at the public API
documentation, you'll not see any such details of the representation exposed.
This is because they are hidden using a special `#[doc(hidden)]` attribute,
but you can see that an
[entire module of
things](https://github.com/mozilla/rust/blob/master/src/libregex/lib.rs#L395)
is exported for the exclusive benefit of the `regex_macros` crate.

Therefore, the "well behaving clients" qualification is necessary. Misbehaving
clients could
[access the internal representation of the Regex
type](https://github.com/mozilla/rust/blob/master/src/libregex/re.rs#L103), and
thereby distinguish between dynamic and native regexes.

(It'd be possible to ameliorate this somewhat by exposing a hidden constructor
function.)


### Performance optimizations

This is still early days for the regex crate in Rust, but there are two primary
optimizations made with native regexes. The first is removing heap allocation.
Briefly, the general VM for dynamic regexes has a few places where it needs to
allocate things on the heap. Primarily, the allocation is for storing capture
groups (i.e., the location of submatches in search text). The heap allocation
is necessary because the dynamic VM doesn't know at compile time how many
capture groups any particular regex might have.

This of course is not the case for native regexes, since the regex is compiled
when your Rust program is compiled. Therefore, we can put all of the capture
group information on the stack using explicitly sized vectors (i.e., `[T,
..N]`). This removes all heap allocation (except for the return value).

The second optimization comes from encoding the instructions directly into one
big `match` expression based on the *position* of the instruction. This removes
a lot of small computations necessary to discover what the "next" instruction
is.

Some other smaller optimizations were made, such as removing conditional
branches based on flags set in a particular instruction (like case
insensitivity, multi-line mode, etc.) or converting a binary search on a
character class to a single `match` expression.


### Benchmarks

As promised, here are some benchmarks run on an Intel i3930K:

<code
  data-gist-id="11161035"
  data-gist-file="bench-cmp"
  data-gist-hide-footer="true"
  data-gist-hide-line-numbers="true"
 ></code>

Native regexes provide a universal speedup.

The more interesting observation is that native regexes drastically reduce
constant factors associated with matching a particular regex against some text.
This is evidenced by the `easy_32`, `easy1_32`, `medium_32` and `hard_32`
benchmarks, where there is an order of magnitude improvement in almost all of
them.

These benchmarks in particular test a few different regexes (based on
particular optimizations related to literal prefixes) on 32 bytes of text.
As one might imagine, with such a small search string, constant factors are
likely to dominate the performance of the matching algorithm.
Not surprisingly, native regexes win big here---likely because there is almost
no heap allocation occurring.


### Future work

There are still many more optimizations that can be performed:

* Creating a "one pass" NFA where no backtracking need occur.
* Attempt to do state compression in the code generator via a more intelligent
  analysis on the sequence of instructions.
* The big one: implement a DFA, similar to how RE2/C++ works.


### Acknowledgements

Many thanks to the following:

* [Steven Fackler](https://github.com/sfackler) for helping
  me navigate the word of syntax extensions.
* [Eduard Burtescu](https://github.com/eddyb) for helping me with the code
  generator and talking through some future ideas for optimization.
* [Chris Morgan](https://github.com/chris-morgan) for [giving me the initial
  idea](https://github.com/rust-lang/rfcs/pull/42#issuecomment-40301587) to
  create native regexes.
* [Alex Crichton](https://github.com/alexcrichton) for holding my hand through
  the pull request process.
* [Russ Cox](http://swtch.com/~rsc/) for blazing the trail and writing
  an **amazing** [series of articles](http://swtch.com/~rsc/regexp) on
  implementing regular expression matching.


### Links

* [regex API
  documentation](http://static.rust-lang.org/doc/master/regex/index.html).
* A [gist of all code samples](https://gist.github.com/BurntSushi/11161035) in
  this article.
* Some rough numbers for the
  [regex-dna](https://github.com/BurntSushi/regexp/tree/master/benchmark/regex-dna)
  benchmark, compared with Go, Python and C.
* The
  [infrastructure used to compile, check and upload](https://github.com/BurntSushi/burntsushi-blog/tree/master/posts/gists/rust-regex-syntax-extensions)
  code samples in this article automatically.

### Other examples of syntax extensions

When documentation is scarce, the next best thing is to look at examples. Here
are the ones I know about:

* [simple-ext](https://github.com/sfackler/syntax-ext-talk/blob/gh-pages/simple-ext/lib.rs)
  is an example of a syntax extension for sorting literal strings. You
  can see it [used
  here](https://github.com/sfackler/syntax-ext-talk/blob/gh-pages/simple-ext/test.rs).
* [hexfloat](https://github.com/mozilla/rust/blob/master/src/libhexfloat/lib.rs)
  for hexadecimal floating-point literals (part of Rust distribution).
* [fourcc](https://github.com/mozilla/rust/blob/master/src/libfourcc/lib.rs)
  four-character code library (part of Rust distribution).
* [rust-phf](https://github.com/sfackler/rust-phf) implements compile time
  static maps.

Please ping me if you know of more examples. I'll add them here.

<script type="text/javascript" src="https://ajax.googleapis.com/ajax/libs/jquery/1.9.1/jquery.min.js"></script>
<script type="text/javascript" src="//cdnjs.cloudflare.com/ajax/libs/gist-embed/1.7/gist-embed.min.js"></script>

