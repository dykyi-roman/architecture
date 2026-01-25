# [Architecture](README.md)

## Security Architecture

Security must be built into the architecture from the start, not bolted on later. This document covers authentication, authorization, secure coding practices, and common vulnerabilities.

## Authentication

### Authentication Methods

| Method | Use Case | Security Level |
|--------|----------|----------------|
| Session-based | Traditional web apps | Medium |
| JWT | SPAs, Mobile apps, Microservices | Medium-High |
| OAuth 2.0 | Third-party integration | High |
| API Keys | Server-to-server | Low-Medium |
| mTLS | Service mesh, high security | Very High |

### JWT Best Practices

```php
final readonly class JwtService
{
    private const ALGORITHM = 'RS256'; // Use asymmetric for microservices

    public function __construct(
        private string $privateKey,
        private string $publicKey,
        private int $accessTokenTtl = 900,    // 15 minutes
        private int $refreshTokenTtl = 604800, // 7 days
    ) {
    }

    public function createAccessToken(User $user): string
    {
        $now = new \DateTimeImmutable();

        return JWT::encode([
            'iss' => 'auth-service',           // Issuer
            'sub' => $user->id()->toString(),  // Subject
            'aud' => 'api',                    // Audience
            'iat' => $now->getTimestamp(),     // Issued at
            'exp' => $now->getTimestamp() + $this->accessTokenTtl, // Expiration
            'jti' => Uuid::v4()->toString(),   // JWT ID (for revocation)
            'roles' => $user->roles(),
        ], $this->privateKey, self::ALGORITHM);
    }

    public function verify(string $token): array
    {
        try {
            $payload = JWT::decode($token, new Key($this->publicKey, self::ALGORITHM));

            // Additional checks
            if ($payload->aud !== 'api') {
                throw new InvalidTokenException('Invalid audience');
            }

            // Check if token is revoked
            if ($this->tokenRepository->isRevoked($payload->jti)) {
                throw new TokenRevokedException();
            }

            return (array) $payload;
        } catch (ExpiredException $e) {
            throw new TokenExpiredException();
        }
    }
}
```

**JWT Security Rules:**
* Use short expiration for access tokens (15 min)
* Store refresh tokens securely (httpOnly cookies or encrypted storage)
* Implement token revocation for logout/password change
* Use asymmetric keys (RS256) for distributed systems
* Never store sensitive data in JWT payload

### OAuth 2.0 Flows

**Authorization Code Flow** (Web apps with backend)
```
User → App → Authorization Server → User Login → Auth Code → App
App (backend) → Token Endpoint (with code + secret) → Tokens
```

**PKCE Flow** (SPAs, Mobile apps)
```
App generates: code_verifier + code_challenge
User → App → Auth Server (with code_challenge) → Auth Code → App
App → Token Endpoint (with code + code_verifier) → Tokens
```

```php
final readonly class PkceService
{
    public function generateVerifier(): string
    {
        return rtrim(strtr(base64_encode(random_bytes(32)), '+/', '-_'), '=');
    }

    public function generateChallenge(string $verifier): string
    {
        return rtrim(strtr(base64_encode(hash('sha256', $verifier, true)), '+/', '-_'), '=');
    }
}
```

## Authorization

### Role-Based Access Control (RBAC)

```php
enum Permission: string
{
    case VIEW_ORDERS = 'orders:view';
    case CREATE_ORDERS = 'orders:create';
    case CANCEL_ORDERS = 'orders:cancel';
    case MANAGE_USERS = 'users:manage';
}

enum Role: string
{
    case ADMIN = 'admin';
    case MANAGER = 'manager';
    case USER = 'user';

    /** @return Permission[] */
    public function permissions(): array
    {
        return match ($this) {
            self::ADMIN => Permission::cases(),
            self::MANAGER => [
                Permission::VIEW_ORDERS,
                Permission::CREATE_ORDERS,
                Permission::CANCEL_ORDERS,
            ],
            self::USER => [
                Permission::VIEW_ORDERS,
                Permission::CREATE_ORDERS,
            ],
        };
    }
}

#[Attribute(Attribute::TARGET_METHOD)]
final readonly class RequiresPermission
{
    public function __construct(
        public Permission $permission,
    ) {
    }
}

// Usage
final readonly class OrderController
{
    #[RequiresPermission(Permission::CANCEL_ORDERS)]
    public function cancel(OrderId $id): Response
    {
        // ...
    }
}
```

