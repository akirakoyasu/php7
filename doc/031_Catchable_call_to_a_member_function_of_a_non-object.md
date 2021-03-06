
====== PHP RFC: Catchable "call to a member function of a non-object" ======
  * Version: 1.1
  * Date: 2014-04-26
  * Author: Timm Friebe <thekid@php.net>
  * Status: Accepted
  * First Published at: https://wiki.php.net/rfc/catchable-call-to-member-of-non-object

**Note: This RFC has been superseded by the [[https://wiki.php.net/rfc/engine_exceptions_for_php7|engine exceptions]] proposal.**

===== Introduction =====
One of the most common fatal errors in PHP is the "call to a member function of a non-object" type. This occurs whenever a method is called on anything other than an object (usually null), e.g.:

<PHP>
// ...when getAction() returns null:
$this->getAction()->invoke();
</PHP>

One situation in which fatal errors are problematic is if you want to run PHP as a webserver itself. For a long story on why you would want to do that in the first place, see http://marcjschmidt.de/blog/2014/02/08/php-high-performance.html.

Other situtations are described in the [[https://wiki.php.net/rfc/engine_exceptions#issues_with_fatal_errors|Engine Exceptions RFC]].

===== Proposal =====
This proposal's essence is to turns fatal errors generated from calls to methods on a non-object into ''E_RECOVERABLE_ERROR''s.

<PHP>
set_error_handler(function($code, $message) {
  var_dump($code, $message);
});

$x= null;
var_dump($x->method());
echo "Alive\n";
</PHP>

The above produces the following output:

  int(4096)
  string(50) "Call to a member function method() on a non-object"
  NULL
  Alive

==== Consistency ====
This behavior is consistent with how type hints work. Framework authors can turn this into exceptions if they wish.

==== Example: Exceptions ====
The following error handler could be embedded into frameworks:

<PHP>
set_error_handler(function($code, $message) {
  if (0 === strncmp('Call', $message, 4)) {
    throw new BadMethodCallException($message);
  } else if (0 === strncmp('Argument', $message, 8)) {
    throw new InvalidArgumentException($message);
  } else {
    trigger_error($message, E_USER_ERROR);
  }
}, E_RECOVERABLE_ERROR);

$x= null;
try {
  $x->method();
} catch (BadMethodCallException $e) {
  echo "Caught expected ", $e->getMessage(), "!\n";
}
echo "Alive\n";
</PHP>

==== Example: Without exceptions ====
This could be a way for people preferring not to use exceptions and instead to exit the script directly, but get a stacktrace instead of just the fatal error message:

<PHP>
set_error_handler(function($code, $message) {
  echo "*** Error #$code: $message\n";
  debug_print_backtrace();
  exit(0xFF);
}, E_RECOVERABLE_ERROR);

$m= new some_db_model();
$row= $m->find(42); // null, deleted concurrently
$row->delete();
</PHP>

==== Differences from Past RFCs ====
This proposal doesn't go as far as the controversial RFC [[engine_exceptions|RFC: Engine exceptions]].

==== Inner workings ====

Taken this code:

<PHP>
function a($comparator) {
  $result= $comparator->compare(1, 2);
  // ...
}
</PHP>

You can unwind this to something like:

  function a($comparator) {
    1: ZEND_INIT_METHOD_CALL $comparator 'compare'
    2: ZEND_SEND_VAL         1
    3: ZEND_SEND_VAL         2
    4: ZEND_DO_FCALL_BY_NAME
    5: ZEND_ASSIGN           $result
    // ...
  }

The handling on checking whether the method is callable happends inside
the opline #1 (''ZEND_INIT_METHOD_CALL''). The opcode handler for it
checks its first argument (here: ''$comparator'') whether it is an object
or not. In case it's not, the following happens:

  - A recoverable error is triggered. Should the script exit here, there's nothing more to be done, this is just as it was before.
  - In case the script continues, all the oplines are skipped until we find the ''FCALL_BY_NAME'' opcode. This is comparable to jumping forward just as, e.g., the ''ZEND_JMP'' instruction would.
  - The return value is set to ''ZVAL_NULL()''.
  - The control is handed back to the executor, which then continues with the ''ASSIGN'' opcode
  - The engine is again in full control; if an exception was raised by the handler, that leads to the known behavior.

===== Other Impact =====

==== On Backward Compatiblity ====
This RFC is backwards compatible with previous PHP releases.

==== On SAPIs ====
There is no impact on any SAPI.

==== On Existing Extensions =====
No impact.

==== On Performance ====
No effect, before, the script terminated.

===== Proposed PHP Version(s) =====
This RFC targets PHP 5.7 or PHP 6.0, whichever comes first.

===== Proposed Voting Choices =====
This RFC modifies the PHP language behaviour and therefore requires a two-third majority of votes.

===== Patches and Tests =====
There is a pull request available over at [[https://github.com/php/php-src/pull/647|GitHub]] which includes tests. Feedback welcome!

===== Future Work =====
Ideas for future work include:

  * Also allowing to catch and handle other fatal errors

===== Vote =====
Voting started 2014-06-29 and ended 2014-07-30.

<doodle title="Catchable Call to a member function bar() on a non-object" auth="thekid" voteType="single" closed="true">
   * Yes
   * No
</doodle>


===== References =====
  * PHP Bugs [[https://bugs.php.net/bug.php?id=46601|46601]], [[https://bugs.php.net/bug.php?id=51882|51882]] and [[https://bugs.php.net/bug.php?id=51848|51848]]- bugs which would be fixed by this
  * PHP Bug [[https://bugs.php.net/bug.php?id=54195|54195]] - related, motivates necessity
  * HHVM [[https://github.com/facebook/hhvm/blob/master/hphp/test/quick/method-non-object.php.expectf|throws a BadMethodCallException]] in these situations
  * [[http://news.php.net/php.internals/73814|Mailing list announcement]]
  * [[rfc/returntypehinting|Return type hinting RFC]] - related work: Some fatal errors can be prevented by setting return types.