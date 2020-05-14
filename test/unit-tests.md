# Unit tests

Unit tests are an important part of a project. They are here to help you see if your code does exactly
what you want, or if you broke something during another development (as part of a CI). 

Each method you implement should be tested with passing and failing values. That means that your functions need to be simple
and perform a single task. 

In symfony, unit tests are executed thanks to PHPUnit (with the symfony/test-pack).

Let's see how this work. Suppose you have a class with a method that says "Hello {name}". 
It should return "error" if a name is not sent.

```php
namespace App\Util;

class Greeting
{
    public function sayHello($name)
    {
        if (null === $name) {
            return 'Error';
        }

        return 'Hello '.$name;
    }
}
```

Now, let's do a passing test

```php
namespace App\Tests\Util;

use App\Util\Greeting;
use PHPUnit\Framework\TestCase;

class GreetingTest extends TestCase
{
    public function testSayHello()
    {
        $greeting = new Greeting();
        $result = $greeting->sayHello('sKyoKun');

        // assert that our strings are equals
        $this->assertEquals('Hello sKyoKun', $result);
    }
}
```

We can now launch our test with 

```Shell
    # execute all the tests in the file
    php bin/phpunit tests/Util/GreetingTest.php 
    
    # or only this test
    php bin/phpunit tests/Util/GreetingTest.php --filter=testSayHello 
```

Now let's test the failing case (without sending a name). We create a new case "testSayHelloError"

```php
public function testSayHelloError()
{
    $greeting = new Greeting();
    $result = $greeting->sayHello();

    $this->assertEquals('Error', $result);
}
```

This is good, but we can simplify these tests by using a data provider. 

Data providers are used to test a method against a set of values. Our tests could now be merged like this.

```php
namespace App\Tests\Util;

use App\Util\Greeting;
use PHPUnit\Framework\TestCase;

class GreetingTest extends TestCase
{
    /**
    * @dataProvider nameProvider
    */
    public function testSayHello($name, $expected)
    {
        $greeting = new Greeting();
        $result = $greeting->sayHello($name);

        // assert that our strings are equals
        $this->assertEquals($expected, $result);
    }

    /**
    * Here the first parameter is the name of the data set. 
    * The array contains the first parameter and the expected return.
    */
    public function nameProvider()
    {
        return [
            'Greeting to sKyoKun'  => ['sKyoKun', 'Hello sKyoKun'],
            'Greeting error' => [null, 'Error'],
            'Greeting to Another name' => ['Another name', 'Hello Another name'],
        ];
    }
}
```

When we'll execute our test, PHPUnit will now do 3 tests, one for each dataset.