====== PHP RFC: Exceptions in the engine (for PHP 7) ======
  * Date: 2014-09-30
  * Author: Nikita Popov <nikic@php.net>
  * Status: Implemented (in PHP 7.0)

===== Introduction =====

This RFC proposes to allow the use of exceptions in the engine and to allow the replacement of existing fatal or recoverable fatal errors with exceptions.

As an example of this change, consider the following code-snippet:

<code php>
function call_method($obj) {
    $obj->method();
}

call_method(null); // oops!
</code>

Currently the above code will throw a fatal error ((Since [[rfc:catchable-call-to-member-of-non-object]] it throws a recoverable fatal error instead, so this particular example no longer applies in pre-release builds of PHP)):

<code>
Fatal error: Call to a member function method() on a non-object in /path/file.php on line 4
</code>

This RFC replaces the fatal error with an ''EngineException''. As such it is now possible to catch this error condition:

<code php>
try {
    call_method(null); // oops!
} catch (EngineException $e) {
    echo "Exception: {$e->getMessage()}\n";
}

// Exception: Call to a member function method() on a non-object
</code>

If the exception is not caught, PHP will continue to throw the same fatal error as it currently does.

===== Motivation =====

==== Summary of current error model ====

PHP currently supports 16 different error types which are listed below, grouped by severity:

<code>
// Fatal errors
E_ERROR
E_CORE_ERROR
E_COMPILE_ERROR
E_USER_ERROR

// Recoverable fatal errors
E_RECOVERABLE_ERROR
// Parse error
E_PARSE

// Warnings
E_WARNING
E_CORE_WARNING
E_COMPILE_WARNING
E_USER_WARNING

// Notices etc.
E_DEPRECATED
E_USER_DEPRECATED
E_NOTICE
E_USER_NOTICE
E_STRICT
</code>

The first four errors are fatal, i.e. they will not invoke the error handler, abort execution in the current context and directly jump (bailout) to the shutdown procedure.

The ''E_RECOVERABLE_ERROR'' error type behaves like a fatal error by default, but it will invoke the error handler, which can instruct the engine to ignore the error and continue execution in the context where the error was raised.

The ''E_PARSE'' error normally behaves like a fatal error (e.g. when ''include'' is used). However when ''eval()'' is used the bailout is not performed (the error handler is still skipped though).

The remaining errors are all non-fatal, i.e. execution continues normally after they occur. The error handler is invoked for all error types apart from ''E_CORE_WARNING'' and ''E_COMPILE_WARNING''.

==== Issues with fatal errors ====

=== Cannot be gracefully handled ===

The most obvious issue with fatal errors is that they immediately abort execution and as such cannot be gracefully recovered from. This behavior is very problematic in some situations.

As an example consider a server or daemon written in PHP. If a fatal error occurs during the handling of a request it will abort not only that individual request but kill the entire server/daemon. It would be much preferable to catch the fatal error and abort the request it originated from, but continue to handle other requests.

Another example is running tests in PHPUnit: If a test throws a fatal error this will abort the whole test-run. It would be more desirable to mark the individual test as failed, but continue running the rest of the testsuite.

=== Error handler is not called ===

Fatal errors do not invoke the error handler and as such it is hard to apply custom error handling procedures (for display, logging, mailing, ...) to them. The only way to handle a fatal error is through a shutdown function:

<code php>
register_shutdown_function(function() { var_dump(error_get_last()); });

$null = null;
$null->foo();

// shutdown function output:
array(4) {
  ["type"]=> int(1)
  ["message"]=> string(47) "Call to a member function foo() on a non-object"
  ["file"]=> ...
  ["line"]=> ...
}
</code>

This allows rudimentary handling of fatal errors, but the available information is very limited. In particular the shutdown function is not able to retrieve a stacktrace for the error (which is possible for other error types going through the error handler.)

=== Finally blocks will not be invoked ===

If a fatal error occurs ''finally'' blocks will not be invoked:

<code php>
$lock->acquire();
try {
    doSomething();
} finally {
    $lock->release();
}
</code>

If ''doSomething()'' in the above example results in a fatal error the ''finally'' block will not be run and the lock is not released.

=== Destructors are not called ===

When a fatal error occurs destructors are not invoked. This means that anything relying on the RAII (Resource Acquisition Is Initialization) will break. Using the lock example again:

<code php>
class LockManager {
    private $lock;
    public function __construct(Lock $lock) {
        $this->lock = $lock;
        $this->lock->acquire();
    }
    public function __destruct() {
        $this->lock->release();
    }
}

function test($lock) {
    $manager = new LockManager($lock); // acquire lock

    doSomething();

    // automatically release lock via dtor
}
</code>

If ''doSomething()'' in the above example throws a fatal error the destructor of ''LockManager'' is not called and as such the lock is not released.

