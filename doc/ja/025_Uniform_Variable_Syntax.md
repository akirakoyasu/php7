====== PHP RFC: Uniform Variable Syntax ======
  * Date: 2014-05-31
  * Author: Nikita Popov <nikic@php.net>
  * Status: Implemented (in PHP 7)
  * Discussion: http://markmail.org/message/mr4ihbubfbdxygci

===== Introduction =====

This RFC proposes the introduction of an internally consistent and complete variable syntax. To achieve this goal the
semantics of some rarely used variable-variable constructions need to be changed.

Examples of expressions that were previously invalid, but will be valid with the uniform variable syntax:

<code php>
// support missing combinations of operations
$foo()['bar']()
[$obj1, $obj2][0]->prop
getStr(){0}

// support nested ::
$foo['bar']::$baz
$foo::$bar::$baz
$foo->bar()::baz()

// support nested ()
foo()()
$foo->bar()()
Foo::bar()()
$foo()()

// support operations on arbitrary (...) expressions
(...)['foo']
(...)->foo
(...)->foo()
(...)::$foo
(...)::foo()
(...)()

// two more practical examples for the last point
(function() { ... })()
($obj->closure)()

// support all operations on dereferencable scalars (not very useful)
"string"->toLower()
[$obj, 'method']()
'Foo'::$bar
</code>

Example of expressions those meaning changes:

<code php>
                        // old meaning            // new meaning
$$foo['bar']['baz']     ${$foo['bar']['baz']}     ($$foo)['bar']['baz']
$foo->$bar['baz']       $foo->{$bar['baz']}       ($foo->$bar)['baz']
$foo->$bar['baz']()     $foo->{$bar['baz']}()     ($foo->$bar)['baz']()
Foo::$bar['baz']()      Foo::{$bar['baz']}()      (Foo::$bar)['baz']()
</code>

Examples of statements which are no longer supported:

<code php>
global $$foo->bar;
// instead use:
global ${$foo->bar};
</code>

===== Issues with the current syntax =====

==== Root cause ====

The root cause for most issues in PHP's current variable syntax are the semantics of the variable-variable syntax
''%%$$foo['bar']%%''. Namely this expression is interpreted as ''%%${$foo['bar']}%%'' (lookup the variable with the name
''%%$foo['bar']%%'') rather than ''%%${$foo}['bar']%%'' (take the ''%%'bar'%%'' offset of the ''%%$$foo%%'' variable).

Why this choice of semantics is problematic to our parser design is explained in the next section. Before getting to
that, I will discuss why this choice is also bad from a language design perspective:

Normally variable accesses are interpreted from left to right. ''%%$foo['bar']->baz%%'' will first fetch a variable
named ''%%$foo%%'', then will take the ''%%'bar%%''' offset of the result and finally access the ''%%baz%%'' property.
The ''%%$$foo['baz']%%'' syntax goes against that basic principle. Rather than fetching ''%%$$foo%%'' and then taking
its ''%%'baz'%%'' offset, it will first fetch ''%%$foo%%'', fetch its ''%%'baz'%%'' offset and then look up a variable
with the name of the result.

This combination of an indirect reference and an offset is the **only** case where the interpretation is inverted. For
example the very similar ''%%$$foo->bar%%'' will be interpreted as ''%%${$foo}->bar%%'' and not as ''%%${$foo->bar}%%''.
It follows normal left-to-right semantics. Similarly ''%%$$foo::$bar%%'' is also interpreted as ''%%${$foo}::$bar%%''
and not as ''%%${$foo::$bar}%%''.

To ensure maximum possible inconsistency there exists an exception to this rule. Namely ''%%global $$foo->bar%%''
**will** be interpreted as ''%%global ${$foo->bar}%%'', even though this is not the case for normal variable accesses.

