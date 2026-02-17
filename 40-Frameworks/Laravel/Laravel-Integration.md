---
created: 2026-02-17
tags:
  - laravel
  - php
  - eloquent
  - integration
aliases:
  - Laravel Integration
  - Laravel Eloquent Auth
related:
  - Laravel-Cheatsheet
  - PHP-MOC
---

# Laravel — Detailed Integration

> [!SUMMARY] Обзор
> Laravel интеграции: Eloquent ORM, Auth (Sanctum, Passport), Queues, Events.

---

## 📦 Eloquent ORM

### Models

```php
// app/Models/User.php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Notifications\Notifiable;
use Illuminate\Foundation\Auth\User as Authenticatable;

class User extends Authenticatable
{
    use Notifiable, SoftDeletes;

    protected $table = 'users';
    
    protected $fillable = [
        'name',
        'email',
        'password',
        'role',
    ];

    protected $hidden = [
        'password',
        'remember_token',
    ];

    protected $casts = [
        'email_verified_at' => 'datetime',
        'password' => 'hashed',
        'is_active' => 'boolean',
    ];

    // Relationships
    public function posts(): HasMany
    {
        return $this->hasMany(Post::class);
    }

    public function roles(): BelongsToMany
    {
        return $this->belongsToMany(Role::class);
    }

    // Scopes
    public function scopeActive($query)
    {
        return $query->where('is_active', true);
    }

    public function scopeAdmin($query)
    {
        return $query->where('role', 'admin');
    }

    // Accessors & Mutators
    public function getFullNameAttribute(): string
    {
        return "{$this->first_name} {$this->last_name}";
    }

    public function setPasswordAttribute(string $value): void
    {
        $this->attributes['password'] = bcrypt($value);
    }
}
```

### Queries

```php
// Basic
$users = User::all();
$users = User::where('active', true)->get();
$user = User::find(1);
$user = User::findOrFail(1);

// With relationships
$users = User::with('posts')->get();
$users = User::with(['posts', 'roles'])->get();

// Eager loading with constraints
$users = User::with(['posts' => function ($query) {
    $query->where('published', true);
}])->get();

// Scopes
$activeUsers = User::active()->get();
$admins = User::admin()->get();

// Chunking
User::chunk(200, function ($users) {
    foreach ($users as $user) {
        // Process
    }
});

// Cursor for large datasets
foreach (User::cursor() as $user) {
    // Process one at a time
}

// Aggregations
$count = User::count();
$avg = User::avg('age');
$max = User::max('age');

// Raw expressions
$users = User::selectRaw('COUNT(*) as count, role')
    ->groupBy('role')
    ->get();
```

---

## 🔐 Authentication

### Laravel Sanctum (API Tokens)

```php
// Install
composer require laravel/sanctum
php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"
php artisan migrate

// app/Models/User.php
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens;
}

// Generate token
$user = User::find(1);
$token = $user->createToken('api-token')->plainTextToken;

// With abilities
$token = $user->createToken('api-token', ['posts:create', 'posts:update']);

// Revoke token
$user->currentAccessToken()->delete();
$user->tokens()->delete();  // Revoke all

// Middleware
Route::middleware(['auth:sanctum'])->group(function () {
    Route::get('/user', [UserController::class, 'me']);
});

// Check ability
if ($user->tokenCan('posts:create')) {
    // ...
}
```

### Laravel Passport (OAuth2)

```php
// Install
composer require laravel/passport
php artisan migrate
php artisan passport:install

// app/Models/User.php
use Laravel\Passport\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens;
}

// auth/config.php
'guards' => [
    'api' => [
        'driver' => 'passport',
        'provider' => 'users',
    ],
],

// Generate token
$user = User::find(1);
$token = $user->createToken('MyApp')->accessToken;

// OAuth2 flows
// POST /oauth/token
// grant_type: password/client_credentials/authorization_code
```

---

## 📬 Queues & Jobs

```php
// Create job
php artisan make:job SendWelcomeEmail

// app/Jobs/SendWelcomeEmail.php
class SendWelcomeEmail implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(
        public User $user,
    ) {}

    public function handle(): void
    {
        Mail::to($this->user->email)
            ->send(new WelcomeMail($this->user));
    }

    public function failed(Throwable $exception): void
    {
        Log::error('Failed to send welcome email', [
            'user_id' => $this->user->id,
            'error' => $exception->getMessage(),
        ]);
    }
}

// Dispatch job
SendWelcomeEmail::dispatch($user);

// Chain jobs
SendWelcomeEmail::withChain([
    new SendNotification($user),
    new LogUserCreation($user),
])->dispatch($user);

// Delay
SendWelcomeEmail::dispatch($user)->delay(now()->addMinutes(5));

// On specific queue
SendWelcomeEmail::dispatch($user)->onQueue('emails');

// Batch jobs
use Illuminate\Bus\Batch;
use Illuminate\Support\Facades\Bus;

$batch = Bus::batch([
    new ProcessUsers(1, 1000),
    new ProcessUsers(1001, 2000),
    new ProcessUsers(2001, 3000),
])->then(function (Batch $batch) {
    // All jobs completed
})->catch(function (Batch $batch, Throwable $e) {
    // First job failed
})->finally(function (Batch $batch) {
    // Batch finished
})->dispatch();
```

---

## 📡 Events & Listeners

```php
// Create event
php artisan make:event UserRegistered

// app/Events/UserRegistered.php
class UserRegistered
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public function __construct(
        public User $user,
    ) {}
}

// Create listener
php artisan make:listener SendWelcomeEmailListener

// app/Listeners/SendWelcomeEmailListener.php
class SendWelcomeEmailListener
{
    public function handle(UserRegistered $event): void
    {
        Mail::to($event->user->email)
            ->send(new WelcomeMail($event->user));
    }
}

// Register in EventServiceProvider
protected $listen = [
    UserRegistered::class => [
        SendWelcomeEmailListener::class,
        LogUserRegistration::class,
    ],
];

// Dispatch event
event(new UserRegistered($user));

// Or on model
// User.php
protected $dispatchesEvents = [
    'created' => UserRegistered::class,
];
```

---

## 📧 Mail

```php
// Create mailable
php artisan make:mail WelcomeMail

// app/Mail/WelcomeMail.php
class WelcomeMail extends Mailable
{
    use Queueable, SerializesModels;

    public function __construct(
        public User $user,
    ) {}

    public function build(): self
    {
        return $this
            ->subject('Welcome!')
            ->view('emails.welcome')
            ->with(['user' => $this->user]);
    }
}

// Send
Mail::to($user->email)->send(new WelcomeMail($user));

// Queue
Mail::to($user->email)->queue(new WelcomeMail($user));

// Markdown
class WelcomeMail extends Mailable
{
    public function build(): self
    {
        return $this->markdown('emails.welcome');
    }
}

// resources/views/emails/welcome.blade.php
@component('mail::message')
# Welcome, {{ $user->name }}!

Thanks for joining us.

@component('mail::button', ['url' => '/dashboard'])
Get Started
@endcomponent

@endcomponent
```

---

## 🔗 Связанные заметки

- [[Laravel-Cheatsheet]] — Laravel overview
- [[PHP-MOC]] — PHP languages
- [[Backend-TypeScript-Cheatsheet]] — Backend patterns
