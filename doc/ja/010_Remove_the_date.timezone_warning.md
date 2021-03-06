====== PHP RFC: Remove the date.timezone warning ======
  * Version: 1.2
  * Date: 2015-01-27
  * Author: Bob Weinand, bobwei9@hotmail.com
  * Status: Implemented
  * First Published at: https://wiki.php.net/rfc/date.timezone_warning_removal

===== Introduction =====
PHP should be usable without configuring it through either an ini file or defining settings on the command line. Currently this is not possible as the date.timezone is required to be set, despite it defaulting to the best option.

"date.timezone" is the only setting that is required to be set. Every other setting, including those that have security implications default to sensible values silently; defaulting to sensible values does not appear to cause any problem for users.

The error message becomes entirely tedious when working with anything other than a single installed version of PHP.

For example when you want to run the unit tests for a library against all the different versions of PHP that library supports, you should be able to do so by downloading each of the PHP versions you want to test against, and invoking the PHP executables from the download location, rather than having to install it first.

===== Proposal =====
The current behaviour when the data.timezone ini entry is not set is to:

  * Default to UTC.
  * Give a warning message.

The proposal is just to remove the warning message. The behaviour of setting the timezone to be UTC would not be changed. This will allow people to run PHP without an ini file without having error messages.

===== Proposed PHP Version(s) =====
Targeting the next minor version. (In this case PHP 7, being a major too.)

===== Unaffected PHP Functionality =====
If the date.timezone ini setting is set but invalid, a warning is still thrown.

===== Feedback =====
==== "Won't this make life more difficult for end users?" ====
No. For users who install PHP through a distribution, the ini file is created by the package maintainer. They should set it to the systems timezone. (Some don't, but that is the fault of the distributions.)

For end users who compile from source they can either modify the ini file that is used by PHP in the compile step or they could add it themselves when PHP is built e.g. For Debian they could easily stick the following in a post-install hook to fix it:

    echo 'date.timezone='`cat /etc/timezone` > /etc/php/config.d/date.ini

People who are compiling PHP from source are perfectly capable of setting all ini settings that need to be set.

The only people who this RFC will affect are:

  * Those who use multiple versions of PHP or otherwise use PHP without going through the install process. This RFC will be beneficial to these users.
  * People who have accidentally deleted their ini files from their production system. These users have bigger issues than this RFC can address.

Still, it is weird, to have that one setting emit a warning, but other much more important settings don't. The ini settings have defaults and they should be used.

==== "Users are idiots and can't be trusted" ====
Well, yes. But there are many things that must be set in an ini file for PHP to be usable in a production environment. The person installing PHP needs to be capable of setting all of these. The timezone setting is not the most important setting that needs to be set so it's not good that PHP gives a warning message when it is not set.

==== "Can't you just define the timezone in the command line? Like: `php -d date.timezone=UTC foo.php`" ====
This is a painful hack. For people who spend a significant amount of time either developing PHP, PHP extensions or testing userland code against multiple PHP versions, they should be able to run PHP code without getting slapped for the temerity of not using an ini file and without being forced to set an ini setting to the same default value it is going to use anyway.


===== Proposed Voting Choices =====
Remove this warning (50%+1 vote) - Vote began 2015-02-16 and ended 2015-02-24.

<doodle title="Should the warning about a not set date.timezone ini setting be removed in master?" auth="bwoebi" voteType="single" closed="true">
   * Yes
   * No
</doodle>

===== Patch =====
https://github.com/php/php-src/pull/1029

===== References =====
There are enough requests to remove this:
  * https://bugs.php.net/bug.php?id=63339
  * https://bugs.php.net/bug.php?id=39142
  * https://bugs.php.net/bug.php?id=53473
  * https://bugs.php.net/bug.php?id=35481

===== Versions =====

  * 1.0: Initial RFC (2015-01-27)
  * 1.1: More neutral wording, discussion of feedback (2015-02-15)
  * 1.2: Put into vote (2015-02-16)