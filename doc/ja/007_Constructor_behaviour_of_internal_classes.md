====== PHP RFC: Constructor behaviour of internal classes ======
  * Version: 1.0
  * Date: 2015-03-01
  * Author: Dan Ackroyd, Danack@php.net
  * Status: Implemented (in PHP 7.0)
  * First Published at: https://wiki.php.net/rfc/internal_constructor_behaviour


===== Introduction =====
Several internal classes in extensions that ship with PHP exhibit some highly surprising behaviour. This RFC has two aims:

i) to make the behaviour of these classes behave more consistently with the behaviour that most people would expect them to have.

ii) setting the behaviour that should be followed by internal classes for their constructors.

If other internal classes that are not listed in this RFC are found to have behaviour that is not consistent with the behaviour listed in this RFC they should be treated as bugs and fixed at the next appropriate release.


==== Constructors returning null rather than throwing an exception ====

For the internal classes:

  * finfo
  * PDO
  * Collator
  * IntlDateFormatter
  * MessageFormatter
  * NumberFormatter
  * ResourceBundle
  * IntlRuleBasedBreakIterator


when their constructor is called with parameters that cannot be used to create a valid instance of the class, the constructor returns null. This is inconsistent with classes which are declared in userland which are not capable of returning null value, but instead have to indicate an error by throwing an exception.

Having a different behaviour for internal classes is inconsistent and highly surprising when users encounter it:

<code>
$mf = new MessageFormatter('en_US', '{this was made intentionally incorrect}');
if ($mf === null) {
    echo "Surprise!";
}
</code>

This RFC proposes setting a standard that internal classes should either return a valid instance of the class or throw an exception. Also for the classes listed to be changed in PHP 7 so that they follow this behaviour.

=== Factory methods ===

The classes IntlDateFormatter, MessageFormatter, NumberFormatter, ResourceBundle and IntlRuleBasedBreakIterator all have a static factory method that can create an instance of the class, as well as the constructor. The factory methods do not currently throw an exception when invalid arguments are passed to it. It is the position of this RFC that it is correct for those factory methods to return either null or a valid usable instance of the class.

PHP is a multi-paradigm programming language which supports writing code in either procedural or OO style. Having the factory methods behave like this is perfectly consistent with userland code.

<code>
class NumberFormatter {
    static function create($locale , $style , $pattern = null) {
        try {
            return new NumberFormatter($locale, $style, $pattern);
        }
        catch(NumberFormatterException $nfe) {
            trigger_error($nfe->getMessage(), E_USER_ERROR);
            return null;
        }
    }
}
</code>

There is also a function `finfo_open` which is also a procedural way of instantiating an finfo object. This function would not be modified.


==== Constructors give warning, but are then in an unusable state ====

The constructors of several classes check the parameters that they are given and just give a warning when they are not acceptable. This leaves the object instantiated but in an unusable state.

<code>
<?php
//Reflection function is not meant to accept an array
$reflection = new ReflectionFunction([]);
//Outputs Warning: ReflectionFunction::__construct() expects parameter 1 to be string, array given in...

var_dump($reflection->getParameters());
//Fatal error: ReflectionFunctionAbstract::getParameters(): Internal error: Failed to retrieve the reflection object in..
</code>

This RFC proposes that this behaviour should be changed so that the constructor should throw an exception if the class cannot be created in a usable state i.e. convert the current warning to an appropriate exception.

The list of classes that show this behaviour is:

  * UConverter
  * SplFixedArray
  * ReflectionFunction
  * ReflectionParameter
  * ReflectionMethod
  * ReflectionProperty
  * ReflectionExtension
  * ReflectionZendExtension
  * Phar
  * PharData
  * PharFileInfo


==== Constructor generates fatal error ====

The class PDORow gives a fatal error when a user attempts to instantiate it.

<code>
$foo = new PDORow();
// PHP Fatal error:  PDORow::__construct(): You should not create a PDOStatement manually in test.php on line 5
</code>

