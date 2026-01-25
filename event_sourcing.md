# [Architecture](README.md)

## Event Sourcing

Event Sourcing is a pattern where the state of an application is determined by a sequence of events rather than just storing the current state. Instead of persisting the current state of an entity, we persist all the events that led to that state.

### Core Concept

Traditional approach stores **current state**:
```
User: { id: 1, balance: 150 }
```

Event Sourcing stores **all events**:
```
1. AccountCreated { userId: 1, initialBalance: 0 }
2. MoneyDeposited { userId: 1, amount: 200 }
3. MoneyWithdrawn { userId: 1, amount: 50 }
// Current state is derived: balance = 0 + 200 - 50 = 150
```

### Key Components

**Event Store** - Append-only storage for domain events. Events are immutable and never deleted or modified.

**Aggregate** - Reconstructs its state by replaying events from the Event Store.

**Projection** - Read model built from events for querying purposes.

**Snapshot** - Optimization technique to avoid replaying all events.

### Implementation

```php
interface DomainEvent
{
    public function occurredAt(): DateTimeImmutable;
    public function aggregateId(): string;
}

final readonly class MoneyDeposited implements DomainEvent
{
    public function __construct(
        private string $accountId,
        private Money $amount,
        private DateTimeImmutable $occurredAt,
    ) {
    }

    public function occurredAt(): DateTimeImmutable
    {
        return $this->occurredAt;
    }

    public function aggregateId(): string
    {
        return $this->accountId;
    }

    public function amount(): Money
    {
        return $this->amount;
    }
}

final class BankAccount
{
    private Money $balance;
    /** @var DomainEvent[] */
    private array $uncommittedEvents = [];

    private function __construct(private string $id)
    {
        $this->balance = Money::zero();
    }

    public static function open(string $id, Money $initialDeposit): self
    {
        $account = new self($id);
        $account->apply(new AccountOpened($id, $initialDeposit, new DateTimeImmutable()));

        return $account;
    }

    public function deposit(Money $amount): void
    {
        $this->apply(new MoneyDeposited($this->id, $amount, new DateTimeImmutable()));
    }

    public function withdraw(Money $amount): void
    {
        if ($this->balance->lessThan($amount)) {
            throw new InsufficientFundsException();
        }

        $this->apply(new MoneyWithdrawn($this->id, $amount, new DateTimeImmutable()));
    }

    private function apply(DomainEvent $event): void
    {
        $this->when($event);
        $this->uncommittedEvents[] = $event;
    }

    private function when(DomainEvent $event): void
    {
        match ($event::class) {
            AccountOpened::class => $this->balance = $event->initialDeposit(),
            MoneyDeposited::class => $this->balance = $this->balance->add($event->amount()),
            MoneyWithdrawn::class => $this->balance = $this->balance->subtract($event->amount()),
            default => null,
        };
    }

    public static function reconstitute(string $id, array $events): self
    {
        $account = new self($id);
        foreach ($events as $event) {
            $account->when($event);
        }

        return $account;
    }

    /** @return DomainEvent[] */
    public function uncommittedEvents(): array
    {
        return $this->uncommittedEvents;
    }
}
```

### Event Store Interface

```php
interface EventStoreInterface
{
    /** @return DomainEvent[] */
    public function load(string $aggregateId): array;

    /** @param DomainEvent[] $events */
    public function append(string $aggregateId, array $events, int $expectedVersion): void;
}

final readonly class PostgresEventStore implements EventStoreInterface
{
    public function append(string $aggregateId, array $events, int $expectedVersion): void
    {
        $this->connection->beginTransaction();

        try {
            // Optimistic locking
            $currentVersion = $this->getCurrentVersion($aggregateId);
            if ($currentVersion !== $expectedVersion) {
                throw new ConcurrencyException('Aggregate was modified');
            }

            foreach ($events as $event) {
                $this->connection->insert('event_store', [
                    'aggregate_id' => $aggregateId,
                    'event_type' => $event::class,
                    'payload' => json_encode($this->serializer->serialize($event)),
                    'version' => ++$currentVersion,
                    'occurred_at' => $event->occurredAt()->format('Y-m-d H:i:s.u'),
                ]);
            }

            $this->connection->commit();
        } catch (\Throwable $e) {
            $this->connection->rollBack();
            throw $e;
        }
    }
}
```

### Snapshots

For aggregates with many events, replaying all events can be slow. Snapshots store the aggregate state at a point in time.

