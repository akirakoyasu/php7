
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

==== Returning by Reference ====

This RFC does not change the location of ''&'' when returning by reference. The following examples are valid:
```php
function &array_sort(array &$data) {
    return $data;
}

function &array_sort(array &$data): array {
    return $data;
}
```

==== Disallowing NULL on Return Types ====
Consider the following function:

```php
function foo(): DateTime {
    return null; // invalid
}
```

It declares that it will return ''DateTime'' but returns ''null''; this type of situation is common in many languages including PHP. By design this RFC does not allow ''null'' to be returned in this situation for two reasons:

  - This aligns with current parameter type behavior. When parameters have a type declared, a value of ''null'' is not allowed ((Except when the parameter has a ''null'' default)).
  - Allowing ''null'' by default works against the purpose of type declarations. Type declarations make it easier to reason about the surrounding code. If ''null'' was allowed the programmer would always have to worry about the ''null'' case.

The [[rfc:nullable_types|Nullable Types RFC]] addresses this shortcoming and more.

==== Methods which cannot declare return types ====

Class constructors, destructors and clone methods may not declare return types. Their respective error messages are:

  * ```Fatal error: Constructor %s::%s() cannot declare a return type in %s on line %s```
  * ```Fatal error: Destructor %s::__destruct() cannot declare a return type in %s on line %s```
  * ```Fatal error: %s::__clone() cannot declare a return type in %s on line %s```

==== Examples ====
Here are some snippets of both valid and invalid usage.

=== Examples of Valid Use ===
```php
// Overriding a method that did not have a return type:
interface Comment {}
interface CommentsIterator extends Iterator {
    function current(): Comment;
}
```
```php
// Using a generator:

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

=== Examples of Invalid Use ===

The error messages are taken from the current patch.
----

```php
// Covariant return-type:

interface Collection {
    function map(callable $fn): Collection;
}

interface Set extends Collection {
    function map(callable $fn): Set;
}
```
''Fatal error: Declaration of Set::map() must be compatible with Collection::map(callable $fn): Collection in %s on line %d''
----
```php
// Returned type does not match the type declaration

function get_config(): array {
    return 42;
}
get_config();
```
''Catchable fatal error: Return value of get_config() must be of the type array, integer returned in %s on line %d''

----

```php
// Int is not a valid type declaration

function answer(): int {
    return 42;
}
answer();
```
''Catchable fatal error: Return value of answer() must be an instance of int, integer returned in %s on line %d''

----

```php
// Cannot return null with a return type declaration

function foo(): DateTime {
    return null;
}
foo();
```
''Catchable fatal error: Return value of foo() must be an instance of DateTime, null returned in %s on line %d''

----

```php
// Missing return type on override

class User {}

interface UserGateway {
    function find($id): User;
}

class UserGateway_MySql implements UserGateway {
    // must return User or subtype of User
    function find($id) {
        return new User();
    }
}
```
''Fatal error: Declaration of UserGateway_MySql::find() must be compatible with UserGateway::find($id): User in %s on line %d''

----

```php
// Generator return types can only be declared as Generator, Iterator or Traversable (compile time check)

function foo(): array {
    yield [];
}
```
''Fatal error: Generators may only declare a return type of Generator, Iterator or Traversable, %s is not permitted in %s on line %d''


==== Multiple Return Types ====
This proposal specifically does not allow declaring multiple return types; this is out of the scope of this RFC and would require a separate RFC if desired.

If you want to use multiple return types in the meantime, simply omit a return type declaration and rely on PHP's excellent dynamic nature.

==== Reflection ====

This RFC purposefully omits reflection support as there is an open RFC about improving type information in reflection: https://wiki.php.net/rfc/reflectionparameter.typehint

==== Differences from Past RFCs ====
This proposal differs from past RFCs in several key ways:

  * **The return type is positioned after the parameter list.** See [[#position_of_type_declaration|Position of Type Declaration]] for more information about this decision.
  * **We keep the current type options.** Past proposals have suggested new types such as ''void'', ''int'', ''string'' or ''scalar''; this RFC does not include any new types. Note that it does allow ''self'' and ''parent'' to be used as return types.
  * **We keep the current search patterns.** You can still search for ```phpfunction foo``` to find ```phpfoo```'s definition; all previous RFCs broke this common workflow.
  * **We allow return type declarations on all function types**. Will Fitch's proposal suggested that we allow it for methods only.
  * **We do not modify or add keywords.** Past RFCs have proposed new keywords such as ''nullable'' and more. We still require the ```phpfunction``` keyword.

===== Other Impact =====

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
