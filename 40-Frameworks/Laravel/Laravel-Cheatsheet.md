---
created: 2026-02-16
tags:
  - cheat-sheet
  - laravel
  - php
  - backend
aliases:
  - Laravel Cheatsheet
  - Laravel Reference
related:
  - PHP-Cheatsheet
  - Database-Design
  - Auth-Patterns
---

# Laravel — Полная шпаргалка

> [!SUMMARY] Обзор
> Laravel — PHP фреймворк с выразительным синтаксисом. Eloquent ORM, миграции, очереди, события, задачи. «Фреймворк для веб-артисанов».

---

## 📚 Теория

### Request Lifecycle

```
┌─────────────────────────────────────────────────────┐
│                   Public / index.php                 │
└─────────────────────┬───────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│              HTTP Kernel (App\Http\Kernel)           │
└─────────────────────┬───────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│              Service Providers Boot                  │
└─────────────────────┬───────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│                  Request → Router                    │
└─────────────────────┬───────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│              Middleware → Controller                 │
└─────────────────────┬───────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│              Response → Middleware                   │
└─────────────────────┬───────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│                   Browser                            │
└─────────────────────────────────────────────────────┘
```

### Service Container

```php
// Автоматическое разрешение зависимостей
class UserController extends Controller {
    public function __construct(
        private UserRepository $users,
        private MailService $mail
    ) {}
}

// Binding
$this->app->bind(Repository::class, EloquentRepository::class);
$this->app->singleton(Database::class, fn($app) => new Database);
```

---

## ⚡ Быстрый старт

```bash
# Установка через Composer
composer create-project laravel/laravel project-name
cd project-name

# Laravel Installer
composer global require laravel/installer
laravel new project-name

# Запуск
php artisan serve              # http://localhost:8000
php artisan serve --host=0.0.0.0 --port=8000

# Artisan команды
php artisan list               # Все команды
php artisan make:controller UserController
php artisan make:model User -mfs  # Модель + migration + factory + seeder
php artisan make:middleware CheckAge
php artisan make:mail OrderShipped
php artisan make:notification InvoicePaid
php artisan make:job ProcessPodcast
php artisan make:event PodcastProcessed
php artisan make:listener ProcessPodcastListener
php artisan make:rule Uppercase
php artisan make:cast CustomCast

# Миграции
php artisan migrate
php artisan migrate:status
php artisan migrate:rollback
php artisan migrate:refresh
php artisan migrate:fresh --seed

# Tinker (REPL)
php artisan tinker
>>> User::find(1)
>>> factory(User::class, 10)->create()

# Кэширование
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan optimize

# Тесты
php artisan test
php artisan test --filter UserTest
```

---

## 🔧 Практические примеры

### Routes

```php
// routes/web.php
use App\Http\Controllers\UserController;
use Illuminate\Support\Facades\Route;

// Basic
Route::get('/', fn() => view('welcome'));
Route::get('/users', [UserController::class, 'index']);

// Resource routes
Route::resource('users', UserController::class);
// GET    /users              → index
// GET    /users/create       → create
// POST   /users              → store
// GET    /users/{user}       → show
// GET    /users/{user}/edit  → edit
// PUT    /users/{user}       → update
// DELETE /users/{user}       → destroy

// Route parameters
Route::get('/users/{user}', fn($id) => "User $id");
Route::get('/users/{user}/posts/{post}', fn($userId, $postId) => "...");

// Optional parameters
Route::get('/user/{name?}', fn($name = null) => "User $name");

// Route groups
Route::middleware(['auth'])->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index']);
});

Route::prefix('admin')->group(function () {
    Route::get('/users', fn() => 'Admin users');
});

// Named routes
Route::get('/profile', fn() => view('profile'))
    ->name('profile');

redirect()->route('profile');
url()->route('profile');

// Route model binding
Route::get('/users/{user}', fn(User $user) => $user);
// Автоматически делает User::find($user)

// Custom binding
Route::bind('user', fn($value) => User::where('slug', $value)->firstOrFail());
```

### Controllers

