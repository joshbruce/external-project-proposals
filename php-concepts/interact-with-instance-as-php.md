Help wanted: As it stands, I do not have the working knowledge or practice to feel comfortable doing the implementation alone. Please do reach out if you'd be interested on (helping) implement this concept (could be answering my questions and teaching me). (After the RFC process and clearly defining what's being done.)

Feedback: Primarily please do use the PHP internals list; otherwise, feel free to submit an issue here. Thank you!

Fair warning: This is my first time, apologies for the bumps.

As of this writing I do not believe I have the [karma points](https://wiki.php.net/rfc/howto) (step 2) required to submit an RFC myself.

As of this writing I do not have the knowledge, practice, and practical understanding of implementing within PHP internals to implement this myself. If you're interested in (helping) implement this concept, please do reach out (help may be in the form guidance and instruction or full implementation, up to you).

(After the RFC process and clearly defining what's being done.)

**Known questions and concerns still outstanding:**

- The drawbacks and potential complexity (or adding bugs) in implementing the cast capability.
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

Goal 1: To maintain consistency within PHP this [concept] recognizes the existence of the `__toString()`, which allows class developers to create objects that can be seamlessly treated as a string. Further, this [concept] is in keeping with the ideas and motives behind the `Stringable` interface.[^1] Therefore, the following signature for the proposed `__toBool()` method:

```php
public function __toBool(): bool
```

This will also to maintain consistency within PHP should the `__toArray()` method approved, which further indicates an established mental conversion for the idea of custom objects being converted to base primitives.

Goal 2: Forward compatibility for type safety and consistency in implementation are maintained through implicent declaration of the interface while also allowing explicit declaration as well using the `bool|Boolaccess` union type.[^1]

Stub of the interface declaraction:

```php
interface BoolAccess
{
  public function __toBool(): bool;
}
```

By explicitly adding the `bool` return type in the implentation, there should be no need to do so at compile time. This will be a point of differentiation with `__toString()` as its declaration doesn't require the existence of a return type.

Goal 3: As this functionality current does not exist, there should be no methods "in the wild" with the same signature, as long as the guidance on naming from the PHP documentation has been followed. (Really want to flesh this out, but don't know any other hinderences to this.)

## Production Code Samples

These code samples could benefit from the availability of this capability. As there is no polyfill or installable extension, they are not live examples of this capability. (As of this writing they are not, as promised, code samples from a production implementations...still gathering.)

Class declaraction with cast:

```php
class MySomething implements BoolAccess
{
  public function __toBool(): bool
  {
    return false;
  }
}

$instance = new MySomething();

$bool = (bool) $instance;

var_dump($bool);
// output: bool(false)
```

Expanded sample (would work as expected as of PHP 8):

```php
class MySomething implements BoolAccess
{
  private $value = 0;
  
  public function increment()
  {
    $this->value++;
    return $this;
  }

  public function string()
  {
    return (string) $this->value;
  }

  public function toBool(): bool
  {
    return $this->value <= 10;
  }
}

$instance = new MySomething();

while($instance->toBool()) {
  print $instance->string();
  $instance->increment();
}

// output: 0 1 2 3 4 5 6 7 8 9 10
```

Adding `__toString()` with explicit declaration of `Stringable` (PHP 8+ compatible).

```php
class MySomething implements Stringable, BoolAccess
{
  private $value = 0;
  
  public function increment()
  {
    $this->value++;
    return $this;
  }

  public function __toString()
  {
    return (string) $this->value;
  }

  public function toBool()
  {
    return $this->value <= 10;
  }
}

$instance = new MySomething();

while($instance->toBool()) {
  print $instance;
  $instance->increment();
}

// output: 0 1 2 3 4 5 6 7 8 9 10
```

Tie the increment logic to the printing thereby making the instance a bit more self-aware, self-managing, and context-aware (PHP 8+ compatible).

```php
class MySomething implements Stringable, BoolAccess
{
  private $value = 0;
  
  public function increment()
  {
    $this->value++;
    return $this;
  }

  public function __toString()
  {
    $string = (string) $this->value ." ";
    $this->increment();
    return $string;
  }

  public function toBool()
  {
    return $this->value <= 10;
  }
}

$instance = new MySomething();

while($instance->toBool()) {
  print $instance;
}

// output: 0 1 2 3 4 5 6 7 8 9 10
```

Implementing the proposed `__toBool()` method and `BoolAccess` interface. (Unavailable in PHP.)

```php
class MySomething implements Stringable, BoolAccess
{
  private $value = 0;
  
  public function increment()
  {
    $this->value++;
    return $this;
  }

  public function __toString()
  {
    $string = (string) $this->value ." ";
    $this->increment();
    return $string;
  }

  public function toBool()
  {
    return $this->value <= 10;
  }
}

$instance = new MySomething();

while($instance) {
  print $instance;
}

// output: 0 1 2 3 4 5 6 7 8 9 10
```

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
