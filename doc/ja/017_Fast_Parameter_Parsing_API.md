====== PHP RFC: Fast Parameter Parsing API ======
  * Version: 0.9
  * Date: 2014-05-23
  * Author: Dmitry Stogov <dmitry@zend.com>, Bob Weinand <bobwei9@hotmail.com>
  * Status: Implemented (PHP 7.0)
  * First Published at: http://wiki.php.net/rfc/fast_zpp

===== Introduction =====
PHP internal functions use zend_parse_parameters() API to receive values of actual parameters into C variables. This function uses scanf() like approach for parameter definition. The number and types of required data defined by a string that contains a list of specifiers ("s" - for string, "l" for long, etc).
Unfortunately, this string has to be parsed each time when php calls this function, and this makes significant performance overhead.

For example zend_parse_parameters() takes ~6% of CPU time in serving wordpress home page.

For some really simple function like is_string() or ord() the overhead of zend_parse_parameters() may be about 90%.

===== Proposal =====
We propose an additional fast API for parameter parsing, that should be used for the most useful functions. The API is based on C macros that lead to inlining of optimized code directly into the body of internal function.

**We don't propose to remove the existing API, and would suggest to use fast API only for most often used functions to get performance boost.**

I'll explain API on the following example:

<code>
if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "al|zb", &input, &offset, &z_length, &preserve_keys) == FAILURE) {
	return;
}
</code>
<code>
ZEND_PARSE_PARAMETERS_START(2, 4)
	Z_PARAM_ARRAY(input)
	Z_PARAM_LONG(offset)
	Z_PARAM_OPTIONAL
	Z_PARAM_ZVAL(z_length)
	Z_PARAM_BOOL(preserve_keys)
ZEND_PARSE_PARAMETERS_END();
</code>

The first code fragment is just taken from PHP_FUNCTION(array_slice), the second is its replacement using new API. The code is actually self explainable.

ZEND_PARSE_PARAMETERS_START() takes two arguments minimal and maximal parameters count.

Z_PARAM_ARRAY() - takes the next argument as array, Z_PARAM_LONG - as long, Z_PARAM_OPTIONAL - tells that the remaining arguments are optional.

The new API covers all possibilities of the existing API. The following table shows the correspondence between old specifiers and new macros.

^ specifier ^ Fast ZPP API macro                ^ args ^
| ''|''     | Z_PARAM_OPTIONAL                  ||
| a         | Z_PARAM_ARRAY(dest)               | dest - zval* |
| A         | Z_PARAM_ARRAY_OR_OBJECT(dest)     | dest - zval* |
| b         | Z_PARAM_BOOL(dest)                | dest - zend_bool |
| C         | Z_PARAM_CLASS(dest)               | dest - zend_class_entry* |
| d         | Z_PARAM_DOUBLE(dest)              | dest - double |
| f         | Z_PARAM_FUNC(fci, fcc)            | fci - zend_fcall_info, fcc - zend_fcall_info_cache |
| h         | Z_PARAM_ARRAY_HT(dest)            | dest - HashTable* |
| H         | Z_PARAM_ARRAY_OR_OBJECT_HT(dest)  | dest - HashTable* |
| l         | Z_PARAM_LONG(dest)                | dest - long |
| L         | Z_PARAM_STRICT_LONG(dest)         | dest - long |
| o         | Z_PARAM_OBJECT(dest)              | dest - zval* |
| O         | Z_PARAM_OBJECT_OF_CLASS(dest, ce) | dest - zval* |
| p         | Z_PARAM_PATH(dest, dest_len)      | dest - char*, dest_len - int |
| P         | Z_PARAM_PATH_STR(dest)            | dest - zend_string* |
| r         | Z_PARAM_RESOURCE(dest)            | dest - zval* |
| s         | Z_PARAM_STRING(dest, dest_len)    | dest - char*, dest_len - int |
| S         | Z_PARAM_STR(dest)                 | dest - zend_string* |
| z         | Z_PARAM_ZVAL(dest)                | dest - zval* |
|           | Z_PARAM_ZVAL_DEREF(dest)          | dest - zval* |
| +         | Z_PARAM_VARIADIC('+', dest, num)  | dest - zval*, num int |
| *         | Z_PARAM_VARIADIC('*', dest, num)  | dest - zval*, num int |


The effect of **!** and reference modifiers may be achieved using extended version of the same macros e.g. Z_PARAM_ZVAL_EX(dest, check_null, separate).

==== Performance Evaluation ====

The following numbers collected for 100 and 1000 requests to wordpress home page.

^                          ^ # of CPU instructions ^ time [sec] ^
| phpng                    |          5,132,181,672|       18.90|
| phpng + fast_zpp         |          4,901,907,204|       18.40|


So the proposal provides 4.5% improvement in terms of CPU instructions and 2.5% in terms of time.

The effect of code size increase is negligible. Just ~20KB for ~70 function (that's below 0.1%)

===== Backward Incompatible Changes =====
The new API provides exactly the same behavior.  Also, we don't propose to remove the old API. So there must not be any compatibility problems.

===== Proposed PHP Version(s) =====
This proposal is targeted to php-7

===== Vote =====

State whether this project requires 50%+1 majority (see [[voting]]). The vote is a straight Yes/No vote.

<doodle title="Should PHP 7 have Fast Parameter Parsing API?" auth="dmitry" voteType="single" closed="true">
   * Yes
   * No
</doodle>

 The vote concluded on January 27th.

===== Patches and Tests =====

  * Original Proposal: https://gist.github.com/dstogov/1131691470a4c37540a6

The patch is already included into PHP-7 and wrapped with ifdef/ifndef FAST_ZPP.
