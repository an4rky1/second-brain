---
created: 2026-02-16
tags:
  - cheat-sheet
  - rust
  - systems
  - performance
aliases:
  - Rust Cheatsheet
  - Rust Reference
related:
  - Docker-Cheatsheet
  - WebAssembly-Basics
  - System-Design-Basics
---

# Rust — Полная шпаргалка

> [!SUMMARY] Обзор
> Rust — системный язык программирования с гарантиями безопасности памяти без сборщика мусора. Идеален для системного программирования, WASM, CLI утилит и high-performance сервисов. Кривая обучения крутая, но окупается надёжностью.

---

## 📚 Теория

### Владение (Ownership)

```rust
// Владелец данных
let s1 = String::from("hello");
let s2 = s1; // Перемещение, s1 больше не валиден

let s1 = String::from("hello");
let s2 = s1.clone(); // Клонирование, оба валидны

let x = 5;
let y = x; // Копирование (Copy trait), оба валидны

// Заимствование
fn print(s: &String) {
    println!("{}", s);
}
// или
fn print(s: &str) {
    println!("{}", s);
}

// Mutable borrow
let mut s = String::from("hello");
let r1 = &mut s;
r1.push_str(", world");
// Нельзя создать другой borrow пока r1 используется

// Правила borrow checker:
// 1. ИЛИ один mutable borrow
// 2. ИЛИ сколько угодно immutable borrow'ов
// 3. Borrow'ы не могут выходить за область видимости владельца
```

### Время жизни (Lifetimes)

```rust
// Аннотация времени жизни
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}

// Struct с lifetimes
struct Excerpt<'a> {
    part: &'a str,
}

// Lifetime elision rules
// 1. Каждый параметр получает своё lifetime
// 2. Если один входной параметр, его lifetime присваивается выходу
// 3. Если &self или &mut self, lifetime self присваивается выходу

impl<'a> Excerpt<'a> {
    fn level(&self) -> i32 {
        3 // lifetime self автоматически
    }
}
```

### Типы данных

```rust
// Скалярные
let x: i32 = 42;           // Знаковое целое
let y: u32 = 42;           // Беззнаковое целое
let f: f64 = 3.14;         // Float
let b: bool = true;        // Boolean
let c: char = 'A';         // Unicode символ
let (): () = ();           // Unit type

// Составные
let arr: [i32; 5] = [1, 2, 3, 4, 5];  // Массив
let tuple: (i32, &str) = (42, "hello");
let slice: &[i32] = &arr[1..3];       // Срез

// String
let s1: &str = "literal";      // String slice (статическая)
let s2: String = String::from("owned");  // Owned String

// Коллекции
let mut vec: Vec<i32> = vec![1, 2, 3];
let mut map: HashMap<String, i32> = HashMap::new();
let mut set: HashSet<i32> = HashSet::new();

// Option
let some: Option<i32> = Some(42);
let none: Option<i32> = None;

// Result
let ok: Result<i32, String> = Ok(42);
let err: Result<i32, String> = Err("error".to_string());

// Pattern matching
let opt: Option<i32> = Some(5);
match opt {
    Some(n) if n > 0 => println!("Positive: {}", n),
    Some(n) => println!("Non-positive: {}", n),
    None => println!("None"),
}

// let-else
let Some(n) = opt else {
    return Err("No value");
};

// if let
if let Some(n) = opt {
    println!("{}", n);
}

// Enum
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}

impl Message {
    fn process(&self) {
        match self {
            Message::Quit => println!("Quit"),
            Message::Move { x, y } => println!("Move to {}, {}", x, y),
            Message::Write(text) => println!("Write: {}", text),
            Message::ChangeColor(r, g, b) => println!("Color: {}, {}, {}", r, g, b),
        }
    }
}

// Struct
#[derive(Debug, Clone, PartialEq)]
struct User {
    id: u64,
    name: String,
    email: String,
}

// Tuple struct
struct Point(i32, i32, i32);

// Unit struct
struct Marker;
```

