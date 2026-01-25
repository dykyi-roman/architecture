# [Architecture](README.md)

## Concurrency Patterns

Concurrency patterns help manage simultaneous access to shared resources, prevent data corruption, and ensure system consistency in multi-user and distributed environments.

## The Problem

When multiple processes access shared data simultaneously:

```
Process A                    Process B
─────────                    ─────────
Read balance: $100           Read balance: $100
Calculate: $100 + $50        Calculate: $100 - $30
Write: $150                  Write: $70

Expected: $120 ($100 + $50 - $30)
Actual: $70 (last write wins)
```

## Locking Strategies

### Optimistic Locking

Assumes conflicts are rare. Allows concurrent reads, validates before write.

```php
// Entity with version
final class Order
{
    public function __construct(
        private OrderId $id,
        private OrderStatus $status,
        private int $version, // Version for optimistic locking
    ) {
    }

    public function version(): int
    {
        return $this->version;
    }

    public function incrementVersion(): void
    {
        $this->version++;
    }
}

// Repository implementation
final readonly class PostgresOrderRepository implements OrderRepositoryInterface
{
    public function save(Order $order): void
    {
        $currentVersion = $order->version();
        $order->incrementVersion();

        $affectedRows = $this->connection->executeStatement(
            'UPDATE orders
             SET status = :status, version = :newVersion
             WHERE id = :id AND version = :currentVersion',
            [
                'id' => $order->id()->toString(),
                'status' => $order->status()->value,
                'newVersion' => $order->version(),
                'currentVersion' => $currentVersion,
            ]
        );

        if ($affectedRows === 0) {
            throw new OptimisticLockException(
                'Order was modified by another process'
            );
        }
    }
}

// Usage with retry
final readonly class OrderService
{
    public function updateOrder(OrderId $id, callable $modifier): void
    {
        $maxRetries = 3;
        $attempt = 0;

        while ($attempt < $maxRetries) {
            try {
                $order = $this->repository->findById($id);
                $modifier($order);
                $this->repository->save($order);
                return;
            } catch (OptimisticLockException $e) {
                $attempt++;
                if ($attempt >= $maxRetries) {
                    throw $e;
                }
                usleep(random_int(10000, 50000)); // Random backoff
            }
        }
    }
}
```

**Use when:**
* Read-heavy workloads
* Low conflict probability
* Short transactions
* Web applications

### Pessimistic Locking

Assumes conflicts are likely. Locks data before modification.

```php
final readonly class PostgresOrderRepository implements OrderRepositoryInterface
{
    public function findByIdForUpdate(OrderId $id): ?Order
    {
        // SELECT ... FOR UPDATE acquires row-level lock
        $row = $this->connection->fetchAssociative(
            'SELECT * FROM orders WHERE id = :id FOR UPDATE',
            ['id' => $id->toString()]
        );

        return $row ? $this->hydrate($row) : null;
    }

    public function findByIdWithShareLock(OrderId $id): ?Order
    {
        // SELECT ... FOR SHARE allows concurrent reads, blocks writes
        $row = $this->connection->fetchAssociative(
            'SELECT * FROM orders WHERE id = :id FOR SHARE',
            ['id' => $id->toString()]
        );

        return $row ? $this->hydrate($row) : null;
    }
}

// Usage
final readonly class OrderService
{
    public function processOrder(OrderId $id): void
    {
        $this->connection->beginTransaction();

        try {
            // Lock the row - other transactions wait
            $order = $this->repository->findByIdForUpdate($id);

            if ($order === null) {
                throw new OrderNotFoundException($id);
            }

            $order->process();
            $this->repository->save($order);

            $this->connection->commit();
        } catch (\Throwable $e) {
            $this->connection->rollBack();
            throw $e;
        }
    }
}
```

**Use when:**
* Write-heavy workloads
* High conflict probability
* Critical data integrity requirements
* Financial transactions

### Lock Types Comparison

| Aspect | Optimistic | Pessimistic |
|--------|------------|-------------|
| Conflict handling | Retry on conflict | Block until released |
| Throughput | Higher (no blocking) | Lower (waiting) |
| Complexity | Version management | Transaction management |
| Deadlock risk | None | Possible |
| Best for | Read-heavy, low conflicts | Write-heavy, high conflicts |

## Distributed Locks

### Redis-based Locking

