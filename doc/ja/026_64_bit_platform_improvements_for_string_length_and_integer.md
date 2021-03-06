
====== PHP RFC: 64 bit platform improvements for string length and integer in zval ======
  * Version: 2.3
  * Date: 2014-05-06
  * Authors: Anatol Belski <ab@php.net>, Matt Ficken <mattficken@php.net>, Stephen A. Zarkos <szarkos@php.net>
  * Status: Accepted
  * First published at: https://wiki.php.net/rfc/size_t_and_int64


===== Introduction =====

Current PHP //zval// datatype implementation uses //long// datatype to handle signed integer and //int// datatype to handle string length. The most 64 bit capable platforms PHP is used on are **LP64** (Linux/most Unix-like), **LLP64** (Windows), **ILP64** (SPARC64). The data model used in PHP for the relevant datatypes on those platforms looks as follows

^  ^ string size ^ signed integer ^
^ Platform ^ //int// ^ //long// ^
| LP64 | 32 bit | 64 bit |
| LLP64 | 32 bit | 32 bit |
| ILP64 | 64 bit | 64 bit |
\\
Regarding PHP that means today, even on 64 bit platforms the following features lack on consistency

  * handling of strings >= 2^31
  * handling of 64 bit integers
  * large file support
  * handling of numeric 64 bit hash keys
\\
Thus, the current situation contains a roadblock on the way to the overall consistent 64 bit platforms support and further improvement of PHP.

To bring everything inline, a dynamic types model is being suggested. Besides the platform inconsistency elimination, this will enable several further moves (see the Future scope). Performance improvements for 64 bit platforms, just to name one, could be then the subject of a new RFC and would have the base for the further development.

===== Proposal =====

The following datatypes are suggested for use in //zval//

^ Platform ^ string size ^ signed integer ^
| LP64 | //size_t// | //int64_t// |
| LLP64 |  //size_t// | //<nowiki>__int64</nowiki>//  |
| ILP64 | //size_t// | //int64_t// |

\\

Those datatypes are guaranteed to be 64 bit which makes PHP work consistent on any 64 bit platforms. The usage of this datatypes is integrated into the appropriate places across PHP. The size of //zval//, with default field alignment, is expected to grow by 4 bytes on LLP64 platforms only.

For the consistent LFS support, a set of portable datatypes and macros was invented and integrated.

For consistent 64 bit numeric hash keys support, the appropriate 64 bit //unsigned// was integrated.

The usage of //long// datatype continues on 32 bit platforms. The only change there is the usage of //size_t// for the string length (which is effectively a 32 bit //unsigned//).

No further configure options are needed, the platform and compiler will be automatically recognized and an appropriate set of datatypes and macros will be activated.
\\