---

## ⚡ Быстрый старт

```bash
# Установка
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Проверка
rustc --version
cargo --version

# Создание проекта
cargo new my_project
cargo new my_lib --lib

# Сборка
cargo build
cargo build --release  # Оптимизированная сборка

# Запуск
cargo run
cargo run --release

# Тесты
cargo test
cargo test -- --nocapture  # Показать вывод

# Проверка кода
cargo check           # Быстрая проверка
cargo clippy          # Линтер
cargo fmt             # Форматирование

# Зависимости
cargo add serde --features derive
cargo add tokio --features full
cargo rm old_dep

# Документация
cargo doc --open
```

### Cargo.toml

```toml
[package]
name = "my_project"
version = "0.1.0"
edition = "2021"
authors = ["Your Name <you@example.com>"]

[dependencies]
# Crates.io
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1.0", features = ["full"] }
reqwest = { version = "0.11", features = ["json"] }

# Git репозиторий
my_crate = { git = "https://github.com/user/repo.git" }

# Локальный путь
local_crate = { path = "../local_crate" }

[dev-dependencies]
mockall = "0.11"
tempfile = "3"

[build-dependencies]
cc = "1.0"

[profile.release]
lto = true
codegen-units = 1
```

---

## 🔧 Практические примеры

### Обработка ошибок

```rust
// Result и ?
fn read_file(path: &str) -> Result<String, std::io::Error> {
    let mut file = std::fs::File::open(path)?;
    let mut contents = String::new();
    file.read_to_string(&mut contents)?;
    Ok(contents)
}

// Преобразование ошибок
fn parse_number(s: &str) -> Result<i32, Box<dyn std::error::Error>> {
    let n: i32 = s.parse()?;
    Ok(n)
}

// Custom error
use thiserror::Error;

#[derive(Error, Debug)]
enum AppError {
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
    
    #[error("Parse error: {0}")]
    Parse(#[from] std::num::ParseIntError),
    
    #[error("Not found: {0}")]
    NotFound(String),
    
    #[error("Validation failed: {0}")]
    Validation(String),
}

fn process(id: &str) -> Result<(), AppError> {
    let n: i32 = id.parse()?;  // Автоматическое преобразование
    if n < 0 {
        return Err(AppError::Validation("ID must be positive".into()));
    }
    Ok(())
}

// anyhow для приложений
use anyhow::{Context, Result};

fn load_config(path: &str) -> Result<String> {
    std::fs::read_to_string(path)
        .with_context(|| format!("Failed to read config from {}", path))
}
```

### Traits

```rust
// Определение trait
trait Drawable {
    fn draw(&self);
    
    fn area(&self) -> f64;
    
    // Метод по умолчанию
    fn describe(&self) -> String {
        format!("An object with area {}", self.area())
    }
}

// Реализация
struct Circle {
    radius: f64,
}

impl Drawable for Circle {
    fn draw(&self) {
        println!("Drawing circle with radius {}", self.radius);
    }
    
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.radius.powi(2)
    }
}

// Trait bounds
fn print_drawable(item: &impl Drawable) {
    item.draw();
}

fn print_both<T: Drawable + std::fmt::Debug>(item: &T) {
    println!("{:?}", item);
    item.draw();
}

// Where clause
fn process<T, U>(t: &T, u: &U) -> f64
where
    T: Drawable,
    U: Drawable,
{
    t.area() + u.area()
}

// Trait objects (динамическая диспетчеризация)
let shapes: Vec<Box<dyn Drawable>> = vec![
    Box::new(Circle { radius: 1.0 }),
];

// Common traits
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
struct User {
    id: u64,
    name: String,
}

// Default trait
impl Default for Config {
    fn default() -> Self {
        Self {
            host: "localhost".into(),
            port: 8080,
        }
    }
}

// From/Into traits
impl From<&str> for Email {
    fn from(s: &str) -> Self {
        Email(s.to_string())
    }
}

let email: Email = "test@example.com".into();

// Display trait
impl std::fmt::Display for User {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{} <id={}>", self.name, self.id)
    }
}
```