As both ''finally'' blocks and destructors fail in face of fatal errors the only reasonably robust way of releasing critical resources is to use a global registry combined with a shutdown function.

==== Issues with recoverable fatal errors ====

After acknowledging that the use of fatal errors is problematic, one might suggest to convert fatal errors to recoverable fatal errors where possible. Sadly this also has several issues:

=== Execution is continued in same context ===

When a recoverable fatal error is dismissed by a custom error handler, execution is continued as if the error never happened. From a core developer perspective this means that a recoverable fatal error needs to be implemented in the same way as a warning is, with the assumption that the following code will still be run.

This makes it technically complicated to convert fatal errors into recoverable errors, because fatal errors are typically thrown in situation where continuing execution in the current codepath is not possible. For example the use of recoverable errors in argument sending would likely require manual stack and call slot cleanup as well as figuring out which code to run after the error.

=== Hard to catch ===

While ''E_RECOVERABLE_ERROR'' is presented as a "Catchable fatal error" to the end user, the error is actually rather hard to catch. In particular the familiar ''try''/''catch'' structure cannot be used and instead an error handler needs to be employed.

To catch a recoverable fatal error non-intrusively code along the following lines is necessary:

<code php>
set_error_handler(function($errno, $errstr, $errfile, $errline) {
    if ($errno === E_RECOVERABLE_ERROR) {
        throw new ErrorException($errstr, $errno, 0, $errfile, $errline);
    }
    return false;
});

try {
    new Closure;
} catch (Exception $e) {
    echo "Caught: {$e->getMessage()}\n";
}

restore_error_handler();
</code>

=== Performance ===

The ability to bypass recoverable fatal errors while still continuing execution in the same code path may cause performance issues in some cases. For example it is currently possible to completely ignore argument type hints with an error handler. As such a JIT compiler may not be able to make strong assumptions about the types of type hinted parameters.

==== Solution: Exceptions ====

Exceptions provide an approach to error handling that does not suffer from the problems of fatal and recoverable fatal errors. In particular exceptions can be gracefully handled, they will invoke ''finally'' blocks and destructors and are easily caught using ''catch'' blocks.

From an implementational point of view they also form a middle ground between fatal errors (abort execution) and recoverable fatal errors (continue in the same codepath). Exceptions typically leave the current codepath right away and make use of automatic cleanup mechanisms (e.g. there is no need to manually clean up the stack). In order to throw an exception from the VM you usually only need to free the opcode operands and invoke ''HANDLE_EXCEPTION()''.

Exceptions have the additional advantage of providing a stack trace.

===== Proposal =====

This proposal introduces two new exception types:

  * ''EngineException'' as the recommended default exception type for exceptions emitted from the executor.
  * ''ParseException'' for use with parse errors in particular.

Additionally the following policy changes are made:

  * It is now allowed to use exceptions in the engine.
  * Existing errors of type ''E_ERROR'', ''E_RECOVERABLE_ERROR'', ''E_PARSE'' or ''E_COMPILE_ERROR'' can be converted to exceptions.
  * It is discouraged to introduce new errors of type ''E_ERROR'' or ''E_RECOVERABLE_ERROR''. Within limits of technical feasibility the use of exceptions is preferred.

In order to avoid updating many tests the current error messages will be retained if the engine/parse exception is not caught. This may be changed in the future.

The patch attached to this RFC already converts a large number of fatal and recoverable fatal errors to exceptions. It also converts parse errors to exceptions.

==== Hierarchy ====

There is some concern that by extending ''EngineException'' directly from ''Exception'', previously fatal errors may be accidentally caught by existing ''catch(Exception $e)'' blocks (aka Pokemon exception handling). To alleviate this concern it is possible to introduce a new ''BaseException'' type with the following inheritance hierarchy:

<code>
BaseException (abstract)
 +- EngineException
 +- ParseException
 +- Exception
     +- ErrorException
     +- RuntimeException
         +- ...
     +- ...
</code>

As such engine/parse exceptions will not be caught by existing ''catch(Exception $e)'' blocks.

Whether such a hierarchy (with a new ''BaseException'' type) should be adopted will be subject to a secondary vote.

===== Potential issues =====

==== E_RECOVERABLE_ERROR compatibility ====

Currently it is possible to silently ignore recoverable fatal errors with a custom error handler. By replacing them with exceptions this capability is removed, thus breaking compatibility.

I have never seen this possibility used in practice outside some weird hacks (which use ignored recoverable type constraint errors to implement scalar typehints). In most cases custom error handlers throw an ''ErrorException'', i.e. they emulate the proposed behavior with a different exception type.

==== E_PARSE compatibility ====

Currently parse errors generated during ''eval()'' (but not ''require'' etc) are non-fatal. This proposal would make ''eval()'' throw an exception instead, which would require some code adjustments in cases where the developer wishes to gracefully handle ''eval()'' errors.

