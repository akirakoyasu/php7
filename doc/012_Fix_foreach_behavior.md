====== PHP RFC: Fix "foreach" behavior ======
  * Version: 0.9
  * Date: 2015-01-29
  * Author: Dmitry Stogov, dmitry@zend.com
  * Status: Implemented (in PHP 7.0)
  * First Published at: http://wiki.php.net/rfc/php7_foreach

===== Introduction =====

The behavior of **foreach** statement in PHP for some edge cases is not exactly defined. Actually, it is implementation driven and quite weird.

Most these edge cases are related to manipulation with internal pointer and modification of elements of array, iterated by **foreach**. The behavior may depend on the value of reference-counter or the fact - if the value is a reference or not. I'll provide just few examples to demonstrate the existing inconsistencies.

Result of current() is undefined
<code>
$ php -r '$a = [1,2,3]; foreach($a as $v) {echo $v . " - " . current($a) . "\n";}'
1 - 2
2 - 2
3 - 2

$ php -r '$a = [1,2,3]; $b = $a; foreach($a as $v) {echo $v . " - " . current($a) . "\n";}'
1 - 1
2 - 1
3 - 1
</code>

unset() may exclude an element from iteration or not
<code>
$ php -r '$a = [1,2,3]; foreach($a as $v) {echo "$v\n"; unset($a[1]);}'
1
2
3

$ php -r '$a = [1,2,3]; $b = &$a; foreach($a as $v) {echo "$v\n"; unset($a[1]);}'
1
3
</code>

It's possible to write more inconsistent or strange examples...

===== Proposal =====
We propose consistent **foreach** statement behavior and provide efficient implementation.

The changes in beahvior affects only edge undefined cases and keeps the conceptual behavior unchanged.

At first, **foreach** statement may iterate:
  * by value - foreach ($a as $v)
  * by reference- foreach ($a as &$v)

At second, **foreach** statement may iterate over:
  * array - $x = [1,2,3]; foreach ($x as $v)
  * plain object - $x = new SomeClass(); foreach ($x as $v)
  * iterator object - $x = new SomeIteratorClass(); foreach ($x as $v)

Iteration over **iterator objects** (by value and by reference) is kept the same.

==== foreach() by value over arrays ====

**foreach** by value over array will never use or modify internal array pointer. It also won't duplicate array, it'll lock it instead (incrementing reference counter). This will lead to copy-on-write on attempt of array modification inside the loop. As result, we will always iterate over elements of array originally passed to **foreach**, ignoring any possible array changes. The following examples demonstrates the changes in behavior (in comparison to examples from introduction).

The value of internal pointer is unaffected
<code>
$ php -r '$a = [1,2,3]; foreach($a as $v) {echo $v . " - " . current($a) . "\n";}'
1 - 1
2 - 1
3 - 1
</code>

Modifications of the original array are ignored
<code>
$ php -r '$a = [1,2,3]; $b = &$a; foreach($a as $v) {echo "$v\n"; unset($a[1]);}'
1
2
3
</code>

==== foreach() by reference over arrays ====

In most cases it repeats the PHP5 behavior.

**foreach** by reference over array modifys internal pointer on each iteration. Exactly as it was implemented in PHP5, it sets the internal pointer to the following element.

<code>
$ php -r '$a = [1,2,3]; foreach($a as &$v) {echo $v . " - " . current($a) . "\n"; }'
1 - 2
2 - 3
3 -
</code>

Modification of internal array pointer through next() and family doesn't affect **foreach** pointer. On next iteration internal pointer is restored to the value of **foreach** pointer. This is also the default PHP5 behavior.

<code>
$ php -r '$a = [1,2,3,4]; foreach($a as &$v) {echo "$v - "; next($a); var_dump(current($a));}'
1 - int(3)
2 - int(4)
3 - bool(false)
4 - bool(false)
</code>

Deletion of the next element referred by **foreach** pointer leads to skipping it (in the same way as as in PHP5).

<code>
$ sapi/cli/php -r '$a = [1,2,3]; foreach($a as &$v) {echo "$v\n"; unset($a[1]);}'
1
3
</code>

Adding new elements after the current **foreach** pointer adds them to iteration (the same as in PHP5)