```php
final readonly class Snapshot
{
    public function __construct(
        public string $aggregateId,
        public int $version,
        public array $state,
        public DateTimeImmutable $createdAt,
    ) {
    }
}

final readonly class SnapshotRepository
{
    private const SNAPSHOT_THRESHOLD = 100;

    public function loadAggregate(string $id): BankAccount
    {
        $snapshot = $this->snapshotStore->load($id);

        if ($snapshot !== null) {
            // Load events AFTER snapshot
            $events = $this->eventStore->loadFromVersion($id, $snapshot->version);
            return BankAccount::reconstituteFromSnapshot($snapshot, $events);
        }

        // No snapshot - replay all events
        return BankAccount::reconstitute($id, $this->eventStore->load($id));
    }

    public function save(BankAccount $account): void
    {
        $events = $account->uncommittedEvents();
        $this->eventStore->append($account->id(), $events, $account->version());

        // Create snapshot every N events
        if ($account->version() % self::SNAPSHOT_THRESHOLD === 0) {
            $this->snapshotStore->save(new Snapshot(
                $account->id(),
                $account->version(),
                $account->toSnapshot(),
                new DateTimeImmutable(),
            ));
        }
    }
}
```

### Event Upcasting

When event schemas change over time, upcasting transforms old events to new format.

```php
interface EventUpcasterInterface
{
    public function supports(string $eventType, int $version): bool;
    public function upcast(array $payload, int $fromVersion): array;
}

final readonly class MoneyDepositedV1ToV2Upcaster implements EventUpcasterInterface
{
    public function supports(string $eventType, int $version): bool
    {
        return $eventType === 'MoneyDeposited' && $version === 1;
    }

    public function upcast(array $payload, int $fromVersion): array
    {
        // V1 had 'amount' as integer cents
        // V2 has 'amount' as Money object with currency
        return [
            ...$payload,
            'amount' => [
                'value' => $payload['amount'],
                'currency' => 'USD', // Default for old events
            ],
        ];
    }
}
```

### Projections (Read Models)

Projections build read-optimized views from events.

```php
final class AccountBalanceProjection
{
    public function __construct(
        private Connection $connection,
    ) {
    }

    public function handle(DomainEvent $event): void
    {
        match ($event::class) {
            AccountOpened::class => $this->onAccountOpened($event),
            MoneyDeposited::class => $this->onMoneyDeposited($event),
            MoneyWithdrawn::class => $this->onMoneyWithdrawn($event),
            default => null,
        };
    }

    private function onAccountOpened(AccountOpened $event): void
    {
        $this->connection->insert('account_balances', [
            'account_id' => $event->aggregateId(),
            'balance' => $event->initialDeposit()->amount(),
            'currency' => $event->initialDeposit()->currency(),
        ]);
    }

    private function onMoneyDeposited(MoneyDeposited $event): void
    {
        $this->connection->executeStatement(
            'UPDATE account_balances SET balance = balance + ? WHERE account_id = ?',
            [$event->amount()->amount(), $event->aggregateId()]
        );
    }
}

// Rebuild projection from scratch
final readonly class ProjectionRebuilder
{
    public function rebuild(string $projectionName): void
    {
        $projection = $this->projectionFactory->create($projectionName);
        $projection->reset(); // Clear existing data

        $events = $this->eventStore->loadAll();
        foreach ($events as $event) {
            $projection->handle($event);
        }
    }
}
```

### When to Apply

* **Audit requirements** - Need complete history of all changes
* **Complex business domains** - Multiple projections needed for different use cases
* **Temporal queries** - "What was the state at time X?"
* **Event-driven integrations** - Events can be published to other systems
* **Debugging** - Can replay events to understand what happened

### When NOT to Apply

* Simple CRUD applications without audit requirements
* When eventual consistency is not acceptable
* Limited team experience with the pattern
* When storage costs for all events are prohibitive
* Simple domains without complex state transitions

### Related Patterns

* [CQRS](cqrs.md) - Often used together; ES provides events for building read models
* [Event-driven Architecture](event_driven_architecture.md) - Domain events can be published externally
* [DDD](ddd.md) - Aggregates are natural fit for Event Sourcing

### Summary

* Complete audit trail of all changes
* Ability to rebuild state at any point in time
* Natural fit for CQRS and event-driven systems
* Increased complexity compared to traditional CRUD
* Requires careful handling of event schema evolution

### Read
* [Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html)
* [Event Sourcing pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing)
* [Event Sourcing in PHP](https://www.youtube.com/watch?v=kpM5gCLF1Zc)
* [Prooph Event Store](https://github.com/prooph/event-store)
* [EventSauce](https://eventsauce.io/)