```php
final readonly class RedisLock implements DistributedLockInterface
{
    private const LOCK_PREFIX = 'lock:';

    public function __construct(
        private \Redis $redis,
        private int $defaultTtlSeconds = 30,
    ) {
    }

    public function acquire(string $resource, ?int $ttlSeconds = null): ?LockToken
    {
        $token = bin2hex(random_bytes(16));
        $key = self::LOCK_PREFIX . $resource;
        $ttl = $ttlSeconds ?? $this->defaultTtlSeconds;

        // SET NX (only if not exists) with expiration
        $acquired = $this->redis->set(
            $key,
            $token,
            ['NX', 'EX' => $ttl]
        );

        return $acquired ? new LockToken($resource, $token, $ttl) : null;
    }

    public function release(LockToken $token): bool
    {
        $key = self::LOCK_PREFIX . $token->resource;

        // Lua script for atomic check-and-delete
        $script = <<<'LUA'
            if redis.call("get", KEYS[1]) == ARGV[1] then
                return redis.call("del", KEYS[1])
            else
                return 0
            end
        LUA;

        return (bool) $this->redis->eval($script, [$key, $token->value], 1);
    }

    public function extend(LockToken $token, int $additionalSeconds): bool
    {
        $key = self::LOCK_PREFIX . $token->resource;

        $script = <<<'LUA'
            if redis.call("get", KEYS[1]) == ARGV[1] then
                return redis.call("expire", KEYS[1], ARGV[2])
            else
                return 0
            end
        LUA;

        return (bool) $this->redis->eval(
            $script,
            [$key, $token->value, $additionalSeconds],
            1
        );
    }
}

// Usage
final readonly class PaymentService
{
    public function processPayment(OrderId $orderId): void
    {
        $lockResource = "order:{$orderId->toString()}";

        $token = $this->lock->acquire($lockResource);

        if ($token === null) {
            throw new ResourceLockedException('Order is being processed');
        }

        try {
            $order = $this->orderRepository->findById($orderId);
            $this->paymentGateway->charge($order);
            $order->markAsPaid();
            $this->orderRepository->save($order);
        } finally {
            $this->lock->release($token);
        }
    }
}
```

### Redlock Algorithm (Multi-node Redis)

For higher availability, use Redlock across multiple Redis instances.

```php
final readonly class Redlock implements DistributedLockInterface
{
    private const CLOCK_DRIFT_FACTOR = 0.01;

    /** @param \Redis[] $redisInstances */
    public function __construct(
        private array $redisInstances,
        private int $retryCount = 3,
        private int $retryDelayMs = 200,
    ) {
    }

    public function acquire(string $resource, int $ttlMs): ?LockToken
    {
        $token = bin2hex(random_bytes(16));
        $quorum = count($this->redisInstances) / 2 + 1;

        for ($attempt = 0; $attempt < $this->retryCount; $attempt++) {
            $startTime = microtime(true) * 1000;
            $acquiredCount = 0;

            foreach ($this->redisInstances as $redis) {
                if ($this->acquireOnInstance($redis, $resource, $token, $ttlMs)) {
                    $acquiredCount++;
                }
            }

            $elapsedMs = (microtime(true) * 1000) - $startTime;
            $drift = ($ttlMs * self::CLOCK_DRIFT_FACTOR) + 2;
            $validity = $ttlMs - $elapsedMs - $drift;

            if ($acquiredCount >= $quorum && $validity > 0) {
                return new LockToken($resource, $token, (int) $validity);
            }

            // Failed to acquire - release all
            $this->releaseAll($resource, $token);

            usleep($this->retryDelayMs * 1000);
        }

        return null;
    }
}
```

## Idempotency Keys

Ensure operations can be safely retried.

```php
final readonly class IdempotencyService
{
    private const TTL_SECONDS = 86400; // 24 hours

    public function __construct(
        private \Redis $redis,
        private SerializerInterface $serializer,
    ) {
    }

    public function executeOnce(
        string $idempotencyKey,
        callable $operation,
    ): mixed {
        $key = "idempotency:{$idempotencyKey}";

        // Check for existing result
        $cached = $this->redis->get($key);
        if ($cached !== false) {
            $data = json_decode($cached, true);

            if ($data['status'] === 'completed') {
                return $this->serializer->deserialize($data['result']);
            }

            if ($data['status'] === 'processing') {
                throw new OperationInProgressException();
            }
        }

        // Mark as processing
        $this->redis->setex($key, self::TTL_SECONDS, json_encode([
            'status' => 'processing',
            'started_at' => time(),
        ]));

        try {
            $result = $operation();

            // Store successful result
            $this->redis->setex($key, self::TTL_SECONDS, json_encode([
                'status' => 'completed',
                'result' => $this->serializer->serialize($result),
                'completed_at' => time(),
            ]));

            return $result;
        } catch (\Throwable $e) {
            // Remove on failure (allow retry)
            $this->redis->del($key);
            throw $e;
        }
    }
}

// Usage
$paymentResult = $this->idempotencyService->executeOnce(
    idempotencyKey: $request->headers->get('Idempotency-Key'),
    operation: fn() => $this->paymentGateway->charge($order),
);
```

