
====== PHP RFC: Return Type Declarations ======
  * Version: 2.0
  * Date: 2014-03-20
  * Author: Levi Morrison <levim@php.net>
  * Status: Implemented (PHP 7.0)
  * First Published at: https://wiki.php.net/rfc/returntypehinting
  * Migrated to: https://wiki.php.net/rfc/return_types

===== Introduction =====
Many developers would like to be able to declare the return type of a function. The basic idea of declaring a return type has been included in at least three RFCs and has been discussed in a few other places (see [[#references]]). This RFC proposes a different approach from previous RFC's to accomplish this goal in a simple way.

Declaring return types has several motivators and use-cases:
  * Prevent sub-types from breaking the expected return type of the super-type((See [[#variance_and_signature_validation|Variance and Signature Validation]] and [[#examples]] for more details on how this works)), especially in interfaces
  * Prevent unintended return values
  * Document return type information in a way that is not easily invalidated (unlike comments)

===== Proposal =====
This proposal adds an optional return type declaration to function declarations including closures, functions, generators,  and methods. This RFC does not change the existing type declarations nor does it add new ones (see [[#differences_from_past_rfcs|differences from past RFCs]]).

Here is a brief example of the syntax in action:
<PHP>
function foo(): array {
    return [];
}
</PHP>
More examples can be found in the [[#examples|Examples]] section.

//Code which does not declare a return type will continue to work// exactly as it currently does. This RFC requires a return type to be declared only when a method inherits from a parent method that declares a return type; in all other cases it may be omitted.

==== Variance and Signature Validation ====
The enforcement of the declared return type during inheritance is invariant; this means that when a sub-type overrides a parent method then the return type of the child must exactly match the parent and may not be omitted. If the parent does not declare a return type then the child is allowed to declare one.

If a mismatch is detected during compile time (e.g. a class improperly overriding a return type) then ''E_COMPILE_ERROR'' will be issued. If a type mismatch is detected when the function returns then ''E_RECOVERABLE_ERROR'' will be issued.

Covariant return types are considered to be type sound and are used in many other languages((C++, Java and others use covariant return types.)). This RFC originally proposed covariant return types but was changed to invariant because of a few issues. It is possible to add covariant return types at some point in the future.

Note that this topic of variance is about the declared return type of the function; this means that the following would be valid for either invariant or covariant return types:

<PHP>
interface A {
    static function make(): A;
}
class B implements A {
    static function make(): A {
        return new B();
    }
}
</PHP>

The class ''B'' implements ''A'' so it is therefore valid. Variance is about the allowed types when overriding the declared types:

<PHP>
interface A {
    static function make(): A;
}
class B implements A {
    static function make(): B { // must exactly match parent; this will error
        return new B();
    }
}
</PHP>

The above sample does not work because this RFC proposes only invariant return types; this could be extended in the future to be allowed.

==== Position of Type Declaration ====
The two major conventions in other programming languages for placing return type information are:

  * Before the function name
  * After the parameter list's closing parenthesis

The former position has been proposed in the past and the RFCs were either declined or withdrawn. One cited issue is that many developers wanted to preserve the ability to search for <php>function foo</php> to be able to find the definition for ''foo''. A recent discussion about [[http://marc.info/?t=141235344900003&r=1&w=2|removing the function keyword]] has several comments that re-emphasized the value in preserving this.

The latter position is used in several languages(([[http://hacklang.org/|Hack]],  [[http://www.haskell.org|Haskell]], [[https://golang.org/|Go]], [[http://www.erlang.org/|Erlang]], [[http://www.adobe.com/devnet/actionscript.html|ActionScript]], [[http://www.typescriptlang.org/|TypeScript]] and more all put the return type after the parameter list)); notably C++11 also places the return type after the parameter lists for certain constructs such as lambdas and auto-deducing return types.

Declaring the return type after the parameter list had no shift/reduce conflicts in the parser.

==== Returning by Reference ====

This RFC does not change the location of ''&'' when returning by reference. The following examples are valid:
<PHP>
function &array_sort(array &$data) {
    return $data;
}

function &array_sort(array &$data): array {
    return $data;
}
</PHP>

==== Disallowing NULL on Return Types ====
Consider the following function:

<PHP>
function foo(): DateTime {
    return null; // invalid
}
</PHP>

It declares that it will return ''DateTime'' but returns ''null''; this type of situation is common in many languages including PHP. By design this RFC does not allow ''null'' to be returned in this situation for two reasons:

  - This aligns with current parameter type behavior. When parameters have a type declared, a value of ''null'' is not allowed ((Except when the parameter has a ''null'' default)).
  - Allowing ''null'' by default works against the purpose of type declarations. Type declarations make it easier to reason about the surrounding code. If ''null'' was allowed the programmer would always have to worry about the ''null'' case.

The [[rfc:nullable_types|Nullable Types RFC]] addresses this shortcoming and more.

==== Methods which cannot declare return types ====

Class constructors, destructors and clone methods may not declare return types. Their respective error messages are:

  * ''<nowiki>Fatal error: Constructor %s::%s() cannot declare a return type in %s on line %s</nowiki>''
  * ''<nowiki>Fatal error: Destructor %s::__destruct() cannot declare a return type in %s on line %s</nowiki>''
  * ''<nowiki>Fatal error: %s::__clone() cannot declare a return type in %s on line %s</nowiki>''

==== Examples ====
Here are some snippets of both valid and invalid usage.

=== Examples of Valid Use ===
<PHP>
// Overriding a method that did not have a return type:
interface Comment {}
interface CommentsIterator extends Iterator {
    function current(): Comment;
}
</PHP>
<PHP>
// Using a generator:

interface Collection extends IteratorAggregate {
    function getIterator(): Iterator;
}

class SomeCollection implements Collection {
    function getIterator(): Iterator {
        foreach ($this->data as $key => $value) {
            yield $key => $value;
        }
    }
}
</PHP>

=== Examples of Invalid Use ===

The error messages are taken from the current patch.
----

<PHP>
// Covariant return-type:

interface Collection {
    function map(callable $fn): Collection;
}

interface Set extends Collection {
    function map(callable $fn): Set;
}
</PHP>
''Fatal error: Declaration of Set::map() must be compatible with Collection::map(callable $fn): Collection in %s on line %d''
----
<PHP>
// Returned type does not match the type declaration

function get_config(): array {
    return 42;
}
get_config();
</PHP>
''Catchable fatal error: Return value of get_config() must be of the type array, integer returned in %s on line %d''

----

<PHP>
// Int is not a valid type declaration

function answer(): int {
    return 42;
}
answer();
</PHP>
''Catchable fatal error: Return value of answer() must be an instance of int, integer returned in %s on line %d''

----

<PHP>
// Cannot return null with a return type declaration

function foo(): DateTime {
    return null;
}
foo();
</PHP>
''Catchable fatal error: Return value of foo() must be an instance of DateTime, null returned in %s on line %d''

----

<PHP>
// Missing return type on override

class User {}

interface UserGateway {
    function find($id): User;
}

class UserGateway_MySql implements UserGateway {
    // must return User or subtype of User
    function find($id) {
        return new User();
    }
}
</PHP>
''Fatal error: Declaration of UserGateway_MySql::find() must be compatible with UserGateway::find($id): User in %s on line %d''

----

<PHP>
// Generator return types can only be declared as Generator, Iterator or Traversable (compile time check)

function foo(): array {
    yield [];
}
</PHP>
''Fatal error: Generators may only declare a return type of Generator, Iterator or Traversable, %s is not permitted in %s on line %d''


==== Multiple Return Types ====
This proposal specifically does not allow declaring multiple return types; this is out of the scope of this RFC and would require a separate RFC if desired.

If you want to use multiple return types in the meantime, simply omit a return type declaration and rely on PHP's excellent dynamic nature.

==== Reflection ====

This RFC purposefully omits reflection support as there is an open RFC about improving type information in reflection: https://wiki.php.net/rfc/reflectionparameter.typehint

==== Differences from Past RFCs ====
This proposal differs from past RFCs in several key ways:

  * **The return type is positioned after the parameter list.** See [[#position_of_type_declaration|Position of Type Declaration]] for more information about this decision.
  * **We keep the current type options.** Past proposals have suggested new types such as ''void'', ''int'', ''string'' or ''scalar''; this RFC does not include any new types. Note that it does allow ''self'' and ''parent'' to be used as return types.
  * **We keep the current search patterns.** You can still search for <php>function foo</php> to find <php>foo</php>'s definition; all previous RFCs broke this common workflow.
  * **We allow return type declarations on all function types**. Will Fitch's proposal suggested that we allow it for methods only.
  * **We do not modify or add keywords.** Past RFCs have proposed new keywords such as ''nullable'' and more. We still require the <php>function</php> keyword.

===== Other Impact =====

==== On Backward Compatiblity ====
This RFC is backwards compatible with previous PHP releases.

==== On SAPIs ====
There is no impact on any SAPI.

==== On Existing Extensions =====
The structs ''zend_function'' and ''zend_op_array'' have been changed; extensions that work directly with these structs may be impacted.

==== On Performance ====
An informal test indicates that performance has not seriously degraded. More formal performance testing can be done before voting phase.

===== Proposed PHP Version(s) =====
This RFC targets PHP 7.

===== Vote =====
This RFC modifies the PHP language syntax and therefore requires a two-third majority of votes.

Should return types as outlined in this RFC be added to the PHP language? Voting will end on January 23, 2015.
<doodle title="Typed Returns" auth="levim" voteType="single" closed="true">
   * Yes
   * No
</doodle>

===== Patches and Tests =====

Dmitry and I have updated the implementation to a more current master branch here: https://github.com/php/php-src/pull/997

This RFC was merged into the master branch (PHP 7) in commit [[https://git.php.net/?p=php-src.git;a=commit;h=638d0cb7531525201e00577d5a77f1da3f84811e|638d0cb7531525201e00577d5a77f1da3f84811e]].

===== Future Work =====
Ideas for future work which are out of the scope of this RFC include:

  * Allow functions to declare that they do not return anything at all (''void'' in Java and C)
  * Allow nullable types (such as <php>?DateTime</php>). This is discussed in the [[rfc:nullable_types|Nullable Types]] RFC.
  * Improve parameter variance. Currently parameter types are invariant while they could be contravariant. Change the E_STRICT on mismatching parameter types to E_COMPILE_ERROR.
  * Improve runtime performance by doing type analysis.
  * Update documentation to use the new return type syntax.

===== References =====
  * [[rfc:returntypehint2|Method Return Type-hints]] by Will Fitch; 2011. [[http://marc.info/?t=132443368800001&r=1&w=2|Mail Archive]].
  * [[rfc:returntypehint|Return Type-hint]] by Felipe; 2010. [[http://marc.info/?l=php-internals&m=128036818909738&w=2|Mail Archive]]
  * [[rfc:typehint|Return value and parameter type hint]] by Felipe; 2008. [[http://marc.info/?l=php-internals&m=120753976214848&w=2|Mail Archive]].
  * [[http://derickrethans.nl/files/meeting-notes.html#type-hinted-properties-and-return-values|Type-hinted properties and return values]] from meeting notes in Paris; Nov 2005.

In the meeting in Paris on November 2005 it was decided that PHP should have return type declarations and some suggestions were made for syntax. Suggestion 5 is nearly compatible with this RFC; however, it requires the addition of a new token ''T_RETURNS''. This RFC opted for a syntax that does not require additional tokens so ''returns'' was replaced by a colon.

The following (tiny) patch would allow the syntax in suggestion 5 to be used alongside the current syntax. This RFC does not propose that both versions of syntax should be used; the patch just shows how similar this RFC is to that suggestion from 2005.

https://gist.github.com/krakjoe/f54f6ba37e3eeab5f705

===== Changelog =====

  * v1.1: Target PHP 7 instead of PHP 5.7
  * v1.2: Disallow return types for constructors, destructors and clone methods.
  * v1.3: Rework Reflection support to use new ''ReflectionType'' class
  * v1.3.1: Rename ''ReflectionType::IS_''* constants to ''TYPE_''*, rename ''->getKind()'' to ''->getTypeConstant()''
  * v2.0: Change to invariant return types and omit reflection support