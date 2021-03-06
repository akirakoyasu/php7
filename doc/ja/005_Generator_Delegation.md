====== PHP RFC: Generator Delegation ======
  * Version: 0.2.0
  * Date: 2015-03-01
  * Author: Daniel Lowrey <rdlowrey@php.net>
  * Contributors: Bob Weinand <bwoebi@php.net>
  * Status: Implemented in 7.0
  * First Published at: http://wiki.php.net/rfc/generator-delegation

====== 概要 ======

このRFCは新しい ```yield from <expr>``` 構文によって、ジェネレータ関数に"トラバーサブル"オブジェクトや
配列へ委譲できるようにすることを提案する。提案する構文は、分離したクラスメソッドがオブジェクト指向の
コードをシンプルにするのと同様に ```yield``` 文をより小さな概念的ユニットにする。提案は先行するRFC、
[Generator Return Expressions](Generator_Delegation.md) によって提案された機能性に概念的に
関連し、依存している。

====== 提案 ======

ジェネレータ関数の本体で次の新しい構文を許す：

```php
    yield from <expr>
```

上記のコードで ```<expr>``` は、"トラバーサブル"オブジェクトや配列を実行するどのような式でもよい。
委譲ジェネレータの呼出元と値を直接授受する間、このトラバーサブルは有効でなくなるまで進められる。
もしこの ```<expr>``` トラバーサブルが"ジェネレータ"であればサブジェネレータとみなされ、最終的な
戻り値が```yield from```式から生じた結果として委譲ジェネレータに返される。

==== 用語 ====

- "委譲ジェネレータ"とは、```yield from <expr>```構文を含む"ジェネレータ"である
- "サブジェネレータ"とは、```yield from <expr>```構文内の```<expr>```で使われる"ジェネレータ"
 である

==== おおまかに ====

- トラバーサブルからyieldされたそれぞれの値は、直接委譲ジェネレータの呼出元に渡される。
- 委譲ジェネレータの"send()"メソッドに送られたそれぞれの値は、サブジェネレータの"send()"メソッドに
 渡される。仮に委譲トラバーサブルがジェネレータでない場合、ジェネレータでないトラバーサブルは
 それを受け取ることができないため、送られた値は全て無視される。
- トラバーサブル/サブジェネレータから投げられた例外はチェーンをたどり、委譲ジェネレータへ伝播する
- トラバーサブルがジェネレータでない場合、終点で"null"が委譲ジェネレータへ返される。トラバーサブルが
 ジェネレータ（サブジェネレータ）であれば、戻り値は```yield from```式の値として委譲ジェネレータへ
 送られる。

==== 正式に ====

提案する構文

```php
$g = function() {
    return yield from <expr>;
};
```

は以下と同じである。

```php
$g = function() {
    $iter = <expr>;
    $isSubgenerator = $iter instanceof Generator;
    $received = null;
    $send = true;

    while ($iter->valid()) {
        if ($isSubgenerator) {
            $next = $send ? $iter->send($received) : $iter->throw($received);
        } else {
            $next = $iter->current();
            $iter->next();
        }
        try {
            $received = yield $next;
            $send = true;
        } catch (Exception $e) {
            if ($isSubgenerator) {
                $received = $e;
                $send = false;
            } else {
                throw $e;
            }
        }
    }

    return $isSubgenerator ? $iter->getReturn() : null;
};
```


===== 根拠 =====

ジェネレータ委譲の主な動機はリファクタリングと可読性である。この中心としては、分離したクラスメソッドから
値を返す場合に利用されるのと同じ信念である。値を返すことができないクラスメソッドを想像してみなさい。
そのようなシナリオでは、我々はインスタンスのプロパティに結果を保持しておき、その後で呼出元の
コード上からそれを受け取るように _できる_ 。しかし、こういった種の余分な状態はすぐに判断を難しくする。

