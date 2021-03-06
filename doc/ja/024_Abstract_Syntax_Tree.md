====== PHP RFC: Abstract syntax tree ======
  * Date: 2014-07-28
  * Author: Nikita Popov <nikic@php.net>
  * Status: Implemented (in PHP 7)
  * Discussion: http://markmail.org/message/br4ixewsnqitrx3n

===== 導入 =====

このRFCは、コンパイル処理中の中間的な構造として抽象構文木（AST）の導入を提案する。これはopcodeを
パーサから直接発行する既存の手法を置き換える。

パーサとコンパイラの分離によって、いくつものハックが不要になり、一般的に実装のメンテナンス性や
わかりやすさが向上する。さらに、シングルパスのコンパイル処理では実現できない構文の実装が可能になる。

===== 抽象構文木の利点 =====

コンパイラ実装の標準的な手法であるという以外に、ASTの利用には２つの主要な利点がある：

==== よりメンテナンス性の高いパーサとコンパイラ ====

新しいASTベースの実装では、コンパイラはパーサと完全に分離されるため、コード品質とメンテナンス性が
向上する。そういった改善の例として以下が議論された：

- 同じ構文に別のコンパイルが必要となるとき、パーサが別々の成果物を定義する必要がなくなる。例えば
 静的なスカラ式は全ての基本的な操作を再定義する必要はなく、標準的な```expr```成果物を再利用する
 ことができる。
- パーサは中間ルールのアクションをより少なくする必要がある。中間ルールによる変換は、以前は至るところで
 使われていたが、現在はdocコメントのバックアップにしか使われていない。コード品質の懸念を除いて、
 中間ルールのアクションはパーサにより早期の変換を強制するため、これは利益をもたらす。すなわち
 変換されるルールを決定するためにパーサはより少ないトークンを検査すればよい。これは実装可能な構文を制限する。
- フロー制御構造の実装は、中間ルールのアクションと呼ばれる複数の機能にわたって広がっている。
 ジャンプ命令のopnumは任意のznodeに支えられており（パーサ、コンパイラとも）、コードを追うのが
 非常に困難となっている。コンパイラのコードを読むとき、常に以下のような疑問を抱くに違いない：
 ```%%close_bracket_token->u.op.opline_num%%```は何を参照しているのか？ このoplineは
 どこに生成され、どのopcodeが持つのか？ ```op2.opline_num```のoplineが変更されることには
 どういう意味があるのか？
-  従来、変数はバックパッチリスト（とスタック）によって実装されていた。そこに```BP_VAR_W```
 フェッチのために必要なoplineが挿入される。その後、最終的に選択されたフェッチモードによって
 oplineは変更あるいは削除される。ASTベースの実装であれば、正しいフェッチモードを使って直接
 コンパイルできる。
- もはや他のいくつもの場所でバックパッチする必要はない。例えば```list()```のコンパイル中など。

==== Decoupling syntax decisions from technical issues ====

With the current single-pass compiler some syntactical elements are very hard, or impossible, to implement. This has influenced a number of syntactical decisions in the past. One example is the restriction that ''yield'' expressions must be wrapped in parentheses when used in expression context:

<code php>
$result = yield fn();   // INVALID
$result = (yield fn()); // VALID
</code>

This choice was made purely due to technical restrictions and is removed in the AST implementation.

Additionally the current compiler architecture prevents us from implementing some types of syntax altogether. Examples include:

  * Array destructuring using ''[$a, $b, $c] = $array'' instead of a dedicated ''list()'' syntax. This is common in other languages, but not possible in PHP.
  * List comprehensions / generator expressions where the result expression comes first, e.g. ''[x * x for x in list]'' in Python. In PHP only the reverse syntax is possible: ''[foreach ($list as $x) yield $x * $x]''
  * C#-style expression trees and LINQ. These require an AST pretty much by definition.

Whether or not we actually want these things, I think it's important that syntax is evaluated based on its merit and not based on technical restrictions.

===== Impact on performance and memory usage =====

