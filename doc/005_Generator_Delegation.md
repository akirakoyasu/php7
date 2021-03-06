====== PHP RFC: Generator Delegation ======
  * Version: 0.2.0
  * Date: 2015-03-01
  * Author: Daniel Lowrey <rdlowrey@php.net>
  * Contributors: Bob Weinand <bwoebi@php.net>
  * Status: Implemented in 7.0
  * First Published at: http://wiki.php.net/rfc/generator-delegation

====== Abstract ======

This RFC proposes new ''yield from <expr>'' syntax allowing ''Generator'' functions to delegate operations to ''Traversable'' objects and arrays. The proposed syntax allows the factoring of ''yield'' statements into smaller conceptual units in the same way that discrete class methods simplify object-oriented code. The proposal is conceptually related to and requires functionality proposed by the forerunning [[rfc:generator-return-expressions|Generator Return Expressions]] RFC.

====== Proposal ======

The following new syntax is allowed in the body of generator functions:

<code php>
    yield from <expr>
</code>

In the above code ''<expr>'' is any expression that evaluates to a ''Traversable'' object or array. This
traversable is advanced until no longer valid, during which time it sends/receives values directly
to/from the delegating generator's caller. If the ''<expr>'' traversable is a ''Generator'' it is
considered a subgenerator whose eventual return value is returned to the delegating generator as the
result of the originating ''yield from'' expression.

==== Terminology ====

  * A "delegating generator" is a ''Generator'' in which the ''yield from <expr>'' syntax appears.

  * A "subgenerator" is a ''Generator'' used in the ''<expr>'' portion of the ''yield from <expr>'' syntax.

==== Prosaically ====

  * Each value yielded by the traversable is passed directly to the delegating generator's caller.

  * Each value sent to the delegating generator's ''send()'' method is passed to the subgenerator's ''send()'' method. If the delegate traversable is not a generator any sent values are ignored as non-generator traversables have no capacity to receive such values.

  * Exceptions thrown by traversable/subgenerator advancement are propagated up the chain to the delegating generator.

  * Upon traversable completion ''null'' is returned to the delegating generator if the traversable is NOT a generator. If the traversable is a generator (subgenerator) its return value is sent to the delegating generator as the value of the ''yield from'' expression.

==== Formally ====

The proposed syntax

<code php>
$g = function() {
    return yield from <expr>;
};
</code>

is equivalent to

<code php>
$g = function() {
    $iter = <expr>;
    $isSubgenerator = $iter instanceof Generator;
    $received = null;
    $send = true;

    while ($iter->valid()) {
        if ($isSubgenerator) {
            $next = $send ? $iter->send($received) : $iter->throw($received);
        } else {
            $next = $iter->current();
            $iter->next();
        }
        try {
            $received = yield $next;
            $send = true;
        } catch (Exception $e) {
            if ($isSubgenerator) {
                $received = $e;
                $send = false;
            } else {
                throw $e;
            }
        }
    }

    return $isSubgenerator ? $iter->getReturn() : null;
};
</code>


===== Rationale =====

A major impetus for generator delegation is refactoring and readability. At its core, this is
the same guiding principle employed when returning values from discrete class methods. Imagine a
class method from which no return value is possible. In such a scenario we //could// store the result
in an instance property and subsequently retrieve it from the context of the calling code. However,
this kind of superfluous state quickly becomes difficult to reason about.

Additionally, we recognize that functional contexts lack the additional stateful context in which
to store results. In the absence of the standard input-output paradigm there's no way to access the
eventual result of a generator's pausable computations. Of course, it //is// possible to work around this
suboptimal situation with references and closure ''use'' binding but these indirect approaches are
instantly eliminated when Generator functions are allowed to return expressions. Return values minimize
cognitive overhead in such cases by allowing programmers to directly associate an individual operation with
its eventual result.

Generator delegation -- at its heart -- is nothing more than the application of standard factoring practices
to allow the decomposition of complex operations into smaller cohesive units.

==== Use-Case: Factored Generator Computations ====

In this simple example we demonstrate the use of ''yield from'' to factor out a more complex operation
into multiple discrete generators. Callers of ''myGeneratorFunction'' do not care from whence the
individual yielded values came (nor should they). Instead, they simply iterate over the yielded values
awaiting the generator function's eventual return.

<code php>
    function myGeneratorFunction($foo) {
        // ... do some stuff with $foo ...
        $bar = yield from factoredComputation1($foo);
        // ... do some stuff with $bar ...
        $baz = yield from factoredComputation2($bar);

        return $baz;
    }
    function factoredComputation1($foo) {
        yield ...; // pseudo-code (something we factored out)
        yield ...; // pseudo-code (something we factored out)
        return 'zanzibar';
    }
    function factoredComputation2($bar) {
        yield ...; // pseudo-code (something we factored out)
        yield ...; // pseudo-code (something we factored out)
        return 42;
    }
</code>


==== Use-Case: Generators as Lightweight Threads ===