しかも我々は、関数的なコンテキストには、結果を保持するような余分にステートフルなコンテキストが不足している
ことを知っている。標準入出力のないパラダイムでは、ジェネレータの逐次処理の最終的な結果にアクセスする
手段がない。もちろん、参照やクロージャの"use"バインディングによってこのそれなりの状況を暫定的に解決する
ことは _できる_ が、この直接的でないアプローチはジェネレータ関数が式を返せるようになった時点で
すぐに排除される。こういった場合に、値を返すことでプログラマにとって個々の処理と最終的な結果とを
直接結びつけられるため、認識のオーバーヘッドを最小化できる。

ジェネレータ委譲 - その中心にあるのは、構造化されていない複雑な処理をより小さく凝集した部品にする
標準的なプラクティスの適用以上の何者でもない。

==== ユースケース： 分解されたジェネレータ計算 ====

この簡単な例では、```yield from``` を利用して、より複雑な処理を複数の別ジェネレータに分解
してみよう。"myGeneratorFunction"の呼出元は、個々のyieldされた値がどこから来たのかは気にしないし、
気にするべきではない。その代わり、ただyieldされた値を繰り返してゆき、ジェネレータ関数の最終的な
戻り値を待つのである。

```php
    function myGeneratorFunction($foo) {
        // ... do some stuff with $foo ...
        $bar = yield from factoredComputation1($foo);
        // ... do some stuff with $bar ...
        $baz = yield from factoredComputation2($bar);

        return $baz;
    }
    function factoredComputation1($foo) {
        yield ...; // pseudo-code (something we factored out)
        yield ...; // pseudo-code (something we factored out)
        return 'zanzibar';
    }
    function factoredComputation2($bar) {
        yield ...; // pseudo-code (something we factored out)
        yield ...; // pseudo-code (something we factored out)
        return 42;
    }
```


==== ユースケース： 軽量なスレッドとしてのジェネレータ ===

ジェネレータ関数の典型的な機能は、のちの再開のための実行の中断である。この能力は、伝統的にシングル
スレッドのPHPのような言語でも、アプリケーションに非同期で並行な機構を実装する仕組みをもたらす。
単純なユーザ領域のタスクスケジュールシステムによって、挟み込まれたジェネレータは並行処理タスクを
実行する軽量なスレッドとなる。

しかしながら、ジェネレータの戻り値がなければ、アプリケーションは最終的な戻り値を返す標準化された
方法を持たないまま、"バックグラウンド"タスクを切り離すという環境を強いられる。
これは、この提案がジェネレータのreturn式RFCが受け入れられることに依存しているという一つの理由である。
戻り値のもう一つの理由は、先に議論したリファクタリング原理から必須の流れである。特に： ジェネレータを
スレッド実行のために使用したコードは、普通の関数のように振る舞うサブジェネレータの恩恵を受けることが
できる。

提案の構文を使えば、普通の関数"foo"
Using the proposed syntax an ordinary function ''foo''

```$baz = foo($bar);```

はサブジェネレータ委譲の形に変形することができ、

```$baz = yield from foo($bar);```

"foo"は中断可能なジェネレータとなる。この作法に則るならば、スレッド化されたマルチタスク処理に
ありがちな認識のオーバーヘッドがない、強力なユーザ領域での並行性の抽象をアプリケーションは
作ることができる。上記の例ではジェネレータ委譲によって、プログラマは入力"$bar"と出力"$baz"を
気にするだけでよく、残りの大きな仕事は言語に任せることができる。

つまり： ジェネレータ委譲は、プログラマに並行コードの振る舞いについて、"yield"文を使って中断させる
ことのできる普通の関数として、単純に"foo()"を考えて判断できるようにする。

__注意：__ 実際のコルーチンのタスクスケジューラ実装はこの文書のスコープ外である。このRFCは
ただユーザ領域でより上手くいきそうなそういったツールを作るために必要な言語レベルの機構にフォーカスする。
単純にジェネレータ関数に移行するだけで、なぜか魔法のように並行実行されたりしないことは明らかである。

===== 基本的な例 =====

他のジェネレータへの委譲（サブジェネレータ）

```php
<?php

function g1() {
  yield 2;
  yield 3;
  yield 4;
}

function g2() {
  yield 1;
  yield from g1();
  yield 5;
}

$g = g2();
foreach ($g as $yielded) {
    var_dump($yielded);
}

/*
int(1)
int(2)
int(3)
int(4)
int(5)
*/
```