## Database-level Concurrency

### Advisory Locks (PostgreSQL)

```php
final readonly class PostgresAdvisoryLock
{
    public function acquireSession(int $lockId): bool
    {
        // Session-level lock (released on disconnect)
        $result = $this->connection->fetchOne(
            'SELECT pg_advisory_lock(:id)',
            ['id' => $lockId]
        );
        return $result !== false;
    }

    public function tryAcquireSession(int $lockId): bool
    {
        // Non-blocking attempt
        return (bool) $this->connection->fetchOne(
            'SELECT pg_try_advisory_lock(:id)',
            ['id' => $lockId]
        );
    }

    public function acquireTransaction(int $lockId): bool
    {
        // Transaction-level lock (released on commit/rollback)
        return (bool) $this->connection->fetchOne(
            'SELECT pg_advisory_xact_lock(:id)',
            ['id' => $lockId]
        );
    }

    public function release(int $lockId): bool
    {
        return (bool) $this->connection->fetchOne(
            'SELECT pg_advisory_unlock(:id)',
            ['id' => $lockId]
        );
    }
}

// Usage: generate lock ID from resource identifier
$lockId = crc32("order:{$orderId->toString()}");
```

### Atomic Operations

```php
// Atomic increment
$this->connection->executeStatement(
    'UPDATE accounts SET balance = balance + :amount WHERE id = :id',
    ['id' => $accountId, 'amount' => $amount]
);

// Atomic compare-and-swap
$affected = $this->connection->executeStatement(
    'UPDATE inventory
     SET quantity = quantity - :requested
     WHERE product_id = :productId AND quantity >= :requested',
    ['productId' => $productId, 'requested' => $quantity]
);

if ($affected === 0) {
    throw new InsufficientInventoryException();
}
```

## Saga Pattern Compensation

Handle distributed transactions with compensation.

```php
final class OrderSaga
{
    private array $completedSteps = [];

    public function execute(CreateOrderCommand $command): void
    {
        try {
            // Step 1: Reserve inventory
            $this->inventoryService->reserve($command->items());
            $this->completedSteps[] = 'inventory_reserved';

            // Step 2: Process payment
            $this->paymentService->charge($command->payment());
            $this->completedSteps[] = 'payment_processed';

            // Step 3: Create order
            $this->orderService->create($command);
            $this->completedSteps[] = 'order_created';

        } catch (\Throwable $e) {
            $this->compensate($command);
            throw $e;
        }
    }

    private function compensate(CreateOrderCommand $command): void
    {
        // Reverse in opposite order
        foreach (array_reverse($this->completedSteps) as $step) {
            match ($step) {
                'order_created' => $this->orderService->cancel($command->orderId()),
                'payment_processed' => $this->paymentService->refund($command->payment()),
                'inventory_reserved' => $this->inventoryService->release($command->items()),
            };
        }
    }
}
```

## Summary

| Pattern | Use Case |
|---------|----------|
| Optimistic Locking | Read-heavy, low conflicts |
| Pessimistic Locking | Write-heavy, critical data |
| Distributed Locks | Cross-service coordination |
| Idempotency Keys | Safe API retries |
| Advisory Locks | Application-level coordination |
| Saga Compensation | Distributed transactions |

### Read
* [Optimistic vs Pessimistic Locking](https://www.baeldung.com/jpa-optimistic-locking)
* [Distributed Locks with Redis](https://redis.io/docs/manual/patterns/distributed-locks/)
* [Redlock Algorithm](https://redis.io/docs/manual/patterns/distributed-locks/#the-redlock-algorithm)
* [Designing Data-Intensive Applications - Chapter 7](https://dataintensive.net/)
* [PostgreSQL Advisory Locks](https://www.postgresql.org/docs/current/explicit-locking.html#ADVISORY-LOCKS)