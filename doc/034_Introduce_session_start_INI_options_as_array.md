
====== PHP RFC: Introduce session_start() options - read_only, unsafe_lock, lazy_write and lazy_destroy ======
  * Version: 1.6
  * Date: 2014-02-02
  * Author: Yasuo Ohgaki, yohgaki@ohgaki.net
  * Status: Passed Proposal 1. 2 and 3 declined.
  * First Published at: http://wiki.php.net/rfc/session-lock-ini

===== Introduction =====

**proposal 0: options by session_start()**

session_start() does not have parameter currently. INI and other options may be passed as parameter. Add array parameter to session_start() to specify options.

Following options are new options for session and these options are not INI option and could only be specified via session_start() option parameter.

**proposal 1: read_only**

Open session for reading. Close session immediately after read. This is the fastest way to read session. This is the fastest way to read session data when update is not needed.

**proposal 2: lazy_write**

Session data does not have to write back to session data storage if session data has not changed. By omitting write, overall system performance may improve. It also helps keep session integrity for unsafe_lock=on session access by omitting writes when data is not updated.

**proposal 3: unsafe_lock**

Add unsafe_lock option for maximum concurrent script execution. Currently, script execution is serialized when session is used. unsafe_lock=on allows script access session data concurrently. When lazy_write=on, only updated data is saved. Applications can maintain consistency by saving updated session data only.

Suppose there is concurrent access from a same client Connection A and B with unsafe_lock=On , lazy_write=On.

  Connection A          Connection B
      ↓                     ↓
  Open Session          Open Session　
      ↓                     ↓
  Login                 Does not care auth status. Just read session.
      ↓                     ↓
  Close　　　　　　　　　Close (Session data unchanged)

Auth info set by Connection A never deleted by Connection B. Read only session achieves similar execution, but it has to wait lock. Therefore, previous access became

  Connection A          Connection B
      ↓                     ↓
  Open Session          Wait lock　
      ↓                     ↓
  Login                      ↓
      ↓                     ↓
  Close　　　　　　　　　     ↓
                        Open Session　
                             ↓
                        Does not care auth status. Just read session.
                             ↓
       　　　　　　　　　Close (Session data unchanged)

**Usage Tip for unsafe_lock=On: ** Update $_SESSION only when update is strictly needed. For example, change $_SESSION only when authentication/authorization information has changed. Alternatively, user may start all session with read_only=TRUE and reopen session with read/write only when authentication/authorization information has to be changed.


**proposal 4: lazy_destroy**

When session ID is renewed, race condition may occur.

  * Script A renewed session ID
  * Script B accessed to server with old session ID

Current session module(session_destroy()/session_regenerate_id()) simply deletes session data with $delete_old_session=true. When $delete_old_session=false(default), session_regenerate_id() simply creates new session ID and leaves old session ID/data.

Even when old session ID is destroyed, script B can access server with old session ID. Without session.strict_mode=On, session module may reinitialize session data with old ID which may be known to attackers.

Lazy session data deletion solves this issue. It also solves reliability issue of session_regenerate_id() on unreliable networks.


**Summary**
There is no new INI option and new options could be specified via session_start(). Users cannot change session behavior by mistake. [0]

Read only session makes sense since there are applications that do not require writing at all except when regenerating session ID/update authentication status. [1]

Currently, programmer has no control which session data is saved. New behavior allows programmer to control save changed session data when there are concurrent access. [2]

Session lock control could be API. e.g. session_lock() However, changing lock status after session started is complex. session_start() option is simpler. [3]

When client access to server concurrently, the order of requests cannot be guaranteed especially under mobile applications. This results in unreliable session_regenerate_id() operation that is closely related to web security. New behavior mitigate race condition. [4]


===== Proposal 0 session_start() option =====

Add option array parameter to session_start(). New option of this RFC are not INI option, but only a option for session_start(). All session INI options may be specified, otherwise INI options are used. (INI option handling is not in the patch)

Example:

  session_start(array('lazy_write'=>true, 'lazy_destroy'=>120, 'unsafe_lock'=>false));

===== Proposal 1 read_only option =====

Just read session data and close session immediately.


===== Proposal 2 - lazy_write =====

Introduce lazy_write option to session_start() that enable/disable lazy session data writing.

  lazy_write - FALSE by default

When lazy_write=FALSE, session module works exactly as it is now.

When lazy_write=TRUE, session module save session data only when session data has been changed.

When reading session data, session module takes MD5 hash for read data. When writing session data, session module compares old MD5 and new MD5 of session data and save session data only when MD5 differs.

