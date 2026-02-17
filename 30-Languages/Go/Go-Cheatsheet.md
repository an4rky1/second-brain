---
created: 2026-02-16
tags:
  - cheat-sheet
  - go
  - golang
  - backend
aliases:
  - Go Cheatsheet
  - Golang Reference
related:
  - Docker-Cheatsheet
  - Kubernetes-Cheatsheet
  - Microservices-Patterns
---

# Go (Golang) — Полная шпаргалка

> [!SUMMARY] Обзор
> Go — компилируемый язык с статической типизацией, созданный Google. Идеален для микросервисов, CLI утилит и high-performance систем. Простой синтаксис, встроенная конкурентность, быстрая компиляция.

---

## 📚 Теория

### Основные принципы Go

1. **Простота** — минимум синтаксического сахара
2. **Явная обработка ошибок** — нет исключений
3. **Встроенная конкурентность** — goroutines и channels
4. **Быстрая компиляция** — секунды, не минуты
5. **Статическая линковка** — один бинарник без зависимостей

### Типы данных

```go
// Примитивы
var name string = "John"
var age int = 30
var age32 int32 = 30
var age64 int64 = 30
var price float64 = 19.99
var isActive bool = true

// Короткое объявление (только внутри функций)
name := "John"
age := 30

// Константы
const Pi = 3.14159
const (
    StatusOK = 200
    StatusNotFound = 404
)

// Массивы (фиксированный размер)
var arr [5]int
arr2 := [3]int{1, 2, 3}

// Слайсы (динамический размер)
slice := []int{1, 2, 3}
slice2 := make([]int, 5, 10) // length, capacity

// Maps
user := map[string]interface{}{
    "name": "John",
    "age":  30,
}

// Structs
type User struct {
    ID    int
    Name  string
    Email string
}

// Pointers
var p *int
x := 10
p = &x
fmt.Println(*p) // 10
```

---

## ⚡ Быстрый старт

```bash
# Установка (macOS)
brew install go

# Проверка версии
go version

# Инициализация модуля
go mod init github.com/user/project

# Установка зависимостей
go get github.com/gin-gonic/gin

# Запуск
go run main.go

# Сборка
go build -o app main.go

# Тесты
go test ./...
go test -v ./...

# Форматирование
go fmt ./...

# Веттинг (проверка стиля)
go vet ./...

# Обновление зависимостей
go get -u ./...
go mod tidy
```

### Структура проекта

```
project/
├── cmd/
│   └── app/
│       └── main.go      # Точка входа
├── internal/
│   ├── handler/         # HTTP handlers
│   ├── service/         # Business logic
│   ├── repository/      # Data access
│   └── model/           # Data models
├── pkg/                 # Публичные пакеты
├── configs/             # Конфигурация
├── migrations/          # DB migrations
├── scripts/             # Build scripts
├── tests/               # Integration tests
├── go.mod
├── go.sum
└── Makefile
```

---

## 🔧 Практические примеры

### Обработка ошибок

```go
// Базовый паттерн
func readFile(path string) (string, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return "", fmt.Errorf("read file: %w", err)
    }
    return string(data), nil
}

// Проверка с errors.Is и errors.As
if errors.Is(err, os.ErrNotExist) {
    // Файл не существует
}

var pathErr *os.PathError
if errors.As(err, &pathErr) {
    // Ошибка пути
}

// Custom errors
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("%s: %s", e.Field, e.Message)
}

// Использование
if err != nil {
    var ve *ValidationError
    if errors.As(err, &ve) {
        // Валидационная ошибка
    }
}
```

### Конкурентность (Goroutines & Channels)

```go
// Goroutine
go func() {
    fmt.Println("Running in background")
}()

// Channel
ch := make(chan int)
go func() {
    ch <- 42 // Send
}()
value := <-ch // Receive

// Buffered channel
ch := make(chan int, 10)

// Range over channel
for value := range ch {
    fmt.Println(value)
}

// Select
select {
case msg1 := <-ch1:
    fmt.Println("Received from ch1:", msg1)
case msg2 := <-ch2:
    fmt.Println("Received from ch2:", msg2)
case <-time.After(time.Second):
    fmt.Println("Timeout")
default:
    fmt.Println("No data ready")
}

// Worker pool
func worker(id int, jobs <-chan int, results chan<- int) {
    for job := range jobs {
        results <- job * 2
    }
}

func main() {
    jobs := make(chan int, 100)
    results := make(chan int, 100)
    
    // Start workers
    for w := 1; w <= 3; w++ {
        go worker(w, jobs, results)
    }
    
    // Send jobs
    for j := 1; j <= 5; j++ {
        jobs <- j
    }
    close(jobs)
    
    // Collect results
    for r := 1; r <= 5; r++ {
        <-results
    }
}

// WaitGroup
var wg sync.WaitGroup
for i := 0; i < 5; i++ {
    wg.Add(1)
    go func(id int) {
        defer wg.Done()
        fmt.Println("Worker", id)
    }(i)
}
wg.Wait()

// Mutex
var mu sync.Mutex
var counter int

func increment() {
    mu.Lock()
    defer mu.Unlock()
    counter++
}

// Atomic
var count atomic.Int64
count.Add(1)
val := count.Load()

// Context
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
resp, err := http.DefaultClient.Do(req)
```

