
====== PHP RFC: Anonymous Classes ======
  * Version: 0.6
  * Date: 2013-09-22
  * Author: Joe Watkins <krakjoe@php.net>, Phil Sturgeon <philstu@php.net>
  * Status: Implemented (in PHP 7.0)
  * First Published at: http://wiki.php.net/rfc/anonymous_classes

===== 導入 =====

かなり以前から、PHPにはクロージャの形式をした特徴的な匿名関数をサポートしている。このパッチは
匿名クラスのオブジェクトのための同様の機能を導入する。

匿名クラスのオブジェクトを作成する能力は他の言語（すなわちC#とJava）のオブジェクト指向プログラミング
で確立されかつよく使われる部分である。

匿名クラスは名前付きクラスの上で使われるだろう：
* そのクラスを文書化する必要がないとき
* そのクラスが実行時に一度しか使われないとき

匿名クラスは（プログラマが宣言した）名前を持たないクラスである。そのオブジェクトの機能は名前付きクラス
のオブジェクトのそれと変わりない。既存のクラスの構文を使い、名前を持たない：

```php
var_dump(new class($i) {
    public function __construct($i) {
        $this->i = $i;
    }
});
```

===== 構文と例 =====

new class (arguments) {definition}

注記： このRFCの以前のバージョンでは引数は定義の後にあったが、最新の議論からのフィードバックを受けて変更された。

```php
<?php
/* 使用しているフレームワークなどから匿名のコンソールオブジェクトを実装 */
(new class extends ConsoleProgram {
    public function main() {
       /* ... */
    }
})->bootstrap();

/* 使用しているMVCフレームワークからページの匿名実装を返却 */
return new class($controller) implements Page {
    public function __construct($controller) {
        /* ... */
    }
    /* ... */
};

/* vs */
class MyPage implements Page {
    public function __construct($controller) {
        /* ... */
    }
    /* ... */
}
return new MyPage($controller);

/* DirectoryIteratorクラスの匿名拡張を返却 */
return new class($path) extends DirectoryIterator {
   /* ... */
};

/* vs */
class MyDirectoryIterator {
    /* .. */
}
return new MyDirectoryIterator($path);

/* 他のクラス内から匿名クラスを返却（PHPで最初のネストされたクラスの導入である）*/
class MyObject extends MyStuff {
    public function getInterface() {
        return new class implements MyInterface {
            /* ... */
        };
    }
}


/* インターフェースを実装したプライベートなオブジェクトを返却 */
class MyObject extends MyStuff {
    /* suitable ctor */

    private function getInterface() {
        return new class(/* suitable ctor args */) extends MyObject implements MyInterface {
            /* ... */
        };
    }
}
```

注記： 匿名クラスでコンストラクタを宣言、また使用する能力はコンストラクト制御を効かせるために必要である

===== 継承/トレイト =====

クラスの拡張はあなたが期待するとおりに動作する。

```php
<?php

class Foo {}

$child = new class extends Foo {};

var_dump($child instanceof Foo); // true
```

トレイトも名前付きクラスの定義と全く同様に動作する。

```php
<?php

trait Foo {
    public function someMethod() {
      return "bar";
    }
}

$anonClass = new class {
    use Foo;
};

var_dump($anonClass->someMethod()); // string(3) "bar"
```

===== リフレクション =====

リフレクションへの唯一の変更は ReflectionClass::isAnonymous() の追加である。

===== 直列化 =====

直列化はサポートされない。匿名関数がエラーとなるのと同様である。

===== クラスへの内部的な命名 =====

匿名クラスの内部的な名前は、そのアドレスを基にした固有な参照から生成される。

```php
function my_factory_function(){
    return new class{};
}
```

get_class(my_factory_function()) は定義が同じであれば何度コールされても "class@0x7fa77f271bd0" を
返すだろう。デフォルトで"class"という語が使用されるが、匿名クラスが名前付きクラスを拡張する場合には
そのクラスの名前が使用される：

```php
class mine {}

new class extends mine {};
```

このクラスの名前は "mine@0x7fc4be471000" となる。

