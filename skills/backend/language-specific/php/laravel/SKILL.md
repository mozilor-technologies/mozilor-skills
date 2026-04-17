---
name: laravel
description: Laravel v12 - The PHP Framework For Web Artisans
---

# Laravel Skill

Comprehensive assistance with Laravel 12.x development, including routing, Eloquent ORM, migrations, authentication, API development, and modern PHP patterns.

## When to Use This Skill

- Building Laravel applications or APIs
- Working with Eloquent models, relationships, and queries
- Setting up authentication, authorization, or API tokens
- Creating database migrations, seeders, or factories
- Implementing middleware, service providers, or events
- Using Laravel's built-in features (queues, cache, validation, etc.)
- Troubleshooting Laravel errors or performance issues
- Implementing RESTful APIs with Laravel Sanctum or Passport

## Quick Reference

### Basic Routing

```php
// Basic routes
Route::get('/users', [UserController::class, 'index']);
Route::post('/users', [UserController::class, 'store']);

// Route parameters
Route::get('/users/{id}', function ($id) {
    return User::find($id);
});

// Named routes
Route::get('/profile', ProfileController::class)->name('profile');

// Route groups with middleware
Route::middleware(['auth'])->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index']);
    Route::resource('posts', PostController::class);
});
```

### Eloquent Model Basics

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Post extends Model
{
    protected $fillable = ['title', 'content', 'user_id'];

    protected $casts = [
        'published_at' => 'datetime',
    ];

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function comments(): HasMany
    {
        return $this->hasMany(Comment::class);
    }
}
```

### Database Migrations

```php
return new class extends Migration
{
    public function up(): void
    {
        Schema::create('posts', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->cascadeOnDelete();
            $table->string('title');
            $table->text('content');
            $table->timestamp('published_at')->nullable();
            $table->timestamps();

            $table->index(['user_id', 'published_at']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('posts');
    }
};
```

### Form Validation

```php
// Inline controller validation
public function store(Request $request)
{
    $validated = $request->validate([
        'title'   => 'required|max:255',
        'content' => 'required',
        'email'   => 'required|email|unique:users',
        'tags'    => 'array|min:1',
        'tags.*'  => 'string|max:50',
    ]);

    return Post::create($validated);
}

// Form Request class (preferred)
class StorePostRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'title'   => 'required|max:255',
            'content' => 'required|min:100',
        ];
    }
}
```

### Eloquent Query Builder

```php
// Eager loading — avoid N+1
$posts = Post::with(['user', 'comments'])
    ->where('published_at', '<=', now())
    ->orderBy('published_at', 'desc')
    ->paginate(15);

// Conditional queries
$query = Post::query();

if ($request->has('search')) {
    $query->where('title', 'like', "%{$request->search}%");
}

if ($request->has('author')) {
    $query->whereHas('user', fn ($q) => $q->where('name', $request->author));
}

$posts = $query->get();
```

### API Resource Controllers

```php
class PostController extends Controller
{
    public function index()
    {
        return PostResource::collection(
            Post::with('user')->latest()->paginate()
        );
    }

    public function store(StorePostRequest $request)
    {
        return new PostResource(Post::create($request->validated()));
    }

    public function show(Post $post)
    {
        return new PostResource($post->load('user', 'comments'));
    }

    public function update(UpdatePostRequest $request, Post $post)
    {
        $post->update($request->validated());
        return new PostResource($post);
    }
}
```

### API Resources (Transformers)

```php
class PostResource extends JsonResource
{
    public function toArray($request): array
    {
        return [
            'id'             => $this->id,
            'title'          => $this->title,
            'slug'           => $this->slug,
            'content'        => $this->when($request->routeIs('posts.show'), $this->content),
            'author'         => new UserResource($this->whenLoaded('user')),
            'comments_count' => $this->when($this->comments_count, $this->comments_count),
            'published_at'   => $this->published_at?->toISOString(),
            'created_at'     => $this->created_at->toISOString(),
        ];
    }
}
```

### Authentication with Sanctum

```php
// User model
class User extends Authenticatable
{
    use HasApiTokens;
}

