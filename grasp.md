# [Architecture](README.md)

## General Responsibility Assignment Software Patterns (GRASP)

It’s a set of recommendations, principles, and patterns that are really good and could make our code much better. Let’s take a look at this list:
* Information expert
* Creator
* Controller
* Low Coupling
* High Cohesion
* Pure Fabrication
* Indirection
* Protected Variations
* Polymorphism

### Information expert
Information expert might be the most important of all the GRASP patterns. This pattern says that all methods that work with data (variables, fields), should be in the same place where data (variables or fields) exist.

#### CODE
### Bad Practice
```php
final readonly class ProductPriceItem
{
    public function __construct(
        public int $amount,
        public int $price,
    ) {
    }
}

final readonly class ProductPriceList
{
    /** @var ProductPriceItem[] */
    private array $items;

    public function __construct(ProductPriceItem ...$items)
    {
        $this->items = $items;
    }
}

$productPriceList = new ProductPriceList(
    new ProductPriceItem(1, 2),
    new ProductPriceItem(2, 3),
    new ProductPriceItem(3, 4),
    new ProductPriceItem(4, 5),
);

// Bad: calculation logic is OUTSIDE the class that owns the data
$sum = 0;
foreach ($productPriceList->getItems() as $item) {
    $sum += $item->amount * $item->price;
}
```

### Good Practice
```php
final readonly class ProductPriceItem
{
    public function __construct(
        private int $amount,
        private int $price,
    ) {
    }

    // Good: calculation is WHERE the data lives
    public function totalPrice(): int
    {
        return $this->amount * $this->price;
    }
}

final readonly class ProductPriceList
{
    /** @var ProductPriceItem[] */
    private array $items;

    public function __construct(ProductPriceItem ...$items)
    {
        $this->items = $items;
    }

    // Good: aggregation logic is in the collection class
    public function total(): int
    {
        $sum = 0;
        foreach ($this->items as $item) {
            $sum += $item->totalPrice();
        }

        return $sum;
    }
}

// Clean usage - no external calculation needed
$total = (new ProductPriceList(
    new ProductPriceItem(1, 2),
    new ProductPriceItem(2, 3),
    new ProductPriceItem(3, 4),
    new ProductPriceItem(4, 5),
))->total();
```

### Creator
The Creator takes the responsibility of creating certain other objects. This principle asks us who creates an Object? Or who should create a new instance of some class?
This is a principle about aggregation too. Not always we need to use DI, in some cases it makes sense to create an object inside an object when it is not a separate part of this object and this object does not have properties.
For example Car and door of this car.

[Creation design patterns](https://refactoring.guru/design-patterns/creational-patterns) provide various object creation mechanisms, which increase flexibility and reuse of existing code.

### Controller
It is deals with how to delegate the request from the UI layer objects to domain layer objects.
I'm a proponent of using [ADR](https://github.com/pmjones/adr) for following this principe.

### Low Coupling and High Cohesion

Low Coupling:
* Follow SRP principe
* How strongly the objects are connected to each other?
* Coupling – object depending on other object, Constant & Model, Shared Kernel
* Modular - small module unite with other in big module like a part one domain
* How can we reduce the impact of change in depended upon elements on dependant elements
* Prefer low coupling – assign responsibilities so that coupling remain low
* Minimizes the dependency hence making system maintainable, efficient and code reusable

High Cohesion:
* How are the operations of any element are functionally related?
* Related responsibilities in to one manageable unit
* Clearly defines the purpose of the element
* Extend on the module level
* Traits, anonymous classes
* Shared Kernel

### Indirection

Help to avoid a direct coupling between two or more elements. 
Indirection introduces an intermediate unit to communicate between the other units, so that the other units are not directly coupled

Patterns that helped to do it: [Adapter](https://refactoring.guru/design-patterns/adapter), [Facade](https://refactoring.guru/design-patterns/facade), [Observer](https://refactoring.guru/design-patterns/observer)

### Polymorphism
Help to handle alternatives based on type. When related alternatives or behaviors vary by type (class), assign responsibility for the behavior (using polymorphic operations) to the types for which the behavior varies.

Patterns that helped to do it: [Strategy](https://refactoring.guru/design-patterns/strategy)

### Protected Variations
Help to avoid impact of variations of some elements on the other elements. Provides flexibility and protection from variations.
As architects and programmers we must be ready for **ever-changing** requirements. Fortunately, we are armed with a lot of design guidelines, principles, patterns and practices to support changes on different levels of abstraction:
* [SOLID](solid.md)
* [GangOfFour](https://martinfowler.com/bliki/GangOfFour.html)
* Encapsulation
* Law of Demeter
* Service Discovery
* Asynchronous messaging
* [Event-driven architectures](event_driven_architecture.md)

As Protected Variations principle says, first step is to identify points of predicted variation or instability

### Read
* [Book(Danya Rao) GRASP Design Principles](https://home.cs.colorado.edu/~kena/classes/5448/f12/presentation-materials/rao.pdf)
* [DESIGN PATTERNS](https://refactoring.guru/design-patterns)