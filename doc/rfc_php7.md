# PHP 7.0

**[Context Sensitive Lexer](Context_Sensitive_Lexer.md)**  
Proposal to have a context sensitive lexer for PHP with support for semi-reserved words (Created 2015-02-15)

**[Add preg_replace_callback_array function](Add_preg_replace_callback_array_function.md)**  
A RFC proposing each pattern can easily have a specific callback.

**[Reliable User-land CSPRNG](Reliable_User-land_CSPRNG.md)**  
A proposal to add a reliable CSPRNG (Created 2015-02-20)

**[Anonymous Class Support](Anonymous_Class_Support.md)**  
This RFC proposes support for anonymous classes.

**[Generator Delegation](Generator_Delegation.md)**  
This RFC proposes new syntax to delegate generator function operations to sub-iterators/generators (Created 2015-03-01)

**[Reserve More Type Names in PHP 7](Reserve_More_Type_Names_in_PHP_7.md)**  
Reserves int, float, string, bool, true, false and null in class names or namespaces. (Created 2015-02-18)

**[Constructor behaviour of internal classes](Constructor_behaviour_of_internal_classes.md)**  
Cleanup undesirable behaviour of constructors internal classes. (Created 2015-03-01)

**[Reclassify E_STRICT notices](Reclassify_E_STRICT_notices.md)**  
This RFC proposes to reclassify our few existing E_STRICT notices to different categories. (Created 2015-02-22)

**[Remove PHP 4 Constructors](Remove_PHP_4_Constructors.md)**  
Stop recognizing methods with the same name as the defining class as constructors. (Created 2014-11-17; Voting closed 2015-03-10)

**[Remove the date.timezone warning](Remove_the_date.timezone_warning.md)**  
(Created 2015-01-27)

**[Combined Comparison (Spaceship) Operator](Combined_Comparison_Spaceship_Operator.md)**  
Proposes a new comparison operator, <=> (Created 2014-02-12, revived 2015-01-19)

**[Fix "foreach" behavior](Fix_foreach_behavior.md)**  
The RFC defines foreach statement behavior for the cases where is wasn't defined before and implementation provided inconsistent results. (Created 2015-01-29)

**[Removal of dead SAPIs and extensions](Removal_of_dead_SAPIs_and_extensions.md)**  
Consideration about removing the unsupported SAPIs and extensions with unsupported dependencies.

**[Jsond](Jsond.md)**  
Whether jsond should replace the current json extension. (Created 2015-01-11)

**[Preserve Fractional Part in JSON encode](Preserve_Fractional_Part_in_JSON_encode.md)**  
Adds a new option for JSON encode to preserve fractional part on float numbers.

**[Return Type Declarations](Return_Type_Declarations.md)**  
Adds return types to functions, methods and closures. (Created 2014-03-20)

**[Fast Parameter Parsing API](Fast_Parameter_Parsing_API.md)**  
Fast API in addition to zend_parse_parameters(). (Created 2014-05-23)

**[Unicode Codepoint Escape Syntax](Unicode_Codepoint_Escape_Syntax.md)**  
Adds an escape sequence syntax for Unicode codepoints to string literals. (Created 2014-11-24)

**[Native TLS](Native_TLS.md)**  
Native TLS for internal globals in TS mode

**[Null Coalesce Operator](Null_Coalesce_Operator.md)**  
Adds the coalesce operator, ??

**[Integer Semantics](Integer_Semantics.md)**  
Improves cross-platform consistency in PHP for some operations dealing with integers

**[ZPP Failure on Overflow](ZPP_Failure_on_Overflow.md)**  
Make zend_parse_parameters fail if a float value out of bounds, or NaN, is passed where an integer is expected. (Created 2014-09-22)

**[Move the phpng branch into master](Move_the_phpng_branch_into_master.md)**  
Embrace the phpng codebase as the basis for the future major version of PHP. (Created 2014-07-20)

**[Abstract Syntax Tree](Abstract_Syntax_Tree.md)**  
Proposes the introduction of an Abstract Syntax Tree (AST) as an intermediary structure in our compilation process.

**[Uniform Variable Syntax](Uniform_Variable_Syntax.md)**  
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
