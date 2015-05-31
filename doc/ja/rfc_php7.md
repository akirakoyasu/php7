# PHP 7.0

**[文脈依存の字句解析](Context_Sensitive_Lexer.md)**  
半予約語をサポートして、PHPに文脈依存の字句解析を導入する提案（2015-02-15作成）

**[preg_replace_callback_array関数の追加](Add_preg_replace_callback_array_function.md)**  
それぞれのパターンに、簡単にコールバック関数を指定できるようにする提案RFC

**[信頼できるユーザ空間CSPRNG（暗号論的擬似乱数生成器）](Reliable_User-land_CSPRNG.md)**  
信頼できるCSPRNG（暗号論的擬似乱数生成器）を追加する提案（2015-02-20作成）

**[匿名クラスのサポート](Anonymous_Class_Support.md)**  
このRFCは匿名クラスのサポートを提案する

**[ジェネレータの委譲](Generator_Delegation.md)**  
このRFCはジェネレータ関数の動作をサブイテレータやサブジェネレータへ委譲する、新しい構文を提案する（2015-03-01作成）

**[PHP7で型名を追加予約する](Reserve_More_Type_Names_in_PHP_7.md)**  
クラス名や名前空間でint, float, string, bool, true, false, nullを予約する（2015-02-18作成）

**[内部クラスのコンストラクタの振る舞い](Constructor_behaviour_of_internal_classes.md)**  
内部クラスのコンストラクタの望ましくない振る舞いをきれいにする（2015-03-01作成）

**[E_STRICT警告を再分類する](Reclassify_E_STRICT_notices.md)**  
このRFCは既存のごく少数のE_STRICT警告を別のカテゴリへ再分類することを提案する（2015-02-22作成）

**[PHP4形式のコンストラクタを削除する](Remove_PHP_4_Constructors.md)**  
定義しているクラスと同名のメソッドをコンストラクタとして認識することをやめる（2014-11-17作成、2015-03-10審議終了）

**[date.timezoneの警告を削除](Remove_the_date.timezone_warning.md)**  
（2015-01-27作成）

**[結合比較（スペースシップ）演算子](Combined_Comparison_Spaceship_Operator.md)**  
新しい比較演算子、<=>を提案する（2014-02-12作成、2015-01-19再開）

**["foreach"の振る舞いを修正する](Fix_foreach_behavior.md)**  
The RFC defines foreach statement behavior for the cases where is wasn't defined before and implementation provided inconsistent results. (Created 2015-01-29)

**[Removal of dead SAPIs and extensions]()**  
Consideration about removing the unsupported SAPIs and extensions with unsupported dependencies.

**[Jsond]()**  
Whether jsond should replace the current json extension. (Created 2015-01-11)

**[Preserve Fractional Part in JSON encode]()**  
Adds a new option for JSON encode to preserve fractional part on float numbers.

**[Return Type Declarations]()**  
Adds return types to functions, methods and closures. (Created 2014-03-20)

**[Fast Parameter Parsing API]()**  
Fast API in addition to zend_parse_parameters(). (Created 2014-05-23)

**[Unicode Codepoint Escape Syntax]()**  
Adds an escape sequence syntax for Unicode codepoints to string literals. (Created 2014-11-24)

**[Native TLS]()**  
Native TLS for internal globals in TS mode

**[Null Coalesce Operator]()**  
Adds the coalesce operator, ??

**[Integer Semantics]()**  
Improves cross-platform consistency in PHP for some operations dealing with integers

**[ZPP Failure on Overflow]()**  
Make zend_parse_parameters fail if a float value out of bounds, or NaN, is passed where an integer is expected. (Created 2014-09-22)

**[Move the phpng branch into master]()**  
Embrace the phpng codebase as the basis for the future major version of PHP. (Created 2014-07-20)

**[Abstract Syntax Tree]()**  
Proposes the introduction of an Abstract Syntax Tree (AST) as an intermediary structure in our compilation process.

**[Uniform Variable Syntax]()**  
Introduces an internally consistent and complete variable syntax.

**[64 bit platform improvements for string length and integer]()**  
Integer and String modifications for full 64 bit support

**[Closure::call]()**  
Proposes a new method on the Closure class to allow calling bound to an object without pre-binding

**[Fix list() behavior inconsistency]()**  
Enable or disable string handling in list() operator

**[Remove alternative PHP tags]()**  
Removes ASP and script tags

**[switch.default.multiple]()**  
Disallow multiple defaults in switch statements

**[Catchable "call to a member function of a non-object"]()**  
Turns this fatal error into E_RECOVERABLE (Created 2014-04-26)

**[Filtered unserialize()]()**  
Add option to ignore all or some objects to unserialize() (Created 2013/03/30)

**[ICU IntlChar class]()**  
Adds an IntlChar class an intl_char_*() functions to the Intl extension.

**[Introduce session_start() INI options as array]()**  
Introduce session_start() options

**[Remove hex support in numeric strings]()**  
Removes support for hexadecimal numbers in numeric string conversions. (Created 2014-08-19)

**[Expectations]()**  
This RFC proposes adding BC zero-cost assertions. (Created 2013-11-01)

**[Group Use Declarations]()**  
The RFC adds improvements to current PHP namespace implementation by introducing group use declarations. (Created 2015-01-28)

**[Exceptions in the engine]()**  
This RFC proposes to allow the use of exceptions in the engine. (Created 2014-09-30)

**[Generator Return Expressions]()**  
This RFC proposes the ability to specify and access return values from Generator functions

**[Scalar Type Hints v0.5]()**  
This RFC proposes a mixed-mode scalar type system

**[Continue output buffering]()**  
Let the output buffer stack be usable despite an aborted connection

**[intdiv()]()**  
This RFC proposes adding an intdiv() function for integer division. (Created 2014-07-15)

**[Fix handling of custom session handler return values]()**  
Make false actually mean failure, not success.

**[Turn gc_collect_cycles into function pointer]()**  
Proposes to turn gc_collect_cycles into function pointer for extensions to hook into. (Created 2012-12-11)