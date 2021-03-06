====== PHP RFC: Filtered unserialize() ======
  * Version: 1.0
  * Date: 2013-03-29
  * Author: Stas Malyshev, stas@php.net
  * Status: Implemented
  * First Published at: http://wiki.php.net/rfc/secure_unserialize
  * Patch: https://github.com/php/php-src/pull/315

===== Introduction =====

unserialize() in PHP has certain security issues, brought by the fact that serialized data can
include objects with data, and once these objects are instantiated, destructors are called when they
are destroyed. This could allow to inject serialized data into application which may perform actions not intended by application writer, such as carelessly assuming unserialized data have certain types and calling functions on them which may have different meaning for different objects. Besides destructors, functions like %%__toString%%, %%__call%%, etc. can be vulnerable.

It is true that best security guidelines recommend to not use unserialize() on untrusted data, but this is mainly because of the missing filtering mechanisms in unserialize(). The fact also is that many applications depend on using unserialize() and are not going to be changed, at least in the short term, to use other mechanisms, especially in areas where object support is still needed. Thus, I think it still makes sense to provide filtered unserialize() in order to allow people reasonable alternative without having them to resort to things like PHP implementation of unserialize() (e.g. see https://github.com/piwik/piwik/blob/master/libs/upgradephp/upgrade.php#L407)

===== Proposal =====

The proposal is to amend unserialize() function, allowing to either completely prohibit restoring objects or restrict the objects being restored to a whitelist of objects.

For this purpose, optional second parameter is added to unserialize(), which can take the following values:

  * true - default value, allows all objects just as before
  * false - no objects allowed
  * array of class names, which list the allowed classes for unserialized objects (empty array here means the same as false)

If the class for the object is not allowed, the object is unserialized as an object of "incomplete class", just as it is done in a case where object's class does not exist. This means that the serialized data are roundtrip-safe with any settings, but with added security settings the unintended objects will not be accessible and their destructors and other methods will not be called.

** Examples **
<code php>
// this will unserialize everything as before
$data = unserialize($foo);
// this will convert all objects into __PHP_Incomplete_Class object
$data = unserialize($foo, false);
// this will convert all objects except ones of MyClass and MyClass2 into __PHP_Incomplete_Class object
$data = unserialize($foo, array("MyClass", "MyClass2"));
</code>

See API Update below.

===== Backward Incompatible Changes =====

   * No user-level BC issues should arise, as the default mode functions exactly as unserialize() works now.

===== Proposed PHP Version(s) =====

Proposed for inclusion into PHP 7 branch, and into  PHP 5.6.x provided the RM approves it and there will be no substantial objections to that.

===== Other issues ====

  * It is not planned that unserialize_callback_func function will be called on prohibited classes as it is done on non-existing classes.
  * This option is not available currently for sessions and any other functions that use unserialization without calling unserialize(). This may be added later if needed, but for sessions it is very unlikely that untrusted user data will be injected as serialized session data - in that case the problems with security are much larger as pretty much any session-based authentication will be immediately broken.

===== Vote =====

The vote is to accept the unserialize() filtering option as described in this RFC for inclusion into PHP, with options being Yes or No. 50%+1 majority is required for acceptance.

Vote started on 2014-11-03 and is open until 2014-11-10 23:59:59 PST.

<doodle title="Approve filtered unserialize() proposal?" auth="stas" voteType="single" closed="true">
   * Yes
   * No
</doodle>


===== API change =====

After some thought and discussion, I have decided to slightly change the API:

<code php>
// this will unserialize everything as before
$data = unserialize($foo);
// this will convert all objects into __PHP_Incomplete_Class object
$data = unserialize($foo, ["allowed_classes" => false]);
// this will convert all objects except ones of MyClass and MyClass2 into __PHP_Incomplete_Class object
$data = unserialize($foo, ["allowed_classes" => ["MyClass", "MyClass2"]);
//accept all classes as in default
$data = unserialize($foo, ["allowed_classes" => true]);
</code>

This will allow to extend the options array in the future if we ever want to add more parameters. No objections were voiced on the list regarding this API change.

===== References =====

   * Unserialize function: http://php.net/unserialize
   * Example of unserialize() security issue: http://heine.familiedeelstra.com/security/unserialize
   * Prior discussion: http://grokbase.com/t/php/php-internals/133z8ns916/rfc-more-secure-unserialize

===== See also =====
   * Joomla unserialize() vulnerability: http://seclists.org/bugtraq/2013/Apr/173
   * CubeCart unserialize() vulnerability: http://karmainsecurity.com/KIS-2013-02
   * TikiWiki unserialize() vulnerability: http://www.securityfocus.com/bid/54298/info
   * Invision Power Board unserialize() vulnerability: http://www.securityfocus.com/bid/56288/info
===== Changelog =====

  - 2013-03-29 First version published