This issue applies not only to simple indirect references, but also to indirected property and method names. For example
''%%Foo::$bar[1][2][3]%%'' is interpreted as an access to the static property ''%%Foo::$bar%%'', followed by fetches of
the 1, 2 and 3 offsets. On the other hand ''%%Foo::$bar[1][2][3]()%%'' (notice the parentheses at the end) has an
entirely different interpretation: This does not call the function stored at ''%%Foo::$bar[1][2][3]%%''. Instead it
calls the static method of class ''%%Foo%%'' with name ''%%$bar[1][2][3]%%''.

The last issue implies that PHP's variable syntax is non-local. It is not possible to parse a PHP variable access with
a fixed finite lookahead, without transplanting the generated syntax tree or instructions after the fact.

==== Impact on parser definition ====

In addition to the problems described above the semantics for indirect references also have far-reaching consequences on
how the variable syntax is defined in our parser. In the following I will outline the kind of issues it causes, for
readers not familiar with parser construction:

The "standard" approach to defining a variable syntax for a LALR parser is to create a left-recursive ''%%variable%%''
rule, which could look roughly as follows (somewhat simplified):

<code>
variable:
        T_VARIABLE                            /* $foo */
    |   variable '[' expr ']'                 /* variable['bar'] */
    |   variable '->' T_STRING                /* variable->baz */
    |   variable '->' T_STRING '(' params ')' /* variable->oof() */
    |   variable '::' T_VARIABLE              /* variable::$rab */
    |   ...
;
</code>

This approach ensures that we can arbitrarily nest different access types (in the following called "dereferencing"). For
example the above definition allows you to write ''%%$foo['bar']->baz->oof()::$rab%%''. This expression is grouped from
left-to-right, i.e. it is interpreted as ''%%(((($foo)['bar'])->baz)->oof())::$rab%%''.

What happens to this scheme if we add a (right-associative) ''%%$$foo['bar']%%'' syntax? One might think that it could be
defined as follows:

<code>
reference_variable:
        T_VARIABLE
    |   reference_variable '[' expr ']'
;

variable:
        T_VARIABLE
    |   '$' reference_variable
    |   variable '[' expr ']'
    |   variable '->' T_STRING
    |   ...
;
</code>

However, this is not possible because it makes the grammar ambiguous. When the parser encounters ''%%$$foo['bar']%%'' it
could either interpret it using the ''%%'$' reference_variable%%'' rule (i.e. ''%%${$foo['bar']}%%'' semantics) or using
the ''%%variable '[' expr ']'%%'' rule (i.e. ''%%${$foo}['bar']%%'' semantics). This kind of issue is called a
"shift/reduce conflict".

How can this issue be resolved? By removing the ''%%variable '[' expr ']'%%'' rule. However, if this rule is removed,
you can no longer write ''%%$foo->bar['baz']%%'' etc either. As such offset access needs to be implemented anew for all
other dereferencing types: You need to implement it for ''%%$foo->bar%%'', for ''%%$foo->bar()%%'' and for
''%%$foo::$bar%%''. Furthermore you need to ensure that you can continue to nest arbitrary types of dereferences after
that.

This is both extremely complicated and fragile. This is the reason why PHP only introduced the ''%%foo()['bar']%%''
syntax in PHP 5.4 and even then the support is not perfect.

==== Incomplete dereferencing support ====

Because of the implementational hurdles described in the previous section, we do not support all combinations of
dereferencing operations to an arbitrary depth. While PHP 5.4 fixed the most glaring issue (support for
''%%$foo->bar()['baz']%%''), other problems still exist.

Basically, there are two classes of issues. The first one is that we do not always properly support nesting of different
dereferencing types. For example, while it is possible to write both ''%%$foo()['bar']%%'' and ''%%$foo['bar']()%%'',
the combination ''%%$foo()['bar']()%%'' results in a parse error. Another example is that the constant dereferencing
syntax implemented in PHP 5.5 allows you to write ''%%[$obj1, $obj2][0]%%'', but ''%%[$obj1, $obj2][0]->prop%%'' is not
possible. Yet another example is that the alternative array syntax ''%%$str{0}%%'' is not supported on function calls,
i.e. ''%%getStr(){0}%%'' is not valid.