同じ場所（例えばループ）で生成された複数の匿名クラスは `==` で比較できるが、他の場所で生成されたものは
別の名前を持つため一致することはないだろう。

```php
$identicalAnonClasses = [];

for ($i = 0; $i < 2; $i++) {
    $identicalAnonClasses[$i] = new class(99) {
        public $i;
        public function __construct($i) {
            $this->i = $i;
        }
    };
}

var_dump($identicalAnonClasses[0] == $identicalAnonClasses[1]); // true

$identicalAnonClasses[2] = new class(99) {
    public $i;
    public function __construct($i) {
        $this->i = $i;
    }
};

var_dump($identicalAnonClasses[0] == $identicalAnonClasses[2]); // false
```

両方のクラスは生成された名前以外は、あらゆる面で同じである。

===== ユースケース =====

コードテストがユースケースの中で最も重要な位置を占めている。しかし、匿名クラスが言語の一部である場合、
テストだけでなく多くのユースケースが発見されている。匿名クラスを使用することが厳密に正しいかどうかは
ほぼそれぞれのアプリケーション次第、もしくはものの見方次第である。

ざっくりしたポイントとしては：
* モックテストが超簡単になる。複雑なモックAPIを使わずに、オンザフライでインターフェースの実装を作る
* クラスが定義されたスコープ外でのクラスの使用を保つ
* ほんの小さな実装のためにオートローダーが起動するのを避ける

