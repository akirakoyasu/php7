
====== PHP RFC: Anonymous Classes ======
  * Version: 0.6
  * Date: 2013-09-22
  * Author: Joe Watkins <krakjoe@php.net>, Phil Sturgeon <philstu@php.net>
  * Status: Implemented (in PHP 7.0)
  * First Published at: http://wiki.php.net/rfc/anonymous_classes

===== Introduction =====

For some time PHP has featured anonymous function support in the shape of Closures; this patch introduces the same kind of functionality for objects of an anonymous class.

The ability to create objects of an anonymous class is an established and well used part of Object Orientated programming in other languages (namely C# and Java).

An anonymous class might be used over a named class:

  * when the class does not need to be documented
  * when the class is used only once during execution

An anonymous class is a class without a (programmer declared) name. The functionality of the object is no different from that of an object of a named class. They use the existing class syntax, with the name missing:

```php
var_dump(new class($i) {
    public function __construct($i) {
        $this->i = $i;
    }
});
```

===== Syntax and Examples =====

new class (arguments) {definition}

Note: in a previous version of this RFC, the arguments were after the definition, this has been changed to reflect the feedback during the last discussion.

```php
<?php
/* implementing an anonymous console object from your framework maybe */
(new class extends ConsoleProgram {
    public function main() {
       /* ... */
    }
})->bootstrap();

/* return an anonymous implementation of a Page for your MVC framework */
return new class($controller) implements Page {
    public function __construct($controller) {
        /* ... */
    }
    /* ... */
};

/* vs */
class MyPage implements Page {
    public function __construct($controller) {
        /* ... */
    }
    /* ... */
}
return new MyPage($controller);

/* return an anonymous extension of the DirectoryIterator class */
return new class($path) extends DirectoryIterator {
   /* ... */
};

/* vs */
class MyDirectoryIterator {
    /* .. */
}
return new MyDirectoryIterator($path);

/* return an anon class from within another class (introduces the first kind of nested class in PHP) */
class MyObject extends MyStuff {
    public function getInterface() {
        return new class implements MyInterface {
            /* ... */
        };
    }
}


/* return a private object implementing an interface */
class MyObject extends MyStuff {
    /* suitable ctor */

    private function getInterface() {
        return new class(/* suitable ctor args */) extends MyObject implements MyInterface {
            /* ... */
        };
    }
}
```

Note: the ability to declare and use a constructor in an anonymous class is necessary where control over construction must be exercised.

===== Inheritance/Traits =====

Extending classes works just as you'd expect.

```php
<?php

class Foo {}

$child = new class extends Foo {};

var_dump($child instanceof Foo); // true
```

Traits work identically as in named class definitions too.

```php
<?php

trait Foo {
    public function someMethod() {
      return "bar";
    }
}

$anonClass = new class {
    use Foo;
};

var_dump($anonClass->someMethod()); // string(3) "bar"
```

===== Reflection =====

The only change to reflection is to add ReflectionClass::isAnonymous().

===== Serialization =====

Serialization is not supported, and will error just as anonymous functions do.

===== Internal Class Naming =====

The internal name of an anonymous class is generated with a unique reference based on its address.

```php
function my_factory_function(){
    return new class{};
}
```

get_class(my_factory_function()) would return "class@0x7fa77f271bd0" even if called multiple times, as it is the same definition. The word "class" is used by default, but if the anonymous class extends a named class it will use that:

```php
class mine {}

new class extends mine {};
```

This class name will be "mine@0x7fc4be471000".

Multiple anonymous classes created in the same position (say, a loop) can be compared with `==`, but those created elsewhere will not match as they will have a different name.

```php
$identicalAnonClasses = [];

for ($i = 0; $i < 2; $i++) {
    $identicalAnonClasses[$i] = new class(99) {
        public $i;
        public function __construct($i) {
            $this->i = $i;
        }
    };
}

var_dump($identicalAnonClasses[0] == $identicalAnonClasses[1]); // true

$identicalAnonClasses[2] = new class(99) {
    public $i;
    public function __construct($i) {
        $this->i = $i;
    }
};

var_dump($identicalAnonClasses[0] == $identicalAnonClasses[2]); // false
```

Both classes where identical in every way, other than their generated name.

===== Use Cases =====

Code testing presents the most significant number of use cases, however, where anonymous classes are a part of a language they do find their way into many use cases, not just testing. Whether it is technically correct to use an anonymous class depends almost entirely on an individual application, or even object depending on perspective.

A few quick points:

  * Mocking tests becomes easy as pie. Create on-the-fly implementations for interfaces, avoiding using complex mocking APIs.
  * Keep usage of these classes outside the scope they are defined in
  * Avoid hitting the autoloader for trivial implementations

Tweaking existing classes which only change a single thing can make this very easy. Taking an example from the [Pusher PHP library](https://github.com/pusher/pusher-http-php#debugging--logging):

```php
// PHP 5.x
class MyLogger {
  public function log($msg) {
    print_r($msg . "\n");
  }
}

$pusher->setLogger( new MyLogger() );

// New Hotness
$pusher->setLogger(new class {
  public function log($msg) {
    print_r($msg . "\n");
  }
});
```

This saved us making a new file, or placing the class definition at the top of the file or somewhere a long way from its usage. For big complex actions, or anything that needs to be reused, that would of course be better off as a named class, but in this case it's nice and handy to not bother.

If you need to implement a very light interface to create a simple dependency:

```php
$subject->attach(new class implements SplObserver {
  function update(SplSubject $s) {
    printf("Got update from: %s\n", $subject);
  }
});
```

Here is one example, which covers converting PSR-7 middleware to Laravel 5-style middleware.

```php
<?php
$conduit->pipe(new class implements MiddlewareInterface {
    public function __invoke($request, $response, $next)
    {
        $laravelRequest = mungePsr7ToLaravelRequest($request);
        $laravelNext    = function ($request) use ($next, $response) {
            return $next(mungeLaravelToPsr7Request($request), $response)
        };
        $laravelMiddleware = new SomeLaravelMiddleware();
        $response = $laravelMiddleware->handle($laravelRequest, $laravelNext);
        return mungeLaravelToPsr2Response($response);
    }
});
```

Anonymous classes do present the opportunity to create the first kind of nested class in PHP. You might nest for slightly different reasons to creating an anonymous class, so that deserves some discussion;

```php
<?php
class Outside {
    protected $data;

    public function __construct($data) {
        $this->data = $data;
    }

    public function getArrayAccess() {
        return new class($this->data) extends Outside implements ArrayAccess {
            public function offsetGet($offset) { return $this->data[$offset]; }
            public function offsetSet($offset, $data) { return ($this->data[$offset] = $data); }
            public function offsetUnset($offset) { unset($this->data[$offset]); }
            public function offsetExists($offset) { return isset($this->data[$offset]); }
        };
    }
}
```

Note: Outer is extended not for access to $this->data - that could just be passed into a constructor; extending Outer allows the nested class implementing ArrayAccess permission to execute protected methods, declared in the Outer class, on the same $this->data, and if by reference, as if they are the Outer class.

In the simple example above Outer::getArrayAccess takes advantage of anonymous classes to declare and create an ArrayAccess interface object for Outer.

By making getArrayAccess private the anonymous class it creates can be said to be a private class.

This increases the possibilities for grouping of your objects functionality, can lead to more readable, some might say more maintainable code.

The alternative to the above is the following:

```php
class Outer implements ArrayAccess {
    public $data;

    public function __construct($data) {
        $this->data;
    }

    public function offsetGet($offset) { return $this->data[$offset]; }
    public function offsetSet($offset, $data) { return ($this->data[$offset] = $data); }
    public function offsetUnset($offset) { unset($this->data[$offset]); }
    public function offsetExists($offset) { return isset($this->data[$offset]); }

}
```

Pass-by-reference is not used in the examples above, so behaviour with regard to $this->data should be implicit.

How you choose to do it for any specific application, whether getArrayAccess is private or not, whether to pass by reference or not, depends on the application.

Various use cases have been suggested on the mailing list: http://php.markmail.org/message/sxqeqgc3fvs3nlpa?q=anonymous+classes+php, and here are a few.

----

The use case is one-time usage of an "implementation", where you currently
probably pass callbacks into a "Callback*"-class like
```php
    $x = new Callback(function() {
        /* do something */
    });

    /* vs */

    $x = new class extends Callback {
        public function doSometing()
        {
            /* do something */
        }
    };
```

Imagine you have several abstract methods in one interface/class, which
would need several callbacks passed to the constructor.
Also '$this' is mapped to the right objects.

Overriding a specific method in a class is one handy use. Instead of creating
a new class that extends the original class, you can just use an anonymous
class and override the methods that you want.

E.G; You can to the following:

```php
use Symfony\Component\Process\Process;

$process = new class extends Process {
    public function start() {
        /* ... */
    }
};
```
instead of the following:
```php
namespace My\Namespace\Process;

use Symfony\Component\Process\Process as Base;

class Process extends Base {
    public function start() {
        /* ... */
    }
}

$process = new \My\Namespace\Process\Process;
```

===== Backward Incompatible Changes =====

New syntax that will fail to parse in previous versions, so no BC breaks.

===== Proposed PHP Version(s) =====

7.0

===== SAPIs Impacted =====

All

===== Impact to Existing Extensions =====

No impact on existing libraries

===== Open Issues =====

None

===== Future Scope =====

The changes made by this patch mean named nested classes are easier to implement (by a tiny bit).

===== References =====

PHP 7 Discussion: http://marc.info/?l=php-internals&m=142478600309800&w=2

===== Proposed Voting Choices =====

The voting choices are yes (in favor for accepting this RFC for PHP 7) or no (against it).

===== Vote =====

Vote starts on March 13th, and will end two weeks later, on March 27th.

This RFC requires a 2/3 majority.

<doodle title="Anonymous Classes" auth="philstu" voteType="single" closed="true">
   * Yes
   * No
</doodle>

 ===== Changelog =====

  * v0.6 Serialization not supported and names added
  * v0.5.2 Updated name examples
  * v0.5.1 Example improvements
  * v0.5: Added traits example and tests
  * v0.4: Comparisons example
  * v0.3: ReflectionClass::isAnonymous() and simple example
  * v0.2: Brought back for discussion
  * v0.1: Initial draft

===== Implementation =====

https://github.com/php/php-src/pull/1118
