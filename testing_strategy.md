# [Architecture](README.md)

## Testing Strategy

A well-defined testing strategy ensures code quality, prevents regressions, and enables confident refactoring. This document covers testing approaches, test types, and best practices.

## Testing Pyramid vs Testing Trophy

### Testing Pyramid (Traditional)

```
        /\
       /  \     E2E Tests (Few, Slow, Expensive)
      /----\
     /      \   Integration Tests (Some)
    /--------\
   /          \ Unit Tests (Many, Fast, Cheap)
  /------------\
```

**Philosophy:** More unit tests, fewer integration and E2E tests.

### Testing Trophy (Modern)

```
     ___________
    |           |  E2E Tests (Few)
    |___________|
   |             |
   |             | Integration Tests (Most)
   |_____________|
    |           |
    |___________| Unit Tests (Some)
       |     |
       |_____| Static Analysis
```

**Philosophy:** Integration tests provide the best confidence-to-effort ratio.

## Test Types

### Unit Tests

Test individual components in isolation.

```php
#[Group('unit')]
#[CoversClass(Money::class)]
final class MoneyTest extends TestCase
{
    #[Test]
    public function it_adds_money_with_same_currency(): void
    {
        $money1 = Money::of(100, Currency::USD);
        $money2 = Money::of(50, Currency::USD);

        $result = $money1->add($money2);

        self::assertEquals(Money::of(150, Currency::USD), $result);
    }

    #[Test]
    public function it_throws_when_adding_different_currencies(): void
    {
        $money1 = Money::of(100, Currency::USD);
        $money2 = Money::of(50, Currency::EUR);

        $this->expectException(CurrencyMismatchException::class);

        $money1->add($money2);
    }
}
```

**When to use:**
* Value Objects
* Domain logic without dependencies
* Pure functions
* Complex algorithms

### Integration Tests

Test multiple components working together.

```php
#[Group('integration')]
final class CreateOrderUseCaseTest extends TestCase
{
    private CreateOrderUseCase $useCase;
    private Connection $connection;

    protected function setUp(): void
    {
        $this->connection = $this->getContainer()->get(Connection::class);
        $this->connection->beginTransaction();

        $this->useCase = $this->getContainer()->get(CreateOrderUseCase::class);
    }

    protected function tearDown(): void
    {
        $this->connection->rollBack();
    }

    #[Test]
    public function it_creates_order_and_reserves_inventory(): void
    {
        // Arrange
        $product = $this->createProduct(sku: 'SKU-001', stock: 10);
        $customer = $this->createCustomer();

        $command = new CreateOrderCommand(
            customerId: $customer->id(),
            items: [
                new OrderItemDto(productId: $product->id(), quantity: 2),
            ],
        );

        // Act
        $orderId = $this->useCase->execute($command);

        // Assert
        $order = $this->orderRepository->findById($orderId);
        self::assertNotNull($order);
        self::assertSame(OrderStatus::PENDING, $order->status());

        $updatedProduct = $this->productRepository->findById($product->id());
        self::assertSame(8, $updatedProduct->availableStock());
    }
}
```

**When to use:**
* Use cases / Application services
* Repository implementations
* API endpoints
* Message handlers

### E2E (End-to-End) Tests

Test complete user flows through the system.

```php
#[Group('e2e')]
final class CheckoutFlowTest extends WebTestCase
{
    #[Test]
    public function user_can_complete_checkout(): void
    {
        // Login
        $this->client->request('POST', '/api/auth/login', [
            'email' => 'user@example.com',
            'password' => 'password',
        ]);
        self::assertResponseIsSuccessful();

        // Add item to cart
        $this->client->request('POST', '/api/cart/items', [
            'product_id' => 'prod_123',
            'quantity' => 2,
        ]);
        self::assertResponseStatusCodeSame(201);

        // Checkout
        $this->client->request('POST', '/api/checkout', [
            'shipping_address' => [...],
            'payment_method' => 'card',
        ]);
        self::assertResponseIsSuccessful();

        // Verify order created
        $response = json_decode($this->client->getResponse()->getContent(), true);
        self::assertArrayHasKey('order_id', $response);
    }
}
```

**When to use:**
* Critical business flows
* User journeys
* Smoke tests

### Contract Tests

Verify API contracts between services.

```php
#[Group('contract')]
final class UserApiContractTest extends TestCase
{
    #[Test]
    public function get_user_response_matches_contract(): void
    {
        $response = $this->client->request('GET', '/api/users/123');

        $this->assertMatchesJsonSchema($response, 'schemas/user.json');
    }

    #[Test]
    public function create_user_accepts_valid_request(): void
    {
        $validRequest = $this->loadFixture('requests/create_user_valid.json');

        $response = $this->client->request('POST', '/api/users', $validRequest);

        self::assertResponseStatusCodeSame(201);
    }
}
```

## Test Doubles

### Types of Test Doubles

| Type | Purpose | Behavior |
|------|---------|----------|
| **Dummy** | Fill parameter lists | No implementation |
| **Stub** | Provide canned answers | Returns predefined values |
| **Spy** | Record interactions | Remembers calls for verification |
| **Mock** | Verify interactions | Expects specific calls |
| **Fake** | Working implementation | Simplified version (e.g., InMemoryRepository) |

