---
created: 2026-02-17
tags:
  - patterns
  - design-patterns
  - gof
  - architecture
aliases:
  - GoF Design Patterns
  - Gang of Four Patterns
related:
  - MOC-Patterns
  - Clean-Architecture
  - Backend-Architecture-MOC
---

# GoF Design Patterns

> [!SUMMARY] Обзор
> 23 паттерна проектирования из книги "Design Patterns" (Gang of Four: Gamma, Helm, Johnson, Vlissides).

---

## 📚 Creational Patterns

### Singleton

```typescript
// Гарантирует один экземпляр класса
export class Database {
  private static instance: Database;
  private connection: any;

  private constructor() {
    this.connection = this.createConnection();
  }

  static getInstance(): Database {
    if (!Database.instance) {
      Database.instance = new Database();
    }
    return Database.instance;
  }

  query(sql: string) {
    return this.connection.execute(sql);
  }
}

// Usage
const db1 = Database.getInstance();
const db2 = Database.getInstance();
console.log(db1 === db2);  // true
```

### Factory Method

```typescript
// Создаёт объекты без указания конкретного класса
interface Document {
  open(): void;
  save(): void;
}

class PdfDocument implements Document {
  open() { console.log('Opening PDF'); }
  save() { console.log('Saving PDF'); }
}

class WordDocument implements Document {
  open() { console.log('Opening Word'); }
  save() { console.log('Saving Word'); }
}

abstract class Application {
  abstract createDocument(): Document;

  openDocument() {
    const doc = this.createDocument();
    doc.open();
  }
}

class PdfApplication extends Application {
  createDocument(): Document {
    return new PdfDocument();
  }
}
```

### Abstract Factory

```typescript
// Семейства связанных объектов
interface Button {
  render(): void;
}

interface Checkbox {
  render(): void;
}

class WindowsButton implements Button {
  render() { console.log('Render Windows Button'); }
}

class MacButton implements Button {
  render() { console.log('Render Mac Button'); }
}

interface GUIFactory {
  createButton(): Button;
  createCheckbox(): Checkbox;
}

class WindowsFactory implements GUIFactory {
  createButton(): Button { return new WindowsButton(); }
  createCheckbox(): Checkbox { return new WindowsCheckbox(); }
}

// Usage
const factory: GUIFactory = new WindowsFactory();
const button = factory.createButton();
button.render();
```

### Builder

```typescript
// Пошаговое создание сложных объектов
class Computer {
  cpu: string;
  ram: string;
  storage: string;
  gpu?: string;

  constructor(builder: ComputerBuilder) {
    this.cpu = builder.cpu;
    this.ram = builder.ram;
    this.storage = builder.storage;
    this.gpu = builder.gpu;
  }
}

class ComputerBuilder {
  cpu: string;
  ram: string;
  storage: string;
  gpu?: string;

  setCpu(cpu: string) { this.cpu = cpu; return this; }
  setRam(ram: string) { this.ram = ram; return this; }
  setStorage(storage: string) { this.storage = storage; return this; }
  setGpu(gpu: string) { this.gpu = gpu; return this; }

  build(): Computer {
    return new Computer(this);
  }
}

// Usage
const gamingPc = new ComputerBuilder()
  .setCpu('Intel i9')
  .setRam('32GB')
  .setStorage('1TB SSD')
  .setGpu('RTX 4090')
  .build();
```

### Prototype

```typescript
// Клонирование объектов
interface Prototype {
  clone(): Prototype;
}

class User implements Prototype {
  constructor(
    public id: number,
    public name: string,
    public email: string,
  ) {}

  clone(): Prototype {
    return new User(this.id, this.name, this.email);
  }
}

// Usage
const user1 = new User(1, 'John', 'john@example.com');
const user2 = user1.clone() as User;
user2.name = 'Jane';  // user1 не изменился
```

---

## 🏗️ Structural Patterns

### Adapter

