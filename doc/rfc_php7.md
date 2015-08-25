# PHP 7.0

**[Context Sensitive Lexer](001_Context_Sensitive_Lexer.md)**  
Proposal to have a context sensitive lexer for PHP with support for semi-reserved words (Created 2015-02-15)

**[Add preg_replace_callback_array function](002_Add_preg_replace_callback_array_function.md)**  
A RFC proposing each pattern can easily have a specific callback.

**[Reliable User-land CSPRNG](003_Reliable_User-land_CSPRNG.md)**  
A proposal to add a reliable CSPRNG (Created 2015-02-20)

**[Anonymous Class Support](004_Anonymous_Class_Support.md)**  
This RFC proposes support for anonymous classes.

**[Generator Delegation](005_Generator_Delegation.md)**  
This RFC proposes new syntax to delegate generator function operations to sub-iterators/generators (Created 2015-03-01)

**[Reserve More Type Names in PHP 7](006_Reserve_More_Type_Names_in_PHP_7.md)**  
Reserves int, float, string, bool, true, false and null in class names or namespaces. (Created 2015-02-18)

**[Constructor behaviour of internal classes](007_Constructor_behaviour_of_internal_classes.md)**  
Cleanup undesirable behaviour of constructors internal classes. (Created 2015-03-01)

**[Reclassify E_STRICT notices](008_Reclassify_E_STRICT_notices.md)**  
This RFC proposes to reclassify our few existing E_STRICT notices to different categories. (Created 2015-02-22)

**[Remove PHP 4 Constructors](009_Remove_PHP_4_Constructors.md)**  
Stop recognizing methods with the same name as the defining class as constructors. (Created 2014-11-17; Voting closed 2015-03-10)

**[Remove the date.timezone warning](010_Remove_the_date.timezone_warning.md)**  
(Created 2015-01-27)

**[Combined Comparison (Spaceship) Operator](011_Combined_Comparison_Spaceship_Operator.md)**  
Proposes a new comparison operator, <=> (Created 2014-02-12, revived 2015-01-19)

**[Fix "foreach" behavior](012_Fix_foreach_behavior.md)**  
The RFC defines foreach statement behavior for the cases where is wasn't defined before and implementation provided inconsistent results. (Created 2015-01-29)

**[Removal of dead SAPIs and extensions](013_Removal_of_dead_SAPIs_and_extensions.md)**  
Consideration about removing the unsupported SAPIs and extensions with unsupported dependencies.

**[Jsond](014_Jsond.md)**  
Whether jsond should replace the current json extension. (Created 2015-01-11)

**[Preserve Fractional Part in JSON encode](015_Preserve_Fractional_Part_in_JSON_encode.md)**  
Adds a new option for JSON encode to preserve fractional part on float numbers.

**[Return Type Declarations](016_Return_Type_Declarations.md)**  
Adds return types to functions, methods and closures. (Created 2014-03-20)

**[Fast Parameter Parsing API](017_Fast_Parameter_Parsing_API.md)**  
Fast API in addition to zend_parse_parameters(). (Created 2014-05-23)

**[Unicode Codepoint Escape Syntax](018_Unicode_Codepoint_Escape_Syntax.md)**  
Adds an escape sequence syntax for Unicode codepoints to string literals. (Created 2014-11-24)

**[Native TLS](019_Native_TLS.md)**  
Native TLS for internal globals in TS mode

**[Null Coalesce Operator](020_Null_Coalesce_Operator.md)**  
Adds the coalesce operator, ??

**[Integer Semantics](021_Integer_Semantics.md)**  
Improves cross-platform consistency in PHP for some operations dealing with integers

**[ZPP Failure on Overflow](022_ZPP_Failure_on_Overflow.md)**  
Make zend_parse_parameters fail if a float value out of bounds, or NaN, is passed where an integer is expected. (Created 2014-09-22)

**[Move the phpng branch into master](023_Move_the_phpng_branch_into_master.md)**  
Embrace the phpng codebase as the basis for the future major version of PHP. (Created 2014-07-20)

**[Abstract Syntax Tree](024_Abstract_Syntax_Tree.md)**  
Proposes the introduction of an Abstract Syntax Tree (AST) as an intermediary structure in our compilation process.

**[Uniform Variable Syntax](025_Uniform_Variable_Syntax.md)**  
Introduces an internally consistent and complete variable syntax.

**[64 bit platform improvements for string length and integer](026_64_bit_platform_improvements_for_string_length_and_integer.md)**  
Integer and String modifications for full 64 bit support

**[Closure::call](027_Closure_call.md)**  
Proposes a new method on the Closure class to allow calling bound to an object without pre-binding

**[Fix list() behavior inconsistency](028_Fix_list_behavior_inconsistency.md)**  
Enable or disable string handling in list() operator

**[Remove alternative PHP tags](029_Remove_alternative_PHP_tags.md)**  
Removes ASP and script tags

**[switch.default.multiple](030_switch.default.multiple.md)**  
Disallow multiple defaults in switch statements

**[Catchable "call to a member function of a non-object"](031_Catchable_call_to_a_member_function_of_a_non-object.md)**  
Turns this fatal error into E_RECOVERABLE (Created 2014-04-26)

**[Filtered unserialize()](032_Filtered_unserialize.md)**  
Add option to ignore all or some objects to unserialize() (Created 2013/03/30)

**[ICU IntlChar class](033_ICU_IntlChar_class.md)**  
Adds an IntlChar class an intl_char_*() functions to the Intl extension.

**[Introduce session_start() INI options as array](034_Introduce_session_start_INI_options_as_array.md)**  
Introduce session_start() options

**[Remove hex support in numeric strings](035_Remove_hex_support_in_numeric_strings.md)**  
Removes support for hexadecimal numbers in numeric string conversions. (Created 2014-08-19)

**[Expectations](036_Expectations.md)**  
This RFC proposes adding BC zero-cost assertions. (Created 2013-11-01)

**[Group Use Declarations](037_Group_Use_Declarations.md)**  
The RFC adds improvements to current PHP namespace implementation by introducing group use declarations. (Created 2015-01-28)

**[Exceptions in the engine](038_Exceptions_in_the_engine.md)**  
This RFC proposes to allow the use of exceptions in the engine. (Created 2014-09-30)

**[Generator Return Expressions](039_Generator_Return_Expressions.md)**  
This RFC proposes the ability to specify and access return values from Generator functions

**[Scalar Type Hints v0.5](040_Scalar_Type_Hints_v0.5.md)**  
This RFC proposes a mixed-mode scalar type system

**[Continue output buffering](041_Continue_output_buffering.md)**  
Let the output buffer stack be usable despite an aborted connection

**[intdiv()](042_intdiv.md)**  
This RFC proposes adding an intdiv() function for integer division. (Created 2014-07-15)

**[Fix handling of custom session handler return values](043_Fix_handling_of_custom_session_handler_return_values.md)**  
Make false actually mean failure, not success.

**[Turn gc_collect_cycles into function pointer](044_Turn_gc_collect_cycles_into_function_pointer.md)**  
Proposes to turn gc_collect_cycles into function pointer for extensions to hook into. (Created 2012-12-11)
