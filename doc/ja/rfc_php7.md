# PHP 7.0

**[文脈依存の字句解析](Context_Sensitive_Lexer.md)**  
半予約語をサポートして、PHPに文脈依存の字句解析を導入する提案（2015-02-15作成）

**[preg_replace_callback_array関数の追加](Add_preg_replace_callback_array_function.md)**  
それぞれのパターンに、簡単にコールバック関数を指定できるようにする提案RFC

**[信頼できるユーザ空間CSPRNG（暗号論的擬似乱数生成器）](Reliable_User-land_CSPRNG.md)**  
信頼できるCSPRNG（暗号論的擬似乱数生成器）を追加する提案（2015-02-20作成） :TODO

**[匿名クラスのサポート](Anonymous_Class_Support.md)**  
このRFCは匿名クラスのサポートを提案する

**[ジェネレータの委譲](Generator_Delegation.md)**  
このRFCはジェネレータ関数の動作をサブイテレータやサブジェネレータへ委譲する、新しい構文を提案する（2015-03-01作成） :TODO

**[PHP7で型名を追加予約する](Reserve_More_Type_Names_in_PHP_7.md)**  
クラス名や名前空間でint, float, string, bool, true, false, nullを予約する（2015-02-18作成） :TODO

**[内部クラスのコンストラクタの振る舞い](Constructor_behaviour_of_internal_classes.md)**  
内部クラスのコンストラクタの望ましくない振る舞いをきれいにする（2015-03-01作成） :TODO

**[E_STRICT警告を再分類する](Reclassify_E_STRICT_notices.md)**  
このRFCは既存のごく少数のE_STRICT警告を別のカテゴリへ再分類することを提案する（2015-02-22作成） :TODO

**[PHP4形式のコンストラクタを削除する](Remove_PHP_4_Constructors.md)**  
定義しているクラスと同名のメソッドをコンストラクタとして認識することをやめる（2014-11-17作成、2015-03-10審議終了） :TODO

**[date.timezoneの警告を削除](Remove_the_date.timezone_warning.md)**  
（2015-01-27作成） :TODO

**[結合比較（スペースシップ）演算子](Combined_Comparison_Spaceship_Operator.md)**  
新しい比較演算子、<=>を提案する（2014-02-12作成、2015-01-19再開） :TODO

**["foreach"の振る舞いを修正する](Fix_foreach_behavior.md)**  
従来定義されていないケースで、実装が一貫性のない結果となっていたforeach文の振る舞いを定義するRFC（2015-01-29作成） :TODO

**[SAPIとエクステンションの削除](Removal_of_dead_SAPIs_and_extensions.md)**  
依存先の非サポートに伴う非サポートSAPIとエクステンションの削除について検討 :TODO

**[Jsond](Jsond.md)**  
jsondが現在のjsonエクステンションを置き換えるべきかどうか（2015-01-11作成） :TODO

**[JSONエンコードで小数部分を保持する](Preserve_Fractional_Part_in_JSON_encode.md)**  
JSONエンコードが浮動小数点数の小数部分を保持するための新しいオプションを追加する :TODO

**[戻り値型の宣言](Return_Type_Declarations.md)**  
関数、メソッド、クロージャに戻り値型を追加する（2014-03-20作成） :TODO

**[パラメータをパースする高速なAPI](Fast_Parameter_Parsing_API.md)**  
zend_parse_parameters()とは別の高速なAPI.（2014-05-23作成） :TODO

**[Unicodeコードポイントのエスケープ構文](Unicode_Codepoint_Escape_Syntax.md)**  
文字列リテラルにUnicodeコードポイントのためのエスケープシーケンス構文を追加する（2014-11-24作成） :TODO

**[ネイティブTLS](Native_TLS.md)**  
TSモードでの内部グローバルのためのネイティブTLS :TODO

**[Null結合演算子](Null_Coalesce_Operator.md)**  
結合演算子、??を追加する :TODO

**[整数セマンティクス](Integer_Semantics.md)**  
整数を扱ういくつかの演算で、PHPのクロスプラットフォーム一貫性を改善する :TODO

