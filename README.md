# Winzou State Machine service provider for Laravel

[![Latest Version on Packagist](https://img.shields.io/packagist/v/sebdesign/laravel-state-machine.svg?style=flat-square)](https://packagist.org/packages/sebdesign/laravel-state-machine)
[![Software License](https://img.shields.io/badge/license-MIT-brightgreen.svg?style=flat-square)](LICENSE.md)
[![Build Status](https://img.shields.io/travis/sebdesign/laravel-state-machine/master.svg?style=flat-square)](https://travis-ci.org/sebdesign/laravel-state-machine)
[![Scrutinizer Code Quality](https://img.shields.io/scrutinizer/g/sebdesign/laravel-state-machine/master.svg?style=flat-square)](https://scrutinizer-ci.com/g/sebdesign/laravel-state-machine/?branch=master)
[![StyleCI](https://styleci.io/repos/78893356/shield?style=flat-square)](https://styleci.io/repos/78893356)

This is a Laravel service provider for [winzou/state-machine](https://github.com/winzou/state-machine). It provides dependency injection for the `StateMachineFactory`. You can also use Laravel's service container to resolve class methods for the callbacks. A facade is also available for convenience.

## Installation

You can install the package via composer. This package requires Laravel 5.1 or higher.

``` bash
composer require sebdesign/laravel-state-machine
```

Since version 5.5, Laravel uses package auto-discovery, so you don't need to manually add the ServiceProvider and the facade. If you don't use auto-discovery or you are using an older version, add the service provider and the facade in config/app.php.

``` php
<?php

'providers' => [
    Sebdesign\SM\ServiceProvider::class,
],

'aliases' => [
    'StateMachine' => Sebdesign\SM\Facade::class,
],
```

## Configuration

Publish the config file in `config/state-machine.php`.

``` bash
php artisan vendor:publish --provider="Sebdesign\SM\ServiceProvider"
```

Please see the documentation of the [StateMachineBundle](https://github.com/winzou/StateMachineBundle) for all the available options.

## Usage

``` php
<?php

// Get the article
$article = App\Article::find($id);

// Get the state machine for this article, and graph called "simple"

// Using the facade
$stateMachine = StateMachine::get($article, 'simple');

// Or using the service container with dependency injection
public function method(SM\Factory\FactoryInterface $factory)
{
    $stateMachine = $factory->get($article, 'simple');
}
```

Now you can use the `$stateMachine` to interact with the state of the `$article`.

``` php
<?php

// Get the actual state of the object
$stateMachine->getState();

// Get all available transitions
$stateMachine->getPossibleTransitions();

// Check if a transition can be applied: returns true or false
$stateMachine->can('approve');

// Apply a transition
$stateMachine->apply('publish');
```

### Callbacks

Callbacks are used to guard transitions or execute some code before or after applying transitions. This package adds the ability to use Laravel's service container to resolve callbacks.

E.g.:

You want to call the `handle` method on the `MyService` class to determine if the state machine can apply the `submit_changes` transition.

```php
<?php

'callbacks' => [
    // will be called when testing a transition
    'guard' => [
        'guard_on_submitting' => [
            // call the callback on a specific transition
            'on' => 'submit_changes',

            // will call the method of this class
            'do' => ['MyService', 'handle'],

            // arguments for the callback
            'args' => ['object'],
        ],
    ],
],
```

You can specify callbacks in array format, e.g. `['Class', 'method']`, or in *@* delimited string format, e.g. `Class@method`.

### Using Gates and Policies

If you don't want to use custom classes/methods for guarding, you can use Laravel's authorization gates or policies for determining if a transition can be applied.

Instead of specifying a `do` key, you must define a `can` key with the name of the ability you want to check. Since Laravel 5.5, you can also specify an array of abilities, and every one will be checked.

By default, the object instance will be passed as an argument to the gate. You can also override the arguments by specifying them in the `args` key.

#### Example using a gate:

In this example, we have defined a gate which will accept the `$article` as and argument. You are not required to define this parameter in your gate if you don't need it.

```php
<?php

use App\User;
use App\Article;

Gate::define('approve', function (User $user, Article $article) {
    //
});
```

```php
<?php

'callbacks' => [
    'guard' => [
        'guard_on_approving' => [
            // call the gate on a specific transition
            'on' => 'approve',
            // will call Gate:allows('approve', $article)
            'can' => 'approve',
        ],
    ],
],
```

#### Example using a policy:

Say you have created an `ArticlePolicy` policy for your `Article` model, which has an `approve` method.

You should define `approve` in the `can` index. This will be the equivalent of calling `$user->can('approve', $article)`.

You can also override the arguments that will be passed in the `approved` method, by specifying the `args` array. You **must** use `object` as the first argument in order for the policy class to be resolved. E.g.: `'args' => ['object', '"final_approval"']` would be the equivalent of calling `$user->can('approve', [$article, 'final_approval'])`.

```php
<?php

namespace App\Policies;

use App\User;
use App\Article;

class ArticlePolicy
{
    public function approve(User $user, Article $article)
    {
        //
    }
}
```

```php
<?php

'callbacks' => [
    'guard' => [
        'guard_on_approving' => [
            // call the policy on a specific transition
            'on' => 'approve',
            // will call Gate:allows('approve', $article)
            'can' => 'approve',
        ],
    ],
],
```

## Events

When checking if a transition can be applied, the `SM\Event\SMEvents::TEST_TRANSITION` event is fired.

Before and after a transition is being applied, the `SM\Event\SMEvents::PRE_TRANSITION` and `SM\Event\SMEvents::POST_TRANSITION` events are fired respectively.

All the events receive an `SM\Event\TransitionEvent` instance.

If you wish to listen to all the events with the same listener, you can use the `winzou.state_machine.*` wildcard parameter.

You can define your own listeners in your app's `EventServiceProvider`. E.g.:

```php
<?php

use SM\Event\SMEvents;

/**
 * The event listener mappings for the application.
 *
 * @var array
 */
protected $listen = [
    SMEvents::TEST_TRANSITION => [
        \App\Listeners\CheckTransition::class,
    ],
    SMEvents::PRE_TRANSITION => [
        \App\Listeners\BeforeTransition::class,
    ],
    SMEvents::POST_TRANSITION => [
        \App\Listeners\AfterTransition::class,
    ],
    'winzou.state_machine.*' => [
        \App\Listeners\Transition::class,
    ],
];
```

## Debug command

An artisan command for debugging graphs is included. It accepts the name of the graph as an argument. If no arguments are passed, the graph name will be asked interactively.

```bash
$ php artisan winzou:state-machine:debug simple

+--------------------+
| Configured States: |
+--------------------+
| new                |
| pending_review     |
| awaiting_changes   |
| accepted           |
| published          |
| rejected           |
+--------------------+

+-----------------+------------------+------------------+
| Transition      | From(s)          | To               |
+-----------------+------------------+------------------+
| create          | new              | pending_review   |
+-----------------+------------------+------------------+
| ask_for_changes | pending_review   | awaiting_changes |
|                 | accepted         |                  |
+-----------------+------------------+------------------+
| cancel_changes  | awaiting_changes | pending_review   |
+-----------------+------------------+------------------+
| submit_changes  | awaiting_changes | pending_review   |
+-----------------+------------------+------------------+
| approve         | pending_review   | accepted         |
|                 | rejected         |                  |
+-----------------+------------------+------------------+
| publish         | accepted         | published        |
+-----------------+------------------+------------------+

+---------------------+--------------------+------------------------+--------+
| Guard Callbacks     | Satisfies          | Do                     | Args   |
+---------------------+--------------------+------------------------+--------+
| guard_on_submitting | On: submit_changes | MyClass::handle()      | object |
| guard_on_approving  | On: approve        | Gate::check("approve") |        |
+---------------------+--------------------+------------------------+--------+

+---------------------+-------------+-------------+-----------------------------+
| Before Callbacks    | Satisfies   | Do          | Args                        |
+---------------------+-------------+-------------+-----------------------------+
| log_before_approval | On: approve | Log::info() | "approving article", object |
+---------------------+-------------+-------------+-----------------------------+

+----------------------+--------------+------------------------------+---------------+
| After Callbacks      | Satisfies    | Do                           | Args          |
+----------------------+--------------+------------------------------+---------------+
| email_after_approval | To: accepted | SendApprovalMail::dispatch() | object, event |
+----------------------+--------------+------------------------------+---------------+
```

## Changelog

Please see [CHANGELOG](CHANGELOG.md) for more information what has changed recently.

## Testing

``` bash
$ composer test
```

## Contributing

Please see [CONTRIBUTING](CONTRIBUTING.md) for details.

## Security

If you discover any security related issues, please email info@sebdesign.eu instead of using the issue tracker.

## Credits

- [Alexandre Bacco](https://github.com/winzou)

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.
