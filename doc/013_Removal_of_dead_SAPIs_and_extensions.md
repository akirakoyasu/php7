
====== PHP RFC: Removal of dead or not yet PHP7 ported SAPIs and extensions ======
  * Version: 2
  * Date: 2015-01-14
  * Author: Anatol Belski, <ab@php.net>
  * Status: Closed
  * First Published at: http://wiki.php.net/rfc/removal_of_dead_sapis

===== Introduction =====

 Currently PHP contains many SAPIs to the servers either completely unavailable or unsupported for a long time. The same can be said about several extensions which are for long deprecated or their dependency libraries are unsupported. Here's a short list:

 Dead SAPIs:
  * aolserver
  * apache
  * apache_hooks
  * caudium
  * continuity
  * isapi
  * milter
  * phttpd
  * pi3web
  * roxen
  * thttpd
  * tux
  * webjames
  * apache2filter - not really dead, but currently broken
  * nsapi


 Extensions with unmaintained dependencies

  * imap
  * mcrypt

 Extensions not yet ported to PHP7

  *  interbase
  *  mssql
  *  oci8
  *  pdo_dblib
  *  pdo_oci
  *  sybase_ct



 Deprecated extensions handled by https://wiki.php.net/rfc/remove_deprecated_functionality_in_php7 (not handled by this RFC)

  * mysql
  * ereg

This leads to the situation where the repository contains a lot of unmaintained code. The code which doesn't become any updates for years or relies on long unmaintained libraries. Besides that code being not maintained, it is also kind of security risk.

\\

===== Research =====

==== sapi/aolserver ====


  * Server home: http://sourceforge.net/projects/aolserver/
  * Server last release: 2012
  * Server debian supported: yes
  * SAPI status: 5.4 builds with the latest server API

=== Voting ===

<doodle title="Remove sapi/aolserver from the core" auth="user" voteType="single" closed="true">
   * Yes
   * No
</doodle>
\\

==== sapi/apache ====

  * Server last release: 2010
  * Server debian supported: no

Apache 1.x branch not developed anymore since 2010. Also it supports no security updates anymore.

=== Voting ===

<doodle title="Remove sapi/apache from the core" auth="user" voteType="single" closed="true">
   * Yes
   * No
</doodle>
\\
==== sapi/apache_hooks ====

Same as sapi/apache.

=== Voting ===

<doodle title="Remove sapi/apache_hooks from the core" auth="user" voteType="single" closed="true">
   * Yes
   * No
</doodle>
\\
==== sapi/apache2filter ====

  * Server debian supported: yes

This Apache 2.x compatible module seems to have been supported in at least PHP5. In master, it will need to be ported.


=== Voting ===

<doodle title="Remove sapi/apache2filter from the core" auth="user" voteType="single" closed="true">
   * Yes
   * No
</doodle>
\\
==== sapi/caudium ====

  * Sevrer home: http://caudium.net/
  * Server last release: 2012
  * Server debian supported: not completely
  * SAPI status: unsupported

The official documentation of Caudium states that PHP can only run as CGI or FCGI http://www.caudium.net/space/documentation/PHP%20Setup . Running PHP as a module is not supported anymore.

=== Voting ===

<doodle title="Remove sapi/caudium from the core" auth="user" voteType="single" closed="true">
   * Yes
   * No
</doodle>
\\
==== sapi/continuity ====

No mortal remains of the corresponding server could be found on the internets.

=== Voting ===

<doodle title="Remove sapi/continuity from the core" auth="user" voteType="single" closed="true">
   * Yes
   * No
</doodle>
\\
==== sapi/isapi ====

  * Server home: http://www.iis.net/
  * SAPI status: unsupported

The ISAPI support was announced to have been dropped already in the 5.3 migration guide. http://us2.php.net/manual/en/migration53.windows.php .

=== Voting ===

<doodle title="Remove sapi/isapi from the core" auth="user" voteType="single" closed="true">
   * Yes
   * No
</doodle>
\\
==== sapi/milter ====

  * Home: https://www.milter.org/
  * Server debian supported: yes
  * SAPI status: 5.4 builds with sendmail 8.14

Milter is not a web server but an API enabling to hook into different MTAs for mail filtering. PHP scripts running under this SAPI are not dedicated to web development, but would be invoked by the corresponding MTA. In this case also, the corresponding PHP program will need to implement whichever rules. An example of another binding to Milter, just to name one, is an interface to Amavis.
=== Voting ===

<doodle title="Remove sapi/milter from the core" auth="user" voteType="single" closed="true">
   * Yes
   * No
</doodle>
\\

==== sapi/phttpd ===

  * Server home: http://www.lysator.liu.se/~pen/phttpd/
  * Server last release: not sure, development seems to have ended around 2000
  * Server debian supported: no
  * SAPI status: development incomplete

Comment from the README: THIS IS BY NO MEANS COMPLETE NOR USABLE RIGHT NOW!

=== Voting ===

<doodle title="Remove sapi/phttpd from the core" auth="user" voteType="single" closed="true">
   * Yes
   * No
</doodle>
\\
==== sapi/pi3web ====

  * Server home: http://pi3web.sourceforge.net/pi3web/
  * Server last release: 2005
  * Server debian supported: no
  * SAPI status: last commit by the author is in 2004

=== Voting ===

<doodle title="Remove sapi/pi3web from the core" auth="user" voteType="single" closed="true">
   * Yes
   * No
</doodle>
\\
==== sapi/roxen ====

  * Server Home: http://www.roxen.com/products/cms/webserver/
  * Server last release: 2014
  * Server debian supported: no
  * SAPI status: experimental

