---
name: laravel-woo
description: Laravel 12.x coding standards, scalable patterns, and WooCommerce/WordPress backend development best practices.
triggers:
  - "laravel"
  - "laravel 12"
  - "php backend"
  - "woocommerce"
  - "wordpress backend"
  - "laravel patterns"
  - "php coding standards"
---

# Laravel 12.x & WooCommerce Backend — Coding Standards & Patterns

## 1) Laravel 12 Project Structure

```
app/
├── Console/Commands/           # Artisan commands
├── Exceptions/                 # Custom exception handlers
├── Http/
│   ├── Controllers/            # Thin — delegate to services
│   │   ├── Api/V1/             # Versioned API controllers
│   │   └── Web/
│   ├── Middleware/             # Request pipeline
│   ├── Requests/               # Form requests (validation)
│   └── Resources/             # API resources (response shaping)
├── Models/                     # Eloquent models
├── Repositories/               # Data access layer (optional, for complex queries)
├── Services/                   # Business logic layer
├── Events/                     # Domain events
├── Listeners/                  # Event listeners
├── Jobs/                       # Queued jobs
└── Providers/                  # Service providers
config/
database/
├── migrations/
├── seeders/
└── factories/
resources/
routes/
├── api.php
└── web.php
tests/
├── Feature/
└── Unit/
```

## 2) Coding Standards

### Laravel Pint (Official Formatter)

```bash
./vendor/bin/pint              # fix all files
./vendor/bin/pint --test       # check only (CI mode)
./vendor/bin/pint app/         # specific directory
```

Configure `pint.json` in project root:

```json
{
  "preset": "laravel",
  "rules": {
    "ordered_imports": { "sort_algorithm": "alpha" },
    "not_operator_with_successor_space": true,
    "trailing_comma_in_multiline": { "elements": ["arrays", "arguments"] }
  }
}
```

### PSR-12 + Laravel Conventions

```php
// Class: PascalCase
class UserService {}

// Methods and variables: camelCase
public function createUser(CreateUserRequest $request): UserResource {}

// Constants: UPPER_SNAKE_CASE
const MAX_LOGIN_ATTEMPTS = 5;

// Blade views: kebab-case
resources/views/user/profile-settings.blade.php

// Database tables: snake_case, plural
users, order_items, product_categories
```

## 3) Controllers — Keep Thin

Controllers should only handle HTTP. All business logic belongs in services.

```php
// Bad: fat controller
class UserController extends Controller
{
    public function store(Request $request)
    {
        $validated = $request->validate([...]);
        $user = User::create($validated);
        Mail::to($user)->send(new WelcomeEmail($user));
        return response()->json($user, 201);
    }
}

// Good: thin controller
class UserController extends Controller
{
    public function __construct(private readonly UserService $userService) {}

    public function store(CreateUserRequest $request): JsonResponse
    {
        $user = $this->userService->create($request->validated());
        return new JsonResponse(new UserResource($user), 201);
    }
}
```

## 4) Form Requests for Validation

```php
// app/Http/Requests/CreateUserRequest.php
class CreateUserRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true; // Or check policy
    }

    public function rules(): array
    {
        return [
            'name'     => ['required', 'string', 'min:2', 'max:100'],
            'email'    => ['required', 'email', 'unique:users,email'],
            'password' => ['required', 'min:8', 'confirmed'],
            'role'     => ['required', Rule::in(['admin', 'user'])],
        ];
    }

    public function messages(): array
    {
        return [
            'email.unique' => 'This email address is already registered.',
        ];
    }
}
```

## 5) Services — Business Logic Layer

```php
// app/Services/UserService.php
class UserService
{
    public function __construct(
        private readonly UserRepository $userRepository,
        private readonly Mailer $mailer,
    ) {}

    public function create(array $data): User
    {
        DB::transaction(function () use ($data, &$user) {
            $user = $this->userRepository->create($data);
            event(new UserCreated($user));
            $this->mailer->sendWelcome($user);
        });

        return $user;
    }
}
```

## 6) Eloquent Best Practices

```php
// Eager load to avoid N+1 queries
$orders = Order::with(['user', 'items.product'])->paginate(20);

// Use scopes for reusable query constraints
class User extends Model
{
    public function scopeActive(Builder $query): Builder
    {
        return $query->where('status', 'active');
    }
}

$activeUsers = User::active()->get();

// Use accessors and mutators (Laravel 9+ syntax)
class User extends Model
{
    protected function fullName(): Attribute
    {
        return Attribute::make(
            get: fn () => "{$this->first_name} {$this->last_name}",
        );
    }
}

// Always use casts for data integrity
class Order extends Model
{
    protected $casts = [
        'total'      => 'decimal:2',
        'metadata'   => 'array',
        'is_paid'    => 'boolean',
        'paid_at'    => 'datetime',
    ];
}
```

## 7) Database Migrations

```php
// Always use migrations — never edit DB manually
Schema::create('orders', function (Blueprint $table) {
    $table->ulid('id')->primary();
    $table->foreignUlid('user_id')->constrained()->cascadeOnDelete();
    $table->decimal('total', 10, 2)->default(0);
    $table->string('status')->default('pending');
    $table->json('metadata')->nullable();
    $table->timestamps();
    $table->softDeletes();

    $table->index(['user_id', 'status']);
    $table->index('created_at');
});

// Rollback support is mandatory
public function down(): void
{
    Schema::dropIfExists('orders');
}
```

## 8) SOLID Principles in Laravel

### Dependency Injection & Service Container