Fatal errors should only be used for fatal errors. This RFC proposes that the constructor for PDORow should be changed to throw an appropriate exception rather than giving a fatal error.






===== Backward Incompatible Changes =====

I don't believe that most users are even aware that the classes listed can show this behaviour. For these people none of the changes would be a BC break.

For people who are aware that the constructor can fail there would be a small BC break.

For the classes that have a factory creation method the code that currently tests against the constructor returning null:

<code>
$mf = new MessageFormatter('en_US', $thisMightBeInvalid);
if ($mf === null) {
    // error handling code
}
</code>

could be changed to using the factory method:


<code>
$mf = MessageFormatter::create('en_US', $thisMightBeInvalid);
if ($mf === null) {
    // error handling code
}
</code>

For the other classes which do not have an equivalent procedural method which will still return null, the user would need to wrap the call to the constructor in a try/catch block.


For the classes where the instance is created but in an unusable state there would theoretically be a small BC break in that the location where the code breaks for invalid parameters would be changed.


===== Proposed PHP Version(s) =====

These changes would be for PHP 7

===== RFC Impact =====
==== To SAPIs ====
None.

==== To Existing Extensions ====

The standard of either returning a usable instance or throwing an exception in an objects constructor would apply to all extensions that ship with the official PHP release. Other extensions (e.g. those that are hosted on PECL) may wish to follow this standard, but it is not required.

===== Open Issues =====

Some of the Intl extension code has always had the behaviour of giving an error notice, and also throwing an exception for the same error. This behaviour should be cleaned up for the release of PHP 7, so that either an error is given, or an exception, but never both.


===== Patches and Tests =====

These classes will be corrected by making the constructor throw an exception rather than return null if the construction of the object fails.

  * finfo
  * PDO
  * Collator
  * IntlDateFormatter
  * MessageFormatter
  * NumberFormatter
  * ResourceBundle
  * IntlRuleBasedBreakIterator


These classes would be corrected by detecting the invalid data in the constructor and throwing an exception at object construction time, rather than giving an error when the created instance is used.
  * UConverter
  * SplFixedArray
  * ReflectionFunction
  * ReflectionParameter
  * ReflectionMethod
  * ReflectionProperty
  * ReflectionExtension
  * ReflectionZendExtension
  * Phar
  * PharData
  * PharFileInfo


The class PDORow will be changed to give an exception if an attempt is made to instantiate it from userland.

The changes have been made in this branch: https://github.com/danack/php-src/tree/InternalClassClean or to view as a PR https://github.com/php/php-src/pull/1178

The list of exceptions used are:

Exception - finfo

IntlException -UConverter, Collator, IntlDateFormatter, MessageFormatter, NumberFormatter3v, ResourceBundle, IntlRuleBasedBreakIterator

InvalidArgumentException - SplFixedArray

PDOException - PDO, PDORow

PharException - Phar, PharData, PharFileInfo

ReflectionException - ReflectionExtension,  ReflectionFunction, ReflectionMethod, ReflectionParameter, ReflectionProperty, ReflectionZendExtension



===== Voting =====


Should the standard paradigm for constructors for internal objects be to return a usable instance of a class on success, and throw an exception if they encounter an error, and should the code for the classes listed below be modified to follow this standard?

<doodle title="Constructor behaviour of internal classes" auth="danack" voteType="single" closed="true">
   * Yes
   * No
</doodle>

Voting will close March 29th 2015 9pm UTC and requires 2/3 in favour to pass.


===== Implementation =====
After the project is implemented, this section should contain
  - the version(s) it was merged to: 7.0
  - a link to the git commit(s): http://git.php.net/?p=php-src.git;a=commit;h=4796e0242b8cdd2a77b552fcbaa74d70d87f6d0a
  - a link to the PHP manual entry for the feature: No new manual entry, the changes are conforming to standard practice.

