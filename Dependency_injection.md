# Dependency Injection

Do you know "Dependency injection"? It is a design pattern that reduces the dependency between classes by passing another object that one class depends on from the outside.
Laravel has a feature called a `service container` that makes it easy to handle dependency injection. 

Let's understand how `service containers` and `dependency injection` work in Laravel through refactoring fictitious code.

## Use case

When you access a URL, you roll the dice twice and return the total number of rolls.

## First, make it work

```php
class DiceController extends Controller
{
    public function rollDouble()
    {
        return mt_rand(1, 6) + mt_rand(1, 6);
    }
}
```

After that, if you set the routing a little, you will have a "page that returns the result of rolling the 6-sided die twice".

In fact, this may be the optimal solution, depending on your requirements and development schedule. However, if the system has complicated requirements , I would like to make it a little more resistant to change . Let's refactor it little by little.

## Classify

`mt_rand(1, 6)` Is a code expression of the action of "rolling a 6-sided die once". However, I don't think that the intention will be conveyed accurately at this rate. So, like object-oriented, let's classify "6-sided dice" and express the action of "rolling once" as a method.

```php
class Dice
{
    public function roll(): int
    {
        return mt_rand(1, 6);
    }
}
```

Modify the controller as follows: `Dice` Create an instance of `roll()` and roll the dice by calling the method.


```php
    public function rollDouble()
    {
        // Init Dice
        $dice = new Dice();

        return $dice->roll() + $dice->roll();
    }
```

The name makes it more descriptive and makes it easier to understand that you are rolling the dice twice.

In addition, `6` the number from the controller has disappeared. 

For example, if the process of rolling the dice is everywhere and there is a specification change that says "I want to make the dice 12 sides instead of 6 sides", it is now possible to deal with it by modifying only one place.

## I want to test the logic

By the way, is this logic of "rolling the dice twice and returning the total value" really working? With this amount of code, you can judge it as "correct" at first glance, but in an actual system, more codes are intertwined and working, so in most cases you will check everything visually. Seems impossible in.

Therefore, we will utilize unit tests . Let's automatically test that the logic is correct in the test code.
Unfortunately, this logic is not easy to test. The result is different each time you run it, so you can't be sure that the logic of "rolling the dice twice and returning the sum" is working correctly.

## Make squid dice

The reason why the result is different every time is that we use "ordinary dice" that return random values. In other words, if you use Ikasama Dice, which gives the same value no matter how many times you shake it, you will be able to test it.

Let's make it right away.

```php
// Ikasama dice
class LoadedDice
{
    public function roll(): int
    {
        return 6; // return only 6
    }
}
```

The usage is the `Dice` same as, `roll()` just instantiate and call. However, in the case of Ikasama Dice, `roll()` the result of is always 6.
Let's incorporate this into the controller.

```php
    public function rollDouble()
    {
        $dice = new LoadedDice(); // replace with squid dice

        return $dice->roll() + $dice->roll();
    }
```

When I run it, it always returns `12`. It's natural `6` because I roll the dice that only come out twice.


## Switch depending on the environment

This time `12`, it came back in environments other than testing .This will not be a game, so try switching the dice instances depending on the environment.

```php
    public function rollDouble()
    {
        // Instantiate squid dice during testing, otherwise normal dice
        if (App::environment('testing')) {
            $dice = new LoadedDice();
        } else {
            $dice = new Dice();
        }

        return $dice->roll() + $dice->roll();
    }
```

You can judge the environment from the result in Laravel `App::environment('testing')`. If you write as above, you will get `12` the result of rolling the 6-sided dice twice in the test environment, but in other environments.

In this state, you can write unit tests. All you have to do `12` is make sure that when you call this method, it will return properly .

## Instantiate outside the controller

Finally, we're talking about dependency injection.
For the sake of explanation, return the controller to its original state.

```php
    public function rollDouble()
    {
        $dice = new Dice();

        return $dice->roll() + $dice->roll();
    }
```

rewrite

