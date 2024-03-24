# Inline PHP Object validation

Object validation is something that many frameworks provide and are generally applied
via a combination of attributes/annotations and ORM validation.

For the cases where DTOs need some level of validation, PHP provides language features
that allow for handling in similar ways to Java, C#, or TypeScript/JavaScript.

## The problem

You are passing data around that is a portion of a repository or ORM data binder. Sometimes,
you are passing around portions of multiple objects and for simplicity you consolidate
them into a DTO.

At what point are these DTOs validated?

Whilst it is nice to have a single source of validation truth, and that lives inside the
repository/ORM code, there are times where a lightweight solution for validation is required.

## The solution

Using PHP attributes (annotations in many other langages) it is possible to provide a DTO
with a minimal level of validation so you can throw errors early and have relatively
swift traceability.

This is particularly important if you are expecting particular property types, or elements
to be required, where pushing invalid data through business rules leads to overly complex code
to handle said missing data and throw appropriate exceptions.

The solution is to use DTOs relatively frequently, and include base level validation
within them.

## Example

In many PHP applications I see examples of:

```php

public function (MyDto $myDto): bool {

    if (!is_null($myDto?->data)) {
        // Do something with the data
        // ...

        // Inform success.
        return true;
    }

    // Inform of failure - to be handled by calling method.
    return false;
}

```

Or:

```php
public function (MyDto $myDto): MyTransformedDto {

    try {
        $myTransformedDto = new MyTransformedDto();
        // Do something with the data which may cause an exception
        // ...

        // Return mutated DTO.
        return $myTransformedDto;
    }
    catch (Exception $exception) {
        throw new CustomDtoException();
    }
    
}
```

This swiftly becomes laborious and repitition is difficult to avoid unless there
is widespread understanding of the available services within the application.

## Solution

An alternativea to embedding basic checks and validation to a method expecting a DTO,
is to add validation to the DTO, where attributes can be used
to determine at run-time, which DTOs should be validated *and how*. This has two benefits:

1. Basic business validation rules are kept with the DTO, not hidden in code in a 
service layer
2. Standardised validation can be achieved in a system at a particular juncture
within the code.
3. Knowledge of the service layer isn't necessary to ensure that similar basic
validation is being carried out across various methods.

### DTO code

Assume we have a `UserSignInDto` which will eventually be passed to the authentication and
user repositories/ORMs.

In this example, the authantication repository will want the 'username' which will be
an email address as far as it is concerned. Similarly, the user repository will expect
the username to be an email address.

There is a risk that the different repositories will have different rules for what is
a valid email address, and the easiest place to manage these differences is when
receiving a request object containing the user's email and password.

Here, we apply some attributes so we can determine some rules that we can interpret 
within various points of the application.

We shall start with a pure PHP implementation, free from any framework code.

Our validation attributes are defined as:

```php


#[Attribute]
class ShouldValidate {

}

#[Attribute]
class IsRequired {

}

#[Attribute]
class IsEmail {

}
```

And our DTO uses them in this way:

```php
#[ShouldValidate]
class UserSignInDto {

  public function __construct(
    #[IsRequired]
    #[IsEmail]
    protected string $username,

    #[IsRequired]
    protected string $password
  ) {
  }

  public function getUsername(): string {
    return $this->username;
  }

  public function getPassword(): string {
    return $this->password;
  }

}
```

Please note that this is an immutable object, the values can only be set once. For the validation that
follows, we are only needing the attribute on the property itself, so getters would not require the same
attribute.

This allows the recipient method, or creating method, to validate an object when it decides to. I will
apply this through a trait for ease:

```php
trait Validator {

  public function exceptionIfInvalid(object $dto) {
    $reflectionClass = new ReflectionClass($dto);
    $shouldValidate = $reflectionClass->getAttributes(ShouldValidate::class);

    if (empty($shouldValidate)) {
      return; // No validation needed
    }

    foreach ($reflectionClass->getProperties() as $property) {
      $isRequired = $property->getAttributes(IsRequired::class);
      $isEmail = $property->getAttributes(IsEmail::class);
      $property->setAccessible(TRUE);
      $value = $property->getValue($dto);

      // Required field validation
      if (!empty($isRequired) && empty($value)) {
        throw new InvalidArgumentException("The {$property->getName()} field is required.");
      }

      // Email format validation
      if (!empty($isEmail) && !filter_var($value, FILTER_VALIDATE_EMAIL)) {
        throw new InvalidArgumentException("The {$property->getName()} field must be a valid email address.");
      }
    }
  }

}
```

