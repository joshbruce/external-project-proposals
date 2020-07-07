# Introduction

This [concept] seeks to implement a for an instance of custom object to be understood as a boolean value, both positive and negative. At present, it appears that PHP will treat any set instance as `true` with no way to return false.

```php
class MyBool
{}

$instance = new MyBool();

// do things that may make a key value go from true to false...

if ($instance) {
  print "This will always be true.";
}
```

There are two ways to do this that PHP already takes advantage of. The first is magic methods. The second is interfaces that are adopted and the methods implemented.

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

The interface version looks similar, but includes the added step of explicitly attaching the interface to the class definition. This should also allow developers to leverage the PHP type system to determine if a given class conforms to the interface.

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

if (! $instance) {
  print "BoolAcces interface method returns false";
}
```

Depending on the use case, the need for creating calculated properties or methods returning boolean values becomes moot, and the resulting code feels cleaner.

For the sake of comparison, without this capability, the previous example would need to look something like this.

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
  print "Instance implement BoolAcces";
  if (! $instance->toBool()) {
    print "BoolAcces interface method returns false";
  }
}
```

The first conditional is required to ensure the method is implemented (could be inferred by PHP). Then we must explicitly call the method by way of added syntax. If the `BoolAccess` interface is not implemented, the same PHP Error we use now would be displayed.

## Personal use

Calling a method that returns a `bool` that is the result of some calculation is something I do quite often. Having the ability to determine its own default boolean value and return the value with no extra steps from me, could allow something like the following (again, totally contrived and not useful - maybe I'll look through my code for simple live examples).

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

Of course, we can use `__toString` to remove one of the method calls.

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

We can move the `increment` call to happen every time the instance is printed; cleaning up the call site even more.

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

Using the proposed `BoolAccess` interface to clean up just a bit more.

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

[PHP RFC:__toArray()](https://wiki.php.net/rfc/to-array)
[PHP RFC: Userspace operator overloading](https://wiki.php.net/rfc/userspace_operator_overloading)