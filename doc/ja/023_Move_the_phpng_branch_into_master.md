====== PHP RFC: Move the phpng branch into master ======
  * Version: 1.0.1
  * Date: 2014-07-20
  * Author: Dmitry Stogov, Zeev Suraski
  * Status: Accepted
  * First Published at: http://wiki.php.net/rfc/phpng


===== Introduction =====
This RFC proposes to accept the phpng refactoring codebase as the basis for the future major version of PHP, and move it into master.

===== Proposal =====
As discussed on internals@ during May 2014, a team of developers led by Dmitry Stogov has conducted research into how to refactor the PHP codebase in order to reduce memory consumption and increase performance.  Their effort resulted in a new updated codebase that is nearly 100% compatible and provides anywhere between 20% and 110% performance gains in real world applications such as Wordpress, Drupal or SugarCRM.  It also significantly reduces memory consumption.

The goal now is to officially accept the new codebase as the basis for the next major version of PHP.  Once that happens, we will generally stop referring to phpng as phpng, and will start referring to it as either PHP 6 or PHP 7, depending on the result of the [[https://wiki.php.net/rfc/php6|PHP Naming]] vote.

Accepting this proposal has no implications on any other RFCs that may go into this next major version.  Other RFCs will be discussed and be voted on separately and independently from this RFC.


===== Implementation =====

The main changes in phpng have to do with compacting of data structures and making smarter choices in representing values.  If you're interested in more information about the changes made to the codebase in the phpng patch, you can find them at the [[https://wiki.php.net/phpng-int|phpng internals]] detailed information page.

===== Performance =====

phpng's key benefits are around performance and memory utilization.  Detailed performance benchmarks against previous (and current) versions of PHP, as well as hhvm, have been published in a [[http://zsuraski.blogspot.co.il/2014/07/benchmarking-phpng.html|blog post]].
In addition, phpng's refactored codebase is an excellent basis for future optimizations, and its performance continued to improve since it was first published in May - as of now, by ~10-25%.  [[https://wiki.php.net/phpng#performance_evaluation|Performance evolution]] data is available.

Memory consumption is also much improved, although it's much more difficult to measure.  The RFC will be updated if & when we have some hard, repeatable data we can share regarding memory utilization.

===== Impact on a future JIT implementation =====

The phpng project followed substantial research into JIT (Just In Time compilation) on top of the existing PHP & Zend Engine codebase.  This research resulted in a highly optimized JIT engine that executed synthetic benchmarks extremely fast - but that showed little or no gains in real world applications.  After much research and experimentation, we reached the conclusion that the non-compact data structures - as well as some other key implementation details of the current engine - were bottlenecks that prevented real world apps from realizing tangible JIT gains.  One of phpng's key goals was therefore to enable a better basis JIT.  We believe it meets that goal, and will be an excellent starting point for future JIT work.

===== Backward Incompatible Changes =====

At this point, approximately 20 tests out of about 11,000 PHPT’s are failing (~0.18%).  They are detailed at [[https://wiki.php.net/phpng#known_problems]] and are work in progress.  We believe that some can be fixed, although there might be some behaviors that were a side-effect of the old implementation that cannot be fixed in the new, refactored implementation.


===== Proposed PHP Version(s) =====
Next major version of PHP

===== RFC Impact =====
==== To Existing Extensions ====
Existing extensions will have to be updated to reflect the new data structures and updated APIs.  Much of this work is already done for most of the extensions bundled with PHP, but there are still extensions that need to be ported, and most of the extensions in PECL will have to be ported.

More details can be found in the [[https://wiki.php.net/phpng-upgrading|Upgrading Guide for Extensions]].

==== To SAPIs ====
No special impact on SAPIs, beyond potential updates to reflect new data structures and updated APIs.


==== To OPcache ====
An updated OPcache is already a part of the phpng codebase.  There will be no impact of any sort on end users.

All performance tests and regression tests were performed with OPcache enabled.

==== New Constants ====
No new constants

==== php.ini Defaults ====
No effect on php.ini

===== Unaffected PHP Functionality =====
All PHP functionality, except for implementation, performance and memory consumption and the minor potential incompatibilities mentioned above should be unaffected.
PHPNG has already been tested with many of the popular off-the-shelf PHP-based applications, including Wordpress, Drupal, SugarCRM, Magento as well as the Zend Framework 2 and Symfony 2 skeleton apps.  It runs all of these applications without any need for modifications.


===== Proposed Voting Choices =====
Two proposed voting choices:

Yes – accept phpng as the basis of the next major version of PHP and merge into master.

No - don't move phpng into master

===== Vote =====

The code changes, while substantial, do not effect the language itself, and therefore a simple 50%+1 majority is required.

Voting started on 2014-08-06 and will end on 2014-08-14.

<doodle title="Move phpng to master?" auth="zeev" voteType="single" closed="true">
   * Yes
   * No
</doodle>

===== Patches and Tests =====
Checkout the phpng branch at https://git.php.net/repository/php-src.git

Additional information on the main changes performed in phpng are available at https://wiki.php.net/phpng-int

The standard PHP test suite can be used to test the updated codebase.

===== Implementation =====
http://wiki.php.net/phpng

===== References =====

[[https://wiki.php.net/phpng#performance_evaluation|Performance evaluation & evolution of the phpng patch]]

[[https://wiki.php.net/phpng-int|phpng internals]]

[[http://zsuraski.blogspot.co.il/2014/07/benchmarking-phpng.html|Benchmarking PHPNG - blog post by Zeev Suraski]]

[[https://wiki.php.net/phpng-upgrading|Upgrading guide for extensions]]