<code>
$ php -r '$a = [1,2]; foreach($a as &$v) {echo "$v\n"; $a[2]=3;}'
1
2
3
</code>

Adding new elements after the current **foreach** pointer when we are already at the end adds them to iteration as well (**this didn't work in PHP5**)

<code>
$ php -r '$a = [1]; foreach($a as &$v) {echo "$v\n"; $a[1]=2;}'
1
2
</code>

Replacing iterated array with another array lead to continue iteration over the new array starting from its internal pointer (the same as in PHP5)

<code>
$ php -r '$a = [1,2]; $b = [3,4]; next($b); foreach($a as &$v) {echo "$v\n"; $a = $b;}'
1
4
</code>

In case we have several **forech** by reference statements over the same array each of them works according to the rules above, independently from the others. (**It didn't work in PHP5**)

<code>
<?php
$a = [0, 1, 2, 3];
foreach ($a as &$x) {
	foreach ($a as &$y) {
		echo "$x - $y\n";
		if ($x == 0 && $y == 1) {
			unset($a[1]);
			unset($a[2]);
		}
	}
}
</code>
<code>
$ php test.php
0 - 0
0 - 1
0 - 3
3 - 0
3 - 3
</code>

Modification of array, iterated through foreach by reference, using internal functions like array_pop(), array_push(), array_shift(), array_unshift() works consistently. These functions preserve the current foreach position or move it to the following element, if the current is deleted. (**It didn't work in PHP5**)

<code>
$ php -r '$a=[1,2,3,4]; foreach($a as &$v) { echo "$v\n"; array_pop($a);}'
1
2
</code>

==== foreach() by value over plain objects ====

It beahves in the same way as **foreach by reference over array**, but using object value instead of reference. As result the object can be modified, but can't be replaced.

==== foreach() by reference over plain objects ====

It beahves in the same way as **foreach by reference over array**.

==== Implementation Details ====

The existing FE_RESET/FE_FETCH opcodes are split into separate FE_RESET_R/FE_FETCH_R opcodes used to implement **foreach by value** and FE_RESET_RW/FE_FETCH_RW to implement **foreach by reference**. The suffix **_R** means that we use array (or object) only for reading, and suffix **_RW** that we also may indirectly modify it. A new FE_FREE opcode is introduced. It's used at the end of **foreach** loops, instead of FREE opcode.

Iteration by value over array doesn't use or modify internal array pointer. The value of the pointer is kept in reserved space of temporary variable used for iteration. It's acceptable through Z_FE_POS() macro.

Iteration by reference or by value over plain object implemented using special **HashTableIterator** structures.

<code>
typedef struct _HashTableIterator {
	HashTable    *ht;
	HashPosition  pos;
} HashTableIterator;
</code>

On entrance into **foreach** loop FE_RESET_R/RW opcode creates and initializes a new iterator and stores its index in reserved space of temporary variable used for iteration. On exit, FE_FREE opcode removes corresponding iterator.

Iterators are actually allocated in a buffer - EG(ht_iterators), represented by plain array. The more nested **foreach by reference** iterators the bigger buffer we will need. We start with small preallocated buffer - EG(ht_iterators_slots), and then extend it if necessary in heap. EG(ht_iterators_count) keeps the number of available slots for iterators, EG(ht_iterators_used) - the number of used slots.

<code>
struct _zend_executor_globals {
	...
	uint32_t           ht_iterators_count;     /* number of allocatd slots */
	uint32_t           ht_iterators_used;      /* number of used slots */
	HashTableIterator *ht_iterators;
	HashTableIterator  ht_iterators_slots[16];
	...
}
</code>

Creation, deletion and accessing iterators position is implemented through special API.

<code>
ZEND_API uint32_t     zend_hash_iterator_add(HashTable *ht);
ZEND_API HashPosition zend_hash_iterator_pos(uint32_t idx, HashTable *ht);
ZEND_API void         zend_hash_iterator_del(uint32_t idx);
</code>

Indirect modification of iterators positions implemented through zend_hash_iterators_update(). It's called when HashTable modification may affects iterator position. For example when element referred by iterator is inserted, or when iterator is set at the end of the array and new element is inserted.

<code>
ZEND_API void         zend_hash_iterators_update(HashTable *ht, HashPosition from, HashPosition to);
</code>

Foe more details see zend_hash_iterators_*() functions implementation in zend_hash.c

===== Backward Incompatible Changes =====
Some rare cases where the **foreach** statement behavior was undefined may be changed. The implementation changes few such PHPT tests. The list and explanation follows:

  * Zend/tests/bug40509.phpt - foreach be value doesn't change internal pointer
  * Zend/tests/bug40705.phpt - foreach be value doesn't change internal pointer
  * tests/lang/bug23624.phpt - foreach be value doesn't change internal pointer
  * tests/lang/foreachLoop.001.phpt - foreach be value doesn't change internal pointer
  * tests/lang/foreachLoop.009.phpt - modification of array in foreach by value doesn't have effect
  * tests/lang/foreachLoop.011.phpt - replacement of array in foreach by value doesn't have effect
  * tests/lang/foreachLoop.013.phpt - modification of array in foreach by reference through internal functions
  * tests/lang/foreachLoop.014.phpt - modification of array in foreach by value doesn't have effect
  * tests/lang/foreachLoop.015.phpt - modification of array in foreach by reference through internal functions
  * tests/lang/foreachLoopObjects.006.phpt - replacement of array in foreach by value doesn't have effect

===== Additional Behavoir Change =====

With new implementation it's quite easy to stop using internal array/object pointer even for *foreach be referece*.
It means that reset/key/current/next/prev function will be completely independent from the sate of *foreach* iterator.
This would change the output of few examples above.

**foreach** (even foreach by  reference) won't affect internal array pointer

<code>
$ php -r '$a = [1,2,3]; foreach($a as &$v) {echo $v . " - " . current($a) . "\n"; }'
1 - 1
2 - 1
3 - 1
</code>

Modification of internal array pointer through next() and family doesn't affect **foreach** pointer. But it also won't be affected by the value of **forech** pointer.

<code>
$ php -r '$a = [1,2,3,4]; foreach($a as &$v) {echo "$v - "; next($a); var_dump(current($a));}'
1 - int(2)
2 - int(3)
3 - int(4)
4 - bool(false)
</code>

===== Proposed PHP Version(s) =====

PHP7

===== RFC Impact =====
==== To Performance ====
New behavior allows elimination of array duplication and this should lead to better performance, on the other hand some HashTable operations require additional check(s). Anyway, for Wordpress-3.6 the proposed patch reduces the number of executed CPU instructions by ~1%. It saves about 200 array duplications and destructions per each request to home page.

==== To Opcache ====
OPCache has to support new opcodes. All necessary OPCache changes are provided with the proposed implementation

===== Open Issues =====
  * implementation optimization for size & speed

===== Future Scope =====
This sections details areas where the feature might be improved in future, but that are not currently proposed in this RFC.

===== Proposed Voting Choices =====
The vote is a straight Yes/No vote, that requires a 2/3 majority

<doodle title="Fix foreach behavoir?" auth="dmitry" voteType="single" closed="true">
   * Yes
   * No
</doodle>

The second (Yes/No 50%+1) question is - if we should stop modifying internal array/object pointer in **foreach**.

<doodle title="Stop using internal array/object pointer in foreach by reference?" auth="dmitry" voteType="single" closed="true">
   * Yes
   * No
</doodle>

The vote will end on February 12.

===== Patches and Tests =====

Pull request for master branch: [[https://github.com/php/php-src/pull/1034]]

The implementation of additional idea is trivial [[https://gist.github.com/dstogov/63b269207ba0aed8b776]]

===== Implementation =====
The RFC implemented in PHP7 with two commits:

[[http://git.php.net/?p=php-src.git;a=commitdiff;h=97fe15db4356f8fa1b3b8eb9bb1baa8141376077|97fe15db4356f8fa1b3b8eb9bb1baa8141376077]]

[[http://git.php.net/?p=php-src.git;a=commitdiff;h=4d2a575db2ac28c9acede4a85152bcec342c4a1d|4d2a575db2ac28c9acede4a85152bcec342c4a1d]]

