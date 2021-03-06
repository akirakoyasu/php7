====== PHP RFC: Scalar Type Declarations ======
  * Version: 0.5.3
  * Date: 2015-02-18
  * Author: Anthony Ferrara <ircmaxell@php.net> (original Andrea Faulds, ajf@ajf.me)
  * Status: Implemented
  * First Published at: http://wiki.php.net/rfc/scalar_type_hints_v5
  * Forked From: http://wiki.php.net/rfc/scalar_type_hints

===== Summary =====

This RFC proposes the addition of four new type declarations for scalar types: ''int'', ''float'', ''string'' and ''bool''. These type declarations would behave identically to the existing mechanisms that built-in PHP functions use.

This RFC further proposes the addition of a new optional per-file directive, ''declare(strict_types=1);'', which makes all function calls and return statements within a file have "strict" type-checking for scalar type declarations, including for extension and built-in PHP functions. In addition, calls to extension and built-in PHP functions with this directive produce an ''E_RECOVERABLE_ERROR'' on parameter parsing failure, bringing them into line with existing userland type declarations.

With these two features, it is hoped that more correct and self-documenting PHP programs can be written.

==== Changes From V0.3 (Andrea's Original Proposal) ====

  * ''declare(strict_types=1)'' (if used) is required to be the first instruction in the file only. No other usages allowed.
  * ''declare(strict_types=1) {}'' (block mode) is specifically disallowed.
  * ''int'' types can resolve a parameter type of ''float''. So calling ''requiresAFloat(10)'' will work. Note that there is no overflow or precision check (see Discussion section for more).
  * aliases are removed (''integer'' and ''boolean'')

===== Details =====

==== Scalar Type Declarations ====

No new reserved words are added. The names ''int'', ''float'', ''string'' and ''bool'' are recognised and allowed as type declarations, and prohibited from use as class/interface/trait names (including with ''use'' and ''class_alias'').

The new userland scalar type declarations are implemented internally by calling the Fast Parameter Parsing API functions.

==== strict_types declare() directive ====

By default, all PHP files are in weak type-checking mode. A new ''declare()'' directive is added, ''strict_types'', which takes either ''1'' or ''0''. If ''1'', strict type-checking mode is used for function calls and return statements in the the file. If ''0'', weak type-checking mode is used.

The ''declare(strict_types=1)'' directive **must** be the first statement in a file. If it appears anywhere else in the file it will generate a compiler error. Block mode is also explicitly disallowed (''declare(strict_types=1);'' is the only allowed form).

Like the ''encoding'' directive, but unlike the ''ticks'' directive, the ''strict_types'' directive only affects the specific file it is used in, and does not affect either other files which include the file nor other files that are included by the file.

The directive is entirely compile-time and cannot be controlled at runtime. It works by setting a flag on the opcodes for function calls (for parameter type declarations) and return type checks (for return type declarations).

=== Parameter type declarations ===

The directive affects any function call, including those within a function or method. For example (strict mode):

<file php strict_types_scope.php>
<?php
declare(strict_types=1);

foo(); // strictly type-checked function call

function foobar() {
    foo(); // strictly type-checked function call
}

class baz {
    function foobar() {
        foo(); // strictly type-checked function call
    }
}
</file>

vs (weak mode):

<file php weak_types_scope.php>
<?php
foo(); // weakly type-checked function call

function foobar() {
    foo(); // weakly type-checked function call
}

class baz {
    function foobar() {
        foo(); // weakly type-checked function call
    }
}
</file>

Whether or not the function being called was declared in a file that uses strict or weak type checking is irrelevant. The type checking mode depends on the file where the function is called.

=== Return type declarations  ===

The directive affects any return statement in any function or method within a file. For example (strict mode):

<file php strict_types_scope2.php>
<?php
declare(strict_types=1);

function foobar(): int {
    return 1.0; // strictly type-checked return
}

class baz {
    function foobar(): int {
        return 1.0; // strictly type-checked return
    }
}
</file>

<file php weak_types_scope2.php>
<?php

function foobar(): int {
    return 1.0; // weakly type-checked return
}

class baz {
    function foobar(): int {
        return 1.0; // weakly type-checked return
    }
}
</file>

Unlike parameter type declarations, the type checking mode used for return types depends on the file where the function is defined, not where the function is called. This is because returning the wrong type is a problem with the callee, while passing the wrong type is a problem with the caller.

==== Behaviour of weak type checks ====

A weakly type-checked call to an extension or built-in PHP function has exactly the same behaviour as it did in previous PHP versions.

The weak type checking rules for the new scalar type declarations are mostly the same as those of extension and built-in PHP functions. The only exception to this is the handling of ''NULL'': in order to be consistent with our existing type declarations for classes, callables and arrays, ''NULL'' is not accepted by default, unless it is a parameter and is explicitly given a default value of ''NULL''.

For the reference of readers who may not be familiar with PHP's existing weak scalar parameter type rules, the following brief summary is provided.

The table shows which types are accepted and converted for scalar type declarations. ''NULL'', arrays and resources are never accepted for scalar type declarations, and so are not included in the table.

^ Type declaration  ^ int  ^ float    ^ string   ^ bool ^ object  ^
^ ''int''           ^ yes  ^ yes*     ^ yes†     ^ yes  ^ no      ^
^ ''float''         ^ yes  ^ yes      ^ yes†     ^ yes  ^ no      ^
^ ''string''        ^ yes  ^ yes      ^ yes      ^ yes  ^ yes‡    ^
^ ''bool''          ^ yes  ^ yes      ^ yes      ^ yes  ^ no      ^

<nowiki>*</nowiki>Only non-NaN floats between ''PHP_INT_MIN'' and ''PHP_INT_MAX'' accepted. (New in PHP 7, see the [[rfc:zpp_fail_on_overflow|ZPP Failure on Overflow]] RFC)

†Non-numeric strings not accepted. Numeric strings with trailing characters are accepted, but produce a notice.

‡Only if it has a ''__toString'' method.

==== Behaviour of strict type checks ====

A strictly type-checked call to an extension or built-in PHP function changes the behaviour of ''zend_parse_parameters''. In particular, it will produce ''E_RECOVERABLE_ERROR'' rather than ''E_WARNING'' on failure, and it follows strict type checking rules for scalar typed parameters, rather than the traditional weak type checking rules.

The strict type checking rules are quite straightforward: when the type of the value matches that specified by the type declaration it is accepted, otherwise it is not.

These strict type checking rules are used for userland scalar type declarations, and for extension and built-in PHP functions.

The one exception is that [[http://docs.oracle.com/javase/specs/jls/se7/html/jls-5.html#jls-5.1.2|widening primitive conversion]] is allowed for ''int'' to ''float''. This means that parameters that declare ''float'' can also accept ''int''.

<file php widening.php>
<?php
declare(strict_types=1);

function add(float $a, float $b): float {
    return $a + $b;
}

add(1, 2); // float(3)
</file>

In this case, we're passing an ''int'' to a function that accepts ''float''. The parameter is converted (widened) to float.

No other conversions are allowed.

==== Error Handler Behavior In Strict Mode ====

Currently it's possible to bypass error check failures using an error handler:

<file php error_handler_fail.php>
<?php
declare(strict_types=1);
set_error_handler(function() {
    return true;
});

function foo(int $abc) {
    var_dump($abc);
}
foo("test"); // string(4) "test"
?>
</file>

This would defeat the purpose of strict typing.

Therefore, this RFC proposes to bypass function execution in strict mode if there's a type mismatch error (just like internal functions do today). The implementation is not complete, as this behavior would be superseded by [[rfc:engine_exceptions]] if it passed. Therefore the implementation will wait for the completion of voting on that RFC.


===== Example =====

Let's create a function that adds two integers together

<file php add.php>
<?php
function add(int $a, int $b): int {
    return $a + $b;
}
</file>

In a separate file, we can call the add function using weak typing

<file php main.php>
<?php
require "add.php";

var_dump(add(1, 2)); // int(3)
// floats are truncated by default
var_dump(add(1.5, 2.5)); // int(3)

//strings convert if there's a number part
var_dump(add("1", "2")); // int(3)
</file>

The types of arguments are "converted" to integer where it makes sense.

By default, weak type declarations that permit some conversions are used, so we could also pass values that are convertible and they'll be converted, just like with extension and built-in PHP functions:

<file php main2.php>
<?php
require "add.php";

var_dump(add("1 foo", "2")); // int(3)
// Notice: A non well formed numeric value encountered
</file>

However, it is also possible to turn on strict type checking with an optional directive. In this mode, the same call would fail:

<file php main3.php>
<?php
declare(strict_types=1);

require "add.php";

var_dump(add(1, 2)); // int(3)

var_dump(add(1.5, 2.5)); // int(3)
// Catchable fatal error: Argument 1 passed to add() must be of the type integer, float given
</file>

The directive affects all function calls in the file, regardless of whether the functions being called were declared in files which used strict type checking. So:

In addition to userland functions, the strict type checking mode also affects extension and built-in PHP functions:

<file php main4.php>
<?php
declare(strict_types=1);

$foo = substr(52, 1);
// Catchable fatal error: substr() expects parameter 1 to be string, integer given
</file>

Scalar type declarations would also work for return values, as does strict type checking mode:

<file php returns.php>
<?php

function foobar(): int {
    return 1.0;
}

var_dump(foobar()); // int(1)
</file>

In weak mode, the float is cast to an integer.

<file php returns_strict.php>
<?php
declare(strict_types=1);

function foobar(): int {
    return 1.0;
}

var_dump(foobar());
// Catchable fatal error: Return value of foobar() must be of the type integer, float returned
</file>

===== Background and Rationale =====

==== History ====

PHP has had parameter type declarations for interface and class names since PHP 5.0, arrays since PHP 5.1 and callables since PHP 5.4. These type declarations allow the PHP runtime to ensure that correctly-typed arguments are passed to functions, and make function signatures more informative. Unfortunately, PHP's scalar types haven't been typeable.

There have been some previous attempts at adding scalar type declarations, such as the [[rfc:scalar_type_hinting_with_cast|Scalar Type Hints with Casts]] RFC. Previous attempts have failed for a variety of reasons:

  * Type conversion and validation behaviour did not match that of extension and built-in PHP functions
  * It followed a weak typing approach
  * Its attempt at "stricter" weak typing failed to placate either strict typing or weak typing fans

This RFC attempts to address all of the issues.

==== Weak typing and strict typing ====

There are three major approaches to how to check parameter and return types in use in modern programming languages:

  * Fully strict type checking (where no conversion happens). This is used by languages such as F#, Go, Haskell, Rust and Facebook's Hack.
  * Widening primitive type checking (where "safe" conversions happen). This is used by languages such as Java, D and Pascal. They allow for [[http://docs.oracle.com/javase/specs/jls/se7/html/jls-5.html#jls-5.1.2|Widening-Primitive-Conversion]] to happen implicitly. That means that a 8-bit integer can be implicitly passed to an argument requiring a 16 bit integer. And an integer can be passed to an argument requiring a floating point number. No other conversion is allowed implicitly.
  * Weak type checking (which all conversions are allowed, with possible warnings raised), which is used to a limited extent by C, C#, C++ and Visual Basic. This tries to "never fail" and always makes a guess at a conversion.

PHP's internal treatment of scalars in ''zend_parse_parameters'' for built-in functions has traditionally followed the weak mode. PHP's treatment of Objects (both internally and externally) uses a form of Widening checking, where an exact match is not required, but children are allowed (also called [[http://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science)|contravariance]]).

Each approach has advantages and disadvantages.

This proposal builds in weak type checking by default (using the same rules), for internal and user functions. It also adds a switch to convert to Widening type checking (called strict mode in this proposal).

==== Why both? ====

So far, most advocates of scalar type declarations have asked for either strict type checking, or weak type checking. Rather than picking one approach or the other, this RFC instead makes weak type checking the default, and adds an optional directive to use strict type checking within a file. There were several reasons behind this choice.

A significant portion of the PHP community appears to favor fully-strict types. However, adding strictly type-checked scalar type declarations would cause a few problems:

  * It creates a glaring inconsistency: extension and built-in PHP functions use weak type checking for scalar typed parameters, yet userland PHP functions would be using //strict// type checking for scalar type declared parameters.
  * The significant population who would like weak type checking would not be in favour of such a proposal, and are likely to block it.
  * Existing code which (perhaps unintentionally) took advantage of PHP's weak typing would break if functions it calls added scalar type declarations to parameters. This would complicate the addition of scalar type declarations to the parameters of functions in existing codebases, particularly libraries.

There is also a significant group of people who are in favour of weak type checking. But, like adding strictly type-checked declarations, adding weakly type-checked scalar type declarations would also cause problems:

  * The large number of people who would like strict type checking would not be in favour of such a proposal, and are likely to block it.
  * It would limit opportunities for static analysis.
  * It can hide subtle bugs where automatic type conversion results in data loss.

A third approach has also been suggested, which is to add separate weakly- and strictly-checked type declarations with different syntax. It would present its own set of issues:

  * People who do not like weak or strict type checking would be forced to deal with strictly or weakly type-checked libraries, respectively.
  * Like adding strict declarations, this would also be inconsistent with extension and built-in PHP functions, which are uniformly weak.

In order to avoid the issues with these three approaches, this RFC proposes a fourth approach: per-file strict or weak type-checking. This has the following advantages:

  * People can choose the type checking model that suits them best, which means this approach should hopefully placate both the strict and weak type checking camps.
  * APIs do not force a type declaration model upon their users.
  * Because files use the weak type checking approach by default, functions in existing codebases (including libraries) should be able to have scalar type declarations added without breaking code that calls them. This enables codebases to add type declarations gradually, or only to portions, which is known as "gradual typing".
  * There only needs to be a single syntax for scalar type declarations.
  * People who would prefer strict type checking get it not only for userland functions, but also for extension and built-in PHP functions. This means users get one model uniformly, rather than having the inconsistency that introducing strict-only scalar declarations would have produced.
  * In strict type checking mode, the error level produced when type checking fails for extension and built-in PHP functions will finally be consistent with the error level produced for userland functions, with both producing ''E_RECOVERABLE_ERROR''.
  * It allows for seamless integration of strict and weak code in a single codebase.

==== Type declaration choices ====

No type declaration for resources is added, as this would prevent moving from resources to objects for existing extensions, which some have already done (e.g. GMP).

===== Discussion Points =====

There are a number of questions around this proposal that have been discussed on-list. I will attempt to curate a list of them here, as well as the stance that this RFC takes:

==== This Proposal Is A Compromise ====

Several people have said that this proposal is a compromise. That it attempts to walk the middle to appease proponents of both strict and weak typing.

=== Current Position ===

This proposal is not a compromise. It is an attempt of allowing strict typing to work in PHP. A mechanism to bridge untyped PHP code with strict typed PHP code, a "weak" bridge, would be required (otherwise explicit ''(type)'' casts would be needed). This proposal unifies the strict and weak typing into a single system that integrates tightly and behaves consistently.

==== Internal Functions Like ceil() Return Unexpected Types ====

Currently, ''ceil()'' returns a ''float''. This results in potentially obscure behavior as the following will fail:

<file php ceil.php>
<?php
declare(strict_types=1);

function foobar(float $abc): int {
    return ceil($abc + 1);
}

foobar(123.0);
</file>

The return types will clash.

There are two ways of solving this issue:

  * Change the type of ''ceil()'' to be ''int'' which is more in line with the 99% use case.
  * Have users cast the type to ''int'' in their functions.

=== Current Position ==

This proposal takes the position that users casting is the correct way forward. The reason is that changing the return type of internal functions to support the 99% use case will undoubtedly make it worse for the 1% use case which would no longer be supported.

The cast makes the intent explicit both to the compiler and to the reader, so that both understand what's happening and what is expected.

==== "37" Should Be Accepted For int Types ====

Currently, if you do ''"37" + 1'' you will get ''int(38)''. Many proponents of weak typing would like to see integer-like strings pass for ''int'' typed functions.

Conversely, many advocates of strict typing point out that this type check is not possible ahead of time, as it relies on values, not types. Therefore it's not a type check, but a runtime value check. This defeats a lot of the point of using strict types in the first place.

=== Current Position ===

This proposal takes the position that numeric strings should be accepted for declarations in weak mode only. In strict mode, types are all that are evaluated.

==== Integers Should Be Accepted For Strict float Arguments ====

In earlier revisions of this RFC, integers were not accepted for float declared functions. This means that the following code would have failed because ''number_format()'' expects a float for its first argument.
:

<file php int_vs_float.php>
<?php
declare(strict_types=1);

echo number_format(50);
</file>

=== Current Position ===

In line with [[http://docs.oracle.com/javase/specs/jls/se7/html/jls-5.html#jls-5.1.2|Java]], D and Pascal, this proposal implements widening-conversion rules. This means that integers are accepted for floating point arguments (the example above works).

It however also means that narrowing conversions (float->int) do not work when passing arguments to functions.

Note: If you read the Java spec, you'll notice that it does mention narrowing conversions. It only allows them in assignment or explicit casts however. So they do not apply in the case this proposal puts forward.

==== Weak Should Error On "10 Birds" Style-Strings Passed To Int Parameters ====

Currently, a notice on malformed numeric string is raised. Some proponents of weak typing would like to see "10 birds" be raised to a warning or recoverable error.

=== Current Position ===

This proposal does not fundamentally change the weak conversion rules that were already implemented for internal functions. It simply exposes them to userland.

Therefore, this proposal's position is that changing weak-type error behavior is outside the scope of this proposal.

==== Int -> Float Conversion Isn't Lossless ====

On a 64 bit platform, integers > 2^53 will not be exactly representable using a float. This can result in subtle issues for function calls as it can result in subtle data loss.

<file php float_precision_loss.php>
<?php
declare(strict_types=1);

echo number_format((1<<61)+1);
</file>

This would output ''2,305,843,009,213,693,952''. The output is incorrect, since the integer representation ends with ''953''. So data is lost in the conversion (since ''number_format'' accepts a float).

=== Current Position ===

This RFC currently takes the position that this is acceptable. There are two reasons for it:

  * This is the current behavior today: http://3v4l.org/0IolN
  * This will not affect a large number of values.

If it does affect the operation of a function significantly, then the function should be modified to accept a ''numeric'' type (a union of ''int'' and ''float''), and make a logical switch between the two to support arbitrarily large data.

Additionally, a number of mainstream strict-typed programming languages behave in this fashion (such as Java, C#, D and Pascal). So it's not unexpected.

==== Int->Float Exception Makes Strict Mode "Flawed" ====

Some people have pointed out that it appears that the int->float widening exception shows that the concept of strict mode is flawed.

=== Current Position ===

The benefits of a strict mode are independent of individual acceptance rules. This is because strict typing depends solely on the type of the argument, not its value.

==== Static Analysis Is Possible With Weak Declarations ====

Several people have said that it's possible to statically analyze weak declarations.

=== Current Position ===

This proposal takes the position that since weak declarations depend on the value being passed instead of just its type, static analysis isn't robust.

That's because any static analysis engine would need to do one of two behaviors:

  * Not warn when passing a ''string'' to an ''int'' parameter, because it *may* work.
  * Warn when passing a ''string'' to an ''int'' parameter, even though it may work.

The first option is useless since errors won't be caught ahead of time. The second option is not ideal since fully functional code may be shown to be incorrect.

Therefore, robust static analysis is not possible in a weak-mode (where the check depends on the value).

==== Errors Should Use Exceptions Instead Of Recoverable Errors ====

It has been brought up that type-mismatch errors should be raised as exceptions instead of recoverable errors.

=== Current Position ===

Current coding standards for Zend mandate that exceptions are not to be used outside of object contexts. Therefore type errors should use recoverable errors everywhere for consistency (since they can be used outside of methods).

Any change to the standard would be out-of-scope for this proposal.

==== Nullable And Union Types ====

Interest has been expressed in a system to allow for union-types: ''int|float'' or nullable-types: ''int?''.

=== Current Position ===

As both of these affect more than just scalar typing, both are considered outside of scope for this proposal.

==== There Should Be A numeric Type ====

The usefulness of a new union type ''numeric'' has been brought up (which would be a built-in union of ''int|float'').

=== Current Position ===

The need for a numeric type is lessened by changing ''float'' parameters to accept ''int''. Therefore, this proposal does not introduce a ''numeric'' type.

==== Internal Functions Should "Opt-In" To Typing ====

This proposal adds the ability to strictly type internal function calls. This has led to several developers wishing that internal functions should have to "opt-in" to this typing (via ArgInfo, etc).

This would allow developers to choose if they want their API to be called strictly or not, and fine tune the behavior if done so.

=== Current Position ===

This proposal takes the standpoint that it's up to the caller to decide how functions should be called. Therefore, the existing types that are exposed via ''zend_parse_parameters()'' should be sufficient to make these types available.

There are several places in core where internal functions accept mixed types (''z'' parameter to ZPP). If internal functions want the ability to be "weak", then they should have already been using the ''z'' type specifier and implemented their own logic (just as it possible in user-land). If types exist already, the conversions are happening already. The only difference is this proposal gives control over those existing conversions to the caller code.

Therefore, this proposal does not allow internal developers to "opt-in" to strict typing.

==== Why Not Use "use strict" Instead Of declare() ====

Several people have suggested alternative forms for "switching on" strict typing, including:

  * ''use strict;''
  * ''<?php strict''
  * ''<?php-strict''

And others.

=== Current Position ===

The declare system was designed precisely for this style of engine switch. Additionally, it leaves room for extending additional "strict" behaviors in the future.

There are also problems with using each of the proposed alternates:

  * ''<?php strict''

This is new syntax, which is potentially ambiguous around what "strictness" is being applied. It limits future compatibility.

Additionally, it's potentially ambiguous if a file starts with ''<?=strict; 4 ?>''. Is that setting strict mode for a file and outputting 4? Or is it outputting the constant "strict"? Sure, this could be "solved" with a rule that it could only follow ''<?php'', but that starts to get arbitrary and potentially confusing, given the other ways to open PHP tags.

  * ''<?php-strict''

This opens the door for potential code disclosure vulnerability if run on an earlier version of PHP (since the ''<?php-strict'' opening tag won't be interpreted properly).

  * ''use strict;''

Re-using namespaces to affect runtime is weird. Not to mention what's the expected behavior of block mode:

<file php use_strict.php>
<?php
namespace Foo {
    use strict;
}

namespace {
    bar();
}
?>
</file>

is bar() called in strict mode? Or in non-strict mode?

  * ''<?php %%//%% strict'' (HHVM style)

Comments should not affect runtime behavior. HHVM uses it as they need to affect behavior while remaining compatible with PHP. We do not have that problem.

  * ''declare(strict=true)'' (exactly as in v0.3)

Which had a number of people against it, with arguments about the odd behavior of declare in blocks, etc. It does not respect scope, so calling it in one function would transparently effect all future functions in the file.

  * ''strict namespace''

<file php strict_namespace.php>
<?php
strict namespace Foo {

}

namespace {
    bar();
}
?>
</file>

This has the same issues as use strict above. However, it also seems to imply that the namespace is strict, where it's only the declarations in the file that are.

==== Why Not Allow Block-Mode For Declare ====

''declare()'' in PHP 5.x supports block modes:

<file php declare_blocks.php>
<?php
declare(ticks=1) {
    ticks_code();
}
</file>

It may be useful to support "strict blocks".

=== Current Proposal ===

Allowing strict "blocks" can create situations where a single file uses several "type modes". This can hamper readability and make working on typed code significantly harder.

Therefore, this proposal explicitly disallows changing the type mode anywhere within the file except the first line. Since the first line is the only allowed type change, block mode does not make sense (as there could only ever be a single block in the file).

Additionally, some technical limitations do make it significantly more difficult: [[http://news.php.net/php.internals/83356|Email describing limitiations]].

==== Internal Functions Do Not Have Declared Return Types ====

Currently, internal functions do not declare return types, and can return ''null'' on error. This limits the type-safety that can be had stringing internal functions together.

=== Current Position ===

This proposal does not necessitate adding return types to internal functions. A future proposal is free to add them, which would then make the type system more robust rather than less without a BC break.

==== Why Not Add Support For Null? ====

Some people would like adding ''null'' support in addition to other primitives.

=== Current Position ===

Without union types, ''null'' makes no sense for parameters. The only useful position would be in return types, which is currently handled by a proposal for ''void''.

==== Why Not Add Support For MIXED? ====

Some people would like adding ''mixed'' support in addition to other primitives.

=== Current Position ===

Currently, there's no mandate for fully typing all functions (even in strict mode). Therefore, there's no functional difference between ''mixed'' and a non-type-declared paramter. For that reason, addition of a ''mixed'' type is outside of the scope for this proposal.

==== Why Not Add An INI Setting For Default Mode ====

It's been asked for the ability to switch the default mode from "weak" to "strict" by a mechanism (ini or compile time flag, etc).

=== Current Position ===

This proposal takes the opinion that behavior modifying switches like ini settings are a death-toll to portability and well designed languages should not change behavior based on "global settings". Therefore, switching strict modes will remain a per-file setting for this proposal.

==== Type Aliases Should Not Be Supported ====

The original proposal had two additional type aliases supported: ''integer'' and ''boolean''.

=== Current Position ===

This proposal takes the stance that there should be one obvious type. Therefore, no aliases are supported.

==== This Proposal Should Have Multiple Vote Options ====

Several people have proposed that this proposal should have 3 or 4 vote options (No, Weak Only, Strict Only, Weak + Strict).

=== Current Position ===
https://wiki.php.net/rfc/reserve_more_types_in_php_7
This is not a two-part proposal. The proposal is of a unified system that was designed to work together. As such, neither part (weak-only or strict-only) is designed to stand on its own without the other part.

Therefore, it only makes sense to vote on this proposal as a whole. Therefore, the voting options this RFC will present will be: ''Yes'' and ''No''.

==== Integer Overflow To Float Behavior ====

Currently, certain integer operations will result in an overflow to a floating point value. Consider the following code:

<file php integer_overflow.php>
<?php
var_dump(2 ** 61); // int(2305843009213693952)
var_dump(2 ** 64); // float(1.844674407371E+19)
</file>

This can result in an error when passing the result of an operation to another function expecting an integer.

There are two prime ways of handling this issue:

  * Allow ''float'' -> ''int'' promotion (narrowing)
  * Error at runtime.

=== Current Position ===

Allowing for "narrowing" (truncating the float back to the closest integer) would result in hard to detect bugs.

Therefore, a runtime error that the overflow occurred is the most appropriate thing to do.

Therefore, the position of this proposal is that overflow situations should generate a runtime error. If you need overflow safety, you should be using a library like GMP.

===== Backward Incompatible Changes =====

''int'', ''float'', ''string'' and ''bool'' are no longer permitted as class/interface/trait names (including with ''use'' and ''class_alias'').

Because the weak type-checking rules for scalar declarations are quite permissive in the values they accept and behave similarly to PHP's type juggling for operators, it should be possible for existing userland libraries to add scalar type declarations without breaking compatibility.

Since the strict type-checking mode is off by default and must be explicitly used, it does not break backwards-compatibility.

===== Proposed PHP Version(s) =====

This proposal targets the 7.0 release of PHP.

===== RFC Impact =====

==== To Existing Extensions ====

''ext/reflection'' will need to be updated in order to support scalar type declaration reflection for parameters. This is left to a follow-up RFC to unify type-declaration information into a uniform reflection API.

==== Unaffected PHP Functionality ====

This doesn't affect the behaviour of cast operators.

When the strict type-checking mode isn't in use (which is the default), function calls to built-in and extension PHP functions behave identically to previous PHP versions.

==== TODO ====

===== Future Scope =====

Because scalar type declarations guarantee that a passed argument will be of a certain type within a function body (at least initially), this could be used in the Zend Engine for optimisations. For example, if a function takes two ''float''-declared arguments and does arithmetic with them, there is no need for the arithmetic operators to check the types of their operands. As I understand it, HHVM already does such optimisations, and might benefit from this RFC.

===== Vote =====

As this is a language change, this RFC requires a 2/3 majority to pass.

<doodle title="Accept Scalar Type Declarations With Optional Strict Mode?" auth="ircmaxell" voteType="single" closed="true">
   * Yes
   * No
</doodle>

This vote is opened on February 26th, 2015 and will close March 16th at 21:00 UTC as announced on list.

===== Patches and Tests =====

There is a working, but possibly buggy php-src branch with tests here: https://github.com/ircmaxell/php-src/compare/scalar_type_hints_v5

There is no language specification patch as yet.

===== Implementation =====
After the project is implemented, this section should contain
  - the version(s) it was merged to
  - a link to the git commit(s)
  - a link to the PHP manual entry for the feature

===== References =====

  * Previous discussions on the internals mailing list about scalar type declarations: [[http://marc.info/?l=php-internals&w=2&r=1&s=scalar+type+hinting&q=t|one]], [[http://marc.info/?w=2&r=1&s=scalar+type+hint&q=t|two]], [[http://marc.info/?t=133056746300001&r=1&w=2|three]], [[http://marc.info/?w=2&r=1&s=scalar+type&q=t|four]]
  * [[http://docs.oracle.com/javase/specs/jls/se7/html/jls-5.html|Java Type Conversion Rules]]
  * [[https://msdn.microsoft.com/en-us/library/y5b434w4.aspx|C# Type Conversion Rules]]

===== Changelog =====

  * v0.5.3 Change version target back and add line about bypassing function execution on type error in strict mode
  * v0.5.2 Change version target
  * v0.5.1 Remove aliases from proposal
  * v0.5 Fork from Andrea's original proposal. Change declare behavior. Add int->float (primitive type widening).