配列への委譲

```php
<?php

function g() {
  yield 1;
  yield from [2, 3, 4];
  yield 5;
}

$g = g();
foreach ($g as $yielded) {
    var_dump($yielded);
}

/*
int(1)
int(2)
int(3)
int(4)
int(5)
*/
```

ジェネレータではないトラバーサブルへの委譲

```php
<?php

function g() {
  yield 1;
  yield from new ArrayIterator([2, 3, 4]);
  yield 5;
}

$g = g();
foreach ($g as $yielded) {
    var_dump($yielded);
}

/*
int(1)
int(2)
int(3)
int(4)
int(5)
*/
```

"yield from"式の値

```php
<?php

function g1() {
  yield 2;
  yield 3;
  return 42;
}

function g2() {
  yield 1;
  $g1result = yield from g1();
  yield 4;
  return $g1result;
}

$g = g2();
foreach ($g as $yielded) {
    var_dump($yielded);
}
var_dump($g->getReturn());

/*
int(1)
int(2)
int(3)
int(4)
int(42)
*/
```

===== 選択された実装の詳細 =====

The delegation implementation builds on the patch submitted as part of the Generator Return Expressions RFC.
This implementation adds the following new parsing token:

''%token T_YIELD_FROM   "yield from (T_YIELD_FROM)"''

The primary advantage of this approach is the addition of a readable and semantically meaningful syntax
without reserving a new ''from'' keyword.


==== Subgenerator Keys ====

As Generator iteration is always associated with an accompanying key there exists the potential that
a given delegation may return the same key multiple times from separate individual generators. This
was deemed unproblematic for three primary reasons.

  * Multiple occurrences of the same key can easily occur in the existing generator implementation should a function ''yield'' the same key more than once.

  * If a caller derives semantic meaning from yielded keys the burden is placed on generator value producers to yield keys sensible for problem domain in which they exist. The burden here is not on the language to avoid duplicate keys.

  * The ''Traversable'' interface represents neither a hash map nor a traditional contiguously indexed array and its implementations have no special requirement to expose data access to API consumers via index keys. As such, key recurrence exposes no risk for overwriting existing internal generator data.


==== Shared Subgenerator Behavior ====

PHP generator functions are implemented as stateful object instances. Although "sharing" valid generator
instances does not present any immediately obvious use-cases, such behavior is still supported. Here
we note some of the characteristics of shared generator functions.

If a "shared" subgenerator that has previously iterated to completion is passed in a ''yield from''
expression its completed return value is immediately returned to the delegating generator. In code:

```php
function subgenerator() {
    yield 1;
    return 42;
}
function delegator(Generator $shared) {
    return yield from $shared;
}

$shared = subgenerator();
while($shared->valid()) { $shared->next(); }
$delegator = delegator($shared);

foreach ($delegator as $value) {
    var_dump($value);
}
var_dump($delegator->getReturn());

/*
int(42)
// This is our only output because no values are yielded
// from the already-completed shared subgenerator
*/
```


Manually advancing a shared subgenerator outside the context of the delegating generator will not
result in an error. In code:

```php
function subgenerator() {
    yield 1;
    yield 2;
    yield 3;
    yield 4;
    return 42;
}
function delegator(Generator $shared) {
    return yield from $shared;
}

$shared = subgenerator();
$shared->next();
$delegator = delegator($shared);
var_dump($delegator->current());
$shared->next();
while($delegator->valid()) {
    var_dump($delegator->current());
    $delegator->next();
}
var_dump($delegator->getReturn());

/*
int(2);
int(3);
int(4);
int(42)
*/
```

==== Error States ====

There are two scenarios in which ''yield from'' usage can result in an ''EngineException'':

  * Using ''yield from <expr>'' where <expr> evaluates to a generator which previously terminated with an uncaught exception results in an ''EngineException''.

  * Using ''yield from <expr>'' where <expr> evaluates to something that is neither ''Traversable'' nor an array throws an ''EngineException''.

