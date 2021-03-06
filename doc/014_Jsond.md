====== PHP RFC: Replacing current json extension with jsond ======
  * Version: 0.2
  * Date: 2015-01-01
  * Author: Jakub Zelenka, bukka@php.net
  * Status: Implemented (PHP 7.0)


===== Introduction =====
The current Json Parser in the json extension does not have a free license which is a problem for many Linux distros. This has been referenced at [[https://bugs.php.net/63520|Bug #63520]]. That results in not packaging json extension in the many Linux distributions.

The extension code is very old and without a maintainer which makes it difficult for further improvements. There also are some implementation decisions that makes the performance worse than it should be (see benchmarks).

===== Proposal =====
The proposal is to replace the current json extension with code that is based on the PECL jsond extension (it's not exactly the same code as there are some modifications for PHP 7).

===== Backward Incompatible Changes =====
  * Rejected ECMA-404 and RFC 7159 incompatible number formats in json_decode string
    * top level (PHP json_decode check): 07, 0xff, .1, -.1
    * all (JSON_Parser): [1.], [1.e1]

The first part (top level) is caused by different handling of integer and double because it's not handled by JSON_Parser. This is clearly a bug.

The second part is happening for all numbers. However it's not valid by the RFC 7159 as it says in  https://tools.ietf.org/html/rfc7159#section-6 :

<code>
number = [ minus ] int [ frac ] [ exp ]
decimal-point = %x2E       ; .
digit1-9 = %x31-39         ; 1-9
e = %x65 / %x45            ; e E
exp = e [ minus / plus ] 1*DIGIT
frac = decimal-point 1*DIGIT
int = zero / ( digit1-9 *DIGIT )
minus = %x2D               ; -
plus = %x2B                ; +
zero = %x30                ; 0
</code>

Please note the definition of frac that specifies that at least one digit after decimal point is required.

===== Proposed PHP Version(s) =====
PHP 7

===== RFC Impact =====
==== To SAPIs ====
No impact

==== To Existing Extensions ====
In addition to the backward incompatible changes for float number format the following internal changes have been done:
  * removed JSON_parser.h header
  * removed utf8_decode.h header
  * macro JSON_PARSER_DEFAULT_DEPTH renamed to PHP_JSON_PARSER_DEFAULT_DEPTH
  * error codes constants moved to php_json.h
    * enum error_codes renamed to php_json_error_codes (typedef)
    * ext global error_code type changed from int to php_json_error_codes

==== To Opcache ====
No impact

==== New Constants ====
None

===== Open Issues =====
None

===== Future Scope =====
  * Improving performance for encoder and decoder
  * Better error reporting (location of the error)

===== Vote =====
50%+1 majority. The vote is a straight Yes/No vote.

<doodle title="Should jsond based extension replace the current json extension in PHP 7?" auth="bukka" voteType="single" closed="true">
   * Yes
   * No
</doodle>

The vote started on 2015-01-26 at 17:00 UTC and ended on 2015-02-01 at 17:00 UTC.

===== Patches and Tests =====
Pull request for master branch: https://github.com/php/php-src/pull/993

The summary of the benchmarks is at https://github.com/bukka/php-jsond-bench/blob/master/reports/0001/summary.md. There also is a link for runs (measured data).

===== Implementation =====
After the project is implemented, this section will contain
  - Merged to the master branch and will be released in PHP 7

===== References =====
There is an old RFC for [[https://wiki.php.net/rfc/free-json-parser|free json parser]] that proposed jsonc as a replacement for json. There were some concerns about the compatibility and performance https://www.mail-archive.com/internals@lists.php.net/msg66658.html .

Executed code showing the current non-conformant number parsing: http://3v4l.org/2D72Q