==== Relevant headers ====

  * [[http://git.php.net/?p=php-src.git;a=blob;f=Zend/zend_int.h;hb=refs/heads/str_size_and_int64 | Zend/zend_int.h]]
  * [[http://git.php.net/?p=php-src.git;a=blob;f=Zend/zend_types.h;hb=refs/heads/str_size_and_int64 | Zend/zend_types.h]]
  * [[http://git.php.net/?p=php-src.git;a=blob;f=Zend/zend_stream.h;hb=refs/heads/str_size_and_int64 | Zend/zend_stream.h]]
  * [[http://git.php.net/?p=php-src.git;a=blob;f=main/php_streams.h;hb=refs/heads/str_size_and_int64 | main/php_streams.h]]

==== New portable datatypes ====

^ Old datatype ^ New datatype ^ Comment ^
| //int//, //uint//, //zend_uint//, //size_t//, //long// | //zend_size_t// | Overall datatype to handle object size and string length in //zval//. Aliased as //php_size_t// in php.h |
| //long// | //zend_int_t// | Overall datatype to handle integers in //zval//.  Aliased with //php_int_t// in php.h |
| //ulong//, //unsigned long// | //zend_uint_t// | Overall datatype to handle numeric hash indexes and other situations with need on an unsigned. Aliased with //php_uint_t// in php.h |
| //off_t//, //_off_t// | //zend_off_t// | Overall datatype to handle file offsets. Corresponding portable macros have to be used. |
| //struct stat//, //struct _stat//, //struct _stat64// | //zend_stat_t// | Overall datatype to handle the FS info. Corresponding portable macros have to be used. |
\\

==== New portable macros for LFS support ====
^ function(s) ^ Alias ^ Comment ^
| stat, _stat64 | zend_stat | for use with //zend_stat_t// |
| fstat, _fstat64 | zend_fstat | for use with //zend_stat_t// |
| lseek, _lseeki64 | zend_lseek | for use with //zend_off_t// |
| ftell, _ftelli64 | zend_ftell | for use with //zend_off_t// |
| fseek, _fseeki64 | zend_fseek | for use with //zend_off_t// |
\\

==== New portable macros for integers ====
^ function(s) ^ Alias ^ Comment ^
| snprintf with "%ld" or "%lld", _ltoa_s, _i64toa_s | ZEND_ITOA | for use with //zend_int_t//|
| atol, atoll, _atoi64 | ZEND_ATOI | for use with //zend_int_t// |
| strtol, strtoll, _strtoi64 | ZEND_STRTOI| for use with //zend_int_t// |
| strtoul, strtoull, _strtoui64 | ZEND_STRTOUI|for use with //zend_int_t// |
| abs, llabs, _abs64 | ZEND_ABS |for use with //zend_int_t// |
| - | ZEND_INT_MAX | Aliased with PHP_INT_MAX in php.h, replaces LONG_MAX where appropriate |
| - | ZEND_INT_MIN | Aliased with PHP_INT_MIN in php.h, replaces LONG_MIN where appropriate |
| - | ZEND_UINT_MAX | ULONG_MAX |
| - | SIZEOF_ZEND_INT | Replaces SIZEOF_ZEND_LONG where appropriate |
| - | ZEND_SIZE_MAX | Max value of //zend_size_t// |
\\

==== Semantical macro renamings ====

^ Old ^ New ^ Comment ^
| Z_STRLEN | Z_STRSIZE | as well the whole Z_STRLEN_* family |
| IS_LONG | IS_INT | |
| RETURN_LONG | RETURN_INT | |
| RETVAL_LONG | RETVAL_INT | |
| Z_LVAL | Z_IVAL | as well the whole Z_LVAL_* family |
| LITERAL_LONG | LITERAL_INT | |
| REGISTER_LONG_CONSTANT | REGISTER_INT_CONSTANT | |
| REGISTER_MAIN_LONG_CONSTANT | REGISTER_MAIN_INT_CONSTANT | |
| ZEND_SIGNED_MULTIPLY_LONG | ZEND_SIGNED_MULTIPLY_INT | |
| ... | ... | |

Generally speaking, every occurence mentioning "long" in macros or function names should be replaced with a corresponding neutral keyword, suggested "int", in further like "lval" with "ival", etc.

\\

==== Accepting values with zend_parse_parameters() ====

^ Old ^ New ^ Comment ^
| "s" | "S" | accept string argument, the length has to be declared as //php_size_t// (or //zend_size_t//) |
| "p" | "P" | accept path argument, the length has to be declared as //php_size_t// (or //zend_size_t//) |
| "l" | "i" | to accept integer argument, the internal var has to be declared as //php_int_t// (inside PHP) or //zend_int_t// (inside Zend) |
| "L" | "I" | to accept integer argument with range check, the internal var has to be declared as //php_int_t// (inside PHP) or //zend_int_t// (inside Zend)|

\\

=== Clang PHP checker ===

Johannes Schlüter did a great work on developing a [[https://github.com/johannes/clang-php-checker | clang plugin]] allowing static analyze of the zpp calls. Steps was undertaken to make it [[http://windows.php.net/downloads/snaps/ostc/clang_checker | available on windows]] as well.

\\

==== spprintf formats ====

New spprintf modifier 'p' was implemented to platform independently output //php_int_t// datatype. That modifier can be used with 'd', 'u', 'x' and 'o' printf format specs with spprintf, snprintf and the wrapping printf implementations.

=== Portable macros to use with printf ==

^ Format spec ^ Macros ^ Comment ^
| %I64d, "%" PRId64, %ld | ZEND_INT_FMT | for use with //zend_int_t// |
| %I64u, "%" PRIu64, %lu | ZEND_UINT_FMT | for use with //zend_uint_t// |

This modifier is of course available in all the spprintf/snprintf derivatives. Any of the introduced new datatypes can be used with the appropriate format spec.


\\

===== Backward Incompatible Changes =====

  * //long// and //int// in //zval// have to be replaced with the new integer datatypes
  * code working with hashes and arrays has to use //php_uint_t//
  * code working with filesystem objects has to use appropriate portable macros and datatypes
  * code working with string lengths has to predominatly use php_size_t
  * 'l', 'L', 's', 'p' parameter formats aren't available anymore
  * Z_STRLEN*, *LONG*, etc. older macros aren't available anymore

===== Proposed PHP Version =====

PHP6 (or whatever next major is called)


===== Impacts =====

  * possible collisions with dependency libraries using 32 bit integer datatypes (range checks needed)
  * existing extensions have to adapt zend_parsing_parameters() format
  * existing extensions have to use new APIs and macros for string length and integer
  * in the user space some function names need to be adapted on the new semantics, for instance long2ip() should be int2ip()


===== Open Issues =====

Some dead SAPIs are present in the core. They was not ported. A decision based on whether the authors are willing to support them has to be met. Then porting or removal of those SAPIs can be scheduled. The separate RFC https://wiki.php.net/rfc/removal_of_dead_sapis was created to handle this issue.

===== Unaffected PHP Functionality =====

It has to do with squeezing anything possible from the 64 bit platforms, for maximal PHP benefit. No real features are going to be changed, removed or added to the PHP language.

===== Performance =====

Performance test done on the base of master and str_size_and_int64 can be found here [[http://windows.php.net/downloads/snaps/ostc/pftt/perf/results-20140411-masterr6c2f7bc-str_size_and_int64raa0c920.html]]. There is no performance difference between two (between 0-2%).

/*===== Some performance comparsion =====


^ PHP Version                   ^ Wordpress            ^ Drupal	           ^ Joomla ^

| str_size_and_int64-x86       | NoCache:     68      | NoCache:     70      | NoCache:     53 |
| -                            | Cache:       284     | Cache:       393     | Cache:       127 |
| php-5.5.8-nts-Win32-VC11-x86 | NoCache:     67      | NoCache:     69      | NoCache:     53 |
| -                            | Cache:       280     | Cache:       390     | Cache:       125 |
| str_size_and_int64-x64       | NoCache:     58      | NoCache:     64      | NoCache:     50 |
| -                            | Cache:       <nowiki>313*</nowiki>    | Cache:       <nowiki>348*</nowiki>    | Cache:       <nowiki>100*</nowiki> |
| php-5.5.8-nts-Win32-VC11-x64 | NoCache:     59      | NoCache:     65      | NoCache:     51 |
| -                            | Cache:       <nowiki>270*</nowiki>    | Cache:       <nowiki>**</nowiki> | Cache:       <nowiki>**</nowiki> |

The numbers here are the test scores one already might have seen in the other [[http://windows.php.net/downloads/snaps/ostc/pftt/perf/ | performance tests]].\\
\\
<nowiki>*, **</nowiki> Some issues with the x64 versions of 5.5.8 and str_size_and_int64 when testing with opcache enabled. However issues of this kind are well known on windows (for instance [[https://bugs.php.net/bug.php?id=64926 | #64926]]) and are due to some unluckily choosen memory address. So the cause persists in the mainstream and is not because of this patch.
\\*/

===== Migration path for PECL extensions =====

[[http://git.php.net/?p=php-src.git;a=tree;f=compat;hb=refs/heads/str_size_and_int64 | Tutorial, tools and compatibility header]] to ease the migration of the PECL extensions are available. The goal is to make the same source in the new semantic compatible with older PHP versions.

==== Example on accepting parameters with zpp ====

<code c>
	php_int_t i0, i1;
	char *s0, p0;
	php_size_t s0_len, p0_len;

	if(zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "iISP", &i0, &i1, &s0, &s0_len, &p0, &p0_len) == FAILURE) {
		return;
	}
</code>

==== Example on printf specs usage ====

<code c>
	php_int_t i0;

	if(zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "i", &i0) == FAILURE) {
		return;
	}

	if (INT_MAX < i0 || INT_MIN > i0) {
		php_error_docref(NULL TSRMLS_CC, E_WARNING, "Value '" ZEND_INT_FMT "' is out of range", i0);
		return;
	}
</code>

==== Example on printf specs usage (no BC) ====

<code c>
	php_error_docref(NULL TSRMLS_CC, E_WARNING, "Value '%pd' is out of range", i0);
</code>

==== Example proper check of string size ====

<code c>
	char *s0;
	php_size_t s0_len;
	php_int_t max_len;

	if(zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "Si", &s0, &s0_len, &max_len) == FAILURE) {
		return;
	}

	if (max_len >= 0 && s0_len > max_len) {
		return;
	}
</code>
\\

==== Example with some renamed macros ====

<code c>
char *dup_substr(zval *s, zval *i)
{
	php_size_t len;
	php_int_t max;
	char ret;

	convert_to_string(s);
	convert_to_int(i);

	len = Z_STRSIZE_P(s);
	max = Z_IVAL(i);

	if (max < 0 || max >= 0 && max > len)
	{
	return NULL;
	}

	ret = emalloc((max + 1) * sizeof(char));

	if (!ret) {
		return NULL;
	}

	memmove(ret, Z_STRVAL_P(s), max);
	ret[max] = '\0';

	return ret;
}
</code>
===== Future Scope =====

  * in far perspective - easier to implement 128 bit support
  * in near perspective - excellent base for 64 bit performance optimization
  * easier integration on rarely used platforms
  * easier integration on new platforms

===== Vote =====


<doodle title="Accept this RFC for PHP6 (or whatever next major is called)" auth="user" voteType="single" closed="true">
   * Yes
   * No
</doodle>
\\
<doodle title="Merge strategy" auth="user" voteType="multi" closed="true">
   * after the vote, master
   * phpng
</doodle>
\\
The RFC is considered approved with 50%+1 acceptance.
\\
The vote starts on May 13th 2014 and ends <del>on May 20th 2014</del> **due to the issues with the mailing lists, the vote period is extended till May 26th 2014**.


===== Patches and Tests =====

[[http://git.php.net/?p=php-src.git;a=shortlog;h=refs/heads/str_size_and_int64 | Feature branch]]\\
[[http://git.php.net/?p=php-src.git;a=shortlog;h=refs/heads/str_size_and_int64_56_backport | Backport branch of PHP 5.6]] (for comparisons only)\\
[[http://windows.php.net/downloads/snaps/str_size_and_int64/ | Windows builds]]\\
[[http://131.107.220.66/PFTT-Results/STR_SIZE_AND_INT64/ | Test reports]]\\

===== References =====

[[rfc/string-size_t/progress| Patch progress page]]\\
[[http://git.php.net/?p=php-src.git;a=tree;f=compat;hb=refs/heads/str_size_and_int64 | PECL porting docs'n'tools]]\\
\\
[[http://grokbase.com/t/php/php-internals/135z59f0kz/5-next-integer-and-string-type-modifications | Initial discussion brought up by Anthony Ferrara]]\\
[[http://grokbase.com/p/php/php-internals/137354x7hf/string-size-refactor-progress| Discussion after implementation start]]\\
[[http://grokbase.com/t/php/php-internals/141a4gns13/rfc-64-bit-platform-improvements-for-string-length-and-integer | Discussions of the previous RFC version]]\\

===== Implementation =====

After the project is implemented, this section should contain
  - the version(s) it was merged to
  - a link to the git commit(s)
  - a link to the PHP manual entry for the feature

===== Rejected Features =====

-