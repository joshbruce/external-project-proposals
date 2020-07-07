# Introduction

For our purposes, we are not talking about converting an instance into a primitive that can then be acted upon by other PHP code. Rather, we are talking about giving developers a way to interact directly with instances as if they were native PHP primitives.

PHP affords developers the means to do this already with many primitives types. For example, to give PHP a way to interact with an instance of a custom object where the inferred type is `string`, developers implement the `__toString()` magic method.

```php
class _String
{
  public function __toString()
  {
    return "Hello, World!";
  }
}

$instance = new _String();

print $instance;

// output: Hello, World!
```

For array access, there is the `ArrayAccess` interface (sample 4), which allows internal values defined in the class to be retrieved using array syntax: `$instance["hello"]` or `$instance[0]`. In the case of at least the `string` primitive, this capability is baked in.

```php
$instance = new _String();

$string = (string) $instance;

print $string[0];

// output: H
```

This [concept] would expand these affordances to instances of custom classes improving flexibility for developers.

## Benefits

Developers can generate packages that act as "language extensions," integrating seamlessly into the PHP syntax without writing and installing extensions or compilers - especially where this is not permitted, such as shared hosting environments. As these language extensions mature they may lay the foundation for future RFC submissions to improve PHP as a whole. (ex. Think of some JavaScript library functions that became part of Javascript itself, then became part of CSS or HTML - fringe to core evolution.)

Further, as developers use more packages and those packages leverage multiple classes, the chances a variable will end up in a position where a `TypeError` can occur increases. These type errors could reduce as developers define a rational default that won't cause a fatal error on the other side when used in the PHP syntax and passed to functions.

## Base types and scope

There are 10 PHP primitives that can be interacted with using PHP syntax. 

Two can be removed from consideration as one (`resource`) doesn't seem in common use and the second (`NULL`) is always NULL, and an instance shouldn't need to specify how to return from a NULL-check, if the instance can return something, it shouldn't be NULL.

### Available as of PHP 7

Of the eight remaining primitives, two can be integrated into common PHP syntax by way of magic methods: `string` and `callable`. There is also a magic method similar to `__toString()` that allows developers to define how an instance will interact with the console and debugger called `__debugInfo()` that will also be used in this sample.

```php
class MyType
{
  public function __toString()
  {
    return "Hello, World!";
  }

  public function __invoke()
  {
    $string = (string) $this;
    return "Invoked: {$string}";
  }

  public function __debugInfo()
  {
    return [
      "output" => "Debugged + {$this()}"
    ];
  }
}

$instance = new MyType();

print $instance; // Hello, World!

print $instance(); // Invoked: Hello, World!

var_dump($instance); // Debugged + Invoked: Hello, World!
```

The important takeaway here is that at no point did the developer call a method on the instance to get the desired result; instead, the developer interacted with the instance just as they would using the PHP primitive. Further, the instance is not limited to only one type of interaction. 

In the first case the instance is usable as a string. In the second, the instance is usable as an anonymous function. (It's worth noting that it doesn't seem possible to cast to a `callable` unlike other types.)

Because these interaction affordances already exist, it doesn't seem necessary to include them in this [concept].

Of the six remaining types, two can be integrated into common PHP syntax by conforming to the `ArrayAccess` and the `Iterator` interfaces (both native to PHP as of PHP5): `array` and the `iterable` pseudo-type. The first allows us to interact with an instance using array notation (`$x[$y]`) and to allow them to be used as traversables typically seen as a variadic argument in a function.

```php
class MyType implements \ArrayAccess, \Iterator
{
  private $array = [];
  private $position = 0;

  public function __construct(...$arrayValues) 
  {
    $this->array = $arrayValues;
  }

// - ArrayAccess
  public function offsetExists($offset): bool
  {}

  public function offsetGet($offset)
  {}

  public function offsetSet($offset, $value): void
  {}

  public function offsetUnset($offset): void
  {}

// - Iterator
  public function rewind() 
  {}

  public function current() 
  {}

  public function key() 
  {}

  public function next() 
  {}

  public function valid() 
  {}
}

$instance = new MyType(1, 2 ,3);

print $instance[0]; // 1 - using ArrayAccess

$instance2 = new MyType(...$instance); // copy - using Traversable via Iterator (to demonstrate variadic)

foreach ($instance2 as $key => $value) {
  print "{$key}: {$value} "; // 0: 1 1: 2 2: 3 - using Iterator
}
```

Note, again, the developer is interacting with the instances as if they were primitives, no extra syntax. Further, it is worth noting that to mirror the full capabilities of the `array` primitive, the object must conform to both the `ArrayAccess` and `Iterator` interfaces.