### When to Use Each

```php
// DUMMY - just fills a parameter, never used
$logger = $this->createMock(LoggerInterface::class);
$service = new MyService($repository, $logger); // logger not relevant to test

// STUB - returns canned responses
$repository = $this->createStub(UserRepository::class);
$repository->method('findById')->willReturn($user);

// SPY - records calls for later verification
$mailer = $this->createMock(MailerInterface::class);
$mailer->expects($this->once())
    ->method('send')
    ->with($this->callback(fn($email) => $email->to() === 'user@example.com'));

// MOCK - strict expectations
$eventBus = $this->createMock(EventBusInterface::class);
$eventBus->expects($this->exactly(2))
    ->method('dispatch')
    ->withConsecutive(
        [$this->isInstanceOf(OrderCreated::class)],
        [$this->isInstanceOf(InventoryReserved::class)],
    );

// FAKE - working simplified implementation
final class InMemoryUserRepository implements UserRepositoryInterface
{
    private array $users = [];

    public function save(User $user): void
    {
        $this->users[$user->id()->toString()] = $user;
    }

    public function findById(UserId $id): ?User
    {
        return $this->users[$id->toString()] ?? null;
    }
}
```

### Rules for Mocking

1. **Mock interfaces, not concrete classes**
2. **Use real objects for Value Objects and Enums**
3. **Don't mock what you don't own** (use adapters)
4. **Prefer fakes over mocks for repositories**

```php
// Bad: mocking concrete class
$user = $this->createMock(User::class); // Don't do this!

// Good: use real Value Objects
$userId = UserId::generate();
$email = Email::fromString('test@example.com');
$user = User::create($userId, $email, 'John');

// Bad: mocking third-party library
$httpClient = $this->createMock(GuzzleClient::class); // Don't do this!

// Good: mock your own adapter
$paymentGateway = $this->createMock(PaymentGatewayInterface::class);
```

## Test Organization

### Directory Structure

```
tests/
├── Unit/
│   ├── Domain/
│   │   ├── User/
│   │   │   ├── UserTest.php
│   │   │   └── EmailTest.php
│   │   └── Order/
│   │       └── MoneyTest.php
│   └── Application/
│       └── CreateUserHandlerTest.php
├── Integration/
│   ├── Repository/
│   │   └── PostgresUserRepositoryTest.php
│   └── UseCase/
│       └── CreateOrderUseCaseTest.php
├── E2E/
│   └── Api/
│       └── UserApiTest.php
├── Contract/
│   └── UserApiContractTest.php
└── Fixtures/
    ├── UserFixture.php
    └── OrderFixture.php
```

### Test Naming

```php
// Pattern: it_does_something_when_condition
#[Test]
public function it_creates_user_with_valid_data(): void

#[Test]
public function it_throws_when_email_is_invalid(): void

#[Test]
public function it_returns_null_when_user_not_found(): void

// Or using test_ prefix
public function test_creates_user_with_valid_data(): void
```

## Coverage Guidelines

| Layer | Minimum Coverage | Focus On |
|-------|-----------------|----------|
| Domain | 90% | Business rules, invariants |
| Application | 90% | Use case flows, error handling |
| Infrastructure | 80% | Repository queries, external integrations |
| Presentation | 80% | Request validation, response mapping |

### What NOT to Test

* Framework code
* Simple getters/setters without logic
* Private methods (test through public interface)
* Third-party libraries

## Test Data Builders

```php
final class UserBuilder
{
    private UserId $id;
    private Email $email;
    private string $name;
    private UserStatus $status;

    public function __construct()
    {
        $this->id = UserId::generate();
        $this->email = Email::fromString('default@example.com');
        $this->name = 'Default User';
        $this->status = UserStatus::ACTIVE;
    }

    public static function aUser(): self
    {
        return new self();
    }

    public function withEmail(string $email): self
    {
        $this->email = Email::fromString($email);
        return $this;
    }

    public function inactive(): self
    {
        $this->status = UserStatus::INACTIVE;
        return $this;
    }

    public function build(): User
    {
        return new User($this->id, $this->email, $this->name, $this->status);
    }
}

// Usage
$user = UserBuilder::aUser()
    ->withEmail('john@example.com')
    ->inactive()
    ->build();
```

## Summary

* Use Testing Trophy approach: focus on integration tests
* Unit test domain logic and value objects
* Integration test use cases and repositories
* E2E test critical business flows only
* Use appropriate test doubles (prefer fakes over mocks)
* Maintain high coverage in Domain and Application layers
* Use test data builders for complex object creation

### Read
* [The Testing Trophy](https://kentcdodds.com/blog/the-testing-trophy-and-testing-classifications)
* [Test Doubles](https://martinfowler.com/bliki/TestDouble.html)
* [Writing Clean Tests](https://www.petrikainulainen.net/writing-clean-tests/)
* [PHPUnit Documentation](https://phpunit.de/documentation.html)
* [Mockery Documentation](https://docs.mockery.io/)