```php
    public function rollDouble(Dice $dice) // Specify Dice as the argument type hint 
    {
        return $dice->roll() + $dice->roll();
    }
```

Attention is the argument part. The controller `Dice $dicenow` takes the argument, and `Dice` the instantiation code has disappeared from the body . I haven't written the instantiation process anywhere yet. However, this still gives the same result.

In Laravel, if you specify an argument whose class name is `type-hinted` for the controller associated with the root, an instance of that class will be automatically generated and passed as an argument.
In this case, the `Dice` class is type-hinted, so it will pass `Dice` an instance of the class `$dice` as.

One of the functions of the `service container` built into Laravel, it helps to inject dependencies . It injects the `instance` that the controller depends on `Dice` from the outside .

If it is a controller, you can also inject it with the constructor.

```php
class DiceController extends Controller
{
    private $dice

    public function __construct(Dice $dice) // ← Injection！
    {
        // Should have as Properties 
        $this->dice = $dice;
    }

    public function rollDouble()
    {
        return $this->dice->roll() + $this->dice->roll();
    }
}
```

This is called `constructor injection`.
On the other hand, the method of injecting with a method is called `method injection`.

Classes built into Laravel's scheme, such as commands and middleware, can generally be injected with constructors. Let's remember.


## Cut out the common part

`Dice` Classes and `LoadedDice` classes return different values, but `roll()` the general structure of "returning an integer value with the method" is the same.
Cut this out as an interface in preparation for the next step.

```php
// Rolling dice interface
interface RollableDice
{
    public function roll(): int; // Defined as roll method with no arguments returns an integer value"
}
```

```php
// Ordinary dice
class Dice implements RollableDice
{
    public function roll(): int
    {
        return mt_rand(1, 6);
    }
}
```

```php
// Ikasama Dice
class LoadedDice implements RollableDice
{
    public function roll(): int
    {
        return 6;
    }
}
```

## Join setting

You can also redefine the logic for the service container to inject the instance. This is called `joining`.

The definition of the join is `ServiceProvider` described in. But if `AppServiceProvider` the number increases, `ServiceProvider` can create your own and divide it.

`\App\Providers\AppServiceProvider`

```php
public function register()
{
    // First argument: Join key ("fully qualified name" of the class you want to inject) 
    // Second argument: Callback function that returns an instance 
    $this->app->bind(Dice::class, function ($app) {
        return new Dice();
    });
}
```

If defined as above, it will be a "simple join" , and each time it is injected, the function of the second argument will be executed and another instance will be created. This is the same behavior as if you did not set the join.

On the other hand, if you define it as follows, it will generate an instance as a singleton . One instance is created when it is first called, and it will be reused thereafter.

```php
    public function register()
    {
        // Using bind rather singleton method, the instance is created only once
        $this->app->singleton(Dice::class, function ($app) {
            return new Dice();
        });
    }
```

## Depend on the interface

In the previous example, that is `Dice::class`, `Dice` the fully qualified name of the class was registered as the join key. The key to this join can actually be any string, but it must be the "fully qualified name of the class that can be specified in the type hint" for automatic injection.

As long as you can specify it as a type hint, you can also use an interface or abstract class as a key for joining. Let's combine `RollableDice` in the `Dice` classes with the interface we created earlier.

```php
    public function register()
    {
        //class names that binds to the interface and returns an instance of Dice
        $this->app->singleton(RollableDice::class, function ($app) {
            return new Dice();
        });
    }
```

Then `RollableDice` change the controller type hint to.

```php
    public function rollDouble(RollableDice $dice) // ←Change type hint from Dice to RollableDice 
    {
        return $dice->roll() + $dice->roll();
    }
```

When I run it, the type hint is an interface, but it's injected, `Dice` so it works fine. 2 3 Identifying an instance from a join key is called resolving a dependency .

At this point, there is a problem of where to write the process to switch to the Ikasama die that I made earlier, but let's move it to the join process for the time being.