### Interfaces

```go
// Определение интерфейса
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

type ReadWriter interface {
    Reader
    Writer
}

// Реализация (неявная)
type File struct{}

func (f *File) Read(p []byte) (int, error) {
    // ...
}

func (f *File) Write(p []byte) (int, error) {
    // ...
}

// File автоматически реализует ReadWriter

// Empty interface (любой тип)
func printAny(v interface{}) {
    fmt.Println(v)
}

// Type assertion
val, ok := v.(string)
if !ok {
    // Не string
}

// Type switch
switch v := v.(type) {
case int:
    fmt.Println("Integer:", v)
case string:
    fmt.Println("String:", v)
default:
    fmt.Println("Unknown type")
}
```

### Generics (Go 1.18+)

```go
// Generic функция
func Map[T, U any](slice []T, fn func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = fn(v)
    }
    return result
}

// Использование
numbers := []int{1, 2, 3}
strings := Map(numbers, func(n int) string {
    return strconv.Itoa(n)
})

// Generic с ограничениями
func Sum[T constraints.Integer | constraints.Float](s []T) T {
    var total T
    for _, v := range s {
        total += v
    }
    return total
}

// Generic интерфейс
type Stringer interface {
    String() string
}

func PrintAll[T Stringer](items []T) {
    for _, item := range items {
        fmt.Println(item.String())
    }
}

// Generic struct
type Container[T any] struct {
    items []T
}

func NewContainer[T any]() *Container[T] {
    return &Container[T]{items: make([]T, 0)}
}

func (c *Container[T]) Add(item T) {
    c.items = append(c.items, item)
}
```

### HTTP Server

```go
// Базовый сервер
package main

import (
    "encoding/json"
    "net/http"
)

type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}

func main() {
    http.HandleFunc("/users", getUsers)
    http.HandleFunc("/users/", getUser)
    http.ListenAndServe(":8080", nil)
}

func getUsers(w http.ResponseWriter, r *http.Request) {
    users := []User{
        {ID: 1, Name: "John"},
        {ID: 2, Name: "Jane"},
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}

func getUser(w http.ResponseWriter, r *http.Request) {
    // GET /users/{id}
    id := r.URL.Path[len("/users/"):]
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(User{ID: 1, Name: "John"})
}

// Middleware
func logging(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        log.Printf("%s %s %v", r.Method, r.URL.Path, time.Since(start))
    })
}

func auth(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if token == "" {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        next.ServeHTTP(w, r)
    })
}

// Использование middleware
mux := http.NewServeMux()
mux.HandleFunc("/users", getUsers)

handler := logging(mux)
handler = auth(handler)

http.ListenAndServe(":8080", handler)
```

### Работа с JSON

```go
// Struct tags
type User struct {
    ID        int       `json:"id"`
    Name      string    `json:"name"`
    Email     string    `json:"email,omitempty"` // Пропустить если пустой
    Password  string    `json:"-"`               // Игнорировать
    CreatedAt time.Time `json:"created_at"`
}

// Marshal (encode)
user := User{ID: 1, Name: "John"}
data, err := json.Marshal(user)
data, err := json.MarshalIndent(user, "", "  ") // С отступами

// Unmarshal (decode)
var user User
err := json.Unmarshal(data, &user)

// Decoder (streaming)
var user User
err := json.NewDecoder(r.Body).Decode(&user)

// Encoder (streaming)
err := json.NewEncoder(w).Encode(user)

// Custom marshal/unmarshal
type Money int64

func (m Money) MarshalJSON() ([]byte, error) {
    return []byte(fmt.Sprintf(`"%d.%02d"`, m/100, m%100)), nil
}

func (m *Money) UnmarshalJSON(data []byte) error {
    var s string
    if err := json.Unmarshal(data, &s); err != nil {
        return err
    }
    // Parse string to Money
    return nil
}
```

