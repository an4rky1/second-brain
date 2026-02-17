---
created: 2026-02-16
tags:
  - cheat-sheet
  - php
  - backend
  - web
aliases:
  - PHP Cheatsheet
  - PHP Reference
related:
  - Laravel-Cheatsheet
  - Security-Hardening
  - Database-Design
---

# PHP — Полная шпаргалка

> [!SUMMARY] Обзор
> PHP — серверный язык программирования для веб-разработки. Современный PHP (8.x) имеет строгую типизацию, JIT компиляцию, атрибуты и отличную производительность. Основа Laravel, Symfony, WordPress.

---

## 📚 Теория

### Основные типы

```php
<?php

// Скалярные типы
$string = "Hello";
$int = 42;
$float = 3.14;
$bool = true;
$null = null;

// Составные типы
$array = [1, 2, 3];
$assocArray = ['key' => 'value'];
$object = new stdClass();

// Специальные типы
$callable = fn($x) => $x * 2;
$iterable = [1, 2, 3];

// Union типы (PHP 8.0+)
function process(int|string $input): int|string {
    return $input;
}

// Mixed тип (PHP 8.0+)
function handle(mixed $data): mixed {
    return $data;
}

// Nullsafe оператор (PHP 8.0+)
$country = $user?->getAddress()?->getCountry()?->getName();

// Named arguments (PHP 8.0+)
function create(array $data = [], bool $validate = true, string $format = 'json') {}
create(validate: false, format: 'xml');

// Match выражение (PHP 8.0+)
$result = match($status) {
    200, 201 => 'success',
    400, 401, 403 => 'client_error',
    500, 502, 503 => 'server_error',
    default => 'unknown',
};

// Constructor property promotion (PHP 8.0+)
class User {
    public function __construct(
        public int $id,
        public string $name,
        private string $email,
    ) {}
}

// Attributes (PHP 8.0+)
#[Route('/api/users', methods: ['GET'])]
#[Cache(ttl: 3600)]
class UserController {}

// Enums (PHP 8.1+)
enum Status: string {
    case PENDING = 'pending';
    case APPROVED = 'approved';
    case REJECTED = 'rejected';
    
    public function color(): string {
        return match($this) {
            self::PENDING => 'yellow',
            self::APPROVED => 'green',
            self::REJECTED => 'red',
        };
    }
}

// Readonly classes (PHP 8.2+)
readonly class User {
    public function __construct(
        public int $id,
        public string $name,
    ) {}
}

// Intersection типы (PHP 8.1+)
function process(Countable & Arrayable $input) {}
```

---

## ⚡ Быстрый старт

```bash
# Установка (Ubuntu)
sudo apt install php php-cli php-fpm php-mysql php-pgsql php-redis php-mbstring php-xml php-curl php-zip

# Проверка версии
php -v

# Запуск встроенного сервера
php -S localhost:8000

# Запуск скрипта
php script.php

# Composer
composer init
composer install
composer require package/name
composer update

# Xdebug (отладка)
pecl install xdebug

# PHPStan (статический анализ)
composer require --dev phpstan/phpstan
./vendor/bin/phpstan analyse src

# PHPUnit (тесты)
composer require --dev phpunit/phpunit
./vendor/bin/phpunit
```

### Структура проекта

```
project/
├── public/
│   └── index.php        # Entry point
├── src/
│   ├── Controller/
│   ├── Service/
│   ├── Repository/
│   ├── Model/
│   └── Middleware/
├── config/
├── tests/
├── migrations/
├── composer.json
└── .env
```

---

## 🔧 Практические примеры

### Работа с массивами

```php
<?php

$array = [1, 2, 3, 4, 5];

// Map
$squared = array_map(fn($n) => $n ** 2, $array);

// Filter
$even = array_filter($array, fn($n) => $n % 2 === 0);

// Reduce
$sum = array_reduce($array, fn($carry, $n) => $carry + $n, 0);

// Find
$found = current(array_filter($array, fn($n) => $n > 3));

// Some
$hasEven = count(array_filter($array, fn($n) => $n % 2 === 0)) > 0;

// Every
$allPositive = count(array_filter($array, fn($n) => $n > 0)) === count($array);

// Collection (Laravel стиль)
$collection = collect($array)
    ->map(fn($n) => $n * 2)
    ->filter(fn($n) => $n > 5)
    ->reduce(fn($carry, $n) => $carry + $n, 0);

// Array destructuring
[$first, $second, ...$rest] = $array;

// Spread operator
$combined = [...$array1, ...$array2];
```

### Классы и ООП

