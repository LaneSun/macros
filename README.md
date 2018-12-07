A more powerful C/C++ macro preprocessor
========================================

The venerable C preprocessor (cpp) – the part of the compilation process that interprets hash-prefixed directives such as `#include` and `#define`, and substitutes for the macros defined by the latter – is undoubtedly one of the backbones of the C/C++ ecosystem. However, its macro functionality suffers from [known limitations](https://gcc.gnu.org/onlinedocs/cpp/Self-Referential-Macros.html): macros may not call themselves recursively, meaningfully work with mutable state or introduce syntax that does not obey the shape of either single keywords or function calls. This severely limits its utility for metaprogramming, necessitating the proliferation of idiosyncratic boilerplate-generation tools of high complexity but limited scope such as [Yacc](https://en.wikipedia.org/wiki/Yacc) or Qt's [moc](http://doc.qt.io/archives/qt-4.8/moc.html).

This project is an attempt to create a Turing-complete general-purpose preprocessor for C and C++ that is powerful enough to subsume all of the above and more: indeed, we shall aim to be able to build on top of either C or C++ in the way the latter was originally built upon the former, while seamlessly blending in with existing code as the C preprocessor does. 

We draw significant inspiration from [Rust's macro system](https://doc.rust-lang.org/nightly/book/second-edition/appendix-04-macros.html), which appears to be the most ambitious such effort this side of LISP, without binding ourselves to its sometimes curious choice of syntax or its demand of [hygiene](https://en.wikipedia.org/wiki/Hygienic_macro) (this is C, after all!). Since all respectable programming language projects are self-hosting and this is not one of them respectable programming language projects, the preprocessor itself is written in [Haskell](https://en.wikipedia.org/wiki/Haskell). The usual warnings about alpha-quality software apply, and all syntax and semantics is subject to change.

[Try it out online here!](http://exosphere.kafuka.org/macros-test/)

Example
-------

```c++
struct {
    int value;
    LinkedList *next;
} LinkedList;

// define recursive macro to create a linked list
@define MakeList {
    ( {@^[,]$head, @^$tail} ) => (
        new LinkedList( {$head, MakeList {$tail}} )
    )
    ( {@^[,]$singleton} ) => (
        new LinkedList( {$singleton, NULL} )
    )
    ( {} ) => ( NULL )
}

// create a linked list with 5 elements
LinkedList *l = MakeList {1,2,3,4,5};
```

Usage
-----

Make sure you have a recent version of `ghc`, `mtl`>=2.2.1 and `parsec` installed. The appropriate packages exist in Debian *testing* repositories as `ghc`, `libghc-mtl-dev` and `libghc-parsec3-dev` respectively; building under Windows or other unixoid OS families is currently untested. Run `make` in the root directory of the repository. This generates an executable file named `macros`. See `INSTALL.windows.md` for Windows installation instructions.
Running
```
./macros <input file>
```
will emit the processed output to `stdout`. If any errors are encountered, they will be printed to `stderr`.
For convenience, a wrapper script `with-macro` is included.
Running
```
./with-macro g++ <input file> -O2 -Dotheroptions
```
(with the input filename being in the second position, i.e. immediately following the compiler executable!) is equivalent to running
`g++ -O2 -Dotheroptions` on the output of the preprocessor on `<input file>`.

Introduction
------------

The fundamental principle of `macros` is **keyword-triggered pattern-matching and substitution on token streams**. A typical macro definition takes the following form:
```c++
@define macroname {
    ( pattern one ) => ( printf("first pattern encountered") )
    ( pattern two ) => ( printf("second pattern encountered") )
    ( pattern three) => ( printf("third pattern encountered") )
    // ...
}
```
This defines a macro that is triggered by encountering the keyword `macroname` anywhere in the program text following the definition. If the keyword is encountered, the macro processor will proceed to try and match the token stream following it against the patterns – the token streams inside the left-hand side parentheses of a pair of the form `(stream) => (stream)` in the definition body in turn. For the first left-hand pattern that fully matches the token stream following the keyword, the corresponding right-hand stream will be substituted in for the keyword *and* the matching tokens. If no patterns match, an error is thrown.
 
So for instance,
```c++
macroname pattern two;
macroname pattern one two three four;
/* will become:
 * printf("second pattern encountered");
 * printf("first pattern encountered") two three four; */
```
The pattern of a macro can also *capture* tokens in compile-time variables, which are recognisable by the sigil `$`. The results of such a capture can be reused on the right-hand side. In the simplest case, we capture a single token:
```
@define greet {
	( $name ) => ( printf("Hello %s!", $name) )
}

greet "world";
```
Multiple tokens can be captured in one variable by using the modifier `@^`, which is followed by any number of *stop patterns* `[token-stream]`. If any stop pattern is encountered or the token stream ends, capture terminates.
```
@define bracketless_printf {
    ( $pattern, @^[;]$args ) => (
        printf($pattern, $args)
    )
}
bracketless_printf "%d %d",1,2;
```

List of directives
------------------

**Macros and pattern matching**

* `@define keyword { ( pattern ) => ( outcome ) ... }`: Defines a new macro `keyword`. The token stream after `keyword` will be matched in turn against each `pattern`. If a pattern is matched against successfully, a new stack frame is instantiated with all variables captured by the pattern, the corresponding `outcome` is processed in the new stack frame and emplaced instead of `keyword` plus the matching token stream.
* `@undef keyword`: Undefines the macro `keyword`.
* `@match ( token-stream ) { ( pattern ) => ( outcome ) ... }`: Processes `token-stream`, and attempts to match the result against each `pattern` in turn, analogously to an `@define`d macro.

**Variable capture**

The following are only valid in `pattern`s.

* `$var`: Matches any single token. Stores the result in `$var`.
* `@#$var`: Matches any *name*, i.e. C/C++ *identifier*.
* `@^[stop-token-stream-1]...[stop-token-stream-n]$var`: Matches an arbitrarily long token stream until any one of the `stop-token-stream`s is encountered, or the token stream ends (via end-of-file or a closing parenthesis). The form `@^$var` is permissible. Each `stop-token-stream` must be *shallow*: behaviour is unspecified if it contains any brackets with nonempty contents. Since all token streams must be bracket-matched, use `@^[{}]$var` to stop when a block of curly braces is encountered.
  * Example:
    ```c++
    @define test { ( @^[a][b][{}]$v ) => () }
    test c d a b { e }; // -> a b { e };
    test c d b a { e }; // -> b a { e };
    test { e }; // -> { e };
    ```
* `@*[separator-token-stream]( pattern )`: Matches zero or more of `pattern`, optionally separated by `separator-token-stream`. (The parameter, along with its `[]`, may be omitted.) Any variables captured in `pattern` will be put into a list.
   * So for instance, `@*[,]( @^[,]$param )` will capture a comma-separated list of token streams, and put them in the list `$param`. Given the input
     `hello world, goodbye world, 1+2`, this will generate a `value-spec` analogous to `@[ (hello world), (goodbye world), (1+2) ]`.
* `@+[separator-token-stream]( pattern )`: As above, but matches one or more.

**Variables**

* `$varname`: Evaluates to the currently visible instance of `$varname`. The visible instance is either the topmost (most recent) definition of `$varname` on the stack,
* `@set $varname value-spec`: Processes the `value-spec`. Then sets the value of the currently visible instance of `$varname` to the result.
  * A `value-spec` is either a singleton `(token-stream)`, or a list `@[ value-spec, ..., value-spec ]`.
* `@var $varname value-spec`: Defines a variable `$varname` in the local stack frame, processes the `value-spec` and sets the value of `$varname` to the result.
* `@global $varname value-spec`: Defines a global variable `$varname`, processes the `value-spec` and sets the value of `$varname` to the result.
* `@push_back $varname value-spec`: Processes the `value-spec`. Then appends the result to the currently visible instance `$varname`. If this instance is not a list, an error is thrown.
* `@for[ separator-token-steam ]( bind-spec )( body-token-stream )`, where `[ separator-token-stream ]` is optional: iterates over lists according to `bind-spec`. For each entry, instantiates a new stack frame with the bound variables, processes `body-token-stream` in the frame and emits the result. If `separator-token-stream` is nonempty, it is processed and emitted between each two `body-token-stream`s.
  * `bind-spec` takes the form `$v1,...,$vn : $l1,...,$ln`, where the `$l1,...,$ln` are lists of identical length.
  * Example usage:
    ```c++
        @var $listone @[ (1), (2), (3) ]
        @var $listtwo @[ (a), (b), (c) ]
        @for[,]( $a,$b : $listone,$listtwo )(
            $a @@ $b
        )
        // outputs: 1a, 2b, 3c
    ```

**Token stream operations**

* `@@`: Concatenates the preceding and following token: `a @@ b` evaluates to `ab`. Compare `##` in `cpp`.
* `@!(token-stream)`/`@![token-stream]`/`@!{token-stream}`/`@!single-token`: Emits `token-stream` or `single-token` without processing.
* `@eval (token-stream)`/`@eval [token-stream]`/`@eval {token-stream}`/`@eval single-token`: Processes `token-stream` or `single-token` twice and emits the result, so e.g. a variable containing an unprocessed token stream is evaluated.
* `@unquote "any string"`: Emits `any string`.
* `@quote (token-stream)`: Emits `"token-stream"` as a string literal.
* `@calc (token-stream)`: Processes `token-stream`, then attempts to evaluate it as an integer arithmetic expression. Substitutes in the result. Supported operators are `+`, `-`, `*` and `+`. Throws an error if the parameter is not an integer arithmetic expression.

**File management**

* `@include "filename"`: Includes and processes the file `filename`, relative to the current file's directory.

