====== PHP RFC: Fix handling of custom session handler return values ======
  * Version: 1.0
  * Date: 2014-05-15
  * Author: Sara Golemon, pollita@php.net
  * Status: Voting
  * First Published at: http://wiki.php.net/rfc/session.user.return-value

===== Introduction =====

The logic in ext/session/mod_user.c is just plain wrong.

http://us2.php.net/session_set_save_handler
   * "[For all callback functions] Return value is TRUE for success, FALSE for failure."

Yet in ext/session/mod_user.c:

   PS_FUNC(user) {
    /* blah blah */
    zval *retval = ps_call_handler(PSF(func), argc, argc);
    if (retval) {
       convert_to_long(retval);
       return Z_LVAL_P(retval);
    }
    return FAILURE;
  }

Remembering that SUCCESS == 0, and FAILURE == -1

So what does that mean for return values?
  * return false => return (int)false => return 0 => return SUCCESS
  * return true => return (int)true) => return 1 => return NeitherSUCCESSnorFAILURE

===== Proposal =====

Change the FINISH macro in session.c to map true to SUCCESS, false to FAILURE, warn (and fail) for integer -1, and warn (but succeed) for anything else.

===== Backward Incompatible Changes =====

  * Anyone currently returning -1 for failure (because that's what ends up working as expected) now gets a warning.
  * Anyone returning false for failure now actually goes down the failure path (and this might be unexpected due to how long this has been wrong).

===== Proposed PHP Version(s) =====

Either 5.next (5.7?) or phpng due to the age of this bug.

===== Vote =====

<doodle title="Fix custom session save handler using the patch as written" auth="user" voteType="multi" closed="true">
   * Yes
   * No
</doodle>


<doodle title="Which version?" auth="user" voteType="multi" closed="true">
   * 5.6 or later
   * 5.7 or later
   * 6.0 or later
</doodle>

Voting Opened Jun 10, 2014
Voting Closes Jun 24, 2014

===== Implementation =====

  * https://github.com/sgolemon/php-src/compare/session.user.return-value?expand=1
