====== PHP RFC: Remove PHP 4 Constructors ======
  * Version: 1.0
  * Date: 2014-11-17
  * Author: Levi Morrison <levim@php.net>
  * Contributors: Andrea Faulds <ajf@ajf.me>
  * Status: Deprecation Implemented (in PHP 7.0)
  * First Published at: http://wiki.php.net/rfc/remove_php4_constructors

===== Introduction =====
PHP 4 constructors are methods that have the same name as the class they are defined in. PHP 5 preserved the ability to use PHP 4 style constructors in some cases; the strangeness of when the constructor is and isn't used increases the mental model for programmers.

<PHP>
class Filter {

    // A constructor in PHP 4 and 5
    function filter($a) {}
}
</PHP>

<PHP>
// Namespaced classes do not recognize PHP 4 constructors
namespace NS;
class Filter {

    // Not a constructor
    function filter($a) {}
}
</PHP>

<PHP>
// Defining __construct and filter makes filter a normal method
class Filter {
    function __construct() {}

    // Not a constructor
    function filter($a) {}
}

//But defining filter first raises an E_STRICT
class Filter {

    // Not a constructor…
    function filter($a) {}

    // This raises E_STRICT
    function __construct() {}
}
</PHP>


===== Proposal =====
PHP 7 will emit ''E_DEPRECATED'' whenever a PHP 4 constructor is defined. When the method name matches the class name, the class is not in a namespace, and a PHP 5 constructor (''<nowiki>__construct</nowiki>'') is not present then an ''E_DEPRECATED'' will be emitted. PHP 8 will stop emitting ''E_DEPRECATED'' and the methods will not be recognized as constructors.

PHP 7 will also stop emitting ''E_STRICT'' when a method with the same name as the class is present as well as ''<nowiki>__construct</nowiki>''.

Refer to the [[#examples|Examples]] section to see how code is impacted.

==== Backward Incompatible Changes ====
Since an ''E_DEPRECATED'' will be emitted in PHP 7 there may be some backwards compatibility breaks when people use custom error handlers.

In PHP 8 recognition for old constructors will be outright removed, meaning that anything without a ''<nowiki>__construct</nowiki>'' will not work the same way it used to. The old-style constructor will be considered a normal method and will not be called when the object is constructed. The fix is to rename the constructor to ''<nowiki>__construct</nowiki>''.

==== Examples ====
<PHP>
class Filter {

    // PHP 5: filter is a constructor
    // PHP 7: filter is a constructor and E_DEPRECATED is raised
    // PHP 8: filter is a normal method and is not a constructor; no E_DEPRECATED is raised
    function filter($a) {}
}

$filter = new ReflectionMethod('Filter', 'filter');

// PHP 5: bool(true)
// PHP 7: bool(true)
// PHP 8: bool(false)
var_dump($filter->isConstructor());
</PHP>

<PHP>
// function filter is not used as constructor in PHP 5+
class Filter {
    // PHP 5: E_STRICT "Redefining already defined constructor"
    // PHP 7: No error is raised
    // PHP 8: No error is raised
    function filter($a) {}
    function __construct() {}
}
</PHP>

<PHP>
// function filter is not used as constructor in PHP 5+
class Filter {
    function __construct() {}

    // PHP 5.0.0 - 5.2.13, 5.3.0 - 5.3.2: E_STRICT "Redefining already defined constructor"
    // PHP 5.2.14 - 5.2.17, 5.3.3 - 5.6: No error is raised
    // PHP 7: No error is raised
    // PHP 8: No error is raised
    function filter($a) {}
}
</PHP>

<PHP>
class Filter {
    // PHP 5: filter is a constructor
    // PHP 7: filter is a constructor and E_DEPRECATED is raised
    // PHP 8: filter is a normal method and is not a constructor; no E_DEPRECATED is raised
    function filter() {}
}

class FilterX extends Filter {

    function __construct() {
        // PHP 5: Filter::filter is called; no error
        // PHP 7: Filter::filter is called; no error
        // PHP 8: "Fatal error: Cannot call constructor"
        parent::__construct();
    }

}

new FilterX();
</PHP>

===== Voting =====
This RFC targets PHP 7 and PHP 8. Please read the RFC to understand what is being proposed.

This RFC requires 2/3 vote in favor of deprecating and removing PHP 4 style constructors to pass.

Do you vote to remove PHP 4 constructors as outlined in this RFC?

<doodle title="remove_php4_constructors" auth="levim" voteType="single" closed="true">
   * Yes
   * No
</doodle>

Voting will close on the evening (UTC-7) of March 6th.

===== Patches and Tests =====
An implementation based on the master branch can be found here: https://github.com/php/php-src/pull/1061