### Attribute-Based Access Control (ABAC)

For complex authorization rules based on attributes.

```php
interface PolicyInterface
{
    public function allows(User $user, string $action, object $resource): bool;
}

final readonly class OrderPolicy implements PolicyInterface
{
    public function allows(User $user, string $action, object $resource): bool
    {
        assert($resource instanceof Order);

        return match ($action) {
            'view' => $this->canView($user, $resource),
            'cancel' => $this->canCancel($user, $resource),
            default => false,
        };
    }

    private function canView(User $user, Order $order): bool
    {
        // Admin can view all
        if ($user->hasRole(Role::ADMIN)) {
            return true;
        }

        // Users can only view their own orders
        return $order->customerId()->equals($user->id());
    }

    private function canCancel(User $user, Order $order): bool
    {
        // Only owner can cancel, and only within 24 hours
        if (!$order->customerId()->equals($user->id())) {
            return false;
        }

        $hoursSinceCreation = $order->createdAt()->diff(new \DateTime())->h;
        return $hoursSinceCreation < 24;
    }
}
```

## OWASP Top 10

### 1. Injection (SQL, Command, LDAP)

```php
// Bad: SQL Injection vulnerable
$query = "SELECT * FROM users WHERE email = '$email'";

// Good: Parameterized query
$stmt = $pdo->prepare('SELECT * FROM users WHERE email = :email');
$stmt->execute(['email' => $email]);

// Good: Using ORM
$user = $this->repository->findOneBy(['email' => $email]);
```

### 2. Broken Authentication

```php
final readonly class AuthService
{
    private const MAX_FAILED_ATTEMPTS = 5;
    private const LOCKOUT_DURATION = 900; // 15 minutes

    public function authenticate(string $email, string $password): User
    {
        // Check rate limiting
        if ($this->isLockedOut($email)) {
            throw new AccountLockedException('Too many failed attempts');
        }

        $user = $this->userRepository->findByEmail($email);

        if ($user === null || !$this->verifyPassword($password, $user->passwordHash())) {
            $this->recordFailedAttempt($email);
            throw new InvalidCredentialsException();
        }

        $this->clearFailedAttempts($email);
        return $user;
    }

    private function verifyPassword(string $password, string $hash): bool
    {
        // Use timing-safe comparison
        return password_verify($password, $hash);
    }
}
```

### 3. Cross-Site Scripting (XSS)

```php
// Bad: Direct output
echo "<p>Welcome, $username</p>";

// Good: Escape output
echo "<p>Welcome, " . htmlspecialchars($username, ENT_QUOTES, 'UTF-8') . "</p>";

// Good: Use templating engine with auto-escaping (Twig, Blade)
{{ username }} {# Auto-escaped #}
{{ username|raw }} {# Only when you KNOW it's safe #}

// Content Security Policy header
header("Content-Security-Policy: default-src 'self'; script-src 'self'");
```

### 4. Insecure Direct Object References (IDOR)

```php
// Bad: No authorization check
public function getOrder(int $orderId): Order
{
    return $this->orderRepository->findById($orderId);
}

// Good: Check ownership
public function getOrder(int $orderId, User $currentUser): Order
{
    $order = $this->orderRepository->findById($orderId);

    if ($order === null) {
        throw new OrderNotFoundException();
    }

    if (!$order->customerId()->equals($currentUser->id()) && !$currentUser->isAdmin()) {
        throw new AccessDeniedException();
    }

    return $order;
}
```