The second class of issues is that some nesting types aren't supported altogether. For example ''%%::%%'' only accepts
simple reference variables on the left hand side. Writing something like ''%%$info['class']::${$info['property']}%%'' is
not possible. Writing ''%%getFunction()()%%'' is not possible either. The ''%%(function() { ... })()%%'' pattern that is
familiar from JavaScript is not allowed as well.

Lack of support for dereferencing parenthesis-expressions also prevents you from properly disambiguating some
expressions. For example, it is a common problem that ''%%$foo->bar()%%'' will always try to call the ''%%bar()%%''
method, rather than calling the closure stored in ''%%$foo->bar%%''. However writing ''%%($foo->bar)()%%'' is not
possible and you need to use a temporary variable. Another example is the case of ''%%Foo::$bar[1][2][3]()%%'' from
above (which is interpreted as ''%%Foo::{$bar[1][2][3]}()%%''). It is currently not possible to force the alternative
behavior ''%%(Foo::$bar[1][2][3])()%%''.

==== Miscellaneous other issues ====

=== Behavior in non-read context ===

The new ''%%(new Foo)['bar']%%'' and ''%%[...]['bar']%%'' syntaxes introduced in PHP 5.4 and PHP 5.5 were implemented as
"non-variable expressions". This means that they are always compiled in "read context", even when they are used in a different context.

For example ''%%empty(['foo' => 42]['bar'])%%'' will generate an "Undefined index" notice, even though ''%%empty()%%'' usually suppresses such warnings. The reason behind this is that proper behavior in isset/empty requires compilation using ''BP_VAR_IS'' rather than ''BP_VAR_R''.

This also means that assignments to dereferences of parenthesis-expressions are never allowed, even when they would be technically possible. E.g. it's not possible to write ''%%(new Foo)['bar'] = 42%%''. Whether this would be particularly useful (the assignment would only be visible through ''offsetSet'') is another question, but it's not much different than writing something like ''%%foo()['bar'] = 42%%'', which is allowed.

=== Superfluous CVs on static property access ===

Upon encountering a static property access ''%%Foo::$bar%%'' PHP will currently emit a compiled variable (CV) for
''%%$bar%%'', even though it is not necessary and never used. This is once again related to the way static member access
needs to be implemented to support our weird indirect reference semantics.

===== Proposal =====

==== Formal definition ====

A formal definition of the new variable syntax is provided in Bison syntax. This is a slightly simplified version of the
grammer used in the actual implementation. Furthermore definitions for ''%%function_name%%'', ''%%class_name%%'',
''%%expr%%'', ''%%function_call_parameter_list%%'' and ''%%array_pair_list%%'' have been omitted.

<code>
variable:
		callable_variable
	|	class_name_or_dereferencable T_PAAMAYIM_NEKUDOTAYIM simple_variable
	|	dereferencable T_OBJECT_OPERATOR member_name
;

callable_variable:
		simple_variable
	|	dereferencable '[' dim_offset ']'
	|	dereferencable '{' expr '}'

	|	function_name function_call_parameter_list
	|	dereferencable T_OBJECT_OPERATOR member_name function_call_parameter_list
	|	class_name_or_dereferencable T_PAAMAYIM_NEKUDOTAYIM member_name function_call_parameter_list
	|	callable_expr function_call_parameter_list
;

simple_variable:
		T_VARIABLE
	|	'$' '{' expr '}'
	|	'$' simple_variable
;

dereferencable:
		variable
	|	'(' expr ')'
	|	dereferencable_scalar
;

dereferencable_scalar:
		T_ARRAY '(' array_pair_list ')'
	|	'[' array_pair_list ']'
	|	T_CONSTANT_ENCAPSED_STRING
;

class_name_or_dereferencable:
        class_name
    |   dereferencable
;

