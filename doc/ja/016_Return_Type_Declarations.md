
====== PHP RFC: Return Type Declarations ======
  * Version: 2.0
  * Date: 2014-03-20
  * Author: Levi Morrison <levim@php.net>
  * Status: Implemented (PHP 7.0)
  * First Published at: https://wiki.php.net/rfc/returntypehinting
  * Migrated to: https://wiki.php.net/rfc/return_types

===== 導入 =====

多くの開発者が関数の戻り値型を宣言できるようにしたいと思っている。戻り値型宣言の基本的な着想は
少なくとも３つのRFCに含まれ、いくつかの他の場所（リファレンスを参照）で議論されてきた。このRFCは
シンプルな方法でこの目標を達成するために、以前のRFCとは異なるアプローチを提案する。

戻り値型を宣言することにはいくつかの動機とユースケースがある：
- サブタイプが、スーパータイプの期待された戻り値型を破壊しないようにする（これがどのように働くか
 詳細には「可変とシグネチャ検証」と「例」を参照）。特にインターフェースについて。
- 意図しない戻り値を防ぐ。
- 戻り値型の情報を、容易には無効化されない方法で文書化する。

===== 提案 =====

この提案はクロージャ、関数、ジェネレータ、メソッドを含む関数宣言に、任意の戻り値型の宣言を追加する。
このRFCは既存の型宣言を変更しないし、追加もしない。（「過去のRFCとの違い」を参照）

これが動く構文の短い例である：

```php
function foo(): array {
    return [];
}
```

他の例は「例」セクションにある。

現在と同様に、 _戻り値型宣言のないコードも継続して動作する。_ このRFCは、メソッドが戻り値型を
宣言した親メソッドを継承する場合のみ、戻り値型宣言を要求する。その他のケースでは省略して構わない。

==== 可変とシグネチャ検証 ====

継承時の戻り値型宣言の強制は不変である。これは、サブタイプが親メソッドをオーバーライドするとき、
子の戻り値型は親と完全に一致しなければならず、省略されないことを意味する。親が戻り値型を宣言していない
場合、子が宣言してもよい。

コンパイル時に不一致を検出した場合、（例えば不適切に戻り値型をオーバーライドしたクラス）```E_COMPILE_ERROR```
となる。関数の戻り時に型不一致を検出した場合、```E_RECOVERABLE_ERROR```となる。

共変戻り値型は適切な型であると見なされ、他の多くの言語（C++、Java他が共変戻り値型を使用している）
で使用されている。このRFCは元々共変戻り値型を提案していたが、いくつかの問題のため不変に変更された。
将来のいつか、共変戻り値型が追加されるかもしれない。

この可変性の話題は関数の戻り値型宣言についてであることに注意してほしい。これは以下が不変、あるいは
共変いずれかの戻り値型が有効であることを意味する：

```php
interface A {
    static function make(): A;
}
class B implements A {
    static function make(): A {
        return new B();
    }
}
```

クラス"B"は"A"を実装しているため、従ってこれは有効である。可変性は宣言された方をオーバーライド
する場合に許される型についてである：

```php
interface A {
    static function make(): A;
}
class B implements A {
    static function make(): B { // 親と完全に一致しなければならない。これはエラーとなる
        return new B();
    }
}
```

このRFCは不変の戻り値型のみを提案するため、上記のサンプルは動作しない。将来拡張されて許される
かもしれない。

==== 型宣言の場所 ====

戻り値型の情報を置くには、他のプログラミング言語で２つの主な慣習がある：
- 関数名の前
- 引数リストの閉じ括弧の後