```php
<?php

// Абстрактный класс
abstract class Animal {
    abstract public function makeSound(): string;
    
    public function sleep(): void {
        echo "Sleeping...\n";
    }
}

// Интерфейс
interface Swimmable {
    public function swim(): void;
}

// Реализация
class Duck extends Animal implements Swimmable {
    public function __construct(
        private string $name,
        private int $age,
    ) {}
    
    public function makeSound(): string {
        return "Quack!";
    }
    
    public function swim(): void {
        echo "Swimming...\n";
    }
}

// Traits
trait Loggable {
    public function log(string $message): void {
        error_log($message);
    }
}

class Service {
    use Loggable;
    
    public function process(): void {
        $this->log("Processing...");
    }
}

// Generics через PHPDoc
/**
 * @template T
 */
class Collection {
    /**
     * @param array<T> $items
     */
    public function __construct(private array $items) {}
    
    /**
     * @return array<T>
     */
    public function all(): array {
        return $this->items;
    }
    
    /**
     * @param callable(T): bool $callback
     * @return array<T>
     */
    public function filter(callable $callback): array {
        return array_values(array_filter($this->items, $callback));
    }
}

// Magic methods
class User {
    public function __get(string $name): mixed {
        return $this->$name ?? null;
    }
    
    public function __set(string $name, mixed $value): void {
        $this->$name = $value;
    }
    
    public function __call(string $name, array $arguments): mixed {
        // Handle undefined method calls
    }
    
    public function __toString(): string {
        return json_encode($this);
    }
    
    public function __clone(): void {
        // Deep clone logic
    }
}
```

### Обработка ошибок

```php
<?php

// Try-catch
try {
    $result = riskyOperation();
} catch (SpecificException $e) {
    error_log("Specific error: " . $e->getMessage());
} catch (Throwable $e) {
    error_log("General error: " . $e->getMessage());
} finally {
    cleanup();
}

// Custom exception
class ValidationException extends Exception {
    public function __construct(
        public array $errors,
        string $message = "Validation failed",
        int $code = 422,
    ) {
        parent::__construct($message, $code);
    }
}

// Throw expression (PHP 8.0+)
$filename = $config['filename'] ?? throw new InvalidArgumentException('Filename required');

// Set error handler
set_error_handler(function($severity, $message, $file, $line) {
    throw new ErrorException($message, 0, $severity, $file, $line);
});

// Set exception handler
set_exception_handler(function(Throwable $e) {
    error_log($e);
    http_response_code(500);
    echo json_encode(['error' => 'Internal server error']);
});
```

### Работа с JSON

```php
<?php

// Encode
$data = ['name' => 'John', 'age' => 30];
$json = json_encode($data);
$json = json_encode($data, JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE);

// Decode
$data = json_decode($json, true); // associative array
$data = json_decode($json); // object

// Error handling
if (json_last_error() !== JSON_ERROR_NONE) {
    throw new JsonException(json_last_error_msg());
}

// JSON_THROW_ON_ERROR (PHP 7.3+)
$data = json_decode($json, true, 512, JSON_THROW_ON_ERROR);

// JsonSerializable
class User implements JsonSerializable {
    public function __construct(
        public int $id,
        public string $name,
        private string $password,
    ) {}
    
    public function jsonSerialize(): array {
        return [
            'id' => $this->id,
            'name' => $this->name,
            // password не включаем
        ];
    }
}
```

### HTTP запросы

```php
<?php

// cURL
$ch = curl_init('https://api.example.com/users');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_HTTPHEADER => ['Content-Type: application/json'],
    CURLOPT_POST => true,
    CURLOPT_POSTFIELDS => json_encode(['name' => 'John']),
]);
$response = curl_exec($ch);
curl_close($ch);

// File_get_contents (простые запросы)
$options = [
    'http' => [
        'method' => 'POST',
        'header' => "Content-Type: application/json\r\n",
        'content' => json_encode(['name' => 'John']),
    ],
];
$response = file_get_contents('https://api.example.com/users', false, stream_context_create($options));

// Guzzle (рекомендуется)
use GuzzleHttp\Client;

$client = new Client(['base_uri' => 'https://api.example.com']);

// GET
$response = $client->get('/users');
$data = json_decode($response->getBody(), true);

// POST
$response = $client->post('/users', [
    'json' => ['name' => 'John'],
]);

// Async
$promises = [
    'users' => $client->getAsync('/users'),
    'posts' => $client->getAsync('/posts'),
];
$results = GuzzleHttp\Promise\Utils::unwrap($promises);

// PSR-7 HTTP Message
use Psr\Http\Client\ClientInterface;
use Psr\Http\Message\RequestFactoryInterface;

$request = $requestFactory->createRequest('GET', '/users');
$response = $httpClient->sendRequest($request);
```

### Database (PDO)