### Database (sqlx)

```go
import (
    "database/sql"
    _ "github.com/lib/pq"
    "github.com/jmoiron/sqlx"
)

type User struct {
    ID    int    `db:"id"`
    Name  string `db:"name"`
    Email string `db:"email"`
}

// Подключение
db, err := sqlx.Connect("postgres", "postgres://user:pass@localhost/db?sslmode=disable")
if err != nil {
    log.Fatal(err)
}

// Query
var users []User
err = db.Select(&users, "SELECT * FROM users WHERE active = $1", true)

// Query row
var user User
err = db.Get(&user, "SELECT * FROM users WHERE id = $1", 1)

// Named query
_, err = db.NamedExec("INSERT INTO users (:name, :email) VALUES (:name, :email)", user)

// Transactions
tx, err := db.Beginx()
if err != nil {
    return err
}
defer tx.Rollback()

_, err = tx.Exec("UPDATE accounts SET balance = balance - $1 WHERE id = $2", amount, fromID)
_, err = tx.Exec("UPDATE accounts SET balance = balance + $1 WHERE id = $2", amount, toID)

if err != nil {
    return err
}

err = tx.Commit()
```

---

## 🎯 Best Practices

### ✅ Делать

```go
// 1. Обработка ошибок
if err != nil {
    return fmt.Errorf("operation failed: %w", err)
}

// 2. Дефер для ресурсов
file, err := os.Open("file.txt")
if err != nil {
    return err
}
defer file.Close()

// 3. Именованные возвращаемые значения (для сложных случаев)
func parse(data []byte) (result *Result, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic: %v", r)
        }
    }()
    // ...
    return result, nil
}

// 4. Контекст для отмены
func process(ctx context.Context, data []byte) error {
    select {
    case <-ctx.Done():
        return ctx.Err()
    default:
        // Process
    }
}

// 5. Тесты
func TestAdd(t *testing.T) {
    t.Parallel()
    
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive", 1, 2, 3},
        {"negative", -1, -2, -3},
    }
    
    for _, tt := range tests {
        tt := tt // capture range variable
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel()
            result := Add(tt.a, tt.b)
            if result != tt.expected {
                t.Errorf("Add(%d, %d) = %d, want %d", tt.a, tt.b, result, tt.expected)
            }
        })
    }
}
```

### ❌ Не делать

```go
// 1. Игнорирование ошибок
result, _ := doSomething() // ❌

// 2. Паника вместо ошибки
if err != nil {
    panic(err) // ❌
}

// 3. Глобальные переменные
var global *Database // ❌

// 4. Игнорирование контекста
func fetchData() ([]byte, error) // ❌
func fetchData(ctx context.Context) ([]byte, error) // ✅

// 5. Сложные дженерики без необходимости
```

---

## 🐛 Частые ошибки и решения

| Ошибка | Причина | Решение |
|--------|---------|---------|
| `variable declared but not used` | Неиспользуемая переменная | Удалите или используйте `_` |
| `cannot use ... as type ...` | Несоответствие типов | Проверьте интерфейс/тип |
| `data race detected` | Гонка данных | Используйте mutex или channels |
| `context deadline exceeded` | Таймаут контекста | Увеличьте таймаут или оптимизируйте |
| `nil pointer dereference` | Нил указатель | Проверяйте перед использованием |

---

## 🔗 Связанные заметки

- [[Docker-Cheatsheet]] — Контейнеризация Go приложений
- [[Kubernetes-Cheatsheet]] — Деплой Go микросервисов
- [[Microservices-Patterns]] — Паттерны микросервисов

---

## 📝 Заметки

> [!TIP] Совет от 15 лет опыта
> 
> 1. **Простота — ключ** — не усложняйте без необходимости
> 2. **Обрабатывайте ошибки явно** — это фича, не баг
> 3. **Используйте контекст** — для отмены и таймаутов
> 4. **Тестируйте параллельно** — `t.Parallel()` ловит гонки
> 5. **Один бинарник** — преимущество Go, используйте его

> [!INFO] Полезные инструменты
> ```bash
> # Линтер
> go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
> golangci-lint run
> 
> # Форматтер
> gofmt -w .
> 
> # Генерация моков
> go install github.com/golang/mock/mockgen@latest
> 
> # Race detector
> go test -race ./...
> 
> # Benchmark
> go test -bench=. -benchmem ./...
> ```
