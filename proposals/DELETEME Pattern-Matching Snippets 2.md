#### Positional Pattern

A positional pattern enables the program to invoke an appropriate `operator is`, and (if the operator has a `void` return type, or returns `true`) perform further pattern matching on the values that are returned from it. It also supports a tuple-like pattern syntax when the static type is the same as the type containing `operator is`, or if the runtime type of the expression implements `ITuple`.

```antlr
positional_pattern
    : type? '(' subpattern_list? ')'
    ;
```

If the *type* is omitted, we take it to be the static type of *e*. In this case it is an error if *e* does not have a type.

Given a match of an expression *e* to the pattern *type* `(` *subpattern_list* `)`, a method is selected by searching in *type* for accessible declarations of `operator is` and selecting one among them using *match operator overload resolution*.

- If a suitable `operator is` exists, it is a compile-time error if the expression *e* is not *pattern compatible* with the type of the first argument of the selected operator. If the *type* is omitted, it is an error if the `operator is` found does not have the static type of *e* as its first parameter. At runtime the value of the expression is tested against the type of the first parameter as in a type pattern. If this fails then the positional pattern match fails and the result is `false`. If it succeeds, the operator is invoked with fresh compiler-generated variables to receive the `out` parameters. Each value that was received is matched against the corresponding *subpattern*, and the match succeeds if all of these succeed. The order in which subpatterns are matched is unspecified, and a failed match may not match all subpatterns.
- If no suitable `operator is` exists, but the expression is *pattern compatible* with the type `System.ITuple`, and no *argument_name* appears among the subpatterns, then we match using `ITuple`. [Note: this needs to be made more precise.]
- Otherwise the pattern is a compile-time error.

If a *subpattern* has an *argument_name*, then every subsequent *subpattern* must have an *argument_name*. In this case each argument name must match a parameter name (of an overloaded `operator is` in the first bullet above). [Note: this needs to be made more precise.]

#### Property Pattern

A property pattern enables the program to recursively match values extracted by the use of properties.

```antlr
property_pattern
    : type? '{' property_subpattern_list? '}'
    | type identifier '{' property_subpattern_list? '}'
    | var identifier '{' property_subpattern_list? '}'
    ;

property_subpattern_list
    : property_subpattern
    | property_subpattern ',' property_subpattern_list
    ;

property_subpattern
    : identifier 'is' pattern
    ;
```

Given a match of an expression *e* to the pattern *type* `{` *property_pattern_list* `}`, it is a compile-time error if the expression *e* is not *pattern compatible* with the type *T* designated by *type*. If the type is absent or designated by `var`, we take it to be the static type of *e*. If the *identifier* is present, it declares a pattern variable of type *type*. Each of the identifiers appearing on the left-hand-side of its *property_pattern_list* must designate a readable property or field of *T*. If the *identifier* of the *property_pattern* is present, it defines a pattern variable of type *T*.

At runtime, the expression is tested against *T*. If this fails then the property pattern match fails and the result is `false`. If it succeeds, then each *property_subpattern* field or property is read and its value matched against its corresponding pattern. The result of the whole match is `false` only if the result of any of these is `false`. The order in which subpatterns are matched is not specified, and a failed match may not match all subpatterns at runtime. If the match succeeds and the *identifier* of the *property_pattern* is present, it is assigned the matched value.

> Note: The property pattern can be used to pattern-match with anonymous types. 



### Match Expression

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

### Some Possible Optimizations

The compilation of pattern matching can take advantage of common parts of patterns. For example, if the top-level type test of two successive patterns in a *switch_statement* is the same type, the generated code can skip the type test for the second pattern.

When some of the patterns are integers or strings, the compiler can generate the same kind of code it generates for a switch-statement in earlier versions of the language.