### 5. Security Misconfiguration

```php
// Production settings
return [
    'debug' => false,
    'display_errors' => false,
    'log_errors' => true,
    'expose_php' => false,

    // Secure session settings
    'session' => [
        'cookie_httponly' => true,
        'cookie_secure' => true,        // HTTPS only
        'cookie_samesite' => 'Strict',
        'use_strict_mode' => true,
    ],

    // Security headers
    'headers' => [
        'X-Content-Type-Options' => 'nosniff',
        'X-Frame-Options' => 'DENY',
        'X-XSS-Protection' => '1; mode=block',
        'Strict-Transport-Security' => 'max-age=31536000; includeSubDomains',
    ],
];
```

### 6. Sensitive Data Exposure

```php
// Encrypt sensitive data at rest
final readonly class EncryptionService
{
    private const CIPHER = 'aes-256-gcm';

    public function encrypt(string $data): string
    {
        $iv = random_bytes(openssl_cipher_iv_length(self::CIPHER));
        $tag = '';

        $encrypted = openssl_encrypt(
            $data,
            self::CIPHER,
            $this->key,
            OPENSSL_RAW_DATA,
            $iv,
            $tag
        );

        return base64_encode($iv . $tag . $encrypted);
    }
}

// Never log sensitive data
$this->logger->info('Payment processed', [
    'order_id' => $order->id(),
    // 'card_number' => $card->number(), // NEVER!
    'card_last_four' => $card->lastFour(),
]);
```

### 7. Cross-Site Request Forgery (CSRF)

```php
final readonly class CsrfProtection
{
    public function generateToken(): string
    {
        $token = bin2hex(random_bytes(32));
        $_SESSION['csrf_token'] = $token;
        return $token;
    }

    public function validateToken(string $token): void
    {
        if (!hash_equals($_SESSION['csrf_token'] ?? '', $token)) {
            throw new CsrfTokenMismatchException();
        }
    }
}

// For APIs: Use SameSite cookies + check Origin header
if ($_SERVER['HTTP_ORIGIN'] !== 'https://myapp.com') {
    throw new InvalidOriginException();
}
```

## Input Validation

```php
final readonly class CreateUserRequest
{
    public function __construct(
        #[Assert\NotBlank]
        #[Assert\Email(mode: Email::VALIDATION_MODE_STRICT)]
        public string $email,

        #[Assert\NotBlank]
        #[Assert\Length(min: 8, max: 128)]
        #[Assert\Regex(
            pattern: '/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/',
            message: 'Password must contain uppercase, lowercase, and number'
        )]
        public string $password,

        #[Assert\NotBlank]
        #[Assert\Length(min: 2, max: 100)]
        #[Assert\Regex(pattern: '/^[\p{L}\s\-]+$/u')]
        public string $name,
    ) {
    }
}
```

## Secrets Management

```php
// Never hardcode secrets
// Bad
$apiKey = 'sk_live_abc123';

// Good: Environment variables
$apiKey = getenv('STRIPE_API_KEY');

// Better: Secrets manager (AWS Secrets Manager, HashiCorp Vault)
$apiKey = $this->secretsManager->get('stripe/api-key');
```

## Summary

* Implement authentication with proper session/token management
* Use RBAC or ABAC for authorization
* Validate all input at system boundaries
* Protect against OWASP Top 10 vulnerabilities
* Encrypt sensitive data at rest and in transit
* Never expose sensitive data in logs or errors
* Use secrets management for credentials

### Read
* [OWASP Top 10](https://owasp.org/www-project-top-ten/)
* [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
* [OAuth 2.0 Security Best Practices](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics)
* [JWT Best Practices](https://curity.io/resources/learn/jwt-best-practices/)
* [PHP Security Best Practices](https://www.php.net/manual/en/security.php)