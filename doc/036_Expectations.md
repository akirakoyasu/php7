====== PHP RFC: Expectations ======

  * Version: 2
  * Date: 2013-10-18
  * Author: Joe <krakjoe@php.net>, Dmitry <dmitry@php.net>
  * Status: Implemented (in PHP 7.0)
  * First Published at: http://wiki.php.net/rfc/expectations

===== Introduction =====
The **assertion** statement has the prototype:

<code>
void assert (mixed $expression [, mixed $message]);
</code>

At execution time, **expression** will be evaluated, if the result is false, an **AssertionException** will be thrown.

In some cases, **expression** will be an expensive evaluation that you do not wish to execute in a production environment, **assertions** can therefore be disabled and enabled via the **PHP_INI_ALL** configuration setting **zend.assertions**. Disabling assertions will almost entirely eliminate the performance penalty making them equivalent to an empty statement.

In any case, **assertions** should never be used to perform tasks **required** for the code to function, nor should they change the internal state of any object //except where that state is used only by other assertions//, these are not rules that are enforced by Zend, but are nonetheless the best rules to follow.

If an object of a class which extends **AssertionException** is used for **message**, it will be thrown if the **assertion** fails, any other expression will be used as the **message** for the **AssertionException**. If no **message** is provided, the statement will be used as the **message** in **AssertionException**.

If **expression** is a constant string, compatibility with the old API is employed, the string is compiled and used as the expression.
===== Scope of Assertions =====

PHP programmers tend to document how their code is supposed to work in comments, this is a fine approach for generating automated documentation, but leaves us a little bewildered, and tired of digging through documentation at runtime when things go wrong:

<code php>
    if ($i % 3 == 0) {
        ...
    } else if ($i % 3 == 1) {
        ...
    } else { // We know ($i % 3 == 2)
        ...
    }
</code>

Becomes:

<code php>
    if ($i % 3 == 0) {
        ...
    } else if ($i % 3 == 1) {
        ...
    } else {
        assert ($i % 3 == 2);
    }
</code>

In a development environment, this forces the executor to make you aware of your mistake.

Another good example for using **assertions** might be a switch block with no default case:

<code php>
switch ($suit) {
    case CLUBS:
        /* ... */
    break;

    case DIAMONDS:
        /* ... */
    break;

    case HEARTS:
        /* ... */
    break;

    case SPADES:
        /* ... */
    break;
}
</code>

The above switch assumes that **suit** can only be one of four values, to test this assumption add the default case:

<code php>
switch ($suit) {
    case CLUBS:
        /* ... */
    break;

    case DIAMONDS:
        /* ... */
    break;

    case HEARTS:
        /* ... */
    break;

    case SPADES:
        /* ... */
    break;

    default:
        assert (false, "Unrecognized suit passed through switch: {$suit}");
}
</code>

The previous example highlights another general area where you should use assertions: **place an assertion at any location you assume will not be reached**. The statement to use is:

<code php>
assert(false);
</code>

Suppose you have a method that looks like:

<code php>
public function method() {
    for (/*...*/) {

        if (/* ... */)
           return true;
    }

}
</code>

The above code assumes that one of the iterations results in a return value being passed back to the caller of **::method()**, to test this assumption:

<code php>
public function method() {
    for (/*...*/) {

        if (/* ... */)
           return true;
    }
    assert(false);
}
</code>

**Assertions** allow the possibility to perform //precondition// and //postcondition// checks:

<code php>
public function setResponseCode($code) {
    $this->code = $code;
}
</code>

Becomes:

<code php>
public function setResponseCode($code) {
    assert($code < 550 && $code > 100, "Invalid response code provided: {$code}");

    $this->code = $code;
}
</code>

The example above performs a //precondition// check on the **code** parameter.

The same kind of logic can be applied to internal object state:

<code php>
public function getResponseCode() {
    assert($this->code,"The response code is not yet set");

    return $this->code;
}
</code>

//postcondition// checks might also be carried out with assert:

<code php>
public function getNext() {
    $data = $this->data[++$this->next];

    assert(preg_match("~^([a-zA-Z0-9-]+)$~", $data["key"]),
        "malformed key found at {$this->next} \"{$data["key"]}\"");

    return $data;
}
</code>

The above method during development would be verbose, not allowing the programmer to make a mistake, while during production where assertions should be disabled, it is //fast//.
===== Managing Failed Assertions =====

When an **assertion** fails, an **AssertionException** is thrown, these can be caught in the normal way, and come with a stack trace and a useful message about the **assertions**. An **AssertionException** extends **ErrorException** and has a **severity** of **E_ERROR**.

The primary purpose of throwing an exception, rather than emitting an error, is an exception comes with a stack trace, this is especially useful during development and or debugging.

You can provide custom exceptions for failed **assertions**:

<code php>
<?php
$next = 1;
$data = array(
    "key" => "X-HTTP ",
    "value" => "testing"
);

class HeaderMalfunctionException extends AssertionException {}

/* ... */
public function getData() {
    /* ... */
    assert(preg_match("~^([a-zA-Z0-9-]+)$~", $data["key"]),
        new HeaderMalfunctionException("malformed key found at {$next} \"{$data["key"]}\""));
    /* ... */
}
/* ... */
?>
</code>

This further improves the stack trace at a glance as well as provides opportunity to structure exceptions (as part of documentation, for example), and secondarily provides the ability for the programmer to catch exceptions by name during development.

//The programmer should never deploy (to production environments) catch blocks for **AssertionExceptions**, as these cannot be removed when assertions are disabled by configuration.//

**The ability to throw custom exceptions is to be a voting option.**

===== Performance =====