### Generics

```rust
// Generic функция
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}

// Generic struct
struct Container<T> {
    item: T,
}

impl<T> Container<T> {
    fn new(item: T) -> Self {
        Self { item }
    }
    
    fn get(&self) -> &T {
        &self.item
    }
}

// Generic с несколькими типами
struct Pair<T, U> {
    first: T,
    second: U,
}

// Associated types
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}

// Generic closures
let process = |x: i32| x * 2;
let with_type: fn(i32) -> i32 = |x| x * 2;
```

### Concurrency

```rust
// Threads
use std::thread;
use std::sync::{Arc, Mutex};

let handle = thread::spawn(|| {
    println!("Hello from thread!");
});

handle.join().unwrap();

// Shared state
let counter = Arc::new(Mutex::new(0));
let mut handles = vec![];

for _ in 0..10 {
    let counter = Arc::clone(&counter);
    let handle = thread::spawn(move || {
        let mut num = counter.lock().unwrap();
        *num += 1;
    });
    handles.push(handle);
}

for handle in handles {
    handle.join().unwrap();
}

// Channels
use std::sync::mpsc;

let (tx, rx) = mpsc::channel();

thread::spawn(move || {
    tx.send("Hello").unwrap();
});

let msg = rx.recv().unwrap();

// Multiple producers
let (tx, rx) = mpsc::channel();
let tx1 = tx.clone();

thread::spawn(move || {
    tx1.send("From tx1").unwrap();
});

thread::spawn(move || {
    tx.send("From tx").unwrap();
});

// Async (tokio)
use tokio::task;

#[tokio::main]
async fn main() {
    let handle = task::spawn(async {
        println!("Async task");
        42
    });
    
    let result = handle.await.unwrap();
}

// Async channels (tokio)
use tokio::sync::{mpsc, broadcast, watch};

let (tx, mut rx) = mpsc::channel(32);

task::spawn(async move {
    tx.send("Hello").await.unwrap();
});

let msg = rx.recv().await;

// RwLock
use std::sync::RwLock;

let lock = RwLock::new(5);
{
    let r1 = lock.read().unwrap();
    let r2 = lock.read().unwrap();
    // Multiple readers
}
{
    let mut w = lock.write().unwrap();
    *w += 1;
    // Single writer
}
```

### Serde (сериализация)

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct User {
    id: u64,
    name: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    email: Option<String>,
}

// JSON
let user = User { id: 1, name: "John".into(), email: None };
let json = serde_json::to_string(&user)?;
let json_pretty = serde_json::to_string_pretty(&user)?;

let user: User = serde_json::from_str(&json)?;

// YAML
let yaml = serde_yaml::to_string(&user)?;
let user: User = serde_yaml::from_str(&yaml)?;

// Custom serialization
impl Serialize for Email {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: serde::Serializer,
    {
        serializer.serialize_str(&self.0)
    }
}
```

### HTTP Client (reqwest)

```rust
use reqwest;
use serde::{Deserialize, Serialize};

#[derive(Serialize)]
struct CreateUser {
    name: String,
    email: String,
}

#[derive(Deserialize)]
struct User {
    id: u64,
    name: String,
}

// GET
let client = reqwest::Client::new();
let user: User = client
    .get("https://api.example.com/users/1")
    .header("Authorization", "Bearer token")
    .send()
    .await?
    .json()
    .await?;

// POST
let new_user = CreateUser {
    name: "John".into(),
    email: "john@example.com".into(),
};

let user: User = client
    .post("https://api.example.com/users")
    .json(&new_user)
    .send()
    .await?
    .json()
    .await?;

// Download file
let bytes = client
    .get("https://example.com/file.bin")
    .send()
    .await?
    .bytes()
    .await?;