```php
    public function register()
    {
        $this->app->singleton(RollableDice::class, function ($app) {
            if (App::environment('testing')) {
                return new LoadedDice();
            }
            return new Dice();
        });
    }
```

Now you can inject in the test environment `LoadedDice` but in other environments `Dice`.

## Check the dependency

Let's compare it with the code when it was instantiated in the controller.

old

```php
    public function rollDouble()
    {
        if (App::environment('testing')) {
            $dice = new LoadedDice();
        } else {
            $dice = new Dice();
        }

        return $dice->roll() + $dice->roll();
    }
```

new

```php
    public function rollDouble(RollableDice $dice)
    {
        return $dice->roll() + $dice->roll();
    }
```

In the old code, the controller had to know both the `Dice` class and the `LoadedDice` class generation process. This process must be rewritten each time there are more types of dice or arguments are needed in the constructor. It depends on these two classes . The role of this method is just to "roll the dice twice and return the total", but I know too much.

On the other hand, the new code `RollableDice` now depends only on the interface (the abstract concept of a swinging dice). `Dice` and `LoadedDice` What do the procedure is necessary to generate it is why no longer need this method know. `RollableDice` The interface guarantees that the class implementing this is a "swinging dice". If you just roll the dice twice and return the result, it is enough if you can guarantee that it is a "rolling dice" .

* If you want to know more deeply, please check "Principle of dependency reversal" and "Depends on abstraction"

## Supplementary information

The following topics have little to do with dependency injection, but are a supplement to Laravel's features and more

### Inject stub

This time I made a separate Ikasama Dice class to explain DI, but if you use Ikasama Dice only for testing, it is easier to create a stub in the test code and override the join setting.

```php
$this->mock(RollableDice::class, function ($mock) {
    $mock->shouldReceive('roll')->andReturn(6);
});
```

If you write it like this in the test case class, it will pass a 6 stub object that always returns when testing, when `RollableDice` resolving the dependency combined with.

### Other ways to retrieve dependent objects

Besides injecting with methods and constructors, there are other ways to get a combined instance.

`Illuminate\Foundation\Application` You can use the method wherever you can access instance of `make()`.

```php
$dice = $this->app->make(RollableDice::class);
```

`Illuminate\Foundation\Application` Even `resolve()` if you don't have access to, you can easily resolve the dependency by using Helper.

```php
$dice = resolve(RollableDice::class);
```

However, these methods should not be used.
With these, it is no longer a "dependency injection", but another pattern called a "service locator". 
Service locators are said to be anti-patterns. Instead of no longer relying on a particular class, it will depend on the service container everywhere.

Only use it in very limited circumstances, such as when you need to pass another instance during the join setup as described below.

## Multiple injection

If the injected class also depends on the processing of another class, it is possible to perform multiple injections.

For example, suppose you `mt_rand` want to use a different logic instead of generating random numbers .
In that case, just like the controller, `Dice` let's inject another instance with method injection.

```php
// Make Dice dependent on MarvelousRandTool class
// Omit MarvelousRandTool implementation 
class Dice implements RollableDice
{
    private $randTool;

    public function __construct(MarvelousRandTool $randTool)
    {
        $this->randTool = $randTool;
    }

    public function roll(): int
    {
        return $this->randTool->rand(1, 12);
    }
}
```

If you omit the join settings, when you `Dice` resolve the dependency , the `Dice` class that is injected in the constructor is also automatically resolved and the required instance is injected.
If you have defined the join settings yourself, you need to properly define the instance to pass to the constructor.

```php
    public function register()
    {
        $this->app->singleton(RollableDice::class, function ($app) {
            // random number generator also be obtained from the service container
            return new Dice($app->make(MarvelousRandTool::class));
        });
    }
```

`MarvelousRandTool` Notice that the instance of is also dependent resolved using the service container. You can reduce the dependence on each other here as well by setting the join settings elsewhere.

[source](https://qiita.com/harunbu/items/079ea728d2c9cf4f44d5)