```php
// app/Http/Controllers/UserController.php
namespace App\Http\Controllers;

use App\Models\User;
use App\Http\Requests\StoreUserRequest;
use Illuminate\Http\Request;

class UserController extends Controller {
    // Конструктор с DI
    public function __construct(
        private UserRepository $users
    ) {
        $this->middleware('auth')->except(['index', 'show']);
    }

    // Index
    public function index() {
        $users = User::paginate(15);
        return view('users.index', compact('users'));
    }

    // Show
    public function show(User $user) {
        return view('users.show', compact('user'));
    }

    // Store с Form Request
    public function store(StoreUserRequest $request) {
        $validated = $request->validated();
        $user = User::create($validated);
        return redirect()->route('users.show', $user);
    }

    // Update
    public function update(StoreUserRequest $request, User $user) {
        $user->update($request->validated());
        return redirect()->route('users.show', $user);
    }

    // Delete
    public function destroy(User $user) {
        $user->delete();
        return redirect()->route('users.index');
    }

    // API Response
    public function apiIndex() {
        return response()->json([
            'data' => User::all(),
            'meta' => ['total' => User::count()]
        ]);
    }
}
```

### Eloquent ORM

```php
// app/Models/User.php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;

class User extends Model {
    // Таблица (по умолчанию plural имени модели)
    protected $table = 'users';
    
    // Primary key
    protected $primaryKey = 'id';
    public $incrementing = true;
    protected $keyType = 'int';
    
    // Timestamps
    public $timestamps = true;
    const CREATED_AT = 'created_at';
    const UPDATED_AT = 'updated_at';
    
    // Fillable / Guarded
    protected $fillable = ['name', 'email', 'password'];
    // или
    protected $guarded = ['id'];
    
    // Casts
    protected $casts = [
        'email_verified_at' => 'datetime',
        'is_active' => 'boolean',
        'settings' => 'array',
        'balance' => 'decimal:2',
    ];
    
    // Appends
    protected $appends = ['full_name'];
    
    // Accessor
    public function getFullNameAttribute(): string {
        return "{$this->first_name} {$this->last_name}";
    }
    
    // Mutator
    public function setPasswordAttribute(string $value): void {
        $this->attributes['password'] = bcrypt($value);
    }
    
    // Relationships
    public function posts(): HasMany {
        return $this->hasMany(Post::class);
    }
    
    public function profile(): HasOne {
        return $this->hasOne(Profile::class);
    }
    
    public function role(): BelongsTo {
        return $this->belongsTo(Role::class);
    }
    
    public function roles(): BelongsToMany {
        return $this->belongsToMany(Role::class)->withTimestamps();
    }
    
    // Scopes
    public function scopeActive($query) {
        return $query->where('is_active', true);
    }
    
    public function scopeAdmin($query) {
        return $query->where('role', 'admin');
    }
    
    // Scopes с параметрами
    public function scopeOfType($query, string $type) {
        return $query->where('type', $type);
    }
}

// Использование
$users = User::active()->admin()->get();
$managers = User::ofType('manager')->get();
```

### Query Builder

```php
// Выборка
DB::table('users')->get();
DB::table('users')->where('active', 1)->get();
DB::table('users')->find($id);
DB::table('users')->value('email');
DB::table('users')->pluck('email');
DB::table('users')->select('name', 'email')->get();

// Where
->where('age', '>=', 18)
->whereBetween('age', [18, 65])
->whereIn('status', ['active', 'pending'])
->whereNull('deleted_at')
->whereExists(fn($q) => $q->select('*')->from('posts')->whereColumn('posts.user_id', 'users.id'))

// Join
DB::table('users')
    ->join('posts', 'users.id', '=', 'posts.user_id')
    ->select('users.*', 'posts.title')
    ->get();

// Aggregates
DB::table('users')->count();
DB::table('orders')->sum('price');
DB::table('orders')->avg('price');
DB::table('users')->max('age');

// Chunking
DB::table('users')->chunk(100, function($users) {
    foreach ($users as $user) {
        // ...
    }
});

// Transactions
DB::transaction(function () {
    DB::table('users')->update(['votes' => DB::raw('votes + 1')]);
    DB::table('posts')->delete();
});

// Manual transaction
DB::beginTransaction();
try {
    // ...
    DB::commit();
} catch (\Exception $e) {
    DB::rollBack();
    throw $e;
}
```

### Migrations