**[オーバーフロー時ZPPを失敗させる](ZPP_Failure_on_Overflow.md)**  
整数を期待している場所で範囲外の浮動小数点数や非数を渡された場合、zend_parse_parametersを失敗させる（2014-09-22作成） :TODO

**[phpngブランチをmasterブランチへ移動する](Move_the_phpng_branch_into_master.md)**  
phpngコードベースを将来のPHPメジャーバージョンの基礎として受け入れる（2014-07-20作成） :TODO

**[抽象構文木](Abstract_Syntax_Tree.md)**  
コンパイル処理の仲介構造として抽象構文木の導入を提案する :TODO

**[均一な変数構文](Uniform_Variable_Syntax.md)**  
内部的に一貫した完全な変数構文を導入する :TODO

**[64ビットプラットフォームの文字列の長さと整数の改善](64_bit_platform_improvements_for_string_length_and_integer.md)**  
64ビットをフルサポートする整数と文字列の変更 :TODO

**[Closure::call](Closure_call.md)**  
事前バインドなしにオブジェクトをバインドして呼び出すための、Closureクラスの新しいメソッドを提案する :TODO

**[list()の一貫性のない振る舞いを修正する](Fix_list_behavior_inconsistency.md)**  
list()演算子で文字列の操作を可能にする、あるいは不可能にする :TODO

**[PHPの代替タグを削除する](Remove_alternative_PHP_tags.md)**  
ASP形式とscript形式のタグを削除する :TODO

**[switch.default.multiple](switch.default.multiple.md)**  
switch文で複数のdefaultを許可しない :TODO

**["call to a member function of a non-object"をキャッチできるようにする](Catchable_call_to_a_member_function_of_a_non-object.md)**  
この致命的なエラーをE_RECOVERABLEに変更する（2014-04-26作成） :TODO

**[フィルタされたunserialize()](Filtered_unserialize.md)**  
全て、あるいはいくつかのオブジェクトを無視するオプションをunserialize()に追加する（2013/03/30作成） :TODO

**[ICU IntlCharクラス](ICU_IntlChar_class.md)**  
IntlエクステンションにIntlCharクラス、intl_char_*()関数を追加する :TODO

**[session_start()のINIオプションを配列として導入する](Introduce_session_start_INI_options_as_array.md)**  
session_start()のオプションを導入する :TODO

**[数値文字列の16進数サポートを削除する](Remove_hex_support_in_numeric_strings.md)**  
数値文字列の変換で16進数のサポートを削除する（2014-08-19作成） :TODO

**[Expectations](Expectations.md)**  
このRFCは、後方互換性がありコストのないアサーションを追加することを提案する（2013-11-01作成） :TODO

**[use宣言をまとめる](Group_Use_Declarations.md)**  
このRFCは、use宣言をまとめられるようにすることで、現在のPHPの名前空間の実装を改善する（2015-01-28作成） :TODO

**[エンジンでの例外](Exceptions_in_the_engine.md)**  
このRFCはエンジンで例外を使用できるようにすることを提案する（2014-09-30作成） :TODO

**[ジェネレータのreturn式](Generator_Return_Expressions.md)**  
このRFCはジェネレータ関数から戻り値を指定し、それにアクセス可能にすることを提案する :TODO

**[スカラ型のタイプヒントv0.5](Scalar_Type_Hints_v0.5.md)**  
このRFCはモードを混合できるスカラ型システムを提案する :TODO

**[出力バッファリングを継続する](Continue_output_buffering.md)**  
接続が中断しても出力バッファスタックを使用できるようにする :TODO

**[intdiv()](intdiv.md)**  
このRFCは整数の除算のためのintdiv()関数を追加することを提案する（2014-07-15作成） :TODO

**[カスタムセッションハンドラの戻り値の処理を修正する](Fix_handling_of_custom_session_handler_return_values.md)**  
falseが実際に成功ではなく失敗を意味するようにする :TODO

**[gc_collect_cyclesを関数ポインタにする](Turn_gc_collect_cycles_into_function_pointer.md)**  
gc_collect_cyclesを、エクステンションがフックできる関数ポインタに変更することを提案する（2012-12-11作成） :TODO