先の場所は過去に提案され、そのRFCは拒否されるか取り消された。挙げられる問題としては、多くの開発者が
```function foo```で検索して"foo"の定義を見つけられるのを維持したいと望んだことである。
[functionキーワードの削除](http://marc.info/?t=141235344900003&r=1&w=2)についての最近の議論
でもこれを維持することの価値を再度強調するコメントがいくつかあった。

後の場所はいくつかの言語で使用されている。（[Hack](http://hacklang.org/)、[Haskell](http://www.haskell.org)、
[Go](https://golang.org/)、[Erlang](http://www.erlang.org/)、[ActionScript](http://www.adobe.com/devnet/actionscript.html)、
[TypeScript](http://www.typescriptlang.org/)、その他戻り値型を引数リストの後に置く言語全て）
特に、C++11もラムダや戻り値型の自動推定など、ある構文では戻り値型を引数リストの後に置く。

引数リストの後で戻り値型を宣言すれば、パーサの移行もなく衝突も少ない。

==== 参照の返却 ====

このRFCは参照の返却をする場合の"&"の場所を変更しない。以下の例は有効である：

```php
function &array_sort(array &$data) {
    return $data;
}

function &array_sort(array &$data): array {
    return $data;
}
```

==== 戻り値型でのNULLの不許可 ====

以下の関数を考えよう：

```php
function foo(): DateTime {
    return null; // 無効
}
```

"DateTime"を返すと宣言しているが、"null"を返している。PHPを含む多くの言語でこの種の状況は
よくある。2つの理由から、このRFCではこういった状況で返す"null"を意図的に許可していない：
- これは現在の引数型の振る舞いに沿うものである。引数に型宣言があるとき、"null"値は許可されていない。
 （引数が"null"をデフォルト値として持つ以外）
- デフォルトで"null"を許可することは型宣言の目的に反するはたらきがある。型宣言はその周りのコードについて
 考えるのを容易にする。もし"null"を許可するならば、プログラマは常に"null"の場合について心配する
 必要が出てくる。

[Nullable Types RFC](https://wiki.php.net/rfc/nullable_types)はこの欠点などに取り組むもの
である。

==== 戻り値型を宣言できないメソッド ====

クラスコンストラクタ、デストラクタ、クローンメソッドは戻り値を宣言しなくてよい。それぞれのエラーメッセージは
以下である：
- ```Fatal error: Constructor %s::%s() cannot declare a return type in %s on line %s```
- ```Fatal error: Destructor %s::__destruct() cannot declare a return type in %s on line %s```
- ```Fatal error: %s::__clone() cannot declare a return type in %s on line %s```

==== 例 ====

有効な使用法と無効な使用法のスニペットをいくつか挙げる。

=== 有効な使用の例 ===
```php
// 戻り値型のないメソッドをオーバーライド：
interface Comment {}
interface CommentsIterator extends Iterator {
    function current(): Comment;
}
```
```php
// ジェネレータを使用する

interface Collection extends IteratorAggregate {
    function getIterator(): Iterator;
}

class SomeCollection implements Collection {
    function getIterator(): Iterator {
        foreach ($this->data as $key => $value) {
            yield $key => $value;
        }
    }
}
```

=== 無効な使用の例 ===

エラーメッセージは現在のパッチに含まれている

----

```php
// 共変戻り値型：

interface Collection {
    function map(callable $fn): Collection;
}

interface Set extends Collection {
    function map(callable $fn): Set;
}
```

```Fatal error: Declaration of Set::map() must be compatible with Collection::map(callable $fn): Collection in %s on line %d```

----
```php
// 戻り型が型宣言と一致しない

function get_config(): array {
    return 42;
}
get_config();
```

```Catchable fatal error: Return value of get_config() must be of the type array, integer returned in %s on line %d```

----

```php
// intは有効な型宣言ではない

function answer(): int {
    return 42;
}
answer();
```

```Catchable fatal error: Return value of answer() must be an instance of int, integer returned in %s on line %d```

----

```php
// 戻り値型宣言のある場所でnullを返すことはできない

function foo(): DateTime {
    return null;
}
foo();
```

```Catchable fatal error: Return value of foo() must be an instance of DateTime, null returned in %s on line %d```

----

```php
// オーバーライド時に戻り値型がない

class User {}

interface UserGateway {
    function find($id): User;
}

class UserGateway_MySql implements UserGateway {
    // UserかUserのサブタイプを返さなければならない
    function find($id) {
        return new User();
    }
}
```

```Fatal error: Declaration of UserGateway_MySql::find() must be compatible with UserGateway::find($id): User in %s on line %d```

----

```php
// ジェネレータの戻り値型はジェネレータ、イテレータ、もしくはトラバーサブルとしてのみ宣言できる（コンパイル時の検査） Iterator or Traversable (compile time check)

function foo(): array {
    yield [];
}
```

```Fatal error: Generators may only declare a return type of Generator, Iterator or Traversable, %s is not permitted in %s on line %d```

==== 複数の戻り値型 ====

この提案は複数の戻り値型宣言を明確に許可しない。これはこのRFCのスコープ外であり、望む場合は
別のRFCが必要になるだろう。

その間もし複数の戻り値型を使用したいなら、単純に戻り値型宣言を省略し、PHPの素晴らしい動的性質に
頼ろう。

==== リフレクション ====

このRFCは、敢えてリフレクションのサポートを除外している。リフレクションの型情報について進行中の
RFCがあるからだ： [Add typehint accessors to ReflectionParameter](https://wiki.php.net/rfc/reflectionparameter.typehint)

==== 過去のRFCとの違い ====

この提案は過去のRFCといくつかの重要な点で異なっている：
- **戻り値型を引数リストの後に置く。** この決定について詳しい情報は「型宣言の場所」を参照してほしい。
- **現在の型オプションを維持する。** 過去の提案は"void"、"int"、"string"、または"scalar"といった
 新しい型を提案してきた。このRFCは新しい型を何も含んでいない。"self"と"parent"を戻り値型として
 使用できることに注意してほしい。
- **現在の検索パターンを維持する。** 今後も```function foo```で```foo```の定義を検索することが
 できる。従来のRFCは全てこの共通したワークフローを破壊するものだった。
- **全ての関数タイプで戻り値型を宣言できる。** Will Fitchの提案はメソッドのみで宣言できることを
 提案した。
- **キーワードを変更したり追加したりしない。** 過去のRFCは"nullable"やその他の新しいキーワードを
 提案してきた。今後も必要なのは```function```キーワードである。

===== その他の影響 =====

==== On Backward Compatiblity ====
This RFC is backwards compatible with previous PHP releases.

==== On SAPIs ====
There is no impact on any SAPI.

==== On Existing Extensions =====
The structs ''zend_function'' and ''zend_op_array'' have been changed; extensions that work directly with these structs may be impacted.

==== On Performance ====
An informal test indicates that performance has not seriously degraded. More formal performance testing can be done before voting phase.

===== Proposed PHP Version(s) =====
This RFC targets PHP 7.

===== Vote =====
This RFC modifies the PHP language syntax and therefore requires a two-third majority of votes.

Should return types as outlined in this RFC be added to the PHP language? Voting will end on January 23, 2015.
<doodle title="Typed Returns" auth="levim" voteType="single" closed="true">
   * Yes
   * No
</doodle>

===== Patches and Tests =====

Dmitry and I have updated the implementation to a more current master branch here: https://github.com/php/php-src/pull/997

This RFC was merged into the master branch (PHP 7) in commit [[https://git.php.net/?p=php-src.git;a=commit;h=638d0cb7531525201e00577d5a77f1da3f84811e|638d0cb7531525201e00577d5a77f1da3f84811e]].

===== Future Work =====
Ideas for future work which are out of the scope of this RFC include:

  * Allow functions to declare that they do not return anything at all (''void'' in Java and C)
  * Allow nullable types (such as ```php?DateTime```). This is discussed in the [[rfc:nullable_types|Nullable Types]] RFC.
  * Improve parameter variance. Currently parameter types are invariant while they could be contravariant. Change the E_STRICT on mismatching parameter types to E_COMPILE_ERROR.
  * Improve runtime performance by doing type analysis.
  * Update documentation to use the new return type syntax.

===== References =====
  * [[rfc:returntypehint2|Method Return Type-hints]] by Will Fitch; 2011. [[http://marc.info/?t=132443368800001&r=1&w=2|Mail Archive]].
  * [[rfc:returntypehint|Return Type-hint]] by Felipe; 2010. [[http://marc.info/?l=php-internals&m=128036818909738&w=2|Mail Archive]]
  * [[rfc:typehint|Return value and parameter type hint]] by Felipe; 2008. [[http://marc.info/?l=php-internals&m=120753976214848&w=2|Mail Archive]].
  * [[http://derickrethans.nl/files/meeting-notes.html#type-hinted-properties-and-return-values|Type-hinted properties and return values]] from meeting notes in Paris; Nov 2005.

In the meeting in Paris on November 2005 it was decided that PHP should have return type declarations and some suggestions were made for syntax. Suggestion 5 is nearly compatible with this RFC; however, it requires the addition of a new token ''T_RETURNS''. This RFC opted for a syntax that does not require additional tokens so ''returns'' was replaced by a colon.

The following (tiny) patch would allow the syntax in suggestion 5 to be used alongside the current syntax. This RFC does not propose that both versions of syntax should be used; the patch just shows how similar this RFC is to that suggestion from 2005.

https://gist.github.com/krakjoe/f54f6ba37e3eeab5f705

===== Changelog =====

  * v1.1: Target PHP 7 instead of PHP 5.7
  * v1.2: Disallow return types for constructors, destructors and clone methods.
  * v1.3: Rework Reflection support to use new ''ReflectionType'' class
  * v1.3.1: Rename ''ReflectionType::IS_''* constants to ''TYPE_''*, rename ''->getKind()'' to ''->getTypeConstant()''
  * v2.0: Change to invariant return types and omit reflection support