The official documentation http://docs.roxen.com/roxen/5.0/system_developer_manual/languages/php.xml call the only supported mode to be CGI. The SAPI module is called experimental there.

=== Voting ===

<doodle title="Remove sapi/roxen from the core" auth="user" voteType="single" closed="true">
   * Yes
   * No
</doodle>
\\
==== sapi/thttpd ====

  * Server home: http://acme.com/software/thttpd/
  * Server last release: 2014
  * Server debian supported: yes
  * SAPI status: not clear

The latest thttpd version is 2.26, which builds fine. But when building PHP, there's an error message \\
\\
"This version only supports thttpd-2.21b and premium thttpd" \\
\\
But thttpd-2.21b doesn't build anymore, at least at some recent distro i've used. The premium thttpd seems to be a commercial product which isn't available to test with.

=== Voting ===

<doodle title="Remove sapi/thttpd from the core" auth="user" voteType="single" closed="true">
   * Yes
   * No
</doodle>
\\
==== sapi/tux ====

  * Server home: http://people.redhat.com/~mingo/TUX-patches/
  * Server last release: 2006
  * Server debian supported: no
  * SAPI status: PHP4 compatible

=== Voting ===

<doodle title="Remove sapi/tux from the core" auth="user" voteType="single" closed="true">
   * Yes
   * No
</doodle>
\\
==== sapi/webjames ====
  * Sevrer home: http://www.cp15.org/webjames/
  * Server last release: 2007
  * Server debian supported: no
  * SAPI status: PHP5.2 compatible

=== Voting ===

<doodle title="Remove sapi/webjames from the core" auth="user" voteType="single" closed="true">
   * Yes
   * No
</doodle>
\\
==== sapi/nsapi ====

  * Server home: there seem to be many, but all require some Sun/Oracle environment, more to read here http://en.wikipedia.org/wiki/Oracle_iPlanet_Web_Server
  * Server last release: 2014
  * Server debian supported: no
  * SAPI status: Seems to be PHP5 compatible, untested

The developers of iPlanet @Oracle wrote back, that they're not intended to support this SAPI starting from PHP7 onwards.

However Uwe Schindler wrote back, that he will port and maintaing this SAPI for PHP7.

**NOTE**: This SAPI was removed from the voting options, because per the current info it's going to be supported for PHP7 in the future.

==== ext/imap ====

  * Dependency home: http://www.washington.edu/imap/
  * Dependency debian supported: yes
  * Dependency last release: 2007

=== Voting ===

<doodle title="Remove ext/imap from the core" auth="user" voteType="single" closed="true">
   * Yes
   * No
</doodle>
\\
==== ext/mcrypt ====

  * Dependency home: http://sourceforge.net/projects/mcrypt/
  * Dependency debian supported: yes
  * Dependency last release: 2007

=== Voting ===

<doodle title="Remove ext/mcrypt from the core" auth="user" voteType="single" closed="true">
   * Yes
   * No
</doodle>
\\
==== ext/interbase ====
  * Dependency home: not available, firebird can be used
  * Dependency debian supported: yes
  * Dependency last release: 2014

Marius Adrian Popa wrote back about an intention to maintain this further in the core together with ext/pdo_firebird. Right now the ext/interbase is half ported to PHP7 (needs cleaning).

**NOTE**: This extension was removed from the voting options, because per the current info it's going to be supported for PHP7 in the future.
\\
==== ext/pdo_oci and ext/oci8 ====
  * Dependency home: oracle.com
  * Dependency debian supported: no
  * Dependency last release: 2014

Christopher Jones stated these extensions to be supported by Oracle. Oracle plans to maintains them further in the core.


**NOTE**: These extensions was removed from the voting options, because they per the current info are going to be supported for PHP7 in the future.
\\
==== ext/mssql ===

  * Dependency home: microsoft.com
  * Dependency debian supported: no
  * Dependency last release: 2014

=== Voting ===

<doodle title="Remove ext/mssql from the core" auth="user" voteType="single" closed="true">
   * Yes
   * No
</doodle>
\\
==== ext/pdo_dblib ====

  * Dependency home: sybase.com, microsoft.com
  * Dependency debian supported: no
  * Dependency last release: 2014

=== Voting ===

<doodle title="Remove ext/pdo_dblib from the core" auth="user" voteType="single" closed="true">
   * Yes
   * No
</doodle>
\\
==== ext/sybase_ct ====

  * Dependency home: sybase.com
  * Dependency debian supported: no
  * Dependency last release: 2014

=== Voting ===

<doodle title="Remove ext/sybase_ct from the core" auth="user" voteType="single" closed="true">
   * Yes
   * No
</doodle>
\\
===== Proposal =====

The SAPIs listed above should be removed from the source tree.
The extensions listed above should be removed from the source tree. Depending on the vote outcome and presence of the maintainers, they could be resurrected in PECL.

===== Backward Incompatible Changes =====

None.

===== Proposed PHP Version(s) =====

7

===== SAPIs Impacted =====

SAPI and extension list in the description.

===== Impact to Existing Extensions =====

Unsupported extensions list in the description.

===== php.ini Defaults =====

php.ini will have to be checked to remove the unavailable config options, if any.

===== Open Issues =====

None.

===== Future Scope =====

Should an SAPI or extension come back, the corresponding code is available in the git history.

===== Voting =====

The voting options are attached to the corresponding items. The voting starts on 2015-02-02 and ends on 2015-02-09 at 23:00 CET respectively. To be accepted/declined each vote requires 50%+1 acceptance.

===== Implementation =====

Implemented in PHP7, see PRE_NATIVE_TLS_MERGE and POST_NATIVE_TLS_MERGE tags.



