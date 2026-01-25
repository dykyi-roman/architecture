# [Architecture](README.md)

## The Dependency Inversion Principle (DIP)
Dependency inversion is one of the popular [SOLID](solid.md) principles, which is an acronym for the first five object-oriented design principles by [Robert C. Martin](http://cleancoder.com/files/about.md)
**Dependency Injection** is a well-known pattern and de-facto standard for implementing a Dependency Inversion Principle.
Most modern frameworks have some level of support for **Dependency Injection**.

### Rules
* A high-level modules should not depend on low-level modules. Both should depend on abstractions ([SOLID](solid.md))
* Abstractions should not depend on details. Details (concrete implementations) should depend on abstractions
* Avoid circular dependencies and keep code testable. The [Acylic Dependency Rule](https://khalilstemmler.com/wiki/acyclic-dependencies-principle/) describes this phenomenon in more detail.

### Summary
When high-level modules depend on abstractions, it promotes loose coupling, making it easier to change the implementation of the low-level modules without affecting the high-level modules.
Loose coupling, where you have minimal interdependence between components or modules of a system, is a sign of a well-structured application.

### CODE

```php
// High-level module depends on ABSTRACTION, not concrete implementation
final readonly class HighLevelComponent
{
    public function __construct(
        private LowLevelComponentInterface $dependency,
    ) {
    }

    public function execute(): void
    {
        $this->dependency->doSomething();
    }
}

// Abstraction (interface) - owned by the high-level module
interface LowLevelComponentInterface
{
    public function doSomething(): void;
}

// Low-level module implements the abstraction
final readonly class LowLevelComponent implements LowLevelComponentInterface
{
    public function doSomething(): void
    {
        // Implementation details
    }
}

// Another implementation can be easily swapped
final readonly class AnotherLowLevelComponent implements LowLevelComponentInterface
{
    public function doSomething(): void
    {
        // Different implementation
    }
}
```

### Practical Example

```php
// Without DIP - tightly coupled
final readonly class OrderService
{
    public function __construct(
        private MySQLOrderRepository $repository, // Depends on concrete class!
    ) {
    }
}

// With DIP - loosely coupled
interface OrderRepositoryInterface
{
    public function save(Order $order): void;
    public function findById(OrderId $id): ?Order;
}

final readonly class OrderService
{
    public function __construct(
        private OrderRepositoryInterface $repository, // Depends on abstraction
    ) {
    }
}

// Now we can have multiple implementations
final readonly class MySQLOrderRepository implements OrderRepositoryInterface { /* ... */ }
final readonly class RedisOrderRepository implements OrderRepositoryInterface { /* ... */ }
final readonly class InMemoryOrderRepository implements OrderRepositoryInterface { /* ... */ } // For tests
```

### Read
* [The Dependency Inversion Principle](https://web.archive.org/web/20110714224327/http://www.objectmentor.com/resources/articles/dip.pdf)
* [The Dependency Rule](https://khalilstemmler.com/wiki/dependency-rule/)
* [THE IMPORTANCE OF THE DEPENDENCY INVERSION PRINCIPLE](https://www.tripled.io/07/05/2019/dependency-inversion-principle/)