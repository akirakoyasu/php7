====== PHP RFC: Context Sensitive Lexer ======
  * Version: 0.4.1
  * Date: 2015-02-15
  * Author: Márcio Almada
  * Status: Implemented (in PHP 7.0)
  * First Published at: http://wiki.php.net/rfc/context_sensitive_lexer

===== Introduction =====

PHP currently has around **64** globally reserved words.
Not infrequently, these reserved words end up clashing with legit alternatives to userland API declarations.
This RFC proposes a partial solution to this by adding minimal changes to have a **context sensitive lexer**
with support for **semi-reserved** words.

For instance, if the RFC gets accepted, code like the following would become possible:

```php
class Collection {
    public function forEach(callable $callback) { /* */ }
    public function list() { /* */ }
}
```

Notice that it's currently **not** possible to have the ''foreach'' and ''list'' method declared without having a syntax error:

<pre>
  PHP Parse error: Syntax error, unexpected T_FOREACH, expecting T_STRING on line 2
  PHP Parse error: Syntax error, unexpected T_LIST, expecting T_STRING on line 3
</pre>

===== Proposal =====

This RFC revisits the topic of [Keywords as Identifiers](https://wiki.php.net/rfc/keywords_as_identifiers) RFC. But this time
presenting a minimal and maintainable [patch](https://github.com/marcioAlmada/php-src/commit/d9d6f0c7e325dcd0d0ff3c3f2dc73c2364c3ad5f),
restricted to OO scope only, consistently comprehending:

  * Properties, constants and methods defined on classes, interfaces and traits
  * Access of properties, constants and methods from objects and classes

The proposed changes could be especially useful to:

  - Reduce the surface of BC breaks whenever new keywords are introduced
  - Avoid restricting userland APIs. Dispensing the need for hacks like unnecessary magic method calls, prefixed identifiers
    or the usage of a [thesaurus](http://en.wikipedia.org/wiki/Thesaurus) to avoid naming conflicts.

This is a list of currently **globally** reserved words that will become **semi-reserved** in case proposed change gets approved:

  callable  class  trait  extends  implements  static  abstract  final  public  protected  private  const
  enddeclare  endfor  endforeach  endif  endwhile  and  global  goto  instanceof  insteadof  interface
  namespace  new  or  xor  try  use  var  exit  list  clone  include  include_once  throw  array
  print  echo  require  require_once  return  else  elseif  default  break  continue  switch  yield
  function  if  endswitch  finally  for  foreach  declare  case  do  while  as  catch  die  self parent

==== Limitations ====

On purpose, it's still forbidden to define a **class constant** named as ''class'' because of the class name resolution ''::class'':

```php
class Foo {
  const class = 'Foo'; // Fatal error
}

// Fatal error: Cannot redefine class constant Foo::CLASS as it is reserved in %s on line %d
```

In practice, it means that we would drop from **64** to only **1** reserved word that affects only class constant names.

''class|object'' properties **can** have any name because PHP has sigils and code like the following has always been allowed:

```php
class Foo {
  public $list = 'list';
}

(new Foo)->list;
```

===== Practical Examples =====

Some practical examples related to the impact this RFC could have on user space code: 

The proposed change, if approved, gives more freedom to userland fluent interfaces or DSL like APIs.

```php
// the following example works with patch
// but currently fails because 'for', 'and', 'or', 'list' are globally reserved words:

$projects =
    Finder::for('project')
        ->where('name')->like('%secret%')
        ->and('priority', '>', 9)
        ->or('code')->in(['4', '5', '7'])
        ->and()->not('created_at')->between([$time1, $time2])
        ->list($limit, $offset);
```

```php
// the following example works with the patch
// but currently fails because 'foreach', 'list' and 'new' are globally reserved words:

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

Globally reserved words end up limiting userland implementations on being the most expressive and semantic as possible:

```php
// the following example works with the patch
// but currently fails because 'include' is a globally reserved word:

class View {
    public function include(View $view) {
        //...
    }
}

$viewA = new View('a.view');
$viewA->include(new View('b.view'));
```

Sometimes there is simply no better name for a class constant. One might want to define an HTTP agent class and would like to have some HTTP status constants:

```php
class HTTP {
    const CONTINUE = 100; // works with patch
                          // but currently fails because 'continue' is a globally reserved word
    const SWITCHING_PROTOCOLS = 101;
    /* ... */
}
```

===== Impact On Other RFCs =====

Some RFCs are proposing to reserve new keywords in order to add features or reserve typehints names:

  * https://wiki.php.net/rfc/in_operator
  * https://wiki.php.net/rfc/reserve_more_types_in_php_7
  * https://wiki.php.net/rfc/reserve_even_more_types_in_php_7

With the approval of the current RFC, BC breaks surface would be much smaller in such cases.

One notable example is the **in** operator RFC. Without a context sensitive lexer, proposed here, the new operator would create a BC break on **Doctrine** library and pretty much many other SQL writers or ORMs out there:

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

===== Proposed PHP Version(s) =====

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