```typescript
// Совместимость несовместимых интерфейсов
interface OldPayment {
  pay(amount: number): boolean;
}

interface NewPayment {
  processPayment(amount: number): Promise<boolean>;
}

class PaymentAdapter implements NewPayment {
  constructor(private oldPayment: OldPayment) {}

  async processPayment(amount: number): Promise<boolean> {
    return this.oldPayment.pay(amount);
  }
}
```

### Decorator

```typescript
// Добавление поведения без изменения класса
interface Coffee {
  cost(): number;
  description(): string;
}

class SimpleCoffee implements Coffee {
  cost() { return 5; }
  description() { return 'Simple coffee'; }
}

class MilkDecorator implements Coffee {
  constructor(private coffee: Coffee) {}

  cost() { return this.coffee.cost() + 2; }
  description() { return `${this.coffee.description()}, milk`; }
}

class SugarDecorator implements Coffee {
  constructor(private coffee: Coffee) {}

  cost() { return this.coffee.cost() + 1; }
  description() { return `${this.coffee.description()}, sugar`; }
}

// Usage
let coffee: Coffee = new SimpleCoffee();
coffee = new MilkDecorator(coffee);
coffee = new SugarDecorator(coffee);
console.log(coffee.description());  // Simple coffee, milk, sugar
console.log(coffee.cost());  // 8
```

### Facade

```typescript
// Упрощённый интерфейс к сложной системе
class AudioMixer {
  fix() { console.log('Audio fixed'); }
}

class VideoCodec {
  fix() { console.log('Video fixed'); }
}

class BitrateReader {
  read(file: string) { return 'data'; }
}

class VideoConverterFacade {
  private mixer = new AudioMixer();
  private codec = new VideoCodec();
  private reader = new BitrateReader();

  convert(file: string, format: string): string {
    this.reader.read(file);
    this.codec.fix();
    this.mixer.fix();
    return 'converted_data';
  }
}

// Usage
const converter = new VideoConverterFacade();
const result = converter.convert('video.mp4', 'avi');
```

### Proxy

```typescript
// Контроль доступа к объекту
interface Image {
  display(): void;
}

class RealImage implements Image {
  constructor(private filename: string) {
    this.loadFromDisk();
  }

  private loadFromDisk() {
    console.log(`Loading ${this.filename}`);
  }

  display() {
    console.log(`Displaying ${this.filename}`);
  }
}

class ProxyImage implements Image {
  private realImage?: RealImage;

  constructor(private filename: string) {}

  display() {
    if (!this.realImage) {
      this.realImage = new RealImage(this.filename);
    }
    this.realImage.display();
  }
}

// Usage
const image = new ProxyImage('photo.jpg');
image.display();  // Загрузит только при первом вызове
```

### Composite

```typescript
// Древовидные структуры
interface Component {
  operation(): string;
}

class Leaf implements Component {
  operation(): string {
    return 'Leaf';
  }
}

class Composite implements Component {
  private children: Component[] = [];

  add(component: Component) {
    this.children.push(component);
  }

  operation(): string {
    return `Branch(${this.children.map(c => c.operation()).join(',')})`;
  }
}

// Usage
const tree = new Composite();
tree.add(new Leaf());
tree.add(new Leaf());
const branch = new Composite();
branch.add(new Leaf());
tree.add(branch);
console.log(tree.operation());
```

---

## 🔄 Behavioral Patterns

### Observer

```typescript
// Подписка на события
interface Observer {
  update(data: any): void;
}

class Subject {
  private observers: Observer[] = [];

  attach(observer: Observer) {
    this.observers.push(observer);
  }

  notify(data: any) {
    this.observers.forEach(o => o.update(data));
  }
}

class EmailObserver implements Observer {
  update(data: any) {
    console.log(`Sending email: ${data}`);
  }
}

class SmsObserver implements Observer {
  update(data: any) {
    console.log(`Sending SMS: ${data}`);
  }
}

// Usage
const subject = new Subject();
subject.attach(new EmailObserver());
subject.attach(new SmsObserver());
subject.notify('New order!');
```