The defining feature of Generator functions is their support for suspending execution
for later resumption. This capability gives applications a mechanism to
implement asynchronous and concurrent architectures even in a traditionally single-threaded language
like PHP. With simple userland task scheduling systems interleaved generators become lightweight
threads of execution for concurrent processing tasks.

In the absence of generator return values, though, applications face an environment where "background"
tasks can be offloaded without a standardized way to return the eventual result.
This is one reason why this proposal depends on the acceptance of the Generator Return Expressions RFC.
The other reason return values are required stems from the previously discussed refactoring principle.
Specifically: code using generators for threaded execution can benefit from subgenerators behaving
like ordinary functions.

Using the proposed syntax an ordinary function ''foo''

''$baz = foo($bar);''

can be transformed into a subgenerator delegation of the form

''$baz = yield from foo($bar);''

where ''foo'' is a pausable generator. In this manner applications can create powerful userland concurrency
abstractions without the cognitive overhead often associated with threaded multitasking. In the
above example generator delegation allows the language to do the heavy lifting while the programmer
need only concern herself with the input, ''$bar'', and the eventual output, ''$baz''.

In short: generator delegation allows programmers to reason about the behaviour of the concurrent code
simply by thinking of ''foo()'' as an ordinary function which can be suspended using a ''yield'' statement.

**NB:** The actual implementation of coroutine task schedulers is outside the scope of
this document. This RFC focuses only on the language-level machinery needed to make such tools more
feasible in userland. It should be obvious that simply moving code into a generator function will
not somehow make it magically concurrent.


===== Basic Examples =====

Delegating to another generator (subgenerator)

<code php>
<?php

function g1() {
  yield 2;
  yield 3;
  yield 4;
}

function g2() {
  yield 1;
  yield from g1();
  yield 5;
}

$g = g2();
foreach ($g as $yielded) {
    var_dump($yielded);
}

/*
int(1)
int(2)
int(3)
int(4)
int(5)
*/
</code>

Delegating to an array

<code php>
<?php

function g() {
  yield 1;
  yield from [2, 3, 4];
  yield 5;
}

$g = g();
foreach ($g as $yielded) {
    var_dump($yielded);
}

/*
int(1)
int(2)
int(3)
int(4)
int(5)
*/
</code>

Delegating to non-generator traversables

<code php>
<?php

function g() {
  yield 1;
  yield from new ArrayIterator([2, 3, 4]);
  yield 5;
}

$g = g();
foreach ($g as $yielded) {
    var_dump($yielded);
}

/*
int(1)
int(2)
int(3)
int(4)
int(5)
*/
</code>

The ''yield from'' expression value

<code php>
<?php

function g1() {
  yield 2;
  yield 3;
  return 42;
}

function g2() {
  yield 1;
  $g1result = yield from g1();
  yield 4;
  return $g1result;
}

$g = g2();
foreach ($g as $yielded) {
    var_dump($yielded);
}
var_dump($g->getReturn());

/*
int(1)
int(2)
int(3)
int(4)
int(42)
*/
</code>



===== Selected Implementation Details =====

The delegation implementation builds on the patch submitted as part of the Generator Return Expressions RFC.
This implementation adds the following new parsing token:

''%token T_YIELD_FROM   "yield from (T_YIELD_FROM)"''

The primary advantage of this approach is the addition of a readable and semantically meaningful syntax
without reserving a new ''from'' keyword.


==== Subgenerator Keys ====

As Generator iteration is always associated with an accompanying key there exists the potential that
a given delegation may return the same key multiple times from separate individual generators. This
was deemed unproblematic for three primary reasons.

  * Multiple occurrences of the same key can easily occur in the existing generator implementation should a function ''yield'' the same key more than once.

  * If a caller derives semantic meaning from yielded keys the burden is placed on generator value producers to yield keys sensible for problem domain in which they exist. The burden here is not on the language to avoid duplicate keys.

  * The ''Traversable'' interface represents neither a hash map nor a traditional contiguously indexed array and its implementations have no special requirement to expose data access to API consumers via index keys. As such, key recurrence exposes no risk for overwriting existing internal generator data.


==== Shared Subgenerator Behavior ====

PHP generator functions are implemented as stateful object instances. Although "sharing" valid generator
instances does not present any immediately obvious use-cases, such behavior is still supported. Here
we note some of the characteristics of shared generator functions.

If a "shared" subgenerator that has previously iterated to completion is passed in a ''yield from''
expression its completed return value is immediately returned to the delegating generator. In code:

<code php>
function subgenerator() {
    yield 1;
    return 42;
}
function delegator(Generator $shared) {
    return yield from $shared;
}

$shared = subgenerator();
while($shared->valid()) { $shared->next(); }
$delegator = delegator($shared);

foreach ($delegator as $value) {
    var_dump($value);
}
var_dump($delegator->getReturn());

/*
int(42)
// This is our only output because no values are yielded
// from the already-completed shared subgenerator
*/
</code>


Manually advancing a shared subgenerator outside the context of the delegating generator will not
result in an error. In code:

<code php>
function subgenerator() {
    yield 1;
    yield 2;
    yield 3;
    yield 4;
    return 42;
}
function delegator(Generator $shared) {
    return yield from $shared;
}

$shared = subgenerator();
$shared->next();
$delegator = delegator($shared);
var_dump($delegator->current());
$shared->next();
while($delegator->valid()) {
    var_dump($delegator->current());
    $delegator->next();
}
var_dump($delegator->getReturn());

/*
int(2);
int(3);
int(4);
int(42)
*/
</code>

==== Error States ====

There are two scenarios in which ''yield from'' usage can result in an ''EngineException'':

  * Using ''yield from <expr>'' where <expr> evaluates to a generator which previously terminated with an uncaught exception results in an ''EngineException''.

  * Using ''yield from <expr>'' where <expr> evaluates to something that is neither ''Traversable'' nor an array throws an ''EngineException''.

===== Rejected Ideas =====

The original version of this RFC proposed a ''yield *'' syntax. The ''yield *'' syntax was rejected in favor of
''yield from'' on the basis that ''*'' would break backwards compatibility. Additionally, the ''yield *''
form was considered less readable than the current proposal.


===== Criticisms =====

It has been suggested during the discussion phase that a mechanism other than ''return'' be used in
subgenerators to establish the value returned to delegating generators by ''yield from'' expressions.
The following counter-arguments are provided to this criticism:

  * Forms other than ''return'' would undermine the proposal's stated goal of conceptualizing subgenerators as suspendable functions. Implementing a different syntax would inhibit this understanding by fostering the idea of generators as being something other than "real" functions.

  * ''return'' expressions have applicable semantics, known characteristics and low cognitive overhead.

  * Other popular dynamic languages inhabiting a similar space to PHP implement generator delegation value resolution using the ''return'' syntax proposed here. Sharing common vernacular for similar features lowers cognitive barriers for developers coming to PHP from diverse backgrounds.


===== Other Languages =====

Other popular dynamic languages currently support variants of the proposed syntax ...

**Python**

Python 3.3 generators support the ''yield from'' syntax:

<code python>
>>> def accumulate():
...     tally = 0
...     while 1:
...         next = yield
...         if next is None:
...             return tally
...         tally += next
...
>>> def gather_tallies(tallies):
...     while 1:
...         tally = yield from accumulate()
...         tallies.append(tally)
...
>>> tallies = []
>>> acc = gather_tallies(tallies)
>>> next(acc) # Ensure the accumulator is ready to accept values
>>> for i in range(4):
...     acc.send(i)
...
>>> acc.send(None) # Finish the first tally
>>> for i in range(5):
...     acc.send(i)
...
>>> acc.send(None) # Finish the second tally
>>> tallies
[6, 10]
</code>

**JavaScript**

Javascript ES6 generators support the ''yield*'' syntax:

<code javascript>
function* g4() {
  yield* [1, 2, 3];
  return "foo";
}

var result;

function* g5() {
  result = yield* g4();
}

var iterator = g5();

console.log(iterator.next()); // { value: 1, done: false }
console.log(iterator.next()); // { value: 2, done: false }
console.log(iterator.next()); // { value: 3, done: false }
console.log(iterator.next()); // { value: undefined, done: true },
                              // g4() returned { value: "foo", done: true } at this point

console.log(result);          // "foo"
</code>

===== Backward Incompatible Changes =====

None

===== Proposed PHP Version(s) =====

PHP7

===== Unaffected PHP Functionality =====

Existing generator semantics are unaffected.


===== Vote =====

A 2/3 "Yes" vote is required to implement this proposal. Voting will continue through March 29, 2015.

<doodle title="Allow Generator delegation in PHP7" auth="rdlowrey" voteType="single" closed="true">
   * Yes
   * No
</doodle>

.

The success of this vote depends on the success of the accompanying [[rfc:generator-return-expressions|Generator Return Expressions]] RFC. Should Generator Return Expressions be rejected the voting outcome of this RFC will be rendered moot.

===== Patches and Tests =====

The current patch is considered "final" and can be found here:

https://github.com/bwoebi/php-src/commits/coroutineDelegation

The patch was written by Bob Weinand and is based upon the implementation branch written by Nikita Popov
for the [[rfc:generator-return-expressions|Generator Return Expressions]] RFC. Extensive .phpt tests
exist in the implementation branch and readers are encouraged both to compile with the proposed
implementation and read the test cases to ascertain the full nature of the proposal.

===== Implementation =====

TBD

===== References =====

  * [[rfc:generator-return-expressions|Generator Return Expressions RFC]]

  * [[https://docs.python.org/3/whatsnew/3.3.html#pep-380|New in Python 3.3: yield from]]

  * [[https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/yield*|JavaScript yield* Syntax]]


===== Changelog =====

  * v0.2.0 Moved to ''yield from'' instead of ''yield *''+ massive textual additions
  * v0.1.0 Initial proposal