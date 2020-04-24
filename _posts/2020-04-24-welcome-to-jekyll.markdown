---
layout: post
title:  "CombinedParsers.jl in pure Julia"
date:   2020-04-24 06:32:36 +0200
categories: julia parsers regex
---
# CombinedParsers in pure Julia

CombinedParsers is a package for parsing into julia types.
Parsimoneously compose parsers with the functional [parser combinator paradigm](https://en.wikipedia.org/wiki/Parser_combinator),
utilize Julia's type inferrence for transformations,
log conveniently for debugging, and let Julia compile your parser for good performance.

Features:
- Clear syntax integrates grammar and transformations with Julia type inference.
- Higher-order parsers depending on the parsing state allow for not context-free parsers.
- All valid parsings can be iterated lazily.
- Interoperable with [TextParse.jl](https://github.com/queryverse/TextParse.jl): existing `TextParse.AbstractToken` implementations can be used with CombinedParsers. `CombinedParsers.AbstractParser` provide `TextParse.tryparsenext` and can be used e.g. in CSV.jl.
- Parametric parser and state types enable Julia compiler optimizations.
- Compiled regular expression parsers in pure julia are provided with the `re_str` macro.
- [AbstractTrees.jl](https://github.com/JuliaCollections/AbstractTrees.jl) interface provides colored and clearly layed out printing in the REPL.
- Convenient logging of the parsing process with `NamedParser`s and `SideeffectParser`s.
- CombinedParsers generalize from strings to parsing any type supporting `getindex`, `nextind`, `prevind` methods.


```julia
"Julia eval for +-*/ operators."
function eval_ops((l,opr))
    for (op,val) in opr
        l = eval( Expr(:call, Symbol(op), l, val))
    end
    l::Rational{Int}
end

using TextParse
@with_names begin
    number = map(Rational{Int}, TextParse.Numeric(Int))
    factor = Either(number)  # or expression in parens, see push! below
    divMul = map(eval_ops,
                 Sequence( factor, Repeat( CharIn("*/"), factor ) ) )
    addSub = map(eval_ops,
		 divMul * Repeat( CharIn("+-") * divMul ) )
    parens = Sequence(2, "(",addSub,")" )
    push!(factor, parens)
    expr = (addSub * AtEnd())[1]
end;
parse(log_names(expr), "1/((1+2)*4+3*(5*2))")

```
1//42

[Is every rational answer ultimately the inverse of a universal question in life?](https://en.wikipedia.org/wiki/Phrases_from_The_Hitchhiker%27s_Guide_to_the_Galaxy#Answer_to_the_Ultimate_Question_of_Life,_the_Universe,_and_Everything_(42))



### Limitations
This is an alpha release.
Collaborations are very welcome!

CombinedParsers.jl was tested against the PCRE C library testset.
Some tests did not pass: 3130 passed, 123 failed, 0 errored, 0 broken.
Failing tests are capture related.


#### Benchmarks
Performance is acceptable although optimizations are currently disabled/incomplete.
```julia
using BenchmarkTools

## PCRE
re = r"a*"
@btime match(re,"a"^3);
#      146.067 ns (5 allocations: 272 bytes)


## CombinedParsers
pc = re"a*"
@btime match(pc,"a"^3);
#      1.049 μs (10 allocations: 480 bytes)
@btime _iterate(pc,"a"^3);
#      283.928 ns (5 allocations: 272 bytes)
```

Preliminary performance tests were even more encouraging, in trivial example above 10x faster than PCRE.


### Next Steps
- [ ] Performance optimizations
    - generated functions
    - parsing memoization
    - backtracking optimization with multiple dispatch on parser and state type.
- [ ] Documentation
- [ ] Publishing packages for parsing wikitext and orgmode markup
- [ ] error backtracking, stepping & debugging
- [ ] test coverage


<!-- Implementation Notes: -->
<!-- - parametric immutable matcher types for compiler optimizations with generated functions (currently inactive) -->
<!-- - small Union{Nothing,T} instead of Nullable{T} -->

<!-- CombinedParsers is a package by Gregor Kappler. If you use and appreciate CombinedParsers.jl, please support development at patreon. -->


Check out the [github repo][CombinedParsers-gh] for more info and start writing wour own parser with CombinedParsers.jl.

[CombinedParsers-gh]: https://github.com/gkappler/CombinedParsers.jl
