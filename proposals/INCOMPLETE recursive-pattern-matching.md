# Pattern-matching changes in C# 8

> Open issues for the design and implementation of this feature can be found at https://github.com/dotnet/csharplang/issues/1054.


## Patterns

We augment the pattern forms [introduced in C# 7](https://github.com/dotnet/csharplang/blob/master/proposals/csharp-7.0/pattern-matching.md), resulting in the following grammar.

``` antlr
pattern
    : var_pattern
    | discard_pattern
    | constant_pattern
    | declaration_pattern
    | recursive_pattern
    ;

var_pattern
    : 'var' designation
    ;

designation
    : simple_designation
    | '(' designation* ')'
    ;

discard_pattern
    : '_'
    ;

constant_pattern
    : expression
    ;

declaration_pattern
    : type simple_designation
    ;

recursive_pattern
    : type? property_subpattern simple_designation?
    | type? positional_subpattern property_subpattern? simple_designation?
    ;

property_subpattern
    : '{' subpatterns? '}'
    ;
```

Each *subpattern* of a *property subpattern* is required to have an *argument_name*.

``` antlr
positional_subpattern
    : '(' subpatterns? ')'
    ;

subpatterns
    : subpattern
    | subpattern ',' subpatterns
    ;

subpattern
    : argument_name? pattern
    ;

argument_name
    : identifier ':'
    ;

simple_designation
    : single_variable_designation
    | discard_designation
    ;

single_variable_designation
    : identifier
    ;

discard_designation
    : '_'
    ;
```


### Var pattern

``` antlr
var_pattern
    : 'var' designation
    ;
```

An expression *e* matches the pattern `var identifier` always. In other words, a match to a *var pattern* always succeeds. At runtime the value of *e* is bounds to a newly introduced local variable. The type of the local variable is the static type of *e*.

It is an error if the name `var` binds to a type.

TODO: needs description, particularly for the parenthesized form. Note that `var` is a contextual keyword. We probably need to define the "context" for it.


### Discard Pattern

``` antlr
discard_pattern
    : '_'
    ;
```

An expression *e* matches the *discard pattern* `_` always.  In other words, every expression matches the *discard_pattern*.


### Recursive Pattern

``` antlr
recursive_pattern
    : type? property_subpattern simple_designation?
    | type? positional_subpattern property_subpattern? simple_designation?
    ;
```

Given a match of an expression *e* to the pattern *type* `{` *property_pattern_list* `}`, it is a compile-time error if the expression *e* is not *pattern compatible* with the type *T* designated by *type*. If the type is absent, we take it to be the static type of *e* if it is not a nullable value type, otherwise the underlying type of that type. If the *identifier* is present, it declares a pattern variable of type *type*. Each of the identifiers appearing on the left-hand-side of its *property_pattern_list* must designate a readable property or field of *T*. If the *identifier* of the *property_pattern* is present, it defines a pattern variable of type *T*.

At runtime, the expression is tested against *T*. If this fails then the property pattern match fails and the result is `false`. If it succeeds, then each *property_subpattern* field or property is read and its value matched against its corresponding pattern. The result of the whole match is `false` only if the result of any of these is `false`. The order in which subpatterns are matched is not specified, and a failed match may not match all subpatterns at runtime. If the match succeeds and the *identifier* of the *property_pattern* is present, it is assigned the matched value.

> Note: The property pattern can be used to pattern-match with anonymous types. 


### Switch Expression

A *match_expression* is added to support `switch`-like semantics for an expression context.

The C# language syntax is augmented with the following syntactic productions:

```antlr
relational_expression
    : match_expression
    ;

match_expression
    : relational_expression 'switch' match_block
    ;

match_block
    : '(' match_sections ','? ')'
    ;

match_sections
	: match_section
	| match_sections ',' match_section
	;

match_section
    : 'case' pattern case_guard? ':' expression
    ;

case_guard
    : 'when' expression
    ;
```

The *match_expression* is not allowed as an *expression_statement*.

The type of the *match_expression* is the *best common type* of the expressions appearing to the right of the `:` tokens of the *match section*s.

It is an error if the compiler can prove (using a set of techniques that has not yet been specified) that some *match_section*'s pattern cannot affect the result because some previous pattern will always match.

At runtime, the result of the *match_expression* is the value of the *expression* of the first *match_section* for which the expression on the left-hand-side of the *match_expression* matches the *match_section*'s pattern, and for which the *case_guard* of the *match_section*, if present, evaluates to `true`.


------------------------------

## Examples


### A match expression in Linq (contributed by @orthoxerox):

```cs
var areas =
    from primitive in primitives
    let area = primitive switch {
        Line l => 0,
        Rectangle r => r.Width * r.Height,
        Circle c => Math.PI * c.Radius * c.Radius,
        _ => throw new ApplicationException()
        }
    select new { Primitive = primitive, Area = area };
```
