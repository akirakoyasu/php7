
====== PHP RFC: Combined Comparison (Spaceship) Operator ======
  * Version: 0.2.1
  * Date: 2014-02-12 (original), 2015-01-19 (v0.2)
  * Authors: Davey Shafik <davey@php.net>, Andrea Faulds <ajf@ajf.me>, Stas Malyshev <stas@php.net>
  * Status: Accepted
  * First Published at: http://wiki.php.net/rfc/combined-comparison-operator

===== Introduction =====

This RFC adds a new operator for combined comparison. Similar to ''strcmp()'' or ''version_compare()'' in behavior, but it can be used on all generic PHP values with the same semantics as ''%%<, <=, ==, >=, >%%''.

===== Proposal =====

Add a new operator ''(expr) %%<=>%% (expr)'',  it returns 0 if both operands are equal, 1 if the left is greater, and -1 if the right is greater. It uses exactly the same comparison rules as used by our existing comparison operators: ''%%<, <=, ==, >=%%'' and ''>''. (See the manual for [[http://php.net/manual/en/language.operators.comparison.php|details]])

This [[https://en.wikipedia.org/wiki/Three-way_comparison|"three-way comparison operator"]], also known as the "spaceship operator" (a common name in other languages), works on all standard PHP values. It exists in other languages: [[http://perldoc.perl.org/perlop.html#Equality-Operators|Perl]], [[http://ruby-doc.org/core-1.9.3/Comparable.html|Ruby]], and Groovy.

For [[http://perldoc.perl.org/perlop.html#Operator-Precedence-and-Associativity|consistency with Perl]], it has the same precedence as ''=='' and ''!=''.

It is implemented by using the result of the existing internal ''compare_function'' that underlies the other comparison operators. The existing comparison operators could be considered mere shorthands for ''%%<=>%%'':

^ operator          ^ ''%%<=>%%'' equivalent     ^
| ''$a < $b''       | ''%%($a <=> $b) === -1%%'' |
| ''%%$a <= $b%%''  | ''%%($a <=> $b) === -1 || ($a <=> $b) === 0%%'' |
| ''$a == $b''      | ''%%($a <=> $b) === 0%%''  |
| ''$a != $b''      | ''%%($a <=> $b) !== 0%%''  |
| ''%%$a >= $b%%''  | ''%%($a <=> $b) === 1 || ($a <=> $b) === 0%%'' |
| ''$a > $b''       | ''%%($a <=> $b) === 1%%''  |

Here are some examples of its behaviour:

<code php>
// Integers
echo 1 <=> 1; // 0
echo 1 <=> 2; // -1
echo 2 <=> 1; // 1

// Floats
echo 1.5 <=> 1.5; // 0
echo 1.5 <=> 2.5; // -1
echo 2.5 <=> 1.5; // 1

// Strings
echo "a" <=> "a"; // 0
echo "a" <=> "b"; // -1
echo "b" <=> "a"; // 1

echo "a" <=> "aa"; // -1
echo "zz" <=> "aa"; // 1

// Arrays
echo [] <=> []; // 0
echo [1, 2, 3] <=> [1, 2, 3]; // 0
echo [1, 2, 3] <=> []; // 1
echo [1, 2, 3] <=> [1, 2, 1]; // 1
echo [1, 2, 3] <=> [1, 2, 4]; // -1

// Objects
$a = (object) ["a" => "b"];
$b = (object) ["a" => "b"];
echo $a <=> $b; // 0

$a = (object) ["a" => "b"];
$b = (object) ["a" => "c"];
echo $a <=> $b; // -1

$a = (object) ["a" => "c"];
$b = (object) ["a" => "b"];
echo $a <=> $b; // 1

// only values are compared
$a = (object) ["a" => "b"];
$b = (object) ["b" => "b"];
echo $a <=> $b; // 0
</code>

Usort Example:

<code php>
if (($handle = fopen("people.csv", "r")) !== FALSE) {
    while (($row = fgetcsv($handle, 1000, ",")) !== FALSE) {
         $data[] = $row;
    }
    fclose($handle);
}

// Sort by last name:
usort($data, function ($left, $right) {
     return $left[1] <=> $right[1];
});
</code>

===== Usefulness =====

It makes writing ordering callbacks for use with ''usort()'' easier. Commonly, users write poor or incorrect ordering functions like this one:

<code php>
function order_func($a, $b) {
    return $a >= $b;
}
</code>

When users do write correct ordering functions, they have to be quite verbose:

<code php>
function order_func($a, $b) {
    return ($a < $b) ? -1 : (($a > $b) ? 1 : 0);
}
</code>

This becomes particularly bad when sorting by multiple columns lexicographically.

With this operator, you can easily write proper ordering functions, like this one:

<code php>
function order_func($a, $b) {
    return $a <=> $b;
}
</code>

Sorting by multiple columns is simpler now, too:

<code php>
function order_func($a, $b) {
    return [$a->x, $a->y, $a->foo] <=> [$b->x, $b->y, $b->foo];
}
</code>

Or:

<code php>
function order_func($a, $b) {
    return ($a->$x <=> $b->x) ?: ($a->y <=> $b->y) ?: ($a->foo <=> $b->foo);
}
</code>

It is also useful in some other contexts.

===== Backward Incompatible Changes =====

This introduces no backwards incompatible changes.

===== Proposed PHP Version(s) =====

The next major version of PHP, currently PHP 7.0.

===== New Constants =====

A ''T_SPACESHIP'' constant for use with ext/tokenizer has been added.

===== Unaffected PHP Functionality =====

All existing comparison operators are unaffected by this addition.

===== Future Scope =====

None.

===== Vote =====

Voting started on 2015-02-02 and will to end on 2015-02-16. As this adds to the PHP language (and hence affects the PHP language specification) a 2/3 majority is required for acceptance. It is a Yes/No vote to accepting the RFC and merging the patch.

<doodle title="Accept the Combined Comparison (Spaceship) Operator RFC and merge patch into master?" auth="stas" voteType="single" closed="true">
   * Yes
   * No
</doodle>


===== Patches and Tests =====

A patch targeting php-src master is here: https://github.com/php/php-src/pull/1007

There is not yet a language specification patch.

Shafik's original patch against 5.6 was here: https://github.com/dshafik/php-src/compare/add-spaceship-operator

===== Changelog =====

  * v0.2.1 - Clarity on type-juggling behaviour and relation to other comparison operators
  * v0.2 - Updated, retargeted to PHP 7 by Andrea
  * v0.1 - Initial version by Shafik