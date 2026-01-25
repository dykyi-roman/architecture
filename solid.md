# [Architecture](README.md)

## SOLID
The SOLID principles are intended to manifest mid-level code structure. The motivation behind these principles is the creation of mid-level software structures that:

## Table of Contents
- [Rules](#rules)
- [Summary](#summary)
- [CODE](#code)
  - [Single responsibility principle](#single-responsibility-principle)
  - [Open close principle](#open-close-principle)
  - [Liskov substitution principle](#liskov-substitution-principle)
  - [Interface segregation principle](#interface-segregation-principle)
  - [Dependency inversion principle](#dependency-inversion-principle)
- [Read](#read)

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

SRP states that a class should have only **one reason to change** — meaning it should be responsible to only one actor (stakeholder). Having multiple methods is not a violation; mixing responsibilities for different actors is.

### Bad Practice
```php
final readonly class UserService
{
    public function __construct(
        private UserRepository $repository,
        private Mailer $mailer,
        private ReportGenerator $reportGenerator,
    ) {
    }

    // For HR department - employee management
    public function updateEmployeeData(User $user): void
    {
        $this->repository->save($user);
    }

    // For Marketing department - notifications
    public function sendPromotionalEmail(User $user): void
    {
        $this->mailer->send($user->email(), 'Promotion!');
    }

    // For Finance department - reporting
    public function generateSalaryReport(): Report
    {
        return $this->reportGenerator->generate();
    }

    // SRP broken: this class serves THREE different actors (HR, Marketing, Finance)
    // Changes requested by one department may affect others
}
```

### Good Practice
```php
// Each class serves ONE actor and has ONE reason to change

final readonly class EmployeeService
{
    public function __construct(private UserRepository $repository)
    {
    }

    public function updateEmployeeData(User $user): void
    {
        $this->repository->save($user);
    }
}

final readonly class MarketingNotificationService
{
    public function __construct(private Mailer $mailer)
    {
    }

    public function sendPromotionalEmail(User $user): void
    {
        $this->mailer->send($user->email(), 'Promotion!');
    }
}

final readonly class SalaryReportService
{
    public function __construct(private ReportGenerator $reportGenerator)
    {
    }

    public function generate(): Report
    {
        return $this->reportGenerator->generate();
    }
}
```

### Open close principle

OCP states that software entities should be **open for extension but closed for modification**. The key is that adding new behavior should not require changing existing, tested code.

**Note:** Factories with `match`/`switch` are acceptable extension points — they exist specifically to know about concrete types. OCP violation occurs when business logic requires modification for new types.

### Bad Practice
```php
final readonly class DiscountCalculator
{
    public function calculate(Order $order): Money
    {
        // OCP violated: adding new customer type requires modifying this method
        return match ($order->customerType()) {
            'regular' => $order->total()->multiply(0.0),
            'premium' => $order->total()->multiply(0.1),
            'vip' => $order->total()->multiply(0.2),
            // Adding 'enterprise' customer requires changing THIS class
            default => throw new \RuntimeException('Unknown customer type'),
        };
    }
}
```

### Good Practice
```php
interface DiscountStrategyInterface
{
    public function calculate(Order $order): Money;
    public function supports(string $customerType): bool;
}

final readonly class RegularCustomerDiscount implements DiscountStrategyInterface
{
    public function calculate(Order $order): Money
    {
        return $order->total()->multiply(0.0);
    }

    public function supports(string $customerType): bool
    {
        return $customerType === 'regular';
    }
}

final readonly class PremiumCustomerDiscount implements DiscountStrategyInterface
{
    public function calculate(Order $order): Money
    {
        return $order->total()->multiply(0.1);
    }

    public function supports(string $customerType): bool
    {
        return $customerType === 'premium';
    }
}

// Adding new discount: just create new class, no modification needed
final readonly class EnterpriseCustomerDiscount implements DiscountStrategyInterface
{
    public function calculate(Order $order): Money
    {
        return $order->total()->multiply(0.25);
    }

    public function supports(string $customerType): bool
    {
        return $customerType === 'enterprise';
    }
}

// This class is CLOSED for modification, OPEN for extension via new strategies
final readonly class DiscountCalculator
{
    /** @param iterable<DiscountStrategyInterface> $strategies */
    public function __construct(private iterable $strategies)
    {
    }

    public function calculate(Order $order): Money
    {
        foreach ($this->strategies as $strategy) {
            if ($strategy->supports($order->customerType())) {
                return $strategy->calculate($order);
            }
        }

        return Money::zero();
    }
}
```

### Liskov substitution principle
### Bad Practice
```php
class FullDaySalaryCalculator
{
    private const HOURS = 8;
    
    public function __construct(
        private int $days,
        private int $amount,
    ) {
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

### Good Practice
```php
interface SalaryCalculatorInterface
{
    public function calculate(int $days, int $amount): int;
}

final readonly class FullDaySalaryCalculator implements SalaryCalculatorInterface
{
    private const HOURS = 8;

    public function calculate(int $days, int $amount): int
    {
        return self::HOURS * $days * $amount;
    }
}

final readonly class HalfDaySalaryCalculator implements SalaryCalculatorInterface
{
    private const HOURS = 4;

    public function calculate(int $days, int $amount): int
    {
        return self::HOURS * $days * $amount;
    }
}
```

### Interface segregation principle
### Bad Practice
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

### Good Practice
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
### Bad Practice
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

### Good Practice
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