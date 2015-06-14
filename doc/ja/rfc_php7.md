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
従来定義されていないケースで、実装が一貫性のない結果となっていたforeach文の振る舞いを定義するRFC（2015-01-29作成）

**[SAPIとエクステンションの削除](Removal_of_dead_SAPIs_and_extensions.md)**  
依存先の非サポートに伴う非サポートSAPIとエクステンションの削除について検討

**[Jsond](Jsond.md)**  
jsondが現在のjsonエクステンションを置き換えるべきかどうか（2015-01-11作成）

**[JSONエンコードで小数部分を保持する](Preserve_Fractional_Part_in_JSON_encode.md)**  
JSONエンコードが浮動小数点数の小数部分を保持するための新しいオプションを追加する

**[戻り値型の宣言](Return_Type_Declarations.md)**  
関数、メソッド、クロージャに戻り値型を追加する（2014-03-20作成）

**[パラメータをパースする高速なAPI](Fast_Parameter_Parsing_API.md)**  
zend_parse_parameters()とは別の高速なAPI.（2014-05-23作成）

**[Unicode Codepoint Escape Syntax](Unicode_Codepoint_Escape_Syntax.md)**  
文字列リテラルにUnicodeコードポイントのためのエスケープシーケンス構文を追加する（2014-11-24作成）

**[ネイティブTLS](Native_TLS.md)**  
TSモードでの内部グローバルのためのネイティブTLS

**[Null結合演算子](Null_Coalesce_Operator.md)**  
結合演算子、??を追加する

**[整数セマンティクス](Integer_Semantics.md)**  
整数を扱ういくつかの演算で、PHPのクロスプラットフォーム一貫性を改善する

**[オーバーフロー時ZPPを失敗させる](ZPP_Failure_on_Overflow.md)**  
整数を期待している場所で範囲外の浮動小数点数や非数を渡された場合、zend_parse_parametersを失敗させる（2014-09-22作成）

**[phpngブランチをmasterブランチへ移動する](Move_the_phpng_branch_into_master.md)**  
phpngコードベースを将来のPHPメジャーバージョンの基礎として受け入れる（2014-07-20作成）

**[抽象構文木](Abstract_Syntax_Tree.md)**  
コンパイル処理の仲介構造として抽象構文木の導入を提案する

**[均一な変数構文](Uniform_Variable_Syntax.md)**  
内部的に一貫した完全な変数構文を導入する

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