**zend.assertions** is a three way switch:

   1 - generate and execute code (development mode)
   0 - generate code and jump around at it at runtime
  -1 - don't generate any code (zero-cost, production mode)

===== Namespaced assert ======

A call to **assert()**, without a fully qualified namespace will call **assert** in the current namespace, if the function exists. An unqualified call to assert is subject to the same optimization configured by **zend.assertions**.

Calling **\assert()** will always invoke the system function.

===== Production Time =====

**Assertions** are a debugging and development feature; the programmer should not take code to production with catch blocks to manage **AssertionExceptions**; the ability to manage the **AssertionExceptions** exists //during development// in order to aid the programmer in debugging the exception, //the only place where it can be raised//.

Library code should not shy away from deploying **Assertions** //everywhere//, use it to literally assert what your code **expects**, rigorously, such that during development the programmer is made aware of every possible mistake //before// production arrives.

This means production library code does not have to manage inconsistencies in usage, because there should, theoretically, be none left; improving it's performance in production by not making those unnecessary checks that stem from inconsistent or incorrect usage.

//prefix everything here with "when deployed and configured properly"//
===== Backward Incompatible Changes =====

This API replaces the old assertion API in a compatible manner.

===== Proposed PHP Version(s) =====

I don't know why this section is suggested since the process is always the same; we vote on merging into master and RM's decide if they will merge into their release.

===== Impact to Existing Extensions =====

None that are obvious (or not taken care of by the patch), this does introduce a new opcode so anything working with opcodes may need adjustment.

Optimizer is impacted, and patched.

===== php.ini =====

  * zend.assertions
  * assert.exception

Two new settings are required to control the new assertion API; The reason for this is that to retain compatibility with the old assert API we need to have an error reporting mode that does not use exceptions.

Exceptions are the superior means of reporting and displaying the error to the programmer, since they come with stack trace information, invaluable for debugging.

Assertions should be enabled (**zend.assertions=1**) on development machines, and disabled (**zend.assertions=0**) in production.

Exceptions should be enabled (**assert.exception=1**) on development machines.

These defaults can be set in the development and production ini files we distribute.

The hardcoded values are:

  * zend.assertions=1
  * assert.exception=0

zend.assertions is an INI_SYSTEM setting, allowing for the safe removal of assertion opcodes.

assert.exception is an INI_ALL setting, allowing for exceptions to be disabled at runtime.

===== Open Issues =====

It has been suggested that **AssertionException** should not extend **Exception**, such that the following code does not catch **AssertionException**:

<code>
try {
    functionUsingAssertAndFailing(10);
} catch(Exception $ex) {
    /* deal with $ex, catches AssertionException */
}
</code>

Right now, we do not have any such exceptions.

The Engine Exceptions RFC deals with introducing a new exception tree, we will wait for that RFC to go ahead before changing the parentage of **AssertionException** if it passes.

===== Unaffected PHP Functionality =====

The current assertion API is unaffected by this addition.

===== Patches and Tests =====

https://github.com/php/php-src/pull/1088

This is a working implementation of Assertions as documented here, with some appropriate tests.

===== References =====

Link to the original internals thread that discussed the proposal to replace assert with new functionality:

http://php.markmail.org/search/?q=net.php.lists.internals+order%3Adate-backward+assertions#query:net.php.lists.internals%20order%3Adate-backward%20assertions+page:1+mid:nxbxke2z5oykztys+state:results

Link to the internals thread discussing this particular RFC:

http://php.markmail.org/search/?q=net.php.lists.internals+expectations+order%3Arelevance#query:net.php.lists.internals%20expectations%20order%3Arelevance+page:1+mid:krr72ib3jwghrc4a+state:results

Link to the earliest bug report I can find requesting this feature:

https://bugs.php.net/bug.php?id=13725

===== Other Languages =====

    Java: http://docs.oracle.com/javase/1.4.2/docs/guide/lang/assert.html
        assert expression : message; evaluates Expression1 and if it is false throws an AssertionError with no detail message, takes message to constructor of AssertionError if present.

.NET (or this implementation for .NET) does not directly result in an exception, more like an exception in a message box, the important part is; it includes the call stack.

    .NET: http://msdn.microsoft.com/en-us/library/system.diagnostics.debug.assert.aspx
        Debug.Assert(expression, message): Checks for a condition; if the condition is false, outputs a specified message and displays a message box that shows the call stack.

Python's implementation is similar to **Assertions** also, but limited

    Python: http://docs.python.org/2/reference/simple_stmts.html
        assert expression raise AssertionError

Javascript has no standard implementation, yet; various implementations exist all the same:

    Chrome: https://developers.google.com/chrome-developer-tools/docs/console-api#consoleassertexpression_object
        console.assert(expression, object): If the specified expression is false, the message is written to the console along with a stack trace.

    Firefox (firebug): http://getfirebug.com/wiki/index.php/Console_API
        console.assert(expression[, object, ...]): Tests that an expression is true. If not, it will write a message to the console and throw an exception.

    Node.js: http://nodejs.org/api/stdio.html#stdio_console_assert_expression_message
        console.assert(expression, [message]): Same as assert.ok() where if the expression evaluates as false throw an AssertionError with message.

These implementations at least include a stack trace; a benefit of using exceptions for failed **Assertions** is that the stack trace is present by default.

===== Vote =====

<doodle title="Merge changes into master?" auth="krakjoe" voteType="single" closed="true">
   * Yes, with custom exceptions
   * Yes, without custom exceptions
   * No
</doodle>

===== Merge =====

The patch was merged with support for custom exceptions: http://git.php.net/?p=php-src.git;a=commitdiff;h=9a20323e1946dff57eae8cd054e0893aefe83092

===== Rejected Features =====

N/A