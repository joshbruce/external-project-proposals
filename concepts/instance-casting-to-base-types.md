"cast" or "type juggle" I'm not sure are the most technically correct terms for the [concept]. The [concept] is:

> The ability to use developer-defined instances as if they were PHP primitives, at least in the syntax of PHP.

# Introduction

PHP affords developers to integrate their custom objects (classes) seamlessly into the PHP syntax in multiple ways.

To inform PHP what to do when an instance is assumed to be a string, there is the `__toString()` magic method.

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

For array access, there is the `ArrayAccess` interface methods (not shown), which allows internal value to be defined in the class and retrieved using array syntax: `$instance["hello"]` or `$instance[0]`. In the case of at least the `string` base type, this capability is native (using the codebase above).

```php
$instance = new _String();

$string = (string) $instance;

print $string[0];

// output: H
```

This [concept] would expand this capability and give more flexibility to developers.

## Benefits of specifying results of juggling to a base type

As developers use more libraries and leverage multiple classes (a developer-defined type), the chances of a variable ends up in a position wherein a `TypeError` can occur increases. By allowing developers to define their intended results could reduce these types of errors.

Developers can also generate language extensions that integrate seamlessly into the PHP syntax without having to write and install extensions or compilers - especially in shared hosting and similar environments may not be permitted. These language extensions can then be matured and possibly become the groundwork for future RFC submissions to improve PHP as a whole. (ex. Think of some JavaScript library functions that become part of Javascript, the become part of CSS or HTML.)

## Base types and scope

10 PHP primitive types and how developer-defined instances can be made to integrate and interact with inside the PHP syntax. Two can be immediately removed from consideration as one (`resource`) doesn't seem to be a common use case and the second (`NULL`) is always NULL, and an instance shouldn't need to specify how to become NULL.

### Available as of PHP 7

Of the eight remaining, two can be integrated into common PHP syntax by way of magic methods: `string` and `callable`. There is also a magic method similar to `__toString()` that allows developers to define how an instance will interact with the console and debugger called `__debugInfo()` that will also be used in this sample.

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

print($instance); // Hello, World!

print($instance()); // Invoked: Hello, World!

var_dump($instance); // Debugged + Invoked: Hello, World!
```

The important takeaway here is that at no point did the developer call a method on the instance to get the desired result from interacting directly with PHP. Further, by using the `__invoke()` magic method, the developer can interact with the instance as though it were an anonymous function (??), which PHP also offers the `Closure` interface to achieve functionality and compplexity not afforded by `__invoke()` alone. It's also worth mentioning that casting to a `callabl` does not seem possible.

Therefore, it does not seem necessary to include these in the scope of this [concept].

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

print($instance[0]); // 1 - using ArrayAccess

$instance2 = new MyType(...$instance); // copy - using Traversable via Iterator (to demonstrate variadic)

foreach ($instance2 as $key => $value) {
  print("{$key}: {$value} "); // 0: 1 1: 2 2: 3 - using Iterator
}
```

Note, again, that the developer is interacting with the instances as if they were primitives, no extra syntax. Further, it is worth noting that to mirror the full capabilities of the `array` primitive, the object must conform to both the `ArrayAccess` and `Iterator` methods.

Of the four remaining, one allows the developer to specify custom results from normal PHP interactions: `object`. Developers can override (or otherwise extend) the normal PHP behavior of interacting with objects, which an instance already is. However, PHP does not give a way to convert an instance of, say, `Foo` to an instance of `stdClass`, and that's okay. This customization is achieved by using various magic methods (the sample uses three that seem to be common): `__call()`, `__get()`, and `__set()`.

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

print($stdClass->hello); // Hello

// MyType
$instance = new MyType();

$instance->hello = "Hello";

print($instance->hello); // Hello

// MyType - extended capability
$instance->suffix(", World!");

print($instance->hello . $instance->suffix); // Hello, World!
```

Again, interacting with a custom instance using standard PHP syntax.

Of the three remaining, one seems like a partial implementation limited to only one use case, which would not help address the goal of interacting with instances as though they were PHP primitives: `integer`. The `Countable` interface allows PHP to automatically convert the instance into an integer, when being called by the `count()` function in the PHP Standard Library.

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

print(count($instance)); // 1 - using Countable

$sum = $instance + $instance; // PHP Notice: Object of class MyType could not be converted to int...
```

Just a quick aside to further illustrate what is being attempted. PHP also offers a way for developers to define how their custom implementations would be converted to a specific type of string, namely JSON by using the `JsonSerialize` serialize interface that allows an instance of the conforming class be passed into the `json_encode()` function from the PHP standard library.

### To be implemented

The three remaining (including `integer` still) primitives - `boolean`, `integer`, and `float` - do not appear to have a way to inform PHP on how to interact with them in situations where that type is required. Numbers can be modified using math operators and booleans are the basis for conditionals.

As it seems the desire of future versions of PHP will possibly move away from magic methods, or at least favor interfaces, the samples will do the same; however, a magic method approach is not ruled out. The root of the name is also still under discussion though `UsableAs` seems to approprately describe the desired outcome.

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

It's presumed `integer` and `float` would require similar methods to be characterized by PHP as an `integer` or `float`; responding to various math operators, for example. For our purposes, instead of something like `ArrayAcess` and `Iterator` where developers are asked to implement multiple methods, for this it would most like be an interjection, the result of which is then processed by PHP per usual - sticking to the one method interface concept seen in `Countable`, `JsonSerialize`, and the `__toString` magic method.

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
	print("Inferred type, if not implemented, TypeError");
}
```

### Consistency

To make the implementation interaction more consistent, it may be worth considering something like the following as well.

```php
// Eventually replaces __toString after being in Parallel
interface [UsableAs]String
{
	public function toString(): string
}

// Uses interfaces already in place
interface [UsableAs]Array implements \ArrayAccess, \Iterator
{
	// Mainly for when an instance is used with someting like `print_r()`
	public function toArray(): array
}
```

Haven't given much thought to the others. Most valuable to me would be the `UsableAsBool`.