```php
// Создание миграции
php artisan make:migration create_users_table

// app/Database/Migrations/xxxx_create_users_table.php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void {
        Schema::create('users', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('email')->unique();
            $table->timestamp('email_verified_at')->nullable();
            $table->string('password');
            $table->boolean('is_active')->default(false);
            $table->json('settings')->nullable();
            $table->rememberToken();
            $table->timestamps();
            $table->softDeletes();  // deleted_at колонка
            
            // Индексы
            $table->index('email');
            $table->index(['status', 'created_at']);
            
            // Foreign keys
            $table->foreignId('role_id')->constrained()->onDelete('cascade');
        });
    }

    public function down(): void {
        Schema::dropIfExists('users');
    }
};

// Изменение таблицы
Schema::table('users', function (Blueprint $table) {
    $table->string('phone')->after('email');
    $table->dropColumn('old_column');
    $table->renameColumn('name', 'full_name');
});

// Индексы
$table->primary('id');
$table->unique('email');
$table->index('status');
$table->fullText('description');
$table->dropIndex('users_status_index');
```

### Relationships

```php
// One to Many
class User extends Model {
    public function posts() {
        return $this->hasMany(Post::class);
    }
}

class Post extends Model {
    public function user() {
        return $this->belongsTo(User::class);
    }
}

// Использование
$user = User::find(1);
$posts = $user->posts;  // Collection
$posts = $user->posts()->where('published', true)->get();

$post = Post::find(1);
$user = $post->user;

// One to One
class User extends Model {
    public function profile() {
        return $this->hasOne(Profile::class);
    }
}

$profile = $user->profile;

// Many to Many
class User extends Model {
    public function roles() {
        return $this->belongsToMany(Role::class)
            ->withTimestamps()
            ->withPivot('created_by');
    }
}

$user->roles()->attach($roleId, ['created_by' => auth()->id()]);
$user->roles()->detach($roleId);
$user->roles()->sync([1, 2, 3]);
$user->roles()->toggle([1, 2, 3]);

// Has Many Through
class Country extends Model {
    public function posts() {
        return $this->hasManyThrough(Post::class, User::class);
    }
}

// Polymorphic
class Post extends Model {
    public function comments() {
        return $this->morphMany(Comment::class, 'commentable');
    }
}

class Video extends Model {
    public function comments() {
        return $this->morphMany(Comment::class, 'commentable');
    }
}

class Comment extends Model {
    public function commentable() {
        return $this->morphTo();
    }
}

// Eager Loading (N+1 проблема)
$users = User::with('posts', 'profile')->get();  // ✅
$users = User::all();  // ❌ N+1 при доступе к $user->posts

// С условиями
$users = User::with(['posts' => fn($q) => $q->where('published', true)])->get();

// Lazy Eager Loading
$users->load('posts');
```

### Validation

```php
// В контроллере
$request->validate([
    'name' => 'required|string|max:255',
    'email' => 'required|email|unique:users,email',
    'password' => 'required|min:8|confirmed',
    'age' => 'nullable|integer|between:18,100',
    'website' => 'url',
    'avatar' => 'image|mimes:jpeg,png|max:2048',
    'tags' => 'array',
    'tags.*' => 'string|distinct',
]);

// Form Request
// app/Http/Requests/StoreUserRequest.php
class StoreUserRequest extends FormRequest {
    public function authorize(): bool {
        return true;  // Или проверка прав
    }
    
    public function rules(): array {
        return [
            'name' => ['required', 'string', 'max:255'],
            'email' => ['required', 'email', 'unique:users'],
            'password' => ['required', 'min:8', 'confirmed'],
        ];
    }
    
    public function messages(): array {
        return [
            'name.required' => 'Имя обязательно',
            'email.unique' => 'Email уже занят',
        ];
    }
}

// Использование в контроллере
public function store(StoreUserRequest $request) {
    $validated = $request->validated();
    // $validated содержит только валидные данные
}

// Manual validation
$validator = Validator::make($data, [
    'email' => 'required|email',
]);

if ($validator->fails()) {
    return redirect()->back()
        ->withErrors($validator)
        ->withInput();
}

// Custom rules
class Uppercase implements Rule {
    public function passes($attribute, $value) {
        return strtoupper($value) === $value;
    }
    
    public function message() {
        return 'The :attribute must be uppercase.';
    }
}

// Использование
'email' => ['required', new Uppercase]
```

### Authentication

