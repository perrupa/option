# mkjpryor/option #

Simple option and result classes for PHP, inspired by Scala's Option, Haskell's Maybe and Rust's Result.

The Option type provides additional functionality over nullable types for operating on values that may or may not be present.

The Result type also includes a reason for the failure (an exception), and allows errors to be propagated without worrying about specifically handling them.


## Installation ##

`mkjpryor/option` can be installed via [Composer](https://getcomposer.org/):

```bash
php composer.phar require mkjpryor/option dev-master
```


## Usage ##

### Creating an Option or a Result ###

```php
<?php

use Mkjp\Option\Option;
use Mkjp\Option\Result;

// Creates a non-empty option containing the value 10
$o1 = Option::just(10);

// Creates an empty option
$o2 = Option::none();

// Creates an option from a nullable value
//   If null is given, an empty option is created
$o3 = Option::from(10);
$o3 = Option::from(null);

// Create a successful result with the given value
$r1 = Result::success(42);

// Create an errored result with the given error
$r2 = Result::error(new \Exception("Some error occurred"));

// Create a result by trying some operation that might fail
//   Creates a success if the function returns successfully
//   Creates an error if the function throws an exception
$r3 = Result::_try(function() {
    // Some operation that might fail with an exception
});
```

### Retrieving a value from an Option or Result ###

The underlying value can be retrieved from an `Option` in an unsafe or safe manner:

```php
<?php

// UNSAFE - throws a LogicException if the option is empty
$val = $option->get();

// Returns the option's value if it is non-empty, 0 otherwise
$val = $option->getOrDefault(0);  

// Returns the option's value if it is non-empty, otherwise the result of evaluating the given function
//   Useful if the default value is expensive to compute
$val = $option->getOrElse(function() { return 0; });  

// Return the option's value if it is non-empty, null otherwise
$val = $option->getOrNull();
```

Similarly, the underlying value (or error) of a `Result` can be retrieved in an unsafe or safe manner:

```php
<?php

// UNSAFE - if the result is an error, the exception that caused the error is thrown
$val = $result->get();

// UNSAFE - if the result is a success, a LogicException is thrown
$err = $result->getError();

// Returns the result's value if it is a success, 0 if it is an error
$val = $result->getOrDefault(0);

// Returns the result's value if it is a success, otherwise the result of evaluating
// the given function with the exception that caused the error
//   Useful if the default value is expensive to compute or depends on the type of error
$val = $result->getOrElse(function(\Exception $error) {
    if( $error instanceof MyException ) return 0;
    return -1;
});  

// Return the result's value if it is a success, null otherwise
$val = $result->getOrNull();
```

### Manipulating options and results ###

`Option`s and `Result`s have several methods for manipulating them in an 'empty-safe' manner, e.g. `map`, `filter`. See the code for more details.


## Examples ##

In the following example, we want to retrieve a user by id from the database and welcome them. If the user does not exist, we want to welcome them as a guest.

```php
<?php

use Mkjp\Option\Option;
use Mkjp\Option\Result;


/**
 * Simple user class
 */
class User {
    public $id;
    public $username;
    public function __construct($id, $username) {
        $this->id = $id;
        $this->username = $username;
    }
}

/**
 * Fetches a user from a database by their id
 *
 * Note how we return a Result containing an Option
 * This is because there are three possible outcomes that are semantically different:
 *   1. We successfully find a user
 *   2. The user doesn't exist in the database (this isn't an error - it is expected and must be handled)
 *   3. There is an error querying the database
 */
function findUserById($id) {
    // Assume DB::execute throws a DBError if there is an error while querying
    $result = Result::_try(function() use($id) {
        return DB::execute("SELECT * FROM users WHERE id = ?", $id);
    });
    
    // Use the error propagation to our advantage
    return $result->map(function($data) {
        if( count($data) > 0 ) {
            return Option::just(new User($data[0]["id"], $data[0]["username"]));
        }
        else {
            return Option::none();
        }
    });
}

$id = 1234;  // This would come from request params or something similar
    
// Print an appropriate welcome message for the user
echo "Hello, " . findUserById($id)
                     // In this case, treat a DB error like not finding a user
                     //
                     // toOption converts the Result<Option<User>> into an
                     // Option<Option<User>>, which we flatten to an Option<User>
                     ->toOption()->flatten()
                     // Get the username from the user
                     ->map(function($u) { return $u->username; })
                     // If we didn't find a user, use a default name
                     ->getOrDefault("Guest");
```

## License ##

This code is licensed under the terms of the MIT licence.
