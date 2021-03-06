====== PHP RFC: Null Coalesce Operator ======
  * Version: 0.2.3
  * Date: 2014-09-06, last updated 2014-09-16
  * Author: Andrea Faulds <ajf@ajf.me>
  * Contributor: Nikita Popov <nikic@php.net> (initial patch)
  * Status: Implemented (PHP 7)
  * First Published at: http://wiki.php.net/rfc/isset_ternary


===== 導入 =====

PHPはWebにフォーカスしたプログラミング言語であり、ユーザデータの処理はよくあることである。
そういった処理では、何かの存在を確認し、それが存在しない場合デフォルトの値を使うすることがよくある。
が、このための最もシンプルな方法はこのようになり、```isset($_GET['mykey']) ? $_GET['mykey'] : ""```
不要に冗長なものである。短い3項演算子、```?:```はこの方法をより便利に提供する： ```$_GET['mykey'] ?: ""```
しかし、値が存在しない場合```E_NOTICE```を発生するため、これは良い方法ではない。この問題のため、
このよくあるパターンをより容易にするために、ifsetor()演算子の幾つかや、```?:```の振る舞いを変更することが
たびたび要求されてきた。（リファレンスを参照）

===== 提案 =====

いわゆるcoalesce、つまり```??```演算子を追加する。これは存在しNULLでなければ、最初のオペランドの結果を
返却し、それ以外では、第2のオペランドを返却する。これは```$_GET['mykey'] ?? ""```が完全に
安全でE_NOTICEを発生しないことを意味する。使い方の例：

```php
// Fetches the request parameter user and results in 'nobody' if it doesn't exist
$username = $_GET['user'] ?? 'nobody';
// equivalent to: $username = isset($_GET['user']) ? $_GET['user'] : 'nobody';

// Calls a hypothetical model-getting function, and uses the provided default if it fails
$model = Model::get($id) ?? $default_model;
// equivalent to: if (($model = Model::get($id)) === NULL) { $model = $default_model; }

// Parse JSON image metadata, and if the width is missing, assume 100
$imageData = json_decode(file_get_contents('php://input'));
$width = $imageData['width'] ?? 100;
// equivalent to: $width = isset($imageData['width']) ? $imageData['width'] : 100;
```

チェーンも可能である：

```php
$x = NULL;
$y = NULL;
$z = 3;
var_dump($x ?? $y ?? $z); // int(3)

$x = ["yarr" => "meaningful_value"];
var_dump($x["aharr"] ?? $x["waharr"] ?? $x["yarr"]); // string(16) "meaningful_value"
```

この例は3項演算子、or論理演算子との優先度を示しており、C#と同じとなっている：

```php
var_dump(2 ?? 3 ? 4 : 5);      // (2 ?? 3) ? 4 : 5        => int(4)
var_dump(0 || 2 ?? 3 ? 4 : 5); // ((0 || 2) ?? 3) ? 4 : 5 => int(4)
```

この例は短絡演算子としての働きを示している：

```php
function foo() {
    echo "executed!", PHP_EOL;
}
var_dump(true ?? foo()); // 短絡であるため、bool(true)を出力し"executed!"は登場しない
```

===== Proposed PHP Version(s) =====

This proposed for the next PHP x, which at the time of this writing would be PHP 7.

===== Patches and Tests =====

A pull request with a working implementation and test, targeting master, is here: https://github.com/php/php-src/pull/824

The original patch was graciously provided by Nikita Popov.

===== Vote =====

As this is a language change, a 2/3 majority is required. A straight Yes/No vote is being held.

Voting started on 2014-09-20 and ended on 2014-09-27.

<doodle title="Approve Null Coalesce Operator RFC and merge patch into master?" auth="ajf" voteType="single" closed="true">
   * Yes
   * No
</doodle>

===== Implementation =====

Merged into master (which will be PHP 7): https://github.com/php/php-src/commit/2d069f640e6cccfa3ba8b1e4f375ade20fb33f64

Not yet documented.

===== References =====

There have been several previous discussions and proposals about adding an ifsetor operator with similar behaviour, or changing the behaviour of ''?:'', indeed there has been at least one each year:

  * 2004: http://marc.info/?t=108214447100002&r=1&w=2 and http://marc.info/?t=108214551500002&r=1&w=2
  * 2005: http://marc.info/?t=111826997700001&r=1&w=2, http://marc.info/?t=111829868900001&r=1&w=2 and http://marc.info/?t=113069916200003&r=1&w=2
  * 2006: http://marc.info/?t=114669117400001&r=1&w=2
  * 2007: http://marc.info/?l=php-internals&m=118208519027980&w=2
  * 2008: [[rfc:ifsetor|RFC: ifsetor()]] - http://marc.info/?t=120136702900003&r=1&w=2
  * 2009: http://marc.info/?l=php-internals&m=124214617109310&w=2
  * 2011: http://marc.info/?l=php-internals&m=130158916712977&w=2
  * 2013: http://marc.info/?t=138547713500001&r=1&w=2

This list is quite probably incomplete.

Operator precedence in C#: http://msdn.microsoft.com/en-us/library/6a71f45d.aspx

===== Rejected Features =====
Keep this updated with features that were discussed on the mail lists.

===== Changelog =====

  * v0.2.3 - Added short-circuit example
  * v0.2.2 - Added precedence example
  * v0.2.1 - Added chaining example
  * v0.2 - Overhauled proposal, proposing new operator ''??'' instead of an extension to ''?:''
  * v0.1 - Created