### Strategy

```typescript
// Смена алгоритмов во время выполнения
interface PaymentStrategy {
  pay(amount: number): void;
}

class CreditCardStrategy implements PaymentStrategy {
  constructor(private cardNumber: string) {}

  pay(amount: number) {
    console.log(`Paid ${amount} with credit card ${this.cardNumber}`);
  }
}

class PaypalStrategy implements PaymentStrategy {
  constructor(private email: string) {}

  pay(amount: number) {
    console.log(`Paid ${amount} with PayPal ${this.email}`);
  }
}

class ShoppingCart {
  private strategy: PaymentStrategy;

  setStrategy(strategy: PaymentStrategy) {
    this.strategy = strategy;
  }

  checkout(amount: number) {
    this.strategy.pay(amount);
  }
}

// Usage
const cart = new ShoppingCart();
cart.setStrategy(new CreditCardStrategy('1234-5678'));
cart.checkout(100);
cart.setStrategy(new PaypalStrategy('john@example.com'));
cart.checkout(50);
```

### Command

```typescript
// Инкапсуляция команд
interface Command {
  execute(): void;
}

class Light {
  turnOn() { console.log('Light on'); }
  turnOff() { console.log('Light off'); }
}

class LightOnCommand implements Command {
  constructor(private light: Light) {}
  execute() { this.light.turnOn(); }
}

class LightOffCommand implements Command {
  constructor(private light: Light) {}
  execute() { this.light.turnOff(); }
}

class RemoteControl {
  private command: Command;

  setCommand(command: Command) {
    this.command = command;
  }

  pressButton() {
    this.command.execute();
  }
}

// Usage
const light = new Light();
const remote = new RemoteControl();
remote.setCommand(new LightOnCommand(light));
remote.pressButton();
```

### State

```typescript
// Поведение от состояния
interface State {
  insertDollar(): void;
  ejectMoney(): void;
  dispense(): void;
}

class NoMoneyState implements State {
  constructor(private machine: VendingMachine) {}

  insertDollar() {
    console.log('Money inserted');
    this.machine.setState(new HasMoneyState(this.machine));
  }

  ejectMoney() { console.log('No money to eject'); }
  dispense() { console.log('Insert money first'); }
}

class HasMoneyState implements State {
  constructor(private machine: VendingMachine) {}

  insertDollar() { console.log('Already has money'); }
  ejectMoney() {
    console.log('Money ejected');
    this.machine.setState(new NoMoneyState(this.machine));
  }
  dispense() {
    console.log('Item dispensed');
    this.machine.setState(new NoMoneyState(this.machine));
  }
}

class VendingMachine {
  private state: State;

  constructor() {
    this.state = new NoMoneyState(this);
  }

  setState(state: State) {
    this.state = state;
  }

  insertDollar() { this.state.insertDollar(); }
  ejectMoney() { this.state.ejectMoney(); }
  dispense() { this.state.dispense(); }
}
```

### Template Method

```typescript
// Скелет алгоритма
abstract class DataProcessor {
  abstract loadData(): void;
  abstract processData(): void;
  abstract saveData(): void;

  process(): void {
    this.loadData();
    this.processData();
    this.saveData();
  }
}

class CsvProcessor extends DataProcessor {
  loadData() { console.log('Load CSV'); }
  processData() { console.log('Process CSV'); }
  saveData() { console.log('Save CSV'); }
}

class JsonProcessor extends DataProcessor {
  loadData() { console.log('Load JSON'); }
  processData() { console.log('Process JSON'); }
  saveData() { console.log('Save JSON'); }
}

// Usage
const processor: DataProcessor = new CsvProcessor();
processor.process();
```

---

## 🔗 Связанные заметки

- [[MOC-Patterns]] — Patterns index
- [[Clean-Architecture]] — Clean Architecture
- [[Backend-Architecture-MOC]] — Backend architecture