```

---

## 🎯 Best Practices

### ✅ Делать

```rust
// 1. Используйте Result с ?
fn read_file(path: &str) -> Result<String, std::io::Error> {
    std::fs::read_to_string(path)
}

// 2. Избегайте unwrap в production
let value = option.ok_or_else(|| Error::new("No value"))?;

// 3. Используйте newtype pattern
struct UserId(u64);
struct Email(String);

// 4. Builder pattern
#[derive(Default)]
struct UserBuilder {
    name: Option<String>,
    email: Option<String>,
}

impl UserBuilder {
    fn name(mut self, name: String) -> Self {
        self.name = Some(name);
        self
    }
    
    fn build(self) -> Result<User> {
        Ok(User {
            name: self.name.ok_or("Name required")?,
            email: self.email.ok_or("Email required")?,
        })
    }
}

// 5. Используйте enum для состояний
enum OrderState {
    Pending,
    Processing { started_at: DateTime<Utc> },
    Shipped { tracking: String },
    Delivered,
}

// 6. Тесты
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_add() {
        assert_eq!(add(2, 3), 5);
    }
    
    #[test]
    #[should_panic(expected = "divide by zero")]
    fn test_divide_by_zero() {
        divide(1, 0);
    }
    
    #[tokio::test]
    async fn test_async() {
        let result = async_func().await;
        assert!(result.is_ok());
    }
}
```

### ❌ Не делать

```rust
// 1. Избегать clone без необходимости
let s2 = s1.clone(); // ❌ если можно заимствовать
let s2 = &s1; // ✅

// 2. Не игнорировать Result
let _ = do_something(); // ❌
do_something()?; // ✅

// 3. Избегать unwrap в production
let value = option.unwrap(); // ❌
let value = option.ok_or(Error::new("..."))?; // ✅

// 4. Не использовать refcell для shared state
let data = Rc::new(RefCell::new(vec![])); // ❌
let data = Arc::new(Mutex::new(vec![])); // ✅ для threads

// 5. Избегать преждевременной оптимизации
// Пишите чистый код сначала, оптимизируйте по профилю
```

---

## 🐛 Частые ошибки и решения

| Ошибка | Причина | Решение |
|--------|---------|---------|
| `borrowed value does not live long enough` | Lifetime слишком короткий | Добавьте явные lifetime аннотации |
| `cannot borrow as mutable` | Нарушение правил borrow | Перестройте код, используйте scope |
| `use of moved value` | Значение перемещено | Используйте clone или borrow |
| `trait bound not satisfied` | Trait не реализован | Добавьте `#[derive(...)]` или impl |
| `lifetime may not live long enough` | Конфликт lifetimes | Используйте `'static` или перестройте |

---

## 🔗 Связанные заметки

- [[Docker-Cheatsheet]] — Контейнеризация Rust приложений
- [[WebAssembly-Basics]] — Rust для WASM
- [[System-Design-Basics]] — Системный дизайн

---

## 📝 Заметки

> [!TIP] Совет от 15 лет опыта
> 
> 1. **Borrow checker — друг** — он предотвращает баги
> 2. **Избегайте unwrap** — используйте ? и proper error handling
> 3. **Clippy знает лучше** — слушайте предупреждения
> 4. **Начинайте с safe Rust** — unsafe только когда необходимо
> 5. **Читайте The Book** — https://doc.rust-lang.org/book/

> [!INFO] Полезные инструменты
> ```bash
> # Clippy (линтер)
> cargo clippy -- -D warnings
> 
> # Format
> cargo fmt
> 
> # Audit зависимости
> cargo install cargo-audit
> cargo audit
> 
> # Outdated зависимости
> cargo install cargo-outdated
> cargo outdated
> 
> # Expand macros
> cargo install cargo-expand
> cargo expand
> 
> # Coverage
> cargo install cargo-tarpaulin
> cargo tarpaulin --out Html
> ```