// Login endpoint
public function login(Request $request)
{
    $credentials = $request->validate([
        'email'    => 'required|email',
        'password' => 'required',
    ]);

    if (!Auth::attempt($credentials)) {
        return response()->json(['message' => 'Invalid credentials'], 401);
    }

    $token = $request->user()->createToken('api-token')->plainTextToken;
    return response()->json(['token' => $token]);
}

// Protected routes
Route::middleware('auth:sanctum')->group(function () {
    Route::get('/user', fn (Request $r) => $r->user());
});
```

### Jobs and Queues

```php
class ProcessVideo implements ShouldQueue
{
    use InteractsWithQueue, Queueable;

    public function __construct(public Video $video) {}

    public function handle(): void
    {
        $this->video->process();
    }
}

// Dispatch
ProcessVideo::dispatch($video);
ProcessVideo::dispatch($video)->onQueue('videos')->delay(now()->addMinutes(5));
```

### Service Container and Dependency Injection

```php
// Register in AppServiceProvider
public function register(): void
{
    $this->app->singleton(PaymentService::class, function ($app) {
        return new PaymentService(config('services.stripe.secret'));
    });
}

// Inject in controllers
public function __construct(protected PaymentService $payment) {}

public function charge(Request $request)
{
    return $this->payment->charge($request->user(), $request->amount);
}
```

## Common Patterns

### Action Classes (Single Responsibility)

```php
class CreatePost
{
    public function execute(array $data): Post
    {
        return DB::transaction(function () use ($data) {
            $post = Post::create($data);
            $post->tags()->attach($data['tag_ids']);
            event(new PostCreated($post));
            return $post;
        });
    }
}
```

### Query Scopes

```php
class Post extends Model
{
    public function scopePublished($query)
    {
        return $query->where('published_at', '<=', now());
    }

    public function scopeByAuthor($query, User $user)
    {
        return $query->where('user_id', $user->id);
    }
}

// Usage
Post::published()->byAuthor($user)->get();
```

### Repository Pattern

```php
interface PostRepositoryInterface
{
    public function all();
    public function find(int $id);
    public function create(array $data);
}

class PostRepository implements PostRepositoryInterface
{
    public function all()
    {
        return Post::with('user')->latest()->get();
    }

    public function find(int $id)
    {
        return Post::with('user', 'comments')->findOrFail($id);
    }
}
```

## Artisan Commands

```bash
php artisan make:model Post -mcr   # Model + migration + controller + resource
php artisan migrate                # Run migrations
php artisan db:seed                # Seed database
php artisan queue:work             # Process queue jobs
php artisan route:list             # View all registered routes
php artisan tinker                 # Interactive REPL
php artisan optimize:clear         # Clear all caches
```

## Best Practices

1. **Use Form Requests** — separate validation logic from controllers
2. **Eager load relationships** — always use `with()` to avoid N+1 queries
3. **Use Resource Controllers** — follow RESTful conventions
4. **Wrap related DB operations in transactions** — use `DB::transaction()`
5. **Queue slow jobs** — never block the HTTP response for heavy work
6. **Use API Resources** — never return raw Eloquent models from APIs
7. **Use Action classes** — keep controllers thin, one action per class
8. **Cache expensive queries** — use `Cache::remember()` for heavy reads
9. **Write tests** — feature tests for HTTP, unit tests for business logic

## Notes

- Laravel 12.x requires **PHP 8.2+**
- Asset compilation uses **Vite** (not Laravel Mix)
- Supports MySQL, PostgreSQL, SQLite, SQL Server
- First-party packages: Sanctum (API auth), Horizon (queue monitoring), Telescope (debugging)
- Official docs: https://laravel.com/docs/12.x
