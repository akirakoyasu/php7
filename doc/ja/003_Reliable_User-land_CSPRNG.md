====== PHP RFC: Easy User-land CSPRNG ======
  * Version: 0.5
  * Date: 2015-02-20
  * Author: Sammy Kaye Powers <me@sammyk.me> & Leigh <leigh@php.net>
  * Status: Implemented (in PHP 7.0)
  * First Published at: http://wiki.php.net/rfc/easy_userland_csprng


===== Introduction =====
This RFC proposes adding a user-land API for an easy to use and reliable [[http://en.wikipedia.org/wiki/Cryptographically_secure_pseudorandom_number_generator|CSPRNG]] in PHP.

==== The Problem ====
By default PHP does not provide an easy mechanism for accessing cryptographically strong random numbers in user-land. Users have a few options like ''openssl_random_pseudo_bytes()'', ''mcrypt_create_iv()'' or directly opening ''/dev/*random'' devices to obtain high quality pseudo-random bytes, but unfortunately system support for these functions and extensions varies between platforms and each come with their own set of problems

  * The ''mcrypt_create_iv()'' function has no dependency on [[http://mcrypt.sourceforge.net/|MCrypt lib]] yet it requires the MCrypt extension to be installed before it can be used. Users are forced to include an entire library for no reason.
  * ''openssl_random_pseudo_bytes()'' is provided by the [[https://www.openssl.org/|OpenSSL lib]]. This function comes with a ''$crypto_strong'' the meaning of which may just confuse users.
  * Falling back to ''/dev/urandom'' is OS-specific.

In addition users may attempt to generate their own streams of random bytes relying on ''rand()'' or ''mt_rand()'', and this is something we absolutely want to avoid.

===== Proposal =====
There should be a user-land API to easily return an arbitrary length of cryptographically secure pseudo-random bytes directly and work on any supported platform.

The initial proposal is to add **two** user-land functions that return the bytes as binary and integer. Arbitrary length strings of random bytes are important for salts, keys and initialisation vectors. Integers based on CS random are important for applications where unbiased results are critical (i.e. shuffling a Poker deck).

Signatures:
<code>
random_bytes(int length);
random_int(int min, int max);
</code>

Examples:
<code php>
$randomStr = random_bytes($length = 16);

$randomInt = random_int($min = 0, $max = 127);
</code>

The sources of random used are as follows:
  * On windows ''CryptGenRandom'' is used exclusively
  * ''arc4random_buf()'' is used if it is available (generally BSD specific)
  * ''/dev/arandom'' is used where available
  * ''/dev/urandom'' is used where none of the above is available
  * An error is thrown in the event that a sufficient source of randomness is unavailable.

===== Backward Incompatible Changes =====
Any user-land code that defines a ''random_bytes()'' or ''random_int()'' function would generate a fatal error, however it is likely that these functions provide the same or similar functionality as desired.

===== Proposed PHP Version(s) =====
PHP 7

===== RFC Impact =====
==== To SAPIs ====
This RFC should not impact the SAPI's.

==== To Existing Extensions ====
No existing extensions are affected.

==== To Opcache ====
Opcache is unaffected.

==== New Constants ====
There would be no new constants.

==== php.ini Defaults ====
There would be no new php.ini defaults.

===== Open Issues =====
  * Nothing yet

===== Unaffected PHP Functionality =====
This change does not affect any of the existing ''rand()'' or ''mt_rand()'' functionality.

===== Future Scope =====
The concepts from the RFC could be used to:

   * Deprecate ''mcrypt_create_iv()''
   * Improve ''session_id'' randomness generation
   * Detect LibreSSL-portable for arc4random() on Linux

===== Patches and Tests =====
The current patch can be found here: https://github.com/php/php-src/pull/1119

===== Proposed Voting Choices =====

The voting choices are yes (in favor for accepting this RFC for PHP 7) or no (against it).

===== Vote =====

Vote starts on March 14th, and will end two weeks later, on March 28th.

This RFC requires a 2/3 majority.

<doodle title="Reliable user-land CSPRNG" auth="SammyK" voteType="single" closed="true">
   * Yes
   * No
</doodle>


===== Changelog =====
   * 0.5: Updated the function header for random_int() to reflect all args as required. - SammyK
   * 0.4: Added BC info. Updated patch link to point to PR. - SammyK
   * 0.3: Changed ''-PHP_INT_MAX'' to ''~PHP_INT_MAX'' (thanks [[https://twitter.com/trevorsuarez/status/570308776185733122|@trevorsuarez]]) - SammyK
   * 0.2: Condensed the problem domain into something more focused. Added function sigs. - Leigh.
   * 0.1: Mmmm drafty  - Leigh
   * 0.0: Initial draft - need Leigh's input - SammyK

===== Acknowledgements =====
Big thanks to Anthony Ferrara, Daniel Lowrey, E. Smith and [[http://chat.stackoverflow.com/rooms/11/php|all the kids in the PHP room]] for all the help with this one!