For more on these kinds of optimizations, see [[Scott and Ramsey (2000)]](http://www.cs.tufts.edu/~nr/cs257/archive/norman-ramsey/match.pdf "When Do Match-Compilation Heuristics Matter?").

It would be possible to support declaring a type hierarchy closed, meaning that all subtypes of the given type are declared in the same assembly. In that case the compiler can generate an internal tag field to distinguish among the different subtypes and reduce the number of type tests required at runtime. Closed hierarchies enable the compiler to detect when a set of matches are complete. It is also possible to provide a slightly weaker form of this optimization while allowing the hierarchy to be open.

----------------------------

Here are my top open issues for pattern matching after C# 7.

# Open LDM Issues in Pattern-Matching

### Recursive pattern forms

The following is the working grammar for recursive patterns. It requires LDM review.

``` antlr
pattern
	: declaration_pattern
	| constant_pattern
	| deconstruction_pattern
	| property_pattern
	;
declaration_pattern
	: type identifier
	;
constant_pattern
	: expression
	;
deconstruction_pattern
	: type? '(' subpatterns? ')' property_subpattern? simple_designation?
	;
subpatterns
	: subpattern
	| subpattern ',' subpatterns
	;
subpattern
	: pattern
	| identifier ':' pattern
	;
property_subpattern
	: '{' subpatterns? '}'
	;
property_pattern
	: type? property_subpattern simple_designation?
	;

simple_designation
	: single_variable_designation
	| discard_designation
        ;
```

It is a semantic error if any _subpattern_ of a _property_pattern_ does not contain an _identifier_ (it must be of the second form, which has an _identifier_).

Note that a null-checking pattern falls out of a trivial property pattern. To check if the string `s` is non-null, you can write any of the following forms

``` c#
if (s is object o) ... // o is of type object
if (s is string x) ... // x is of type string
if (s is {} x) ... // x is of type string
if (s is {}) ...
```

I suspect that the LDM will prefer `is` instead of `:` in the _property_subpattern_.

**Resolution 2017-11-20 LDM**: we want to forbid a deconstruction pattern with a single subpattern but an omitted type. We may consider ways to relax this later. The `identifier`s in the syntax above should be designations.]

### Matching via `ITuple`

We need to specify/decide under what conditions the compiler will attempt to match a positional pattern using `ITuple`.

**Resolution 2017-11-20 LDM** permit matching via `ITuple` only for `object`, `ITuple`, and types that are declared to implement `ITuple` but contain no `Deconstruct` methods.]

### Parenthesized expression vs tuple pattern

There is an ambiguity between a constant pattern expressed using a parenthesized expression, and a “positional” (Deconstruct) recursive pattern with one element:

``` c#
case (2):
```

Here are the two possible meanings:
``` c#
switch (2)
{
    case (2): // a parenthesized expression
```
and
``` c#
class C { public void Deconstruct(out int x) … }

switch (new C())
{
    case (2): // Call Deconstruct once.
```
While it is possible for the programmer to disambiguate (e.g. `case +(2):` or `case (2) {}`), we still need a policy for the compiler to disambiguate when the programmer has not done so. The proposed policy is that this will be parsed as a parenthesized expression. However, if the type of the parenthesized expression is not suitable given the switch expression, semantic analysis will notice the set of parentheses and fall back to trying `Deconstruct`. That will allow both of the above examples to “work” as written.

It does this recursively, so this will work too:
``` c#
class C { public void Deconstruct(out D d) … }
class D { public void Deconstruct(out int x) … }

switch (new C())
{
    case ((2)): // Call Deconstruct twice.
```
A semantic ambiguity arises between these two interpretations when the switched type is, for example, `object` (we deconstruct using `ITuple`). Another way the ambiguity could arise would be if someone were to write an (extension) `Deconstruct` method for one of the built-in types. In either case we will just treat it as a parenthesized expression (i.e. the simplest interpretation that is semantically meaningful), and require explicit disambiguation if another interpretation is intended. 

**Resolution 2017-11-20 LDM**: disallow a deconstruction pattern that contains a single subpattern but for which the type is omitted. We'll look at other ways of disambiguating later, such as perhaps permitting `var` to infer the type, or a trailing comma. This also keeps the design space open for using parens for grouping patterns in the future, e.g. if we introduce `or` and `and` patterns.]

### Cast expression ambiguity

There is a syntactic ambiguity between a cast expression and a positional pattern. The switch case
``` c#
case (A)B:
```
could be either a cast (i.e. existing code, if A is the name of an enum type, `Int32` or a using-alias)
``` c#
case (Int32)MyConstantValue:
```
or a single-element deconstruction pattern with the constant subpattern A being named B.
``` c#
case (MyConstantValue) x:
```
When such an ambiguity occurs, it (is proposed that it) will be parsed as a cast expression (for compatibility with existing code). To disambiguate, you can add an empty property pattern part:
``` c#
case (MyConstantValue) {} x;
```

**Resolution 2017-11-20 LDM**: this issue is moot given the resolution to the previous issue. It is a cast if it can be a cast, otherwise it is an error.]

### Short discard