Since write is omitted, save handler may not change "last updated time". This may result deleting session data by timeout. To prevent unwanted expiration of session, introduce new save handler API that update "last updated time" of session data.

  PS_UPDATE_FUNC()

Session module calls PS_UPDATE_FUNC() instead of PS_WRITE_FUNC() when lazy_write=On.


**NOTE: save handler implementation**

For files save handler, PS_UPDATE_FUNC() is not needed as it can update mtime at PS_OPEN_FUNC(). Files save handler's PS_UPDATE_FUNC() will be empty function that simply return true.

For memcache/memcached/redis/etc save handlers, PS_UPDATE_FUNC() is not needed since read access(PS_READ_FUNC()) updates last access time stamp. PS_UPDATE_FUNC() may return success simply.

Other save handlers should support PS_UPDATE_FUNC() API to update last access time. Otherwise, garbage correction may remove active sessions. Alternatively, save handler may update last access time when PS_OPEN()/PS_READ_FUNC() is called, then PS_UPDATE_FUNC() may return success simply.

===== Proposal 3 unsafe_lock option=====

Session module locks session data while executing script by default. This make sure session data integrity by serializing script execution. i.e. Only a script may have access to session data at the same time.

Current behavior ensures session data integrity, but it serializes script execution and slows down application performance.

For user defined save handlers, locking is up to users. Memcached session save handler has lock ini option for better performance by sacrificing integrity a little. Second proposal mitigates broken integrity by unlocked session data access.

Introduce unsafe_lock option to session_start() that enable/disable session data lock.

  unsafe_lock - FALSE by default.

When unsafe_lock=FALSE, save handler works exactly as it is now.

When unsafe_lock=TRUE, save handler lock session data only when locking is strictly needed. For files save handler, it locks only when reading/writing data.

Current behavior:

  Open session data
   ↓
  Read session data   -----------------
   ↓                         ↑
  （Script execution）      Locked
   ↓                         ↓
  Write session data  ------------------
   ↓　
  Close session data

New behavior with unsafe_lock=On:


  Open session data
   ↓
  Read session data   ←   Locked
   ↓
  （Script execution）
   ↓
  Write session data  ←   Locked
   ↓　
  Close session data

===== Proposal 4 - lazy_destroy =====

Introduce lazy_destroy option to session_start() that enable/disable lazy session data deletion. **Delayed deletion is required for secure/reliable session_regenerate_id() operation**.

  lazy_destroy - 60 (seconds) by default. Seconds that app may access session marked to be deleted.

In order to session module to know if the session data is already deleted and accessible, there must be flag/time stamp for deletion/access control.

**First option:**

<nowiki>
Save time stamp in $_SESSION['__PHP_SESSION_DESTROYED__'].
</nowiki>

  * Pros - simple and fast.
  * Cons - exposed internal data.

**Alternative option:**

Introduce new save handler API that saves flag/time stamp else where.

  * Pros - clean. no exposed data.
  * Cons - complex and slow.

Since alternative option could impact performance a lot. This RFC chooses first option.

By introducing this feature, session_regenerate_id(true) ($delete_old_session) can be a default.

session_destroy() no longer deletes session data, but set destroyed flag. To force deletion, add $force_deletion option to session_destroy(). (Default: FALSE)


**Difference between lazy destroy and leaving old session data**

  * Destroying session data with lazy destroy, it can only accessible defined time after destroy. (60 sec by default)

  * Leaving old data may keep old data forever. There is GC expire (default 1440 sec), but data may survive forever by accessing periodically.

It's clear that lazy_destroy is much secure than leaving old session data while it is keeping reliable session ID regeneration.





===== Backward Incompatible Changes =====

None for most applications.

lazy_destroy is enabled by default for reliable session_regenerate_id() operation. This could be issue for test scripts. For this case set $delete_old_session=TRUE.

session_destroy() no longer deletes session data. Like session_regenerate_id(), this could be a issue for test scripts. Use session_destroy(TRUE) to delete data actually.

===== Proposed PHP Version(s) =====

PHP 5.6 and later.


===== Impact to Existing Extensions =====

None other than session.

Old save handler works without new INI or API support.

Save handler module needs to be recompiled, but recompile is required for new release anyway.


===== php.ini Defaults =====

No INI option is exposed.

===== Open Issues =====

None.

===== Related Features ====

Implemented
  - session_commit()/session_write_close() - Implemented (PHP 4.0 and up). End session and write session data.
  - session_abort() - Implemented (5.6 and up). End session without saving session data.
  - session_regenerate_id() - It cannot be delete old session reliably without session.lazy_destroy. "$delete_old_session" option default changed to TRUE by default.
  - session_destroy() - Added "$force_deletion" option(Default: FALSE).
  - session_create_id() - Create session ID using session module code. Added by as part of RFC patch.
  - session_set_save_handler() - Additional functions/methods added. Added by this RFC.

