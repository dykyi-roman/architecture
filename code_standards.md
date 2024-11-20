# [Architecture](README.md)

## Coding Standard and Best Practices

## Table of Contents
- [PSR-12 Coding Style](#psr-12-coding-style)
- [Naming Conventions](#naming-conventions)
- [PHPDoc Standards](#phpdoc-standards)
- [Error Handling](#error-handling)
- [Testing Requirements](#testing-requirements)
- [Read](#read)

## PSR-12 Coding Style

[PSR-12](https://www.php-fig.org/psr/psr-12/) extends and replaces PSR-2, providing a comprehensive coding style guide.

### Key Rules

**Files:**
* Use only `<?php` and `<?=` tags
* Use UTF-8 without BOM for PHP code
* Files should either declare symbols OR cause side-effects, not both
* Use Unix LF line endings
* End files with a single blank line

**Lines:**
* No hard limit on line length; soft limit is 120 characters
* No trailing whitespace at the end of lines
* Blank lines may be added for readability

**Indentation:**
* Use 4 spaces for indenting, not tabs

**Keywords:**
* PHP keywords must be in lowercase
* Type declarations must be in lowercase (`bool`, `int`, `string`, `array`, `null`)

```php
<?php

declare(strict_types=1);

namespace App\Domain\User;

final readonly class User
{
    public function __construct(
        private UserId $id,
        private Email $email,
        private string $name,
    ) {
    }

    public function changeEmail(Email $newEmail): self
    {
        return new self($this->id, $newEmail, $this->name);
    }
}
```

## Naming Conventions

### Classes and Interfaces
* Use **PascalCase** for class names
* Interfaces should have `Interface` suffix: `UserRepositoryInterface`
* Abstract classes should have `Abstract` prefix: `AbstractController`
* Traits should have `Trait` suffix: `LoggableTrait`
* Exceptions should have `Exception` suffix: `UserNotFoundException`

### Methods and Functions
* Use **camelCase** for method names
* Methods should be verbs or verb phrases: `save()`, `findById()`, `calculateTotal()`
* Boolean methods should start with `is`, `has`, `can`, `should`: `isActive()`, `hasPermission()`
* Getters don't need `get` prefix for simple property access: `name()` instead of `getName()`

### Variables and Properties
* Use **camelCase** for variables and properties
* Use descriptive names: `$userRepository` instead of `$repo`
* Avoid abbreviations: `$configuration` instead of `$cfg`
* Collections should be plural: `$users`, `$orderItems`

### Constants
* Use **UPPER_SNAKE_CASE** for constants
* Use class constants instead of global `define()`

```php
final class HttpStatus
{
    public const OK = 200;
    public const NOT_FOUND = 404;
    public const INTERNAL_SERVER_ERROR = 500;
}
```

## PHPDoc Standards

### When to Use PHPDoc
* For complex parameter types that can't be expressed with type hints
* For documenting exceptions thrown by methods
* For providing additional context about method behavior

### When NOT to Use PHPDoc
* When the type hint fully describes the parameter/return type
* For simple getters and setters
* When the code is self-documenting

### Good Example
```php
/**
 * @param array<string, mixed> $filters Associative array of filter criteria
 * @throws UserNotFoundException When user with given criteria doesn't exist
 */
public function findOneBy(array $filters): User
{
    // ...
}
```

### Bad Example (Redundant)
```php
/**
 * Get user name.
 *
 * @return string The name
 */
public function name(): string // PHPDoc is redundant here
{
    return $this->name;
}
```

## Error Handling

### Exceptions
* Create custom exceptions for domain-specific errors
* Extend from specific exception types (`InvalidArgumentException`, `RuntimeException`)
* Include context in exception messages

```php
final class UserNotFoundException extends RuntimeException
{
    public static function withId(UserId $id): self
    {
        return new self(
            sprintf('User with ID "%s" was not found', $id->toString())
        );
    }

    public static function withEmail(Email $email): self
    {
        return new self(
            sprintf('User with email "%s" was not found', $email->toString())
        );
    }
}
```

### Validation
* Validate at system boundaries (controllers, API handlers)
* Use Value Objects for domain validation
* Fail fast - validate early, fail early

```php
final readonly class Email
{
    private function __construct(private string $value)
    {
    }

    public static function fromString(string $email): self
    {
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            throw new InvalidArgumentException(
                sprintf('Invalid email format: %s', $email)
            );
        }

        return new self($email);
    }
}
```

### Logging
* Log exceptions with sufficient context
* Use appropriate log levels (ERROR, WARNING, INFO, DEBUG)
* Include correlation IDs for request tracing

## Testing Requirements

### Test Structure
* One test class per production class
* Test class names end with `Test`: `UserServiceTest`
* Test methods start with `test_` or use `#[Test]` attribute

### Coverage Requirements
* Domain Layer: 90% minimum
* Application Layer: 90% minimum
* Infrastructure Layer: 80% minimum
* Presentation Layer: 80% minimum

### Test Principles
* Each test verifies one behavior
* Tests should be independent and isolated
* Use meaningful test names that describe the scenario

```php
#[Group('unit')]
#[CoversClass(User::class)]
final class UserTest extends TestCase
{
    #[Test]
    public function it_creates_user_with_valid_data(): void
    {
        $user = User::create(
            UserId::generate(),
            Email::fromString('test@example.com'),
            'John Doe',
        );

        self::assertSame('test@example.com', $user->email()->toString());
        self::assertSame('John Doe', $user->name());
    }

    #[Test]
    public function it_changes_email(): void
    {
        $user = $this->createUser();
        $newEmail = Email::fromString('new@example.com');

        $updatedUser = $user->changeEmail($newEmail);

        self::assertSame('new@example.com', $updatedUser->email()->toString());
    }
}
```

### Mocking Guidelines
* Mock interfaces, not concrete classes
* Use real objects for Value Objects and Enums
* Keep mocks simple - avoid complex mock setups

## Read
* [PSR-12: Extended Coding Style](https://www.php-fig.org/psr/psr-12/)
* [PSR-4: Autoloader](https://www.php-fig.org/psr/psr-4/)
* [Clean Code PHP](https://github.com/jupeter/clean-code-php)
* [PHP: The Right Way](https://phptherightway.com/)
* [Object Calisthenics](https://williamdurand.fr/2013/06/03/object-calisthenics/)