```php
// app/Providers/AppServiceProvider.php
public function register(): void
{
    $this->app->bind(PaymentGatewayInterface::class, function () {
        return match(config('payment.driver')) {
            'stripe' => new StripeGateway(config('payment.stripe_key')),
            'paypal' => new PayPalGateway(config('payment.paypal_client')),
        };
    });
}

// Controller receives interface, not concrete class
class PaymentController extends Controller
{
    public function __construct(
        private readonly PaymentGatewayInterface $gateway
    ) {}
}
```

## 9) API Resources for Response Shaping

```php
// app/Http/Resources/UserResource.php
class UserResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id'         => $this->id,
            'name'       => $this->name,
            'email'      => $this->email,
            'role'       => $this->role,
            'created_at' => $this->created_at->toISOString(),
            'orders'     => OrderResource::collection($this->whenLoaded('orders')),
        ];
    }
}
```

## 10) WooCommerce Backend Development

### Plugin Structure

```
my-plugin/
├── my-plugin.php              # Main plugin file (header + bootstrap)
├── includes/
│   ├── class-plugin.php       # Core plugin class
│   ├── admin/                 # Admin-specific code
│   ├── api/                   # REST API endpoints
│   ├── models/                # Data models
│   └── services/              # Business logic
├── assets/
├── templates/
└── tests/
```

### WooCommerce Hooks Architecture

```php
// Never directly modify WooCommerce core — use hooks
class MyPlugin
{
    public function __construct()
    {
        add_action('woocommerce_order_status_completed', [$this, 'onOrderComplete'], 10, 1);
        add_filter('woocommerce_product_tabs', [$this, 'addProductTab']);
        add_action('rest_api_init', [$this, 'registerRestRoutes']);
    }

    public function onOrderComplete(int $orderId): void
    {
        $order = wc_get_order($orderId);
        if (!$order instanceof \WC_Order) return;

        // Process completed order
        $this->fulfillmentService->process($order);
    }
}
```

### WooCommerce REST API Extensions

```php
class ProductEndpoint
{
    public function register(): void
    {
        register_rest_route('my-plugin/v1', '/products/(?P<id>\d+)/stock', [
            'methods'             => \WP_REST_Server::READABLE,
            'callback'            => [$this, 'getStock'],
            'permission_callback' => [$this, 'checkPermission'],
            'args'                => [
                'id' => [
                    'required'          => true,
                    'validate_callback' => fn($v) => is_numeric($v),
                    'sanitize_callback' => 'absint',
                ],
            ],
        ]);
    }

    public function checkPermission(\WP_REST_Request $request): bool
    {
        return current_user_can('manage_woocommerce');
    }
}
```

### WooCommerce Data Layer — Use CRUD APIs

```php
// BAD: direct database access
global $wpdb;
$order_total = $wpdb->get_var("SELECT total FROM orders WHERE id = {$order_id}");

// GOOD: WooCommerce CRUD
$order = wc_get_order($order_id);
$total = $order->get_total();

// GOOD: Order data manipulation
$order = wc_create_order();
$order->add_product(wc_get_product($product_id), 2);
$order->calculate_totals();
$order->save();
```

### WordPress Coding Standards

```bash
# Install PHP_CodeSniffer with WordPress standards
composer require --dev wp-coding-standards/wpcs squizlabs/php_codesniffer
./vendor/bin/phpcs --standard=WordPress src/

# Never: 
# - use $_REQUEST directly
# - output user data without escaping
# - use raw SQL without $wpdb->prepare()
```

```php
// Always escape output
echo esc_html($user_name);
echo esc_url($url);
echo esc_attr($attribute);

// Always sanitize input
$name = sanitize_text_field($_POST['name'] ?? '');
$email = sanitize_email($_POST['email'] ?? '');

// Always use nonces for forms
wp_nonce_field('my_action', 'my_nonce');
check_admin_referer('my_action', 'my_nonce');
```

## 11) Testing

```bash
# Laravel testing
php artisan test                  # run all tests
php artisan test --parallel       # parallel execution
php artisan test --coverage       # with coverage
```

```php
// Feature test
class CreateUserTest extends TestCase
{
    use RefreshDatabase;

    public function test_creates_user_and_sends_welcome_email(): void
    {
        Mail::fake();

        $response = $this->postJson('/api/v1/users', [
            'name'                  => 'Test User',
            'email'                 => 'test@example.com',
            'password'              => 'password123',
            'password_confirmation' => 'password123',
        ]);

        $response->assertCreated()
                 ->assertJsonPath('data.email', 'test@example.com');

        $this->assertDatabaseHas('users', ['email' => 'test@example.com']);
        Mail::assertQueued(WelcomeEmail::class);
    }
}
```

## 12) Quality Checklist

- [ ] Laravel Pint passes (`./vendor/bin/pint --test`)
- [ ] PHPStan or Larastan at level 8+ passes
- [ ] All tests green (`php artisan test`)
- [ ] No N+1 queries (use Laravel Debugbar in dev)
- [ ] All inputs validated via Form Requests
- [ ] All outputs escaped (WooCommerce/WordPress)
- [ ] Migrations are reversible
- [ ] No raw SQL without parameter binding

## References

- [Laravel 12 Official Docs](https://laravel.com/docs/12.x)
- [Laravel Best Practices](https://www.hamidkodez.com/blog/laravel-best-practices)
- [WordPress Coding Standards](https://developer.wordpress.org/coding-standards/)
- [WooCommerce REST API](https://woocommerce.github.io/woocommerce-rest-api-docs/)