```php
// Breeze (простая аутентификация)
composer require laravel/breeze --dev
php artisan breeze:install
npm install && npm run dev

// Jetstream (полноценная с командами)
composer require laravel/jetstream
php artisan jetstream:install livewire  // или inertia

// Ручная реализация
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Hash;

// Login
if (Auth::attempt(['email' => $email, 'password' => $password])) {
    $request->session()->regenerate();
    return redirect()->intended('/dashboard');
}

// Logout
Auth::logout();
$request->session()->invalidate();
$request->session()->regenerateToken();

// Проверка
if (Auth::check()) { /* авторизован */ }
if (Auth::guest()) { /* гость */ }

// Текущий пользователь
$user = Auth::user();
$id = Auth::id();

// API Token (Sanctum)
use Laravel\Sanctum\Sanctum;

$token = $user->createToken('token-name')->plainTextToken;
// Заголовок: Authorization: Bearer {token}

$user->tokens()->delete();  // Отозвать все токены
```

### Middleware

```php
// Создание
php artisan make:middleware CheckAge

// app/Http/Middleware/CheckAge.php
class CheckAge {
    public function handle(Request $request, Closure $next) {
        if ($request->age <= 18) {
            return redirect('home');
        }
        return $next($request);
    }
}

// Регистрация (bootstrap/app.php или app/Http/Kernel.php)
// Global
$kernel->globalMiddleware([\App\Http\Middleware\CheckAge::class]);

// Route groups
$kernel->middlewareGroups([
    'web' => [...],
    'api' => [...],
]);

// Single routes
$kernel->routeMiddleware([
    'age' => \App\Http\Middleware\CheckAge::class,
    'auth' => \App\Http\Middleware\Authenticate::class,
]);

// Использование
Route::get('/adult', fn() => '...')->middleware('age');

Route::middleware(['auth', 'age'])->group(function () {
    // ...
});
```

---

## 🎯 Best Practices

### ✅ Делать

```php
// 1. Используйте Eloquent
User::find(1);  // ✅
DB::table('users')->where('id', 1)->first();  // ❌ только для сложных запросов

// 2. Массовое присваивание
User::create($request->validated());  // ✅

// 3. Eager loading
$users = User::with('posts')->get();  // ✅

// 4. Scopes
User::active()->get();  // ✅

// 5. Resources для API
php artisan make:resource UserResource

// app/Http/Resources/UserResource.php
class UserResource extends JsonResource {
    public function toArray($request) {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->when($request->user()->isAdmin(), $this->email),
        ];
    }
}

// 6. Jobs для долгих задач
ProcessEmail::dispatch($user);

// 7. Events/Listeners
event(new UserRegistered($user));
```

### ❌ Не делать

```php
// 1. N+1 запросы
foreach (User::all() as $user) {
    echo $user->posts->count();  // ❌ N+1
}

foreach (User::with('posts')->get() as $user) {  // ✅
    echo $user->posts->count();
}

// 2. Логика в контроллерах
public function store(Request $request) {
    // ❌ 50 строк логики
}

public function store(StoreUserRequest $request) {
    $user = $this->userService->create($request->validated());  // ✅
}

// 3. Жёстко закодированные значения
if ($user->role === 'admin') {  // ❌
if ($user->isAdmin()) {  // ✅
}

// 4. Игнорирование транзакций
DB::table('users')->update(...);
DB::table('posts')->delete();  // ❌

DB::transaction(fn() => {  // ✅
    DB::table('users')->update(...);
    DB::table('posts')->delete();
});
```

---

## 🔗 Связанные заметки

- [[PHP-Cheatsheet]] — PHP основы
- [[Database-Design]] — Проектирование БД
- [[Auth-Patterns]] — Паттерны аутентификации

---

## 📝 Заметки

> [!TIP] Совет от 15 лет опыта
> 
> 1. **Eloquent > Query Builder** — проще и выразительнее
> 2. **Form Requests** — валидация вне контроллеров
> 3. **Eager Loading** — всегда проверяйте N+1
> 4. **Jobs/Queues** — для долгих операций
> 5. **Artisan** — учите常用 команды

> [!INFO] Полезные пакеты
> ```bash
> # Debug
> composer require barryvdh/laravel-debugbar --dev
> composer require spatie/laravel-ignition --dev
> 
> # IDE Helper
> composer require barryvdh/laravel-ide-helper --dev
> 
> # Security
> composer require laravel/sanctum
> composer require spatie/laravel-permission
> 
> # Utils
> composer require spatie/laravel-medialibrary
> composer require intervention/image
> composer require maatwebsite/excel
> ```