Not implemented
  - session_unlock() - Unlock active session data. Session data is saved at close. i.e. session_commit() or module shutdown. This will not be implemented as it could be complex and slow. 'unsafe_lock' should be used instead.



===== Proposed Voting Choices =====

  * Proposal 1,2: Yes / No
  * Proposal 3: Yes / No
  * Proposal 4: Yes / No

**1) read_only** for reading only.

**2) lazy_write** that writes only session data has changed.

**Pros.**
  - lazy_write=TRUE could improved 2 times or better performance by removing write() calls.
  - lazy_write=TRUE provides better data consistency since only modified session data is written when session.lock=FALSE,


**Cons.**
  - Session behavior is changed. When lazy_write=Off, "last session written" wins. When lazy_write=TRUE, "last session modified" wins.
  - Programmers are responsible for ensuring data consistency when unsafe_lock=TRUE.

**3) unsafe_lock** that control session data locking.

**Pros**
  - Maximum concurrency (= better performance)
  - Programmer does not have to selectively start session. (i.e. read only or not) Simplify code.
  - Allow concurrency without explicit calls of session_commit(), session_write_close(), session_abort().
  - Simpler and faster than session_lock() API. e.g. This API requires additional access to data storage which make data handling more complex and slower.

**Cons**
  - Misuse of this feature could be a cause of bugs.
  - Open all session with read_only=TRUE and reopen session for writing as it is needed, would be safer for average users. (We may better to promote this usage pattern)

**4) lazy_destroy** that allows delayed session data deletion for concurrent accesses and reliable session_regenerate_id() operation.

**Pros**
  - session_regenerate_id() becomes more reliable for reasons described below.
  - Ensures proper application behavior even when network connection is unreliable. This is important for mobile applications especially.
  - Mitigates race condition that creates unneeded sessions. NOTE: Unneeded session created by race condition could be known session to attacker. Unneeded session increase system load.

**Cons**
  - Deleted session is accessible by specified time. NOTE: Access to deleted session is required for reliable session_regenerate_id() operation.
  - Deleted session contains time stamp variable in $_SESSION. NOTE: Time stamp is mandatory and storing it in $_SESSION is the fastest and simplest way.


Please note that **lazy_destroy is mandatory for reliable session_regenerate_id()/session_destroy()** operation. Please refer to referenced discussion for details.

Please propose alternative solution if you vote no to this proposal. This is security related change so there must be alternative solution. session_regenerate_id() cannot delete old session without delayed deletion or alternative solution if there is.


===== Benchmark =====

Note: read_only=on will yield better result than this.

== lazy_write: on, unsafe_lock: on ==
  Requests per second:    1894.61 [#/sec] (mean)
  100%     67 (longest request)

== lazy_write: off, unsafe_lock: off ==
  Requests per second:    445.19 [#/sec] (mean)
  100%   1547 (longest request)

Benchmark detail is blow
https://wiki.php.net/rfc/session-lock-ini#benchmark_detail

===== Vote =====

Please vote feature by feature.

The voting period is 2014/02/12 until 2014/02/19.

1,2) Introduce read_only, lazy_write to session_start() option.
<doodle title="Read only, lazy write option" auth="user" voteType="single" closed="true">
   * Yes
   * No
</doodle>

3) Introduce unsafe_lock option to session_start() option.
<doodle title="Unsafe lock option" auth="user" voteType="single" closed="true">
   * Yes
   * No
</doodle>

4) Introduce lazy_destroy option to session_start() option.
<doodle title="Lazy destroy option" auth="user" voteType="single" closed="true">
   * Yes
   * No
</doodle>


Thank you for voting!


===== Patches and Tests =====

https://github.com/yohgaki/php-src/compare/PHP-5.6-rfc-session-lock


===== Implementation =====

After the project is implemented, this section should contain
  - the version(s) it was merged to
  - a link to the git commit(s)
  - a link to the PHP manual entry for the feature

===== References =====

  - session_regenerated_id() discussion - http://marc.info/?l=php-internals&m=138242492914526&w=2
  - session.lock discussion - http://marc.info/?l=php-internals&m=138445826032002&w=2
  - VOTE discussion - http://marc.info/?l=php-internals&m=139017836113664&w=2


===== Benchmark Detail ======

 Since files save handler does not flush data to disk,
it's like a accessing memory and unsafe_lock=on could be a bit slower.
lazy_write=on may not change performance since disk access is not a
performance bottle neck.

To  see the benefit, database storage and more complex script would be needed.