The abstract syntax tree has little impact on runtime performance or memory usage. While the AST does allow us to generate slightly better and smaller instruction sequences in some cases (e.g. for ''for'' loops), the practical impact is not worth mentioning.

The introduction of an AST does however impact the performance and memory usage of the compilation process itself. It should be emphasized that this difference is //only relevant when no opcode cache is in use//. If an opcode cache is used then each file is only compiled once, as such any difference does not have a practical impact.

The script used to measure the following numbers is available as a [[https://gist.github.com/nikic/289b0c7538b46c2220bc|Gist]]. Tests were performed on three files, with different sizes. The "small" one has about 100 lines of code, the "medium" one about 700 and the "large" one about 2800.

The following table shows the time that is needed to compile each of the files 1000 times:

|        ^ php-ng ^ php-ast ^   diff |
^ small  | 0.180s | 0.160s  | -12.5% |
^ medium | 1.492s | 1.268s  | -17.7% |
^ large  | 6.703s | 5.736s  | -16.9% |

The following table shows the peak memory usage during a single compilation((Doing multiple compiles here is pointless, because the memory needed for compilation is much lower than the memory needed for the opcodes of 1000 compiled files)):

|        ^ php-ng ^ php-ast ^   diff |
^ small  |  378kB |   414kB |  +9.5% |
^ medium |  507kB |   643kB | +26.8% |
^ large  | 1084kB |  1857kB | +71.3% |

As compilation of individual files is not very representative of practical usage, I additionally measured the time and memory necessary to include (compile) all files from the [[https://github.com/nikic/PHP-Parser/tree/master/lib/PhpParser|PhpParser project]]. The results are shown in the following table:

|        ^ php-ng ^ php-ast ^   diff |
^ time   | 25.5ms |  22.8ms | -11.8% |
^ memory | 2360kB |  2482kB |  +5.1% |

To summarize the results: The AST based implementation is 10-15% faster, but requires more memory. The amount of additional memory heavily depends on the file size. A small file needs an additional 10% of memory, whereas a very large file needs 70%. In the more realistic case where multiple files are compiled, the difference in memory usage drops to 5%, because at this point the memory necessary to hold all the compiled scripts is much larger than the memory necessary to compile one.

===== Changes to syntax or behavior =====

The introduction of the AST comes with minor changes to syntax and behavior, which are listed in the following:

==== yield does not require parentheses ====

This is the only syntax related change and was already mentioned previously. ''yield'' in expression use no longer needs parentheses, so all of the following are valid:

<code php>
$result = yield;
$result = yield $v;
$result = yield $k => $v;
</code>

==== Parentheses do not influence behavior ====

An open issue of the [[rfc:uniform_variable_syntax|Uniform Variable Syntax RFC]] was that ''%%($foo)['bar'] = 'baz'%%'' and ''%%$foo['bar'] = 'baz'%%'' did not exhibit the same behavior, because they were compiled with different fetch modes.

Similarly ''byRef(func())'' and ''%%byRef((func()))%%'' will now both throw a strict-standards notice if ''byRef'' takes its parameter by-reference, but ''func'' does not return by reference.

==== Changes to list() ====

''list()'' currently assigns variables right-to-left, the AST implementation will assign them left-to-right instead:

<code php>
list($array[], $array[], $array[]) = [1, 2, 3];
var_dump($array);

// OLD: $array = [3, 2, 1]
// NEW: $array = [1, 2, 3]
</code>

Another example where the assignment order is relevant is if both the left and right side of the list assignment use the same variable:

<code php>
$a = [1, 2];
list($a, $b) = $a;

// OLD: $a = 1, $b = 2
// NEW: $a = 1, $b = null + "Undefined index 1"

$b = [1, 2];
list($a, $b) = $b;
// OLD: $a = null + "Undefined index 0", $b = 2
// NEW: $a = 1, $b = 2
</code>

''list()'' will now access every offset only once:

<code php>
list(list($a, $b)) = $array;

// OLD:
$b = $array[0][1];
$a = $array[0][0];

// NEW:
$_tmp = $array[0];
$a = $_tmp[0];
$b = $_tmp[1];
</code>

The only visible change this has for most purposes is that an "Undefined index" notice is only thrown once per index, not many times.

Empty ''list()''s are now always disallowed. Previously they were only forbidden in some places:

<code php>
list() = $a;           // INVALID
list($b, list()) = $a; // INVALID
foreach ($a as list()) // INVALID (was also invalid previously)
</code>

==== Auto-vivification order for by-reference assignments ====

While by-reference assignments are (CVs notwithstanding) evaluated left-to-right, auto-vivification currently occurs right-to-left. In the AST implementation this will happen left-to-right instead:

<code php>
$obj = new stdClass;
$obj->a = &$obj->b;
$obj->b = 1;
var_dump($obj);

// OLD:
object(stdClass)#1 (2) {
  ["b"]=>
  &int(1)
  ["a"]=>
  &int(1)
}

// NEW:
object(stdClass)#1 (2) {
  ["a"]=>
  &int(1)
  ["b"]=>
  &int(1)
}
</code>

Note: The order can easily be changed, but the old behavior looks like a bug to me, so I decided to keep the new behavior.

==== Directly calling __clone is allowed ====

Doing calls like ''%%$obj->__clone()%%'' is now allowed. This was the only magic method that had a compile-time check preventing some calls to it, which doesn't make sense. If we allow all other magic methods to be called, there's no reason to forbid this one.

===== Implementation =====

==== Overview ====

The process for converting a PHP file into opcodes now consists of three phases:

  - Lexing: the generation of a token stream from the source code
  - Parsing: the generation of an abstract syntax tree from the token stream.
  - Compilation: the generation of op arrays from the abstract syntax tree.

The lexer is defined in ''zend_language_scanner.l'' and generated using re2c. The lexer returns token IDs one at a time and optionally provides a semantic value zval (e.g. containing the name of a variable token).

The parser is defined in ''zend_language_parser.y'' and generated using bison. The parser consumes the tokens provided by the lexer and executes semantic actions based on the encountered token sequence. These semantic actions generate the abstract syntax tree, which is finally written into ''CG(ast)''.

The parser uses the LALR(1) parsing algorithm, which means that it only has one token of lookahead to distinguish different syntactic structures.

The compiler consumes the AST and generates op arrays from it (one per file and function). This happens by recursively walking the AST by invoking ''zend_compile_*'' functions. These compilation functions emit opcodes, i.e. instructions for the Zend VM.

In the following the AST API is outlined, as well as its usage in the parser and the compiler.

==== AST API ====

=== AST node structure and creation ===

A standard AST node is defined as follows:

<code c>
typedef unsigned short zend_ast_kind;
typedef unsigned short zend_ast_attr;

typedef struct _zend_ast {
    zend_ast_kind kind;
    zend_ast_attr attr;
    zend_uint lineno;
    struct _zend_ast *child[1];
} zend_ast;
</code>

''kind'' is a ''ZEND_AST_*'' enum constant indicating the type of the AST node, e.g. ''ZEND_AST_BINARY_OP'' for a binary operation. ''attr'' is a unsigned short that can be used to store kind-specific flags. ''lineno'' is the start line number of the node.

Child nodes are stored in the ''child'' array. The size of this array is determined during allocation based on the kind. Nodes are created using ''zend_ast_create'' or ''zend_ast_create_ex'', depending on whether you want to make use of ''attr'':

<code c>
zend_ast *zend_ast_create_ex(zend_ast_kind kind, zend_ast_attr attr, ...);
zend_ast *zend_ast_create(zend_ast_kind kind, ...);
</code>

For example:

<code c>
zend_ast *ast = zend_ast_create_ex(ZEND_AST_BINARY_OP, ZEND_ADD, left_ast, right_ast);
</code>

AST nodes created this way have a fixed number of children determined by the AST kind. For cases where the number of children is determined dynamically (e.g. arrays, argument lists, statement lists, etc) the type ``zend_ast_list`` is used instead. It is identical to ordinary AST nodes, but contains an additional children count:

<code c>
typedef struct _zend_ast_list {
    zend_ast_kind kind;
    zend_ast_attr attr;
    zend_uint lineno;
    zend_uint children;
    zend_ast *child[1];
} zend_ast_list;
</code>

List nodes are created using ''zend_ast_create_list'' and children are appended using ''zend_ast_list_add''.

<code c>
zend_ast_list *zend_ast_create_list(zend_uint init_children, zend_ast_kind kind, ...);
zend_ast_list *zend_ast_list_add(zend_ast_list *list, zend_ast *op);
</code>

For example, creating and appending to an array AST:

<code c>
/* Initialize array with two elems */
zend_ast_list *list = zend_ast_create_list(2, ZEND_AST_ARRAY,
    zend_ast_create(ZEND_AST_ARRAY_ELEM, value1_ast, key1_ast),
    zend_ast_create(ZEND_AST_ARRAY_ELEM, value2_ast, key2_ast));

/* Add another element afterwards */
list = zend_ast_list_add(list,
    zend_ast_create(ZEND_AST_ARRAY_ELEM(value3_ast, key3_ast));
</code>

Lastly an AST node can store a ''zval''. For this purpose the ''ZEND_AST_ZVAL'' kind is used in conjunction with the ''zend_ast_zval'' structure:

<code c>
/* Lineno is stored in val.u2.lineno */
typedef struct _zend_ast_zval {
    zend_ast_kind kind;
    zend_ast_attr attr;
    zval val;
} zend_ast_zval;
</code>

Zval AST nodes are created using ''zend_ast_create_zval'' or ''zend_ast_create_zval_ex'' (in case ''attr'' is used). There are two additional convenience functions which create a zval AST node from a string or a long:

<code c>
zend_ast *zend_ast_create_zval_ex(zval *zv, zend_ast_attr attr);
zend_ast *zend_ast_create_zval(zval *zv);

zend_ast *zend_ast_create_zval_from_str(zend_string *str);
zend_ast *zend_ast_create_zval_from_long(long lval);
</code>

These functions return the node cast to ''zend_ast*'' for practical purposes. The ''zend_ast_zval'' structure is only used internally and all external code works through ''zend_ast*''.

There is another special node type for class and function declarations, which is not documented here.

=== Usage in the parser ===

The parser stack now uses ''zend_parser_stack_elem'' unions rather than ''znode''s. The union is defined as follows:

<code c>
typedef union _zend_parser_stack_elem {
	zend_ast *ast;
	zend_ast_list *list;
	zend_string *str;
	zend_ulong num;
} zend_parser_stack_elem;
</code>

The ''ast'' member is used when creating ordinary AST nodes:

<code>
callable_variable:
        simple_variable
            { $$.ast = zend_ast_create(ZEND_AST_VAR, $1.ast); }
    |   dereferencable '[' dim_offset ']'
            { $$.ast = zend_ast_create(ZEND_AST_DIM, $1.ast, $3.ast); }
    |   dereferencable '{' expr '}'
            { $$.ast = zend_ast_create(ZEND_AST_DIM, $1.ast, $3.ast); }
    |   dereferencable T_OBJECT_OPERATOR member_name argument_list
            { $$.ast = zend_ast_create(ZEND_AST_METHOD_CALL, $1.ast, $3.ast, $4.ast); }
    |   function_call { $$.ast = $1.ast; }
;
</code>

When lists are created or modified, the ''list'' member is used instead:

<code>
inner_statement_list:
        inner_statement_list inner_statement
            { $$.list = zend_ast_list_add($1.list, $2.ast); }
    |   /* empty */
            { $$.list = zend_ast_create_list(0, ZEND_AST_STMT_LIST); }
;
</code>

The ''str'' member is used to back up doc comments during parsing and ''num'' is utilized to back up line numbers or store flags.

=== Retrieving information from AST nodes ===

For ordinary AST nodes you can directly access the children using ''%%ast->child[0]%%'' and so on.

When working with a list node you must first retrieve the list using ''zend_ast_get_list'' (this is effectively just a cast to the ''zend_ast_list*'' type). Afterwards you can iterate through the list as follows:

<code c>
zend_ast_list *list = zend_ast_get_list(ast);

zend_uint i;
for (i = 0; i < list->children; ++i) {
    zend_ast *elem = list->child[i];
    /* ... */
}
</code>

The zval from a zval AST node is fetched using ''zend_ast_get_zval''. As the zval is commonly known to be a string an additional ''zend_ast_get_str'' function is provided, which returns a ''zend_string*''.

Apart from these, there are a number of introspection function, which are useful work generic code working on AST nodes:

  * ''zend_ast_get_lineno'' will return the starting line number for all AST node types.
  * ''zend_ast_is_list'' returns whether an AST node is a list
  * ''zend_ast_get_num_children'' returns the number of children a **non-list** node has.

=== AST allocation, destruction and copy ===

As the abstract syntax tree is only necessary during compilation and discarded afterwards, AST nodes make use of an arena allocator. The arena is stored in ''CG(ast_arena)''. Before invoking ''zendparse'' this arena must be allocated using ''zend_arena_create''.

Due to the usage of an arena allocator AST nodes do not need to be individually freed, however zvals held by them still need to be destroyed. This is accomplished using ''zend_ast_destroy'', which will recursively walk the AST and dtor all held values. The following code features a sample invocation of the parser, including arena handling:

<code c>
CG(ast_arena) = zend_arena_create(1024 * 32);
compiler_result = zendparse(TSRMLS_C);
if (compiler_result != 0) {
    zend_bailout();
}
zend_compile_top_stmt(CG(ast) TSRMLS_CC);
zend_ast_destroy(CG(ast));
zend_arena_destroy(CG(ast_arena));
</code>

For constant scalar expressions AST nodes need to be preserved after compilation. For this purpose they need to be copied from the arena into ZMM allocated memory. This is accomplished using the ''zend_ast_copy'' function. The heap-allocated AST can then be destroyed using ''zend_ast_destroy_and_free''.

==== Compiler implementation ====

=== Emitting oplines ===

The compiler continues to use ''znode''s to store operands during compilation. Oplines are usually created using the ''zend_emit_op'' or ''zend_emit_op_tmp'' functions:

<code c>
zend_op *zend_emit_op(znode *result, zend_uchar opcode, znode *op1, znode *op2 TSRMLS_DC);
zend_op *zend_emit_op_tmp(znode *result, zend_uchar opcode, znode *op1, znode *op2 TSRMLS_DC);
</code>

Both will emit (and return) an opline with the given opcode and operands. ''zend_emit_op_tmp'' will use an ''IS_TMP_VAR'' variable for the result. ''zend_emit_op'' uses an ''IS_VAR'' variable instead. ''zend_emit_op'' also accepts ''NULL'' for the result, in which case the opline will have no result (''IS_UNUSED'').

Both ''op1'' and ''op2'' can also be ''NULL'', in which case they are unused.

There are additional functions for handling jump instructions. ''zend_emit_jump'' and ''zend_emit_cond_jump'' will emit jumps / conditional jumps. In many cases the jump target is not known at the time the opline is emitted. In this case ''0'' should be passed for ''opnum_target'' and the opnum returned by the function backed up. Afterwards the returned opnum can be used to update the jump target using ''zend_update_jump_target'':

<code c>
zend_uint zend_emit_jump(zend_uint opnum_target TSRMLS_DC);
zend_uint zend_emit_cond_jump(zend_uchar opcode, znode *cond, zend_uint opnum_target TSRMLS_DC);
void zend_update_jump_target(zend_uint opnum_jump, zend_uint opnum_target TSRMLS_DC);
</code>

=== Compiling expressions ===

Expressions are compiled using the ''zend_compile_expr'' function, which accepts a ''result'' znode and an ''ast''. This function merely dispatches to a more concrete compilation function based on the AST kind:

<code c>
switch (ast->kind) {
    /* ... */
    case ZEND_AST_ASSIGN_OP:
        zend_compile_compound_assign(result, ast TSRMLS_CC);
        return;
    case ZEND_AST_BINARY_OP:
        zend_compile_binary_op(result, ast TSRMLS_CC);
        return;
    /* ... */
}
</code>

The ''zend_compile_binary_op'' function referenced above is defined as follows (compile-time pre-evaluation code has been removed for conciseness):

<code c>
void zend_compile_binary_op(znode *result, zend_ast *ast TSRMLS_DC) {
    zend_ast *left_ast = ast->child[0];
    zend_ast *right_ast = ast->child[1];
    zend_uint opcode = ast->attr;

    znode left_node, right_node;
    zend_compile_expr(&left_node, left_ast TSRMLS_CC);
    zend_compile_expr(&right_node, right_ast TSRMLS_CC);

    zend_emit_op_tmp(result, opcode, &left_node, &right_node TSRMLS_CC);
}
</code>

By convention, all ''zend_compile_*'' functions start by extracting child nodes into named local variables with an ''_ast'' suffix. Afterwards the function compiles the child nodes into ''znode''s (which have the same name as the corresponding AST nodes, but using a ''_node'' suffix). Finally an opline is emitted, which has a TMP_VAR result and uses the two znodes as operands.

When ''zend_compile_expr'' is passed the AST of a variable expression, it will compile it using the ''BP_VAR_R'' fetch mode. To explicitly specify the fetch mode the function ''zend_compile_var'' can be used instead, which accepts a ''BP_VAR_*'' mode as the last argument:

<code c>
void zend_compile_pre_incdec(znode *result, zend_ast *ast TSRMLS_DC) {
    zend_ast *var_ast = ast->child[0];
    ZEND_ASSERT(ast->kind == ZEND_AST_PRE_INC || ast->kind == ZEND_AST_PRE_DEC);

    if (var_ast->kind == ZEND_AST_PROP) {
        zend_op *opline = zend_compile_prop_common(result, var_ast, BP_VAR_RW TSRMLS_CC);
        opline->opcode = ast->kind == ZEND_AST_PRE_INC ? ZEND_PRE_INC_OBJ : ZEND_PRE_DEC_OBJ;
    } else {
        znode var_node;
        zend_compile_var(&var_node, var_ast, BP_VAR_RW TSRMLS_CC);
        zend_emit_op(result, ast->kind == ZEND_AST_PRE_INC ? ZEND_PRE_INC : ZEND_PRE_DEC,
            &var_node, NULL TSRMLS_CC);
    }
}
</code>

The preceding code sample also illustrates another pattern that sometimes occurs when dealing with operations on variables: A property increment has the same structure as a property fetch, just using a different opcode. Such situations are handled by providing a ''_common'' variant of variable-compilation functions, which return the generated opline for further adjustments.

=== Compiling statements ===

Statements are compiled using the ''zend_compile_stmt'' function, which accepts an ''ast'' (but no longer a result znode, as statements have no result). Once again this function will delegate to more specific compilation functions based on the AST kind. For example, this is the compilation function for a ''while'' loop:

<code c>
void zend_compile_while(zend_ast *ast TSRMLS_DC) {
    zend_ast *cond_ast = ast->child[0];
    zend_ast *stmt_ast = ast->child[1];

    znode cond_node;
    zend_uint opnum_start, opnum_jmpz;

    opnum_start = get_next_op_number(CG(active_op_array));
    zend_compile_expr(&cond_node, cond_ast TSRMLS_CC);

    opnum_jmpz = zend_emit_cond_jump(ZEND_JMPZ, &cond_node, 0 TSRMLS_CC);

    zend_begin_loop(TSRMLS_C);

    zend_compile_stmt(stmt_ast TSRMLS_CC);
    zend_emit_jump(opnum_start TSRMLS_CC);

    zend_update_jump_target_to_next(opnum_jmpz TSRMLS_CC);
    zend_end_loop(opnum_start, 0 TSRMLS_CC);
}
</code>

After extracting the child nodes into local variables, this function compiles the loop condition and generates a conditional jump of type ''JMPZ'' based on it. Both the opnum of the condition and of the JMPZ instructions are backed up.

Afterwards the body of the loop is compiled (''stmt_ast'') and a jump back to the condition generated. Lastly the jump target of the JMPZ instruction is updated to point after the loop (i.e. if the condition is false, execution should continue after the loop.)

The ''zend_begin_loop'' and ''zend_end_loop'' functions store information for break/continue and try/catch.

===== Additional possibilities (not implemented) =====

The generated AST can be exposed to userland via an extension, for use by static analysers. This should be relatively easy to implement and we might even want to provide this as a bundled extension (like ext/tokenizer).

More interestingly, we could allow extensions to hook into the compilation process (the current AST implementation does not provide hooks, but they can be added if we want them). This would allow extensions to implement some types of "language features".

As an example, this is roughly how an implementation of the [[rfc:ifsetor|ifsetor RFC]] //could// look like using an AST hook:

<code c>
/* Works by rewriting ifsetor($foo, 'bar') to isset($foo) ? $foo : 'bar' */
void ext_ifsetor_hook(zend_ast **ast_ptr TSRMLS_DC) {
    zend_ast *ast = *ast_ptr;

    if (ast->kind == ZEND_AST_CALL && ast->child[0]->kind == ZEND_AST_ZVAL) {
        zend_string *name = zval_get_string(zend_ast_get_zval(ast->child[0]));
        zend_ast_list *args = zend_ast_get_list(ast->child[1]);

        if (zend_str_equals_literal_ci(name, "ifsetor")
            && args->children == 2 && !zend_args_contain_unpack(args)
        ) {
            if (!zend_is_variable(args->child[0])) {
                zend_error_noreturn(E_COMPILE_ERROR, "First argument of ifsetor "
                    "must be a variable");
            }

            /* Note: One would need a function for adding refs to args->child[0] here,
             * as it is used two times - as written here it won't work correctly. */
            *ast_ptr = zend_ast_create(ZEND_AST_CONDITIONAL,
                zend_ast_create(ZEND_AST_ISSET, args->child[0]),
                args->child[0],
                args->child[1]
            );
        }

        STR_RELEASE(name);
    }
}
</code>

I don't know how useful this is and how many things can be implemented in this way, but I think it's worth considering.

An additional possibility is to drop the keywords for ''isset'' and ''empty'' and just compile them as special function calls (using similar checks as the code above). Maybe other keywords can be dropped as well.

===== Patch =====

The AST implementation can be found at https://github.com/nikic/php-src/tree/ast. Some quick links to the most important files:

  * ''[[https://github.com/nikic/php-src/blob/ast/Zend/zend_language_parser.y|zend_language_parser.y]]''
  * ''[[https://github.com/nikic/php-src/blob/ast/Zend/zend_ast.h|zend_ast.h]]''
  * ''[[https://github.com/nikic/php-src/blob/ast/Zend/zend_ast.c|zend_ast.c]]''
  * ''[[https://github.com/nikic/php-src/blob/ast/Zend/zend_compile.c|zend_compile.c]]'' (The code relevant to the AST begins somewhere around line 3200, everything before that is largely untouched.)

//The branch already includes the Uniform Variable Syntax RFC//, as it was a necessary prerequisite for the implementation.

The implementation has everything ported, but probably still has some bugs and needs some cleanup :)

===== Vote =====
The vote started on 2014-08-18 and ended on 2014-08-25. The necessary 2/3 majority was reached, as such the RFC is accepted.

<doodle title="Use AST implementation in PHP 7?" auth="nikic" voteType="single" closed="true">
   * Yes
   * No
</doodle>