This validator can be improved, but is useful to highlight how straightforward reflection
and attribute retrieval is in PHP.

When used, we can instruct a controller, or another method, to validate a DTO at any point,
secure in the knowledge that consistent validation rules are being followed:

```php
class MyClass {

  use Validator;

  /**
   * @param string $username
   * @param string $password
   *
   * @return void
   * @throws \HttpException
   */
  public function signIn(string $username, string $password): void {
    $dto = new UserSignInDto(
      $username,
      $password
    );
    $this->exceptionIfInvalid($dto);
    // Do something...
  }

}
```

Some examples of how this causes exceptions are:

```php
$myClass = new MyClass();
$myClass->signIn(
  'invalid.email',
  ''
);
```

Throws:

```
PHP Fatal error:  Uncaught InvalidArgumentException: The username field must be a valid email address. 
```

```php
$myClass = new MyClass();
$myClass->signIn(
  'invalid.email@mail.com',
  ''
);
```

Throws:

```
PHP Fatal error:  Uncaught InvalidArgumentException: The password field is required.
```

Whereas:

```php
$myClass = new MyClass();
$myClass->signIn(
  'invalid.email@mail.com',
  '123'
);
```

Throws no error.

Now we have a common framework for validating DTOs and can expand on the validation logic with ease.

## Extensions

This is a basic example of tying validation into an application.

Further application of this approach could be to:

1. Implement interfaces to show that a controller, or some other object, expects validated data
2. Use a framework to implement the interceptor pattern, so that any method with an attribute goes through validation
3. Add customised error text for the validation failure messages
4. Add Swagger compatibility

## Framework implementations

Many of the most common frameworks have implemented similar validator functionality.

### Symfony

DTO with validation:

```php
use Symfony\Component\Validator\Constraints as Assert;

class UserSignInDto {
    /**
     * @Assert\NotBlank(message="Username should not be blank.")
     * @Assert\Email(message = "The email '{{ value }}' is not a valid email.")
     */
    private $username;

    /**
     * @Assert\NotBlank(message="Password should not be blank.")
     */
    private $password;

    // Constructor, getters, and setters...
}

```

Is used in a controller or service:

```php
use Symfony\Component\Validator\Validator\ValidatorInterface;

class YourController {
    private $validator;

    public function __construct(ValidatorInterface $validator) {
        $this->validator = $validator;
    }

    public function someAction() {
        $dto = new UserSignInDto();
        $errors = $this->validator->validate($dto);

        if (count($errors) > 0) {
            // Handle validation errors...
        }

        // Proceed with valid DTO...
    }
}

```

### Laravel

Laravel provides a different approach, integrating validation directly into the HTTP layer, often using Form Request classes. 
However, you can also apply similar principles to DTOs. Here's a basic example of how you might validate a DTO in Laravel:

With a DTO:

```php

namespace App\DTOs;

use Illuminate\Support\Facades\Validator;

class UserSignInDto {
    private $username;
    private $password;

    public function __construct(array $attributes) {
        $validator = Validator::make($attributes, [
            'username' => 'required|email',
            'password' => 'required',
        ]);

        if ($validator->fails()) {
            throw new \Illuminate\Validation\ValidationException($validator);
        }

        $this->username = $attributes['username'];
        $this->password = $attributes['password'];
    }

    // Getters...
}
```

Can then be used in controllers:

```php
namespace App\Http\Controllers;

use Illuminate\Http\Request;
use App\DTOs\UserSignInDto;

class YourController extends Controller {
    public function signIn(Request $request) {
        $dto = new UserSignInDto($request->all());

        // Proceed with your business logic...
    }
}
```

## Conclusion

Validation of DTOs is easily possible and something that you may wish to integrate into an application
if your architecture demands it. Many of the common PHP frameworks provide something similar, however
in Laravel your validation is often tied to form data that a user is inputting, whereas Symfony
offers greater validation on generic objects, even if through DocBlocks.

Where applications are:

1. Performing their own significant business rules
2. Not directly saving data from one input into a specific entity (REST style)
3. Data is extracted from an entity or user input for another process via a DTO
4. Validation is tightly coupled to business language

It may be useful to integrate your own validation syntax for DTOs or requests to your application.
