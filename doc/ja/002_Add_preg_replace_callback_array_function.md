====== PHP RFC: Add preg_replace_callback_array function ======
  * Version: 1.0
  * Date: 2015-03-10
  * Author: Wei Dai, demon@php.net
  * Status: Implemented (in PHP 7.0)
  * First Published at: https://wiki.php.net/rfc/preg_replace_callback_array

===== 導入 =====

5.5.0以前、我々は'/e'修飾子を使用することができた。5.4.xでのコードが以下である：

```php
Zend/zend_vm_gen.php

$code = preg_replace(
                array(
                    "/EXECUTE_DATA/m",
                    "/ZEND_VM_DISPATCH_TO_HANDLER\(\s*([A-Z_]*)\s*\)/m",
                    "/ZEND_VM_DISPATCH_TO_HELPER\(\s*([A-Za-z_]*)\s*\)/me",
                    "/ZEND_VM_DISPATCH_TO_HELPER_EX\(\s*([A-Za-z_]*)\s*,\s*[A-Za-z_]*\s*,\s*(.*)\s*\);/me",
                ),
                array(
                    "execute_data",
                    "return \\1".($spec?"_SPEC":"").$prefix[$op1].$prefix[$op2]."_HANDLER(ZEND_OPCODE_HANDLER_ARGS_PASSTHRU)",
                    "'return '.helper_name('\\1',$spec,'$op1','$op2').'(ZEND_OPCODE_HANDLER_ARGS_PASSTHRU)'",
                    "'return '.helper_name('\\1',$spec,'$op1','$op2').'(\\2, ZEND_OPCODE_HANDLER_ARGS_PASSTHRU);'",
                ),
                $code);
```

5.5.0以降、我々は /e 修飾子を非推奨としたため、このように変更する必要がある：
```php
Zend/zend_vm_gen.php

$code = preg_replace_callback(
    array(
        "/EXECUTE_DATA/m",
	"/ZEND_VM_DISPATCH_TO_HANDLER\(\s*([A-Z_]*)\s*\)/m",
	"/ZEND_VM_DISPATCH_TO_HELPER\(\s*([A-Za-z_]*)\s*\)/m",
	"/ZEND_VM_DISPATCH_TO_HELPER_EX\(\s*([A-Za-z_]*)\s*,\s*[A-Za-z_]*\s*,\s*(.*)\s*\);/m",
    ),
    function($matches) use ($spec, $prefix, $op1, $op2) {
	if (strncasecmp($matches[0], "EXECUTE_DATA", strlen("EXECUTE_DATA")) == 0) {
	    return "execute_data";
        } else if (strncasecmp($matches[0], "ZEND_VM_DISPATCH_TO_HANDLER", strlen("ZEND_VM_DISPATCH_TO_HANDLER")) == 0) {
	    return "return " . $matches[1] . ($spec?"_SPEC":"") . $prefix[$op1] . $prefix[$op2] . "_HANDLER(ZEND_OPCODE_HANDLER_ARGS_PASSTHRU)";
        } else if (strncasecmp($matches[0], "ZEND_VM_DISPATCH_TO_HELPER_EX", strlen("ZEND_VM_DISPATCH_TO_HELPER_EX")) == 0) {
	    return "return " . helper_name($matches[1], $spec, $op1, $op2) . "(" . $matches[2]. ", ZEND_OPCODE_HANDLER_ARGS_PASSTHRU);";
        } else {
	    return "return " . helper_name($matches[1], $spec, $op1, $op2) . "(ZEND_OPCODE_HANDLER_ARGS_PASSTHRU)";
        }
    }, $code);
```

===== 提案 =====

preg_replace_callback_array関数はpreg_replace_callbackの拡張である。この関数を使えば、それぞれのパターンに
容易に特定のコールバックを持たせられる。

これは複数のパターンがあるときの最適な実装方法である。

preg_replace_callback_arrayによって、コードはより読みやすくなり、ロジックで満たされる。
おかしな分岐の解決はより少なくなる。

preg_replace_callback_arrayを使えば、コードはこのようになる：

```php
Zend/zend_vm_gen.php

$code = preg_replace_callback_array(
	array(
		"/EXECUTE_DATA/m" => function($matches) {
                        return "execute_data";
                 },
		"/ZEND_VM_DISPATCH_TO_HANDLER\(\s*([A-Z_]*)\s*\)/m" => function($matches) use ($spec, $prefix, $op1, $op2) {
		       return "return " . $matches[1] . ($spec?"_SPEC":"") . $prefix[$op1] . $prefix[$op2] . "_HANDLER(ZEND_OPCODE_HANDLER_ARGS_PASSTHRU)";
		},
		"/ZEND_VM_DISPATCH_TO_HELPER\(\s*([A-Za-z_]*)\s*\)/m" => function($matches) use ($spec, $prefix, $op1, $op2) {
			return "return " . helper_name($matches[1], $spec, $op1, $op2) . "(ZEND_OPCODE_HANDLER_ARGS_PASSTHRU)";
		},
		"/ZEND_VM_DISPATCH_TO_HELPER_EX\(\s*([A-Za-z_]*)\s*,\s*[A-Za-z_]*\s*,\s*(.*)\s*\);/m" => function($matches) use ($spec, $prefix, $op1, $op2) {
			return "return " . helper_name($matches[1], $spec, $op1, $op2) . "(" . $matches[2]. ", ZEND_OPCODE_HANDLER_ARGS_PASSTHRU);";
		},
	), $code);
```

この関数の1番目の引数は配列である。この引数のうち、キーがパターンであり、値がコールバックである。
対象は繰り返しそれぞれのキーにマッチするか検査される。もしマッチすれば、コールバックが呼ばれる。その間に、
結果が新しい対象として次の検査に渡される。

===== 提案するPHPバージョン =====

This is proposed for PHP7

===== 影響のないPHP機能 =====

preg_filter(), preg_replace(), preg_replace_callback() will stay the same.

===== Proposed Voting Choices =====

Include these so readers know where you are heading and can discuss the proposed voting options.

State whether this project requires a 2/3 or 50%+1 majority (see [[voting]])

===== Patches and Tests =====

Currently implemented on https://github.com/php/php-src/pull/1171