member_name:
		T_STRING
	|	'{' expr '}'
	|	simple_variable
;

dim_offset:
		/* empty */
	|	expr
;

callable_expr:
		callable_variable
	|	'(' expr ')'
	|	dereferencable_scalar
;
</code>

==== Semantic differences in existing syntax ====

The main difference to the existing variable syntax, is that indirect variable, property and method references are now
interpreted with left-to-right semantics. Examples:

<code>
$$foo['bar']['baz']   interpreted as   ($$foo)['bar']['baz']
$foo->$bar['baz']     interpreted as   ($foo->$bar)['baz']
$foo->$bar['baz']()   interpreted as   ($foo->$bar)['baz']()
Foo::$bar['baz']()    interpreted as   (Foo::$bar)['baz']()
</code>

This change is **backwards incompatible** (with low practical impact), which is the reason why this RFC targets PHP 7.
However it is always possible to recreate the old behavior by explicitly using braces:

<code php>
${$foo['bar']['baz']}
Foo::{$bar['baz']}()
$foo->{$bar['baz']}()
</code>

This syntax will have guaranteed same behavior in both PHP 5 and PHP 7.

==== Newly added and generalized syntax ====

  * There are no longer any restrictions on nesting of dereferencing operations. In particular the examples ''%%$foo()['bar']()%%'', ''%%[$obj1, $obj2][0]->prop%%'' and ''%%getStr(){0}%%'' from the previous sections are all supported now.
  * Static property fetches and method calls can now be applied to any dereferencable expression. E.g. ''%%$foo['bar']::$baz%%'', ''%%$foo::$bar::$baz%%'' and ''%%$foo->bar()::baz()%%'' are all valid now.
  * The result of a call can now be directly called again, i.e. all of ''%%foo()()%%'', ''%%$foo->bar()()%%'', ''%%Foo::bar()()%%'' and ''%%$foo()()%%'' are valid now.
  * All dereferencing operations can now be applied to arbitrary parenthesis-expressions. I.e. all of ''%%(...)['foo']%%'', ''%%(...)->foo%%'', ''%%(...)->foo()%%'', ''%%(...)::$foo%%'', ''%%(...)::foo()%%'' and ''%%(...)()%%'' are supported now. In particular this also includes ''%%(function() { ... })()%%''.
  * All dereferencing operations can now be applied to dereferencable scalars (array and string literals as of PHP 5.5). E.g. it is possible to write ''%%"string"->toLower()%%'', ''[$obj, 'method']()'' and ''%%'Foo'::$bar%%''. Note that these don't necessarily make sense by themselves, but the syntax is supported nonetheless. Extensions can then use it to implement the actual behavior for something like ''%%"string"->toLower()%%''.

==== Global keyword takes only simple variables ====

Previously the ''%%global%%'' keyword accepted variables of the form ''%%global $$foo->bar%%'', where an arbitrary
variable could follow after the ''$'' character. This is inconsistent with the usual variable syntax.

The ''%%global%%'' keyword now only accepts simple variables. This means that you need to write
''%%global ${$foo->bar}%%'' instead. ''%%global $$foo->bar%%'' will result in a parse error.

==== Behavior in write context ====

Expressions of type ''%%(...)[42]%%'' and ''%%[...][42]%%'' (and other cases where some dereferencing operation is
applied to a non-variable) are now parsed as a ''%%variable%%'' (rather than ''%%expr_without_variable%%'', as it was
previously).

This means that the expression will behave correctly in ''%%empty()%%''. E.g. ''%%empty([]['a'])%%'' will no longer
throw an undefined offset notice. Furthermore it is now possible to assign to expressions of this kind, for example
''%%(new Foo)->bar = 'baz'%%'' is now possible (thought doesn't make much sense unless ''%%Foo%%'' implements
''%%__set%%'').