To make benchmark is more realistic, following script generates 10 sessions, client
concurrently connects to apache using 50 connection. (5 connections/session)

http://stackoverflow.com/questions/985431/max-parallel-http-connections-in-a-browser

Most scripts are much more complex, so there is usleep() simulate it.

== lazy_write: on, unsafe_lock: on ==
  Requests per second:    1894.61 [#/sec] (mean)
  100%     67 (longest request)

== lazy_write: off, unsafe_lock: off ==
  Requests per second:    445.19 [#/sec] (mean)
  100%   1547 (longest request)

This test case is rather extreme, but if everything in web page (i.e. images, etc) is
access controlled by session, this can be a usual case.

<code php>
<?php
ini_set('session.use_trans_sid',TRUE);
ini_set('session.use_cookies',FALSE);
ini_set('session.use_only_cookies',FALSE);
ini_set('session.use_strict_mode',FALSE);
ini_set('session.save_path', '/usr/tmp/');

session_id('TESTID'.mt_rand(1, 10));
session_start(['lazy_write'=>$_GET['lazy_write'], 'unsafe_lock'=>$_GET['unsafe_lock']]);

if (!isset($session['x'])) {
  $_SESSION['x'] = str_repeat('x', 1024*10);
}

echo session_id();
var_dump(session_module_feature());

usleep(20000);
</code>

=============== lazy_write: on, unsafe_lock: on ===============

<code>
$ ab -c 50 -n 20000 'http://localhost:1080/session.php?lazy_write=1&unsafe_lock=1'
This is ApacheBench, Version 2.3 <$Revision: 1430300 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking localhost (be patient)
Completed 2000 requests
Completed 4000 requests
Completed 6000 requests
Completed 8000 requests
Completed 10000 requests
Completed 12000 requests
Completed 14000 requests
Completed 16000 requests
Completed 18000 requests
Completed 20000 requests
Finished 20000 requests


Server Software:        Apache/2.2.22
Server Hostname:        localhost
Server Port:            1080

Document Path:          /session.php?lazy_write=1&unsafe_lock=1
Document Length:        246 bytes

Concurrency Level:      50
Time taken for tests:   10.556 seconds
Complete requests:      20000
Failed requests:        1983
   (Connect: 0, Receive: 0, Length: 1983, Exceptions: 0)
Write errors:           0
Total transferred:      11661983 bytes
HTML transferred:       4921983 bytes
Requests per second:    1894.61 [#/sec] (mean)
Time per request:       26.391 [ms] (mean)
Time per request:       0.528 [ms] (mean, across all concurrent requests)
Transfer rate:          1078.85 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.2      0       6
Processing:    21   26   4.4     25      66
Waiting:       18   26   4.2     25      66
Total:         21   26   4.4     26      67

Percentage of the requests served within a certain time (ms)
  50%     26
  66%     28
  75%     29
  80%     30
  90%     32
  95%     34
  98%     37
  99%     40
 100%     67 (longest request)
</code>

=============== lazy_write: off, unsafe_lock: off ===============

<code>
$ ab -c 50 -n 20000 'http://localhost:1080/session.php?lazy_write=0&unsafe_lock=0'
This is ApacheBench, Version 2.3 <$Revision: 1430300 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking localhost (be patient)
Completed 2000 requests
Completed 4000 requests
Completed 6000 requests
Completed 8000 requests
Completed 10000 requests
Completed 12000 requests
Completed 14000 requests
Completed 16000 requests
Completed 18000 requests
Completed 20000 requests
Finished 20000 requests


Server Software:        Apache/2.2.22
Server Hostname:        localhost
Server Port:            1080

Document Path:          /session.php?lazy_write=0&unsafe_lock=0
Document Length:        246 bytes

Concurrency Level:      50
Time taken for tests:   44.924 seconds
Complete requests:      20000
Failed requests:        1938
   (Connect: 0, Receive: 0, Length: 1938, Exceptions: 0)
Write errors:           0
Total transferred:      11661938 bytes
HTML transferred:       4921938 bytes
Requests per second:    445.19 [#/sec] (mean)
Time per request:       112.310 [ms] (mean)
Time per request:       2.246 [ms] (mean, across all concurrent requests)
Transfer rate:          253.51 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.1      0       2
Processing:    21  112 131.0     61    1547
Waiting:       20  112 131.0     61    1547
Total:         21  112 131.0     61    1547

Percentage of the requests served within a certain time (ms)
  50%     61
  66%     90
  75%    124
  80%    160
  90%    275
  95%    390
  98%    545
  99%    647
 100%   1547 (longest request)
</code>

===== Rejected Features =====