I think that in recursive patterns the syntax for a discard `var _` will be a bit too verbose. I'd rather see `_` used as an alternative, e.g. `o is Point(3, _)`. But that conflicts with its use as a constant pattern. That also gets mixed up with the parenthesized expression ambiguity described above. My proposal would be to say that `_` in a pattern is *always* a discard, but with an error reported if there is a constant named `_` in scope. That error prevents a silent change to existing code, where `x is _` may treat the `_` as a constant.

This was part of the motivation for #1064, but that may be going too far.

**Resolution 2017-11-20 LDM**: we want to permit the short discard. We don't want to break existing code that tests for a type, e.g. `expr is _` where there is a type named `_`. So that will continue to work. However, we do not want to permit a short discard at the top level in an is-pattern expression. We also want to forbid matching a named constant named `_`. The latter may be a breaking change, but only from a recently permitted coding pattern, so it is probably a break we can take.]

### Match expression

We need to define the _syntax_ and _semantics_ of the match expression.

There are many syntax options listed below. Here is one proposal to get the discussion started:

``` c#
    state = (state, action) switch {
        (DoorState.Closed, Action.Open) => DoorState.Opened,
        (DoorState.Opened, Action.Close) => DoorState.Closed,
        (DoorState.Closed, Action.Lock) => DoorState.Locked,
        (DoorState.Locked, Action.Unlock) => DoorState.Closed,
        _ => state};
```

this is based on the following grammar

``` antlr
switch_expression
    : null_coalescing_expression switch '{' switch_expression_case_list '}'
    ;
switch_expression_case_list
    : switch_expression_case
    | switch_expression_case_list ',' switch_expression_case
    ;
switch_expression_case
    : pattern when_clause? '=>' expression
    ;
```

With *switch_expression* added at roughly the same precedence as a conditional expression but left associative.

``` antlr
non_assignment_expression
    : conditional_expression
    | lambda_expression
    | query_expression
    | switch_expression
    ;
```

The main semantic question is what should happen when the set of patterns is incomplete: should the compiler silently accept the code (or perhaps with a warning) and allow it to throw a runtime exception when the input is unmatched, or should it be a compile-time error?

**Discussion 2018-04-04 LDM**: Syntax OK for now. We will probably revisit. Warning and exception OK.

### Scope of expression variables in initializers

See https://github.com/dotnet/csharplang/issues/32 for details.

### Order of evaluation in pattern-matching

Giving the compiler flexibility in reordering the operations executed during pattern-matching can permit flexibility that can be used to improve the efficiency of pattern-matching. The (unenforced) requirement would be that properties accessed in a pattern, and the Deconstruct methods, are required to be "pure" (side-effect free, idempotent, etc). That doesn't mean that we would add purity as a language concept, only that we would allow the compiler flexibility in reordering operations.

**Resolution 2018-04-04 LDM**: confirmed: the compiler is permitted to reorder calls to `Deconstruct`, property accesses, and invocations of methods in `ITuple`, and may assume that returned values are the same from multiple calls. The compiler should not invoke functions that cannot affect the result, and we will be very careful before making any changes to the compiler-generated order of evaluation in the future.

### `default` pattern

Once we start having a match expression that doesn't use the keyword `case`, a constant pattern that uses the simple literal `default` will look like a default branch of the switch expression. To avoid any confusion, I would like to disallow a simple expression `default` as a constant pattern. We could leave it as a warning, as today, or make it an error in a switch statement. Note that you can always write either `null`, `0`, or `'\0'` instead of `default`.

**Resolution**: See https://github.com/dotnet/roslyn/issues/23499 which documents our tentative decision, taken via email, to forbid `default` as a pattern.

### Range Pattern

If we have a range operator `1..10`, would we similarly have a range pattern? How would it work?

``` c#
    if (ch is in 'a' to 'z')
    switch (ch) {
        case in 'a' to 'z':
```
    
**Discussion 2018-04-04 LDM**: Keep this on the back burner. It is not clear how important it is. We would want to discuss this in the context of also extending other constructs:
``` c#
from i in 1 to 100 select ...

foreach (var i in 0 to s.Length-1) ...
```

### `var` deconstruct pattern

It would be nice if there were a way to pattern-match a tuple (or Deconstructable) into a set of variables declared only by their designator, e.g. the last line in this match expression

