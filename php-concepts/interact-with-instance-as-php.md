Help wanted: RFC submission and implementation.

As of this writing I do not believe I have the [karma points](https://wiki.php.net/rfc/howto) (step 2) required to submit an RFC myself.

As of this writing I do not have the knowledge, practice, and practical understanding of implementing within PHP internals to implement this myself. If you're interested in (helping) implement this concept, please do reach out (help may be in the form guidance and instruction or full implementation, up to you).

(After the RFC process and clearly defining what's being done.)

**Perceived roadblocks**

I have tried multiple times to regisster to the wiki per [the form](https://wiki.php.net/start?do=register). The error message received is not detailed enough and I'm not sure who to contact - the error message reads:

> That wasn't the answer we were expecting

I've tried multiple usernames in case that was the problem. I've an @gmail.com address and multiple self hosted email addresses. 

Any assistance would be greatly appreciated.

**Known questions and concerns still outstanding:**

- ~The drawbacks and potential complexity (or adding bugs) in implementing the cast capability.~
- Whether to use magic method only, interface only, or both (as of now, we are going with both to increase the potential for future consistency based on what is in PHP 8, and RFCs under review).
- What to name the interface as it does not fit the *-able pattern well.
- Rumors of magic methods going away - not from folks in the deep end of PHP development near as I can tell.

Please feel free to add more.

***

This [concept] introduces a new `__toBool()` method and `BoolAccess` interface, which is automatically added to classes implementing the `__toBool()` method.

It has # goals:

1. provide single method implementation by way of the `__toBool()` method, without explictly adding the BoolAccess interface to the class definition,
2. allow `bool|BoolAccess` to express `bool|object-with-__toBool()`, and
3. no BC breaks (no polyfill is planned at this time).

## Introduction

Goal 1: To maintain consistency within PHP this [concept] recognizes the existence of the `__toString()`, which allows class developers to create objects that can be seamlessly treated as a string. Further, this [concept] is in keeping with the ideas and motives behind the `Stringable` interface.[^1] Therefore, the following signature is proposed for the `__toBool()` method:

```php
public function __toBool(): bool
```

This also maintains consistency within PHP should the `__toArray()` method be approved, which further indicates an established desire for the idea of custom objects being converted to base primitives.[^2]

Goal 2: Forward compatibility for type safety and consistency in implementation are maintained through implicent declaration of the interface while also allowing explicit declaration as well using the `bool|Boolaccess` union type.[^1]

Stub of the interface declaraction:

```php
interface BoolAccess
{
  public function __toBool(): bool;
}
```

By explicitly adding the `bool` return type in the implentation, there should be no need to do so at compile time. This will be a point of differentiation with `__toString()` as its declaration doesn't require the existence of a return type.

Goal 3: As this functionality does not exist currently, there should be no methods "in the wild" with the same signature, as long as the guidance on naming from the PHP documentation has been followed. (Really want to flesh this out, but don't know any other hinderences at this time.)

## Usage

Status quote on casting for base primitives:

```php
$bool = (bool) "Hello"; // true

$bool = (bool) ""; // false

$bool = (bool) [1, 2, 3]; // true

$bool = (bool) []; // false

$bool = (bool) 1; // true

$bool = (bool) -1; // true

$bool = (bool) 0; // false

$bool = (bool) new \stdClass(); // true

// There is no false variant for an object instance.
```

Class using `__toBool()` only:

```php
class MyClass
{
  public function __toBool(): bool
  {
    return false;
  }
}

$bool = (bool) (new MyClass()); // output: bool(false)

if ((new MyClass())) {
  // will NOT be reached
}

if (! (new MyClass())) {
  // will be reached
}

if ($i = (new MyClass())) {
  // will NOT be reached
}

if (! $i = (new MyClass())) {
  // will be reached
}
```

Explicitly implementing the interface:

```php
class MyClass implements BoolAccess
{}
```

## Rationale

The following could be considered a rational implementation.

```php
class MyClass
{
  public function aFunction($return = true): ?string
  {
    if ($return) {
      return "Hello, World!";
    }
    return null;
  }
}

$instance = new MyClass();

if ($instance->aFunction() !== null) {
  print "Is true";
}

if ($instance->aFunction(false) === null) {
  print "Is false";
}
```

Some developers (including its inventor) view `null` as a necessary evil (if not a "billion dollar mistake"). Some developers return empty (read `false`) representations of the type in order to avoid returning something representative of nothing and maintaining stricter type safety.

```php
class MyClass
{
  public function aFunction($return = true): string
  {
    if ($return) {
      return "Hello, World!";
    }
    return "";
  }
}

$instance = new MyClass();

// Still wise to check (defensive)
if (strlen($instance->aFunction()) > 0) {
  print "Is true";
}

if (strlen($instance->aFunction(false)) === 0) {
  print "Is false";
}
```

The checks almost become boilerplate and can be seen throughout codebases.

The following sample returns a custom object. At present there is no way to have a valid instance of an "empty" custom object (??) and tell PHP (at least not without calling a method on the object, which requires the developer to know more about the object's API). 

In these cases, we often revert to using `null` - either the instance exists and is usable or it doesn't exist - nothing for the potential third option.

```php
class MyOtherClass
{}

class MyClass
{
  public function aFunction($return = true): ?MyOtherClass
  {
    if ($return) {
      return new MyOtherClass();
    }
    return null;
  }
}

$instance = new MyClass();

// Null-checks
if ($instance->aFunction() !== null) {
  print "Is true";
}

if ($instance->aFunction(false) === null) {
  print "Is false";
}
```

An instantiated object always resolves to `true`; therefore, the method could return `false`, instead of `null`. 

For this sample, it wouldn't matter as much as `null` (in PHP) resolves to `false`. However, explicitly returning `false` affords the developer debugging code to differentiate between something unusable and something that doesn't actually exist.

```php
class MyOtherClass
{}

class MyClass
{
  public function aFunction($return = true): ?MyOtherClass|bool
  {
    if ($return) {
      return new MyOtherClass();
    }

    // Could return null - which resolves to false
    return false;
  }
}

$instance = new MyClass();

if ($instance->aFunction()) {
  print "Is true";
}

if (! $instance->aFunction(false)) {
  print "Is false";
}
```

For the previous sample we need to implement union types for the purposes of type safety, or use the optional flag (?), or both.

Which raises the following questions:

1. Why does the calling object need to know what's going on with the new instance (defensive)? 
2. Why can't the new instance know, based on constuction arguments, what constitutes a usable state for itself beyond occupying memory or throwing exceptions (defensive)?

```php
class MyOtherClass
{
  private $bool = true;

  public function __construct(bool $bool = true)
  {
    $this->bool = $bool;
  }

  public function __toBool()
  {
    return $this->bool;
  }
}

$castable = new MyOtherClass();

$bool = (bool) $castable;

var_dump($bool); // output: bool(true)

class MyClass
{
  public function aFunction($return = true): MyOtherClass
  {
    return new MyOtherClass($return);
  }
}

$instance = new MyClass();

if ($instance->aFunction()) {
  print "Is true";
}

if ($instance->aFunction(false)) {
  print "Is always true, because not null";
}

if (! $instance->aFunction(false)->__toBool()) {
  print "Is false, would not have required method call.";
}
```

In the previous sample `MyOtherClass` is more self-aware. The `aFunction()` method returns one and only one type, which is never `null` unless an error occurs at instantiation.

Declaring a variable inside a condition could be productive as well, though will add an unknown degree of complexity to the implementation.

```php
class MyOtherClass
{
  private $bool = true;

  public function __construct(bool $bool = true)
  {
    $this->bool = $bool;
  }

  public function __toBool()
  {
    return $this->bool;
  }

  public function __toString()
  {
    return ($this->bool) ? "Hello, World!" : "To bool could be helpful";
  }
}

class MyClass
{
  public function aFunction($return = true): MyOtherClass
  {
    return new MyOtherClass($return);
  }
}

$instance = new MyClass();

if ($i = $instance->aFunction()) {
  print $i; // Hello, World!
}

// Without __toBool as part of PHP
if (! $instance->aFunction(false)->__toBool() and $i = $instance->aFunction(false)) {
    print $i; // To bool could be helpful
}

// Preferred with __toBool
if (! $i = $instance->aFunction(false)) {
    print $i; // To bool could be helpful
}

// And, we remove the need for `MyClass` - at least for this purpose
if (! $i = (new MyOtherClass(false))) {
  print $i; // To bool could be helpful
}
```

Applying the Not operator would:

1. call the instance method,
2. if not `null` AND cast-able to bool, cast the return value to `bool` (otherwise, exit conditional)
3. if `false` the return value is assigned to `$i`.

This could be useful when triggering errors and similar guard operations while simultaneously reducing syntactic boilerplate.

## Backward Incompatible Changes

No known incompatibilities.

## Proposed PHP Version(s)

PHP 8.1 or later (as I can't imagine it being approved and implemented for the PHP 8 code freeze)

## RFC Impact

No known negative impacts.

## Unaffected PHP Functionality

No known affect on current fuctionality.

## Future Scope

As of PHP 8 it is possible to convert (or at minimum interact with) custom objects as though they were PHP base primitives for:

- String: `__toString()` and `Stringable` (optional)
- Array: `ArrayAccess` and `Iterator`
- Object: `__get()`, `__set()`, `__call()`, and others
- Closure|Callable: `__invoke()`

This [concept] would add `bool` to the list. If userspace operator overloading is approved, that would effectively add `int` and `float` to the list.[^3] The magic method seems to be the most consistent base approach for this type of interaction.

## See also

The following RFCs are identified as similar in spirit to this [concept], possibly helped by this [concept], or this [concept] is potentially helped by the result of the linked RFC.

- [PHP RFC: Pipe Operator v2](https://wiki.php.net/rfc/pipe-operator-v2)
- (approved for PHP 8) [PHP RFC: Union Types 2.0](https://wiki.php.net/rfc/union_types_v2)

## Footnotes

- [^1]: [PHP RFC: Add Stringable interface](https://wiki.php.net/rfc/stringable)
- [^2]: [PHP RFC:__toArray()](https://wiki.php.net/rfc/to-array)
- [^3]: [PHP RFC: Userspace operator overloading](https://wiki.php.net/rfc/userspace_operator_overloading)
