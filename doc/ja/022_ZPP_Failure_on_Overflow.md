====== PHP RFC: ZPP Failure on Overflow ======
  * Version: 0.1.1
  * Date: 2014-09-22, Last Updated 2014-12-02
  * Author: Andrea Faulds, ajf@ajf.me
  * Status: Implemented (PHP 7.0)
  * First Published at: http://wiki.php.net/rfc/zpp_fail_on_overflow

===== Introduction =====

PHP is a weakly-typed language, and so implicitly converts between integers and floats when such values are passed to internal functions. Currently, when a float that is beyond the range of an integer (outside [PHP_INT_MIN, PHP_INT_MAX]) is passed to an internal function expecting an integer argument, it will silently truncate (e.g. ''3221225470.5'' becomes ''-1073741826'' on 32-bit platforms), causing a loss of magnitude and sign information, though technically preserving the "lower bits". This mangling of input happens without warning, is unintuitive, and can lead to subtle bugs.

===== Proposal =====

''zend_parse_parameters'' and its fast macro counterparts are modified to fail (which usually causes the PHP function that invoked it to bail out with an ''E_WARNING'' and return ''NULL'') if a float that is outside of the range of an integer, or is NaN, is passed for the ''l'' (''Z_PARAM_LONG'') parameter type. Functions which use the ''L'' type (aka ''strict'' mode in the macros) will retain their previous behaviour of saturating (capping at ''PHP_INT_MAX''/''MIN'') without erroring when a float is out of bounds, but will now fail if NaN is passed.

The special floating-point values ''INF'' and ''-INF'' fail the check like any other number that is too large.

This proposal would complement the [[rfc:bigint|Big Integer Support RFC]], as it would be desirable to have functions which only accept platform-native integers (32-bit or 64-bit) error instead of silently truncating when a bigint that is too large is passed. This is particularly important given that the bigint RFC strives for there to be no user-visible difference between the internal `IS_LONG` and `IS_BIGINT` types. Integers being silently mangled when passed to other functions can only cause bugs and programmer frustration. Thus it is quite important that something is done about this before bigints can be implement.

===== Backward Incompatible Changes =====

This is an inherently backwards-incompatible change. However, the previous behaviour was dangerous, and this will only affect uncommon edge cases. In the unusual event that the truncation/NaN-tolerant behaviour is desired by the programmer, an explicit ''(int)'' cast can be used.

===== Proposed PHP Version(s) =====

As this is a backwards-compatibility break, it is targeted to the next major release of PHP, currently PHP 7.

===== Unaffected PHP Functionality =====

This does not affect other implicit integer casts, such as those done with array keys or by the bitwise operators.

===== Vote =====

This is arguably not a language change. However, as it breaks backwards-compatibility, and because some may argue it //is// a language change, a 2/3 majority will be required. The vote is a straight Yes/No vote.

Voting opened on 2014-12-02 and ended on 2014-12-12.

<doodle title="Accept the ZPP Failure on Overflow RFC and merge into master?" auth="ajf" voteType="single" closed="true">
   * Yes
   * No
</doodle>

===== Patches and Tests =====

There is a working pull request containing a patch here: https://github.com/php/php-src/pull/835

===== Implementation =====

This was merged into master here: https://github.com/php/php-src/commit/0ea0b591d79ae0ee18d33533a5c701330836ff6b

It will form part of PHP 7.

===== References =====

  * [[rfc:bigint|Big Integer Support RFC]]

===== Rejected Features =====

Keep this updated with features that were discussed on the mail lists.

===== Changelog =====

  * v0.1.1 - Some cleanup and clarity
  * v0.1 - Initial version