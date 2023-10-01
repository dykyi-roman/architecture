# [Architecture](README.md)

## SOLID
The SOLID principles are intended to manifest mid-level code structure. The motivation behind these principles is the creation of mid-level software structures that:

### Rules
* Single responsibility principle (SRP) – a module should be responsible to one, and only one actor
* Open close principle (OCP) – a software artifact should be open for extension but closed for modification
* Liskov substitution principle (LSP) – states that objects in a program should be replaceable with instances of their subtypes without changing the correctness of that program
* Interface segregation principle (ISP) – make fine-grained interfaces that are client specific
* Dependency inversion principle ([DIP](dip.md)) – source code dependencies refer only to abstractions, not to concretions

### Summary
Set of five software design principles intended to make software design more understandable, flexible, and maintainable.
These principles are part of a larger set of principles promoted by Robert C. Martin, which are intended to help developers create better software architecture

### CODE

#### Single responsibility principle
:poop:
```php
final readonly class Job
{
    public function partTimeJob(): void
    {
    }
    
    public function fullDayJob(): void
    {   
    }
    
    //SRP broken, available methods for two actors
}

```
:heart:
```php
interface JobInterface
{
    public function execute(object $value): void;
}

final readonly class PartTimeJob implements JobInterface
{
    public function execute(object $value): void
    {
    }
}

final readonly class FullDatyJob implements JobInterface
{
    public function execute(object $value): void
    {
    }
}
```

### Open close principle
:poop:
```php
final readonly class Employment
{
    public function office(): void
    {
    }
    
    public function remote(): void
    {   
    }
    
    // OCP will be broken when we add a new employment type here
}

final readonly class EmploymentFactory
{
    public function __constructor(
        private Employment $employment,
    );

    public function create(string $type): Employment
    {
        return match ($type) {
            'office' => $this->employment->office(),
            'remote' => $this->employment->remote(),
            default => throw new \RuntimeException(sprintf('Unsupported employment type: %s'), $type),
        };
    }
}
```

:heart:
```php
interface EmploymentInterface
{
    public function job(): void;
}

final readonly class Office implements EmploymentInterface
{
     public function job(): void
     {
     }
}

final readonly class Remote implements EmploymentInterface
{
     public function job(): void
     {
     }
}

final readonly class EmploymentFactory
{
    public static function create(string $type): EmploymentInterface
    {
        return match ($type) {
            'office' => new Office(),
            'remote' => new Remote()
            default => throw new \RuntimeException(sprintf('Unsupported employment type: %s'), $type),
        };
    }
}

EmploymentFactory::create($request->tyype)->job();
```

### Liskov substitution principle
:poop:
```php
class FullDaySalaryCalculator
{
    private const HOURS = 8;
    
    public function __constructor(
        private int $days,
        private int $amount,
    ): void {
    }
    
    public function calculate(): int
    {
        return self::HOURS * $this->days * $this->amount;
    }
}

class HalfDaySalaryCalculator extends FullDaySalaryCalculator
{
}

// LSP will be broken   
(new HalfDaySalaryCalculator(30, 1000))->calculate();
```

:heart:
```php
interface SalaryCalculatorInterface
{
    public function calculate(int $days, int $amount): int
}

final readonly FullDaySalaryCalculator implements SalaryCalculatorInterface
{
    private const HOURS = 8;
    
    public function calculate(int $days, int $amount): int
    {
        return self::HOURS * $days * $amount; 
    }
}

final readonly HalfDaySalaryCalculator implements SalaryCalculatorInterface
{
    private const HOURS = 4;
    
    public function calculate(int $days, int $amount): int
    {
        return self::HOURS * $days * $amount; 
    }
}
```

### Interface segregation principle
:poop:
```php
// ISP is broken because doing too many things
interface CVInterface
{
    public function upload(object $value): void;
    public function download(): object;
    public function deactivate(): void;
    public function activate(): void;
}    
```

:heart:
```php
interface CVStorageInterface
{
    public function upload(object $value): void;
    public function download(): object;
}    

interface CVActivationInterface
{
    public function deactivate(): void;
    public function activate(): void;
}
```

### Dependency inversion principle
:poop:
```php
final readonly class ExportToPDF
{
    public function export(): object;
}   

final readonly CV
{
    // DIP is broken because we depend on from realization
    public function __construct(private ExportToPDF $exporter);
}
```

:heart:
```php
interface ExporterInterface
{
    public function export(): object;
}

final readonly class ExportToPDF implements ExporterInterface
{
    public function export(): object;
}   

final readonly class CV
{
    public function __construct(private ExporterInterface $exporter);
}
```

### Read
* [The Clean Code Blog - Solid Relevance](https://blog.cleancoder.com/uncle-bob/2020/10/18/Solid-Relevance.html)
* [SOLID Principles: The Software Developer's Framework to Robust & Maintainable Code](https://khalilstemmler.com/articles/solid-principles/solid-typescript/)
* [What are the SOLID Principles?](https://kessler.tech/software-architecture/solid/#aioseo-the-solid-principles-mid-level-architecture)