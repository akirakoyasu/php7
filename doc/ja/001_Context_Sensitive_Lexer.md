====== PHP RFC: Context Sensitive Lexer ======
  * Version: 0.4.1
  * Date: 2015-02-15
  * Author: Márcio Almada
  * Status: Implemented (in PHP 7.0)
  * First Published at: http://wiki.php.net/rfc/context_sensitive_lexer

===== 導入 =====

PHPには現在**64**のグローバルな予約語がある。
まれでなく、これらの予約語は正当なユーザ空間のAPI宣言と衝突することになる。
このRFCは、 **半予約語** をサポートする **文脈依存の字句解析** を導入する極小の変更を加えることで、この問題に対する部分的な解決策を提案する。

例えば、このRFCが受け入れられれば、以下のようなコードが可能になるだろう。

```php
class Collection {
    public function forEach(callable $callback) { /* */ }
    public function list() { /* */ }
}
```

現在は以下の構文エラーなしに ''foreach'' や ''list'' というメソッドを宣言することは **不可能** であることに注意してほしい。

<pre>
  PHP Parse error: Syntax error, unexpected T_FOREACH, expecting T_STRING on line 2
  PHP Parse error: Syntax error, unexpected T_LIST, expecting T_STRING on line 3
</pre>

===== 提案 =====