Of the four remaining, one allows developers to specify custom results from normal PHP interactions: `object`. Developers can override (or otherwise extend) the normal PHP behavior of interacting with objects, which an instance already is. However, PHP does not give a way to convert an instance of, say, `Foo` to an instance of `stdClass`, and that's okay. This customization is achieved by using various magic methods (the sample uses three that seem common): `__call()`, `__get()`, and `__set()`.

```php
class MyType
{
  private $values = [];

  public function __set($name, $value)
  {
    $this->values[$name] = $value;
  }

  public function __get($name) 
  {
    return $this->values[$name];
  }

  public function __call($name, $args)
  {
    if (count($args) === 0) {
      return $this->{$name};
    }
    $this->{$name} = $args[0];
  }
}

// PHP
$stdClass = new \stdClass();

$stdClass->hello = "Hello";

print $stdClass->hello; // Hello

// MyType
$instance = new MyType();

$instance->hello = "Hello";

print $instance->hello; // Hello

// MyType - extended capability
$instance->suffix(", World!");

print $instance->hello . $instance->suffix; // Hello, World!
```

Again, interacting with a custom instance using standard PHP syntax.

Of the three remaining, one seems like a partial implementation limited to only one use case, which would not help address the goal of interacting with instances as though they were PHP primitives: `integer`. The `Countable` interface allows PHP to automatically convert the instance into an integer, but only when being called by the `count()` function in the PHP Standard Library.

```php
class MyType implements \Countable
{
  private $count = 0;

  public function __construct(int $count) 
  {
    $this->count = $count;
  }

// - Countable
  public function count()
  {
    return $this->count;
  }
}

$instance = new MyType(1);

print count($instance); // 1 - using Countable

$sum = $instance + $instance; // PHP Notice: Object of class MyType could not be converted to int...
```

Just a quick aside to further illustrate what is being attempted and how much this is concept is already a part of the PHP ecosystem. PHP also offers a way for developers to define how their instance would be converted to a specific type of string, namely JSON by using the `JsonSerialize` interface allowin an instance of the conforming class to be passed into the `json_encode()` function from the PHP standard library without a `TypeError`.

### To be implemented

The three remaining primitives (including `integer`) - `boolean`, `integer`, and `float` - do not appear to have a way to inform PHP on how to interact with them in situations where that type is required. Numbers can be modified using math operators and booleans are the basis for conditionals.

As it seems there is momentum for future PHP versions to move away from magic methods, or at least favor interfaces, the samples will use interfaces. The from an implementation perspective the difference should be negligible as one requires no interface reference while the other does require such a reference. With that said, with a magic method, the PHP type system may requiring updating account for such a method; however, when it comes to interfaces, the PHP type system already checks for those (??). 

The root of the name is still under discussion though `UsableAs` seems to approprately describe the desired outcome.

```php
interface [UsableAs]Bool
{
  public function toBool(): bool;
}

class MyType implements [UsableAs]Bool
{
  $value = true;

  public function toggle()
  {
    $this->value = ! $value;
    return $this;
  }

// - UsableAsBool
  public function toBool(): bool
  {
    return $this->value;
  }
}

if (true) {
  print("Hello!"); // Hello!
}

$instance = new MyType();

if ($instance) {
  print("Hello!"); // Hello! - could occur just because not null
  $instance->toggle();

  if (! $instance) {
    print("Value is false.");
  }
}
```

It's presumed `integer` and `float` would require similar methods to be characterized by PHP as an `integer` or `float`; responding to various math operators, for example. For our purposes, instead of something like `ArrayAcess` and `Iterator` where developers are asked to implement multiple methods, for these it would most likely be an interjection, the result of which is then processed by PHP per usual - sticking to the one method interface concept seen in `Countable`, `JsonSerialize`, and the `__toString` magic method.

```php
interface [UsableAs]Integer
{
  public function toInteger(): int;
}

class MyType implements [UsableAs]Integer
{
  $value = 0;

  public function increment()
  {
    $this->value++; // The value could be an instance employ UsableAsInteger
    return $this;
  }

// - UsableAsInteger
  public function toInteger(): int
  {
    return $this->value;
  }
}

$instance = new MyType();

$two = $instance + $instance;

print($two); // 2

$instance->increment();

if ($instance > 1) {
  print "Inferred type, if not implemented, TypeError";
}
```

### Consistency

To make the implementation interaction more consistent, it may be worth considering something like the following as well.

```php
// Eventually replaces __toString after being in Parallel or __toString can be left as is
interface [UsableAs]String
{
  public function toString(): string
}

// Uses interfaces already available then adds one method
interface [UsableAs]Array implements \ArrayAccess, \Iterator
{
  // Mainly for when an instance is used with someting like `print_r()`
  public function toArray(): array
}
```

Haven't given much thought to the other primitives. Most valuable to me would be the `UsableAsBool`.

## See also

[PHP RFC:__toArray()](https://wiki.php.net/rfc/to-array)