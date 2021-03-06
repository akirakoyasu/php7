====== PHP RFC: Add preg_replace_callback_array function ======
  * Version: 1.0
  * Date: 2015-03-10
  * Author: Wei Dai, demon@php.net
  * Status: Implemented (in PHP 7.0)
  * First Published at: https://wiki.php.net/rfc/preg_replace_callback_array

===== Introduction =====
Before 5.5.0, we can use the '/e' modifier. Here is a piece of code with 5.4.x:

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

Since 5.5.0, we deprecated the /e modifier, so we have to change it to:

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

===== Proposal =====
The preg_replace_callback_array function is an extension to preg_replace_callback. With the function, each pattern can easily have a specific callback.

This is the best way to implement when there are multiple patterns.

With preg_replace_callback_array, the code is more readable, and full of logic. There is less crazy branch determination.

With the preg_replace_callback_array function, the code is:

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

The first parameter is an array in this function. In this parameter, Key is pattern, and value is callback. Subject will iterate and match each key. If it is matched, the callback will be called. In the meanwhile, the result will be the new subject and passed to the next match.


===== Proposed PHP Version(s) =====
This is proposed for PHP7

===== Unaffected PHP Functionality =====
preg_filter(), preg_replace(), preg_replace_callback() will stay the same.

===== Proposed Voting Choices =====
Include these so readers know where you are heading and can discuss the proposed voting options.

State whether this project requires a 2/3 or 50%+1 majority (see [[voting]])

===== Patches and Tests =====
Currently implemented on https://github.com/php/php-src/pull/1171