既存のクラスをほんの僅か変更するだけで、これはごく容易になる。[Pusher PHP library](https://github.com/pusher/pusher-http-php#debugging--logging)
から例を参照してほしい：

```php
// PHP 5.x
class MyLogger {
  public function log($msg) {
    print_r($msg . "\n");
  }
}

$pusher->setLogger( new MyLogger() );

// New Hotness
$pusher->setLogger(new class {
  public function log($msg) {
    print_r($msg . "\n");
  }
});
```

新しいファイルを作成したり、クラス定義をファイルの先頭、あるいは使用される場所から遠く離れた場所に
配置したりしなくてよくなる。大きく複雑なアクション、あるいは繰り返し使用する必要のあるものは
もちろん名前付きクラスにしたほうが良いが、この場合には煩雑でないことがすばらしく、手軽である。

もし簡単な依存性を作成して、とても軽いインターフェースを実装する必要があるなら：

```php
$subject->attach(new class implements SplObserver {
  function update(SplSubject $s) {
    printf("Got update from: %s\n", $subject);
  }
});
```
これはひとつの例だが、PSR-7 のミドルウェアを Laravel5 スタイルのミドルウェアに変換するものだ。

```php
<?php
$conduit->pipe(new class implements MiddlewareInterface {
    public function __invoke($request, $response, $next)
    {
        $laravelRequest = mungePsr7ToLaravelRequest($request);
        $laravelNext    = function ($request) use ($next, $response) {
            return $next(mungeLaravelToPsr7Request($request), $response)
        };
        $laravelMiddleware = new SomeLaravelMiddleware();
        $response = $laravelMiddleware->handle($laravelRequest, $laravelNext);
        return mungeLaravelToPsr2Response($response);
    }
});
```

匿名クラスは、PHPで初のネストされたクラスを作成する機会を示している。あなたはわずかに異なる理由で
匿名クラスをネストするかもしれず、それはいくらか議論するにふさわしい。

```php
<?php
class Outside {
    protected $data;

    public function __construct($data) {
        $this->data = $data;
    }

    public function getArrayAccess() {
        return new class($this->data) extends Outside implements ArrayAccess {
            public function offsetGet($offset) { return $this->data[$offset]; }
            public function offsetSet($offset, $data) { return ($this->data[$offset] = $data); }
            public function offsetUnset($offset) { unset($this->data[$offset]); }
            public function offsetExists($offset) { return isset($this->data[$offset]); }
        };
    }
}
```

注記： Outer は $this->data のアクセスのために拡張されたのではない ー それであれば単にコンストラクタに
渡せばよい。 Outer を拡張することで、 ArrayAccess を実装したネストされたクラスは Outerクラスで
宣言された protected なメソッドを同じ $this->data 上で実行することができ、参照としては
まるでOuterクラスであるかのようである。

上記の単純な例では、Outer::getArrayAccess は Outer のための ArrayAccessインターフェースオブジェクトを
宣言し作成する匿名クラスを利用している。

getArrayAccess を private にすることで、作成される匿名クラスは private なクラスと言えるようになる。

これはオブジェクトの機能をまとめられる可能性を高め、より可読性が高く、メンテナンス性が高いとも
言えるコードをもたらすだろう。

上記の代わりとしては以下もありうる：

```php
class Outer implements ArrayAccess {
    public $data;

    public function __construct($data) {
        $this->data;
    }

    public function offsetGet($offset) { return $this->data[$offset]; }
    public function offsetSet($offset, $data) { return ($this->data[$offset] = $data); }
    public function offsetUnset($offset) { unset($this->data[$offset]); }
    public function offsetExists($offset) { return isset($this->data[$offset]); }

}
```

上の例では参照渡しは使われていないため、$this->data に関する振る舞いは暗黙的となる。

あるアプリケーションでどのように選択するか、getArrayAccess は private か否か、参照渡しを
行うか否かは、そのアプリケーション次第である。

様々なユースケースがメーリングリストで提案されている：
http://php.markmail.org/message/sxqeqgc3fvs3nlpa?q=anonymous+classes+php, いくつかを示す。

----

ユースケースは"実装"を一度だけ使用する場合である。現在はまず"Callback*"-クラスにコールバックを
渡すような。

```php
    $x = new Callback(function() {
        /* do something */
    });

    /* vs */

    $x = new class extends Callback {
        public function doSometing()
        {
            /* do something */
        }
    };
```

いくつかの抽象メソッドがひとつのインターフェース/クラスにあり、いくつかのコールバックをコンストラクタに
渡す必要があると想像てみよう。
また '$this' は正しいオブジェクトにマップされるとして。

クラスのあるメソッドをオーバーライドすることは簡単な使用法のひとつである。元のクラスを拡張した
新しいクラスを作成する代わりに、匿名クラスを使い、オーバーライドしたいメソッドをオーバーライド
することができる。

例えば、以下のようにすることができる：

```php
use Symfony\Component\Process\Process;

$process = new class extends Process {
    public function start() {
        /* ... */
    }
};
```

以下のようにする代わりに：

```php
namespace My\Namespace\Process;

use Symfony\Component\Process\Process as Base;

class Process extends Base {
    public function start() {
        /* ... */
    }
}

$process = new \My\Namespace\Process\Process;
```

===== 後方互換性のない変更 =====

新しい構文は以前のバージョンではパースに失敗するため、後方互換性を損なうことはない。

===== 提案するPHPバージョン =====

7.0

===== SAPIへの影響 =====

All

===== 既存のエクステンションへの影響 =====

No impact on existing libraries

===== Open Issues =====

None

===== Future Scope =====

The changes made by this patch mean named nested classes are easier to implement (by a tiny bit).

===== References =====

PHP 7 Discussion: http://marc.info/?l=php-internals&m=142478600309800&w=2

===== Proposed Voting Choices =====

The voting choices are yes (in favor for accepting this RFC for PHP 7) or no (against it).

===== Vote =====

Vote starts on March 13th, and will end two weeks later, on March 27th.

This RFC requires a 2/3 majority.

<doodle title="Anonymous Classes" auth="philstu" voteType="single" closed="true">
   * Yes
   * No
</doodle>

 ===== Changelog =====

  * v0.6 Serialization not supported and names added
  * v0.5.2 Updated name examples
  * v0.5.1 Example improvements
  * v0.5: Added traits example and tests
  * v0.4: Comparisons example
  * v0.3: ReflectionClass::isAnonymous() and simple example
  * v0.2: Brought back for discussion
  * v0.1: Initial draft

===== Implementation =====

https://github.com/php/php-src/pull/1118
