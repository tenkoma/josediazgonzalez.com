---
  title:       "CakePHP Testing 102"
  date:        2013-12-18 14:17
  description: "Some personal thoughts on testing your CakePHP code to get maximum efficiency"
  category:    cakephp
  tags:
    - cakeadvent-2013
    - cakephp
    - testing
  comments:    true
  sharing:     true
  published:   true
  layout:      post
  series:      CakeAdvent-2013
---

> Note: I have typically approached testing as `TODO: test this`. Meaning I cowboy code a lot. So I take lots of shortcuts. Keep that in mind as you read this post.

Here are a few tips that will help guide you on your testing path:

## Use composer to install PHPUnit

I hate having to deal with Pear and pear discovery etc. Instead, lets just use composer. Add the following to your `composer.json`:

```javascript
{
  "config": {
    "vendor-dir": "vendors"
  },
  "require-dev": {
    "phpunit/phpunit": "3.7.*"
  }
}
```

Next, run `composer update` to install dependencies. You'll finally need the following line in your `app/Config/bootstrap.php`:


```php
<?php
App::import('Vendor', array('file' => 'autoload'));
?>
```

Presto-chango! You now have PHPUnit setup and ready to go.

## Don't test your controllers (mostly)

I *hate* testing my controllers, because they all look like this:

```php
<?php
class OrdersController extends AppController {
  public function checkout () {
    $order = $this->OrderSession->get();
    $order->validate();
    $order->saveOrder();
    return $this->redirect($order->successUrl());
  }
}
?>
```

I use exceptions to handle error states, which makes my controller logic look a bit plain. Instead, I test the output of the methods I call. Test Components, Behaviors and Models, not controllers.

> Note: This does not mean you should ignore controller testing altogether. Rather, you want to avoid testing things that would be outside of the scope of your function, such as testing that authentication works in all your controller tests. That should be factored out into it's own set of tests, otherwise you are polluting your application logic tests.

> Testing controller logic requires mocking out the full request/response cycle, which is a pain to do. You retain some sanity if you don't do that.

## Learn to Mock

Mocking is incredibly powerful. It allows you to pretend a class exists and acts one way or another. For instance, you would mock a database, or mock an external service, which allows you to test code where there might otherwise not be a staging or testing environment.

Here is a mock deconstructed:

```php
<?php
// Create a mock within your test
$mock = $this->getMock(
  // (Required) Name of the class to mock
  'ClassYouAreMocking',
  // (Optional) Array of names of the methods to mock. Empty array means all methods
  array('names', 'of', 'methods', 'to', 'mock'),
  // (Optional) Parameters for the constructor
  array($args, $for, $constructor),
  // (Optional) Name of the created class. If empty, it is autogenerated
  'NameOfCreatedClass',
  // (Optional) Whether to call the original constructor
  $booleanCallConstructorOrNot
);

// (Required) How often do we expect this to run?
// Options: [never(), once(), any(), atLeastOnce(), exactly(3)]
// You can also specify the order at which a call is made
// - at(0) for the first call
// - at(6) for the seventh call
$mock->expects($this->once())
     // (Required) name of the method. This *must* be a mocked method
     ->method('methodYouExpectToBeCalled')
     // (Optional) required parameters. Omit this call when using `never()`
     ->with($args, $for, $method)
     // (Optional) Return value for the method.
     ->will($this->returnValue(TRUE));
?>
```

There are many good examples of mocking in the [Crud component](https://github.com/friendsofcake/crud)

## Run tests automatically

Use [TravisCI](http://travis-ci.com/) to automatically run tests. I detailed this in a [blog post for plugins earlier](/2013/12/01/testing-your-cakephp-plugins-with-travis/), but you should definitely consider doing it for your application as well.

## Use githooks for checking syntax

Testing should include whether your code is valid PHP. Rather than pushing bad code, test it before you make a commit in GIT. Here is a `pre-commit` hook you can use from [Phil Barker](http://www.phil-barker.com/2013/07/syntax-check-your-php-before-git-commit/):

```php
#!/usr/bin/php
<?php
// Grab all added, copied or modified files into $output array
exec('git diff --cached --name-status --diff-filter=ACM', $output);

foreach ($output as $file) {
    $fileName = trim(substr($file, 1));
    // We only want to check PHP files
    if (pathinfo($fileName,PATHINFO_EXTENSION) == "php") {
        $lint_output = array();
        // Check syntax with PHP lint
        exec("php -l " . escapeshellarg($fileName), $lint_output, $return);

        if ($return == 0) {
            continue;
        } else {
            echo implode("\n", $lint_output), "\n";
            exit(1);
        }
    }
}

exit(0);
?>
```

Simply add the above to your `.git/hooks/pre-commit` and you'll be off to the races!

## Find failing commits

Do you have a test that fails and you don't know when the regression was introduced? Use `git-bisect` to find it.

```shell
# start a bisect
git bisect state
# mark a commit as passing
git bisect good b0ac325
# mark a commit as failing
git bisect bad d09cf21
# tell git how to run your tests
git bisect run ./Console/cake test app Model/Post
```