However assignment is not allowed if the left hand expression yields an ''%%IS_CONST%%'' or ''%%IS_TMP_VAR%%'' operand.
For example the expression ''%%'foo'[0] = 'b'%%'' will result in a "Cannot use temporary expression in write context"
compile error. For ''%%BP_VAR_FUNC_ARG%%'' fetches a runtime fatal error is thrown instead.

==== Class name variable for new expression ====

It has always been possible to create classes using a dynamically specified class name by writing ''%%new $className()%%'' or similar.
However the supported variables are more limited in this case: They may not include calls anywhere, as this would cause
ambiguity with the constructor parameter list. New variables are now defined as follows:

<code>
new_variable:
        simple_variable
    |   new_variable '[' dim_offset ']'
    |   new_variable '{' expr '}'
    |   new_variable T_OBJECT_OPERATOR member_name
    |   class_name T_PAAMAYIM_NEKUDOTAYIM simple_variable
    |   new_variable T_PAAMAYIM_NEKUDOTAYIM simple_variable
;
</code>

This matches the previously allowed variable expressions, with the minor extension of allowing chaining of ''%%::%%''.
For example ''%%new $foo['bar']::$baz%%'' would now be possible.

===== Backward Incompatible Changes =====

The changes described in [[#semantic_differences_in_existing_syntax|Semantic differences in existing syntax]] and
[[#global_keyword_takes_only_simple_variables|Global keyword takes only simple variables]] are backwards compatibility
breaks.

The former is a change in the behavior of currently existing syntax. Examples:

<code php>
                        // old meaning            // new meaning
$$foo['bar']['baz']     ${$foo['bar']['baz']}     ($$foo)['bar']['baz']
$foo->$bar['baz']       $foo->{$bar['baz']}       ($foo->$bar)['baz']
$foo->$bar['baz']()     $foo->{$bar['baz']}()     ($foo->$bar)['baz']()
Foo::$bar['baz']()      Foo::{$bar['baz']}()      (Foo::$bar)['baz']()
</code>

An analysis of the Zend Framework and Symfony projects (including standard dependencies) showed that only a single
occurrence of ''%%$loader[0]::$loader[1]($className)%%'' in the Doctrine class loader will be affected by this change.
This occurrence must be replaced with ''%%$loader[0]::{$loader[1]}($className)%%'' to achieve compatibility with both
PHP 5 and PHP 7.

The latter change turns currently valid syntax into a parse error. Expressions like ''%%global $$foo->bar%%'' are no
longer valid and ''%%global ${$foo->bar}%%'' must be used instead.

As these changes only apply to some very rarely used syntax, the breakage seems acceptable for PHP 7.

===== Open issues =====

The current patch introduces a new "write context" issue. Namely ''%%($foo)['bar'] = 'baz'%%'' will not behave this same
was as ''%%$foo['bar'] = 'baz'%%''. In the former case an undefined variable notice will be thrown if ''%%$foo%%'' does
not exist, whereas the latter does not throw a notice.

The reason for this is that ''%%(...)%%'' is always interpreted as an expression, and not a variable. This means that
the part in parentheses will always be compiled in "read context", even though write context is desired.

This issue already exists currently, in a different context: The expression ''%%byRef(func())%%'' will throw a strict
standards notice, if ''%%byRef()%%'' accepts the first argument by-reference, but ''%%func()%%'' does not return by
reference. On the other hand ''%%byRef((func()))%%'' will throw no such notice, because ''%%(func())%%'' is recognized
as an expression instead of a variable.

===== Patch =====

An implementation of this proposal against the phpng branch is available at https://github.com/php/php-src/pull/686.

The main changes are limited to the language parser and compiler. Furthermore some opcode handlers had to be modified
to support ''CONST'' and ''TMP'' operands.

===== Vote =====

As this is a language change, a 2/3 majority is required for acceptance. The vote started on 2014-07-07 and ended on 2014-07-14.

<doodle title="Implement Uniform Variable Syntax in PHP 6?" auth="nikic" voteType="single" closed="true">
   * Yes
   * No
</doodle>