===== Rejected Ideas =====

The original version of this RFC proposed a "yield \*" syntax. The "yield \*" syntax was rejected in favor of
"yield \*" on the basis that "\*" would break backwards compatibility. Additionally, the "yield \*"
form was considered less readable than the current proposal.


===== Criticisms =====

It has been suggested during the discussion phase that a mechanism other than ''return'' be used in
subgenerators to establish the value returned to delegating generators by ''yield from'' expressions.
The following counter-arguments are provided to this criticism:

  * Forms other than ''return'' would undermine the proposal's stated goal of conceptualizing subgenerators as suspendable functions. Implementing a different syntax would inhibit this understanding by fostering the idea of generators as being something other than "real" functions.

  * ''return'' expressions have applicable semantics, known characteristics and low cognitive overhead.

  * Other popular dynamic languages inhabiting a similar space to PHP implement generator delegation value resolution using the ''return'' syntax proposed here. Sharing common vernacular for similar features lowers cognitive barriers for developers coming to PHP from diverse backgrounds.


===== Other Languages =====

Other popular dynamic languages currently support variants of the proposed syntax ...

**Python**

Python 3.3 generators support the ''yield from'' syntax:

```python
>>> def accumulate():
...     tally = 0
...     while 1:
...         next = yield
...         if next is None:
...             return tally
...         tally += next
...
>>> def gather_tallies(tallies):
...     while 1:
...         tally = yield from accumulate()
...         tallies.append(tally)
...
>>> tallies = []
>>> acc = gather_tallies(tallies)
>>> next(acc) # Ensure the accumulator is ready to accept values
>>> for i in range(4):
...     acc.send(i)
...
>>> acc.send(None) # Finish the first tally
>>> for i in range(5):
...     acc.send(i)
...
>>> acc.send(None) # Finish the second tally
>>> tallies
[6, 10]
```

**JavaScript**

Javascript ES6 generators support the ''yield*'' syntax:

```javascript
function* g4() {
  yield* [1, 2, 3];
  return "foo";
}

var result;

function* g5() {
  result = yield* g4();
}

var iterator = g5();

console.log(iterator.next()); // { value: 1, done: false }
console.log(iterator.next()); // { value: 2, done: false }
console.log(iterator.next()); // { value: 3, done: false }
console.log(iterator.next()); // { value: undefined, done: true },
                              // g4() returned { value: "foo", done: true } at this point

console.log(result);          // "foo"
```

===== Backward Incompatible Changes =====

None

===== Proposed PHP Version(s) =====

PHP7

===== Unaffected PHP Functionality =====

Existing generator semantics are unaffected.


===== Vote =====

A 2/3 "Yes" vote is required to implement this proposal. Voting will continue through March 29, 2015.

<doodle title="Allow Generator delegation in PHP7" auth="rdlowrey" voteType="single" closed="true">
   * Yes
   * No
</doodle>

.

The success of this vote depends on the success of the accompanying [[rfc:generator-return-expressions|Generator Return Expressions]] RFC. Should Generator Return Expressions be rejected the voting outcome of this RFC will be rendered moot.

===== Patches and Tests =====

The current patch is considered "final" and can be found here:

https://github.com/bwoebi/php-src/commits/coroutineDelegation

The patch was written by Bob Weinand and is based upon the implementation branch written by Nikita Popov
for the [[rfc:generator-return-expressions|Generator Return Expressions]] RFC. Extensive .phpt tests
exist in the implementation branch and readers are encouraged both to compile with the proposed
implementation and read the test cases to ascertain the full nature of the proposal.

===== Implementation =====

TBD

===== References =====

  * [[rfc:generator-return-expressions|Generator Return Expressions RFC]]

  * [[https://docs.python.org/3/whatsnew/3.3.html#pep-380|New in Python 3.3: yield from]]

  * [[https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/yield*|JavaScript yield* Syntax]]


===== Changelog =====

  * v0.2.0 Moved to ''yield from'' instead of ''yield *''+ massive textual additions
  * v0.1.0 Initial proposal
