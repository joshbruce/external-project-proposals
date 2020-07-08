Help wanted: As it stands, I do not have the working knowledge or practice to feel comfortable doing the implementation alone. Please do reach out if you'd be interested on (helping) implement this concept. (After the RFC process and clearly defining what's being done.)

# Introduction

This [concept] allows an instance of a custom object to define a default boolean value, both `true` and `false`. At present, PHP treats any set instance as `true` with no way to return `false` without explicit request by the developer.

```php 
class MyBool
{}

$instance = new MyBool();

// do things that may make a key value go from true to false...

if ($instance) {
  print "This will always be true.";
}
```

There are two precedents already established for implementing such an affordance: a magic method, an interface, or both.
The magic method for defining a `boolean` value for an instance might look something like this.

```php
class MyBool
{
  public function __toBool()
  {
    return false;
  }
}

$instance = new MyBool();

if (! $instance) {
  print "Because we return false from the magic method, we end up here.";
}
```

A more true-to-life (albeit insanely contrived) scenario might look like the following.

```php
class MyBool
{
  private $x = 1;
  private $y = 0;

  public function __construct($y = 0)
  {
    $this->y = $y;
  }

  public function __toBool()
  {
    return ($this->x > $this->y);
  }
}

$instance = new MyBool();

if ($instance) {
  print "Magic method return is true.";
}

$instance = new MyBool(3);

if (! $instance) {
  print "Magic method return is false.";
}
```

The interface version looks similar, but includes the added step of explicitly attaching the interface to the class definition. 

When passing an instance as an argument to a function or method the PHP type system should facilitate type safety without the added complexity or development required by a magic method implementation (??). I am not sure which would be easier to implement when it comes to this concept: union types, available as of PHP 8, may help facilitate.

```php
interface BoolAccess
{
  public function toBool(): bool;
}

class MyBool implements BoolAccess
{
  public function toBool(): bool
  {
    return false; // Result of some calculation or check    
  }
}

$instance = new MyBool();

if (! $instance) {
  print "BoolAcces interface method returns false";
}
```
For the sake of comparison, without this capability, the previous example might look like this.

```php
interface BoolAcces
{
  public function toBool(): bool;
}

class MyBool implements BoolAcces
{
  public function toBool(): bool
  {
    return false; // Result of some calculation or check    
  }
}

$instance = new MyBool();

if ($instance instanceof BoolAcces) {
  print "Instance implements BoolAcces";

  if (! $instance->toBool()) {
    print "BoolAcces interface method returns false";

  }
}
```

First, we ensure the method is implemented (this check could be performed by PHP). 

Second, we explicitly call the method by way of added syntax. If the `BoolAccess` interface is not implemented in this scenario, the same PHP Error we use now would be displayed.

## Personal use

Calling a method that returns a `bool` that is the result of some calculation is something I do quite often. Being able to let the instance determine its own boolean value and return that value, with no extra steps from me (or additional syntax), means I, as the developer, don't need to maintain as much knowledge about the class (property and method names) and could allow something like the following (again, totally contrived and not very practical - maybe I'll look through my code for simple live examples).

```php
class MySomething
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

  public function toBool()
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

We can use `__toString` to remove one of the method calls.

```php
class MySomething
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

We can move the `increment` call to happen every time the instance is printed; cleaning up the call site even more and making `MySomething` a bit more self-aware, self-managing, and context-aware.

```php
class MySomething
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

Next we'll use the proposed `BoolAccess` interface to clean up just a bit more.

```php
class MySomething implements BoolAccess
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

## See also

The following RFCs are identified as similar in spirit to this [concept], possibly helped by this [concept], or this [concept] is potentially helped by the result of the linked RFC.
- [PHP RFC:__toArray()](https://wiki.php.net/rfc/to-array)
- [PHP RFC: Pipe Operator v2](https://wiki.php.net/rfc/pipe-operator-v2)
- [PHP RFC: Userspace operator overloading](https://wiki.php.net/rfc/userspace_operator_overloading)
- [PHP RFC: Union Types 2.0](https://wiki.php.net/rfc/pipe-operator-v2)
- (approved for PHP 8) [PHP RFC: Union Types 2.0](https://wiki.php.net/rfc/union_types_v2)