``` c#
    var newState = (GetState(), action, hasKey) switch {
        (DoorState.Closed, Action.Open, _) => DoorState.Opened,
        (DoorState.Opened, Action.Close, _) => DoorState.Closed,
        (DoorState.Closed, Action.Lock, true) => DoorState.Locked,
        (DoorState.Locked, Action.Unlock, true) => DoorState.Closed,
        var (state, _, _) => state };
```

(Perhaps not the best example since it only declares one thing on the last line)

This would be based on some grammar like this

``` antlr
var_pattern
    : 'var' variable_designation
    ;
```

where the _variable_designation_ could be a _parenthesized_variable_designation_, i.e. generalizing the current construct.

To make this syntactically unambiguous, we would no longer allow `var` to bind to a user-declared type in a pattern. Forbidding it from binding to a constant would also simplify things, but probably isn't strictly necessary.

I’m suggesting this now because at a recent LDM it was suggested that `var` could perhaps be used as a placeholder for an unknown type taken from context. But that would conflict with this usage because this usage changes the syntax of what is permitted between the parens (designators vs patterns).

**Resolution 2018-04-04 LDM**: YES! Approved.

### ref/lvalue-producing pattern switch expression

As currently designed, the “switch expression” yields an rvalue.

``` c#
    e switch { p1 when c1 => v1, p2 when c2 => v2 }
```
@agocke pointed out that it might be valuable for there to be a variant that produces a ref or an lvalue.

1.	Should we pursue this?
2.	What would the syntax be?
    `e switch { p1 when c1 => ref v1, p2 when c2 => ref v2 }`
3.	Would the same syntax be used to produce a ref and an lvalue?

**Discussion 2018-04-04 LDM**: Lets keep this on the back burner and see if there are requests based on actual use cases.

### switching on a tuple literal

In order to switch on a tuple literal, you have to write what appear to be redundant parens

``` c#
switch ((a, b))
{
```

It has been proposed that we permit

``` c#
switch (a, b)
{
```

**Resolution 2018-04-04 LDM**: Yes. We discussed a couple of ways of doing this, and settled on making the parens of the switch statement optional when the expression being switched on is a tuple literal.

### Short discard diagnostics

Regarding the previous short discard diagnostics approved by the LDM (see above), there are two possibly breaking situations.

`case _` where there is a named constant `_` is now an error.

`e is _` is proposed to be a warning: `The name '_' refers to the type '{0}', not the discard pattern. Use '@_' for the type, or 'var _' to discard.`

### Nullable Reference Types vs switch analysis

We give a warning when a switch expression does not exhaustively handle all possible inputs. What if the input is a non-nullable string `s`, in the code

`s switch { string t => t }`