```php
<?php

// Подключение
$pdo = new PDO(
    'mysql:host=localhost;dbname=mydb;charset=utf8mb4',
    'user',
    'pass',
    [
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
        PDO::ATTR_EMULATE_PREPARES => false,
    ]
);

// Prepared statements (защита от SQL injection)
$stmt = $pdo->prepare('SELECT * FROM users WHERE email = :email');
$stmt->execute(['email' => $email]);
$user = $stmt->fetch();

// Insert
$stmt = $pdo->prepare('INSERT INTO users (name, email) VALUES (:name, :email)');
$stmt->execute(['name' => $name, 'email' => $email]);
$lastId = $pdo->lastInsertId();

// Transaction
try {
    $pdo->beginTransaction();
    
    $pdo->exec("UPDATE accounts SET balance = balance - 100 WHERE id = 1");
    $pdo->exec("UPDATE accounts SET balance = balance + 100 WHERE id = 2");
    
    $pdo->commit();
} catch (Throwable $e) {
    $pdo->rollBack();
    throw $e;
}

// Fetch modes
$all = $stmt->fetchAll(PDO::FETCH_ASSOC);
$objects = $stmt->fetchAll(PDO::FETCH_CLASS, User::class);
$column = $stmt->fetchAll(PDO::FETCH_COLUMN);
```

---

## 🎯 Best Practices

### ✅ Делать

```php
<?php

// 1. Строгая типизация
declare(strict_types=1);

// 2. Типизация параметров и возвращаемых значений
function calculate(int $a, int $b): int {
    return $a + $b;
}

// 3. Использование value objects
readonly class Email {
    public function __construct(public string $value) {
        if (!filter_var($value, FILTER_VALIDATE_EMAIL)) {
            throw new InvalidArgumentException('Invalid email');
        }
    }
}

// 4. Dependency Injection
class UserService {
    public function __construct(
        private UserRepository $repository,
        private EmailService $emailService,
    ) {}
}

// 5. Repository pattern
interface UserRepository {
    public function findById(int $id): ?User;
    public function save(User $user): void;
}

// 6. DTO для передачи данных
readonly class CreateUserDTO {
    public function __construct(
        public string $name,
        public string $email,
        public string $password,
    ) {}
    
    public static function fromRequest(array $data): self {
        return new self(
            $data['name'],
            $data['email'],
            $data['password'],
        );
    }
}
```

### ❌ Не делать

```php
<?php

// 1. Избегать mysql_* функций (устарели)
mysql_connect(...); // ❌
PDO или mysqli; // ✅

// 2. Не использовать глобальные переменные
global $db; // ❌
Dependency Injection; // ✅

// 3. Не игнорировать ошибки
$result = @file_get_contents($url); // ❌
try { /* обработка */ } catch (...) {} // ✅

// 4. Не делать SQL injection
$sql = "SELECT * FROM users WHERE id = $id"; // ❌
$stmt = $pdo->prepare('SELECT * FROM users WHERE id = ?'); // ✅

// 5. Не использовать eval
eval($code); // ❌ (опасно)

// 6. Избегать глубокой вложенности
if ($a) {
    if ($b) {
        if ($c) {
            // ...
        }
    }
}
// ✅ Используйте guard clauses
if (!$a) return;
if (!$b) return;
if (!$c) return;
```

---

## 🐛 Частые ошибки и решения

| Ошибка | Причина | Решение |
|--------|---------|---------|
| `Cannot modify header information` | Вывод до header() | Уберите пробелы/echo до header() |
| `Undefined array key` | Доступ к несуществующему ключу | Используйте `??` оператор |
| `Trying to access array offset on null` | Null вместо массива | Проверяйте на null перед доступом |
| `SQLSTATE[HY000]` | Ошибка подключения к БД | Проверьте credentials и соединение |
| `Memory exhausted` | Слишком много данных | Используйте генераторы, pagination |

---

## 🔗 Связанные заметки

- [[Laravel-Cheatsheet]] — PHP фреймворк
- [[Security-Hardening]] — Безопасность PHP приложений
- [[Database-Design]] — Проектирование БД

---

## 📝 Заметки

> [!TIP] Совет от 15 лет опыта
> 
> 1. **Всегда strict_types=1** — ловит ошибки типов
> 2. **Используйте Composer** — не reinvent the wheel
> 3. **PDO с prepared statements** — защита от SQL injection
> 4. **PHPStan/Psalm** — статический анализ ловит баги
> 5. **Современный PHP (8.x)** — намного лучше старого

> [!INFO] Полезные инструменты
> ```bash
> # PHPStan (статический анализ)
> composer require --dev phpstan/phpstan
> ./vendor/bin/phpstan analyse src --level max
> 
> # Psalm (статический анализ)
> composer require --dev vimeo/psalm
> ./vendor/bin/psalm
> 
> # PHP CS Fixer (код стайл)
> composer require --dev friendsofphp/php-cs-fixer
> ./vendor/bin/php-cs-fixer fix
> 
> # PHPUnit (тесты)
> composer require --dev phpunit/phpunit
> ./vendor/bin/phpunit --coverage-html coverage
> 
> # Xdebug (отладка)
> pecl install xdebug
> # Добавьте в php.ini: zend_extension=xdebug.so
> ```