As parse errors do not invoke the error handler, handling eval errors is tricky and requires code looking roughly as follows  ((This does not handle all cases, but should give you a rough idea of how eval errors are handled)):

<code php>
set_error_handler(function() { return false; }, 0);
@$undefinedVariable;
restore_error_handler();

$oldErrorReporting = error_reporting();
error_reporting($oldErrorReporting & ~E_PARSE);
$result = eval($code);
error_reporting($oldErrorReporting);

$error = error_get_last();
if ($result === false && $error['type'] === E_PARSE) {
    // Handle $error
}
</code>

After this RFC errors should be handled as follows instead:

<code php>
try {
    $result = eval($code);
} catch (\ParseException $exception) {
    // Handle $exception
}
</code>

==== Not all errors converted ====

The Zend Engine currently (master on 2014-09-30) contains the following number of fatal-y errors:

<code>
E_ERROR:            182    (note: not counting 636 occurrences in zend_vm_execute.h)
E_CORE_ERROR:        12
E_COMPILE_ERROR:    146
E_PARSE:              1
E_RECOVERABLE_ERROR: 17
</code>

The count was obtained using ''git grep "error[^(]*(E_ERROR_TYPE" Zend | wc -l'' and as such may not be totally accurate, but should be a good approximation.

The patch attached to the RFC currently (as of 2014-09-30) removes 75 ''E_ERROR''s, 13 ''E_RECOVERABLE_ERROR''s and the one ''E_PARSE'' error. While I hope to port more errors to exceptions before the patch is merged, the process is rather time consuming and I will not be able to convert all errors. (Note: The number of occurrences in the source code says rather little about what percentage of "actually thrown" errors this constitutes.)

Some errors are easy to change to exceptions, others are more complicated. Some are impossible, like the memory limit or execution time limit errors. The ''E_CORE_ERROR'' type can't be converted to use exceptions because it occurs during startup (at least if used correctly). Converting ''E_COMPILE_ERROR'' to exceptions would also require some significant changes to the compiler.

This means that there will always be some truly fatal errors and to a userland developer the distinction between what results in an exception and what in a fatal error may be non-obvious. I don't think that this is really a problem. Not being able to make everything an exception is no reason to avoid exceptions in the cases where they //can// be used.

===== Backwards compatibility =====

The ''E_ERROR'' portion of this proposal does not break backwards compatibility: All code that was previously working, will continue to work. The change only relaxes error conditions, which is generally not regarded as breaking BC.

The ''E_RECOVERABLE_ERROR'' part of the proposal may introduce a minor BC break, because it will no longer allow to silently ignore recoverable errors with a custom error handler. As this point is somewhat controversial I'll have a separate voting option for this.

The ''E_PARSE'' part of the proposal may introduce a minor BC break, because ''E_PARSE'' exhibits non-fatal behavior when used with ''eval()''.

===== Patch =====

A patch for this RFC is available at https://github.com/php/php-src/pull/1095.

The patch introduces basic infrastructure for this change, replaces ''E_PARSE'' with ''ParseException'' and a number of ''E_ERROR'' and ''E_RECOVERABLE_ERROR'' with ''EngineException''.

To simplify porting to exceptions it is possible to throw engine exceptions by passing an additional ''E_EXCEPTION'' flag to ''zend_error'' (instead of using ''zend_throw_exception''). To free unfetched operands in the VM the ''FREE_UNFETCHED_OPn'' pseudo-macros are introduced. An example of the necessary change to port a fatal error to an exception:

<code>
- zend_error_noreturn(E_ERROR, "Cannot pass parameter %d by reference", opline->op2.num);
+ zend_error(E_EXCEPTION | E_ERROR, "Cannot pass parameter %d by reference", opline->op2.num);
+ FREE_UNFETCHED_OP1();
+ HANDLE_EXCEPTION();
</code>

The current patch continues using "Recoverable fatal error" messages even for errors that are not recoverable anymore in the previous sense of the word. These messages will be adjusted afterwards. (The patch tries to separate important functional changes from cosmetic tweaks.)

===== Vote =====

The primary vote decides if the proposal as outlined in the RFC is accepted and requires a 2/3 majority.

<doodle title="Allow exceptions in the engine and conversion of existing fatals?" auth="nikic" voteType="single" closed="true">
   * Yes
   * No
</doodle>

The secondary vote decides whether or not a ''BaseException'' based hierarchy should be introduced, as described in the [[#hierarchy|Hierarchy]] section of the proposal. Whichever option has more votes wins.

<doodle title="Introduce and use BaseException?" auth="nikic" voteType="single" closed="true">
   * Yes
   * No
</doodle>

Voting started on 2015-02-23 and ends on 2015-03-08.