Is a warning deserved because the switch expression does not handle all possible inputs (it doesn't handle a possible null)? Note that although we *think* s cannot be null, we still generate code for a null check and throw an exception when the null input is not handled.

If we decide *not* to generate a warning here, we may have to carefully structure the phases of the compiler so we do this exhaustiveness analysis after the nullable pass of the compiler, which will now have to record its conclusions in a rewritten bound tree.

Similarly, in the code

``` c#
int i;
switch (s)
{
    case string t: i = 0; break;
}
Console.WriteLine(i); // is i definitely assigned?
```

Is the last line an error because `i` is not definitely assigned?

### Use type unification to rule out some pattern-matching situations?

See https://github.com/dotnet/roslyn/issues/26100. The possible new warnings for the is-type operator correspond to possible errors for the is-pattern operator. We could use type unification subroutines, which are already in the compiler, to detect some situations and make them errors. Should we? Sooner is better.

### Permit optionally omitting the pattern on the last branch of the switch expression

We wonder if it would be helpful to sometime omit the pattern on the last branch of a switch expression, in cases where the pattern would be `_`.

### Should exhaustiveness affect definite assignment

In a non-pattern switch statement, [we do not currently (in any previous C# version) do "exhaustiveness" analysis](https://github.com/dotnet/roslyn/issues/24865):

``` c#
    int M(bool b)
    {
        int result;
        switch (b)
        {
            case false:
                result = 0;
                break;
            case true:
                result = 1;
                break;
        }
        return result; // error: not definitely assigned
    }
```

But in a pattern-based switch statement we do:

``` c#
    int M(bool b)
    {
        int result;
        switch (b)
        {
            case false when true:
                result = 0;
                break;
            case true:
                result = 1;
                break;
        }
        return result; // ok: definitely assigned
    }
```

We should decide and confirm the intended behavior for both cases. This also includes [reachability of statements](https://github.com/dotnet/roslyn/issues/11438). The current prototype does exhaustiveness analysis for switch statements based on a [previous informal decision](https://github.com/dotnet/roslyn/issues/11438#issuecomment-232770018). We should confirm that in the LDM.

### Switch expression as a statement expression

It has been requested that we permit a switch expression to be used as a statement expression. In order to make this work, we'd
1. Permit its type to be `void`.
2. Permit a switch expression to be used as a statement expression if all of the switch arm expressions are statement expressions.
3. Adjust type inference so that `void` is more specific than any other type, so it can be inferred as the result type.

### Single-element positional deconstruction

The current LDM decision on single-element positional deconstruction pattern is that the type is required.

This is particularly inconvenient for tuple types and other generic types, and perhaps impossible for values deconstructed by a dynamic check via `ITuple`.

We should consider if other disambiguation should be permitted (e.g. the presence of a designation).

``` c#
if (o is (3) _)
```

### Disambiguating deconstruction using parameter names

We could permit more than one `Deconstruct` method if we permit disambiguating by parameter names.

``` c#
class Point
{
    public void Deconstruct(double Angle, double Length) => ...;
    public void Deconstruct(int X, int Y) => ...;
}

Point p = ...;
if (p is (X: 3, Y: 4)) ...;
```

Should we do this?

### var pattern for 0 and 1 elements

The var pattern currently requires 2 or more elements because it inherits the grammar for *variable_designation*. However, both 0-element and 1-element var patterns would make sense and are syntactically unambiguous.

``` c#
public class C {
    public void Deconstruct() => throw null;
    public void Deconstruct(out int i) => throw null;
    public void Deconstruct(out int i, out int j) => throw null;
    void M() {
        if (this is var ()) { }          // error
        if (this is var (x1)) { }        // error
        if (this is var (x2, y2)) { }    // ok
    }
}
```

I propose we relax the grammar to permit 0-element and 1-element var patterns.

We could do the same for deconstruction.

### Syntax model for var patterns

In C# 7, we treat both `Type identifier` and `var identifier` as a DeclarationPatternSyntax, even though they have quite different semantics.

In C# 8, we introduce a more general *var pattern* that accepts any designation on the right-hand-side.

We would prefer to use the new node for `var identifier` even though those were represented by a declaration pattern previously. Clients of the syntax API would see a change (to using a new node) when parsing existing code.

Is this an acceptable change to make?

### Kind of member in a property pattern

A property pattern `{ PropName : Pattern }` is used to check if the value of a property or field matches the given pattern.

Besides readable properties and fields, what other kinds of members should be permitted here?
- An indexed property? Possible indexed with a constant?
- An event reference?
- Anything else?

### Restricted types

A recent bug (https://github.com/dotnet/roslyn/pull/27803) revealed how pattern matching interacts with restricted types in interesting ways (when doing type checks).
We should discuss with LDM on whether this should be banned (like pointer types), and if there are actual use cases that need it.

### Matching using ITuple

I assume that we only intend to permit matching using `ITuple` when the type is omitted?

``` c#
IComparable x = ...;
if (x is ITuple(3, 4)) // (1) permitted?
if (x is object(3, 4)) // (2) permitted?
if (x is SomeTypeThatImplementsITuple(3, 4)) // (3) permitted?
```

### Matching using `ITuple` in the presence of an extension `Deconstruct`

When an extension `Deconstruct` method is present and a type also implements the `ITuple` interface, it is not clear which should take priority. I believe the current LDM position (probably not intentional) is that the extension method is used for the deconstruction declaration or deconstruction assignment, and `ITuple` is used for pattern-matching. We probably want to reconcile these to be the same.

``` c#
public class C : ITuple
{
    int ITuple.Length => 3;
    object ITuple.this[int i] => i + 3;
    public static void Main()
    {
        var t = new C();
        Console.WriteLine(t is (3, 4)); // true? or a compile-time error?
        Console.WriteLine(t is (3, 4, 5)); // true? or a compile-time error?
    }
}
static class Extensions
{
    public static void Deconstruct(this C c, out int X, out int Y) => (X, Y) = (3, 4);
}
```

### ITuple vs unconstrained type parameter

Our current rules require an error in the following, and it will not attempt to match using `ITuple`. Is that what we want?

``` c#
    public static void M<T>(T t)
    {
        Console.WriteLine(t is (3, 4)); // error: no Deconstruct
    }
```