このRFCは[Keywords as Identifiers：キーワードと識別子）](https://wiki.php.net/rfc/keywords_as_identifiers|Keywords as Identifiers)
RFCの話題に再訪する。が、今回はOOのスコープのみに限定し、極小でメンテンナンス可能な[パッチ](https://github.com/marcioAlmada/php-src/commit/d9d6f0c7e325dcd0d0ff3c3f2dc73c2364c3ad5f)を
示す。これは一貫性のため以下を含む：

  * クラスに定義されたプロパティ、定数、メソッド、インタフェースとトレイト
  * オブジェクトとクラスからプロパティ、定数、メソッドへのアクセス

提案する変更は特に以下の効果を見込む：

  1. 新しいキーワードが導入される際につきものの後方互換性を破壊するリスクを減らす
  1. ユーザ空間のAPIを制限することを避ける。不要なマジックメソッド呼び出しや、接頭辞付きの識別子や、命名の衝突をさけるために類義語辞典を使う
     ようなハックの必要をなくす

これは、提案する変更が承認された場合に**半予約語**となる現在は**グローバルな**予約語の一覧である。

  callable  class  trait  extends  implements  static  abstract  final  public  protected  private  const
  enddeclare  endfor  endforeach  endif  endwhile  and  global  goto  instanceof  insteadof  interface
  namespace  new  or  xor  try  use  var  exit  list  clone  include  include_once  throw  array
  print  echo  require  require_once  return  else  elseif  default  break  continue  switch  yield
  function  if  endswitch  finally  for  foreach  declare  case  do  while  as  catch  die  self parent

==== 制約 ====

意図的に、''::class''によるクラス名解決のため''class''という名前で **クラス定数** を定義することは依然として禁止する：

```php
class Foo {
  const class = 'Foo'; // Fatal error
}

// Fatal error: Cannot redefine class constant Foo::CLASS as it is reserved in %s on line %d
```

実際のところ、これはクラス定数の名前のみに影響する予約語を **64個** からわずか **1個** へ減らすことを意味する。

PHPでは以下のような記法とコードが常に許されるため、''class|object''プロパティはどのような名前でも良い：

```php
class Foo {
  public $list = 'list';
}

(new Foo)->list;
```

===== 実際的な例 =====

このRFCがユーザ空間のコードに与える影響に関連する、いくつかの実際的な例：

提案する変更が承認された場合、APIのようにユーザ空間の流れるインタフェースやDSLの自由度が高くなる。
The proposed change, if approved, gives more freedom to userland fluent interfaces or DSL like APIs.

```php
// 以下の例はパッチを当てれば動作するが、
// 現在は'for', 'and', 'or', 'list'がグローバルな予約語であるため失敗する：

$projects =
    Finder::for('project')
        ->where('name')->like('%secret%')
        ->and('priority', '>', 9)
        ->or('code')->in(['4', '5', '7'])
        ->and()->not('created_at')->between([$time1, $time2])
        ->list($limit, $offset);
```

```php
// 以下の例はパッチを当てれば動作するが、
// 現在は'foreach', 'list' and 'new'がグローバルな予約語であるため失敗する：

class Collection extends \ArrayAccess, \Countable, \IteratorAggregate {

    public function forEach(callable $callback) {
        //...
    }

    public function list() {
        //...
    }

    public static function new(array $itens) {
        return new self($itens);
    }
}

Collection::new(['foo', 'bar'])->forEach(function($index, $item){
  /* callback */
})->list();
```

グローバルな予約語は、可能な限り最も表現豊かで意味論的であろうとするユーザ空間の実装を、結果的に制限していることになる：

```php
// 以下の例はパッチを当てれば動作するが、
// 現在は'include'がグローバルな予約語であるため失敗する：

class View {
    public function include(View $view) {
        //...
    }
}

$viewA = new View('a.view');
$viewA->include(new View('b.view'));
```

ときに、単にクラス定数としてより良い名前がないこともある。HTTPエージェントクラスを定義しようとするとき、いくつかのHTTPステータス定数を使いたいかもしれない：

```php
class HTTP {
    const CONTINUE = 100; // パッチを当てれば動作するが、
                          // 現在は'continue'がグローバルな予約語であるため失敗する
    const SWITCHING_PROTOCOLS = 101;
    /* ... */
}
```

===== 他のRFCへの影響 =====

いくつかのRFCが、機能の追加やタイプヒント名の予約のために、新しい予約語を提案している。

  * https://wiki.php.net/rfc/in_operator
  * https://wiki.php.net/rfc/reserve_more_types_in_php_7
  * https://wiki.php.net/rfc/reserve_even_more_types_in_php_7

このRFCの承認によって、このようなケースで後方互換性を破壊するリスクはかなり小さくなるだろう。

一つの重要な例は **in** 演算子のRFCである。ここで提案する文脈依存の字句解析なしでは、新しい演算子は以下に示すような **Doctrine** ライブラリやその他多くのSQLライタ、またはORMの後方互換性を破壊することになるだろう：

https://github.com/doctrine/doctrine2/blob/master/lib/Doctrine/ORM/Query/Expr.php#L443

===== Implementation Details =====

==== Patch 1 - Discarded ====

The lexer now keeps track of the context needed to have unreserved words on OO scope and makes use of a minimal amount of RE2C lookahead capabilities when disambiguation becomes inevitable.

For instance, the lexing rules to disambiguate ''::class'' (class name resolution operator) from a ''class constant'' or ''static method'' access is:

```c++
<ST_IN_SCRIPTING>"::"/{OPTIONAL_WHITESPACE}"class" {
  return T_PAAMAYIM_NEKUDOTAYIM;
}

<ST_IN_SCRIPTING>"::"/{OPTIONAL_WHITESPACE}("$"|{LABEL}){OPTIONAL_WHITESPACE}"("? {
  yy_push_state(ST_LOOKING_FOR_SEMI_RESERVED_NAME);
  return T_PAAMAYIM_NEKUDOTAYIM;
}
```

A few additional compile time check were created:

```c
if(ZEND_NOT_RESERVED != zend_check_reserved_method_name(decl->name)) {
  zend_error_noreturn(E_COMPILE_ERROR,
    "Cannot use '%s' as class method name as it is reserved", decl->name->val);
}
```

==== Patch 2 ====

A new patch has been added during the voting phase. It's a different approach that proved to have many advantages over the first patch and therefore it is intended to supersede it.

The new patch just requires the maintenance of a single inclusive parser rule listing all tokens that should be matched as a ''T_STRING'' on specific places:

  - It offers no regression | forward compatibility risks and is highly predictable
  - It has a very small footprint when compared to the previous attempt involving a pure lexical approach
  - Requires no compile time checks
  - Is highly configurable, to make a word semi-reserved you only have to edit an inclusive parser rule.

In order to send information to the lexer about the context change, we just have to use ''identifier'' instead of ''T_STRING'' when applicable. For instance this is the needed changes on the parser grammar to allow semi reserved words on method names:

```c
// before
method_modifiers function returns_ref T_STRING '(' parameter_list ')' //...

// after
method_modifiers function returns_ref identifier '(' parameter_list ')' //...
```

===== Future Work And Maintenance =====

  * All php-src tests are passing with the new patch, some work still has to be done. There is a better possibility to expand semi reserved words support to namespaces and class names with the new patch, but this more ambitious proposal will be tailored only for PHP 7.1 by the RFC author.

=> The first patch has been discarded during discussion on voting phase. It was considered too "ad-hoc" and could cause issues for PHP 7.1 and ahead.

===== 提案するPHPバージョン =====

This is proposed for the next PHP x, which at the time of this writing would be PHP 7.

===== Votes =====

This voting requires a 2/3 majority. The implementation will be evaluated on internals mailing list and will only be merged if it's
considered good enough, independently of the voting results. The RCF author encourages voting for the feature.

<doodle title="Should PHP7 have a context sensitive lexer?" auth="marcio" voteType="single" closed="true">
   * Yes
   * No
</doodle>

Voting started on 2015-02-28 and ends on 2015-03-14.

===== Patch =====

==== Patch 1 - Discarded ====

  - Pull request with all the tests and regenerated ext tokenizer is at [[https://github.com/php/php-src/pull/1054]]

==== Patch 2 ====

  - Pull request with all the tests is at [[https://github.com/php/php-src/pull/1221/]]

==== Later Changes ===

The *Patch 2* was merged and, later, method modifiers were allowed as class member names. This was a limitation from the older implementation candidate - Patch 1 - and there was no reason to keep it. The **Limitations** section was updated accordingly. Only the keyword **class** for class constants is reserved now.

===== References =====

This is the previous rejected RFC that attempted to remove reserved words on all contexts: https://wiki.php.net/rfc/keywords_as_identifiers.

===== Rejected Features =====

 * Prior to voting, the support for ''namespaces|classes|traits|interfaces'' names has been removed from the first patch as it could create some possible issues.

=> The RFC author will try to solve the wider problem on PHP 7.1

===== Changelog =====
  * 0.1: Initial draft with support for class, interfaces and trait members
  * 0.2: Additional support to namespaces, classes, interafces and traits names
  * 0.3: Oops. Add forgotten support for typehints
  * 0.4: Reverts to 0.1 feature set because class name support created undesired situations regarding the future addition of a future short lambda syntax and possibly block other language changes.
  * 0.4.1: A new compatible implementation has been introduced

===== Acknowledgements =====

Thanks to:

  * Bob Weinand, author of the last [[https://wiki.php.net/rfc/keywords_as_identifiers|rejected]] RFC on the same topic, for giving honest feedback and being cooperative all the time.
  * Nikita Popov for providing accurate information about the PHP implementation and constructive criticism.
  * Anthony Ferrara, Joe Watkins and Daniel Ackroyd for the quick reviews.
  * All people on http://chat.stackoverflow.com/rooms/11/php
