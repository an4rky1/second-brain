---
created: 2026-02-16
tags:
  - cheat-sheet
  - angular
  - frontend
  - typescript
aliases:
  - Angular Cheatsheet
  - Angular Reference
related:
  - TypeScript-Cheatsheet
  - React-Cheatsheet
  - RxJS-Basics
---

# Angular — Полная шпаргалка

> [!SUMMARY] Обзор
> Angular — полноценный TypeScript фреймворк от Google. Двустороннее绑定, dependency injection, RxJS, модульная архитектура. Для enterprise приложений.

---

## 📚 Теория

### Архитектура

```
┌─────────────────────────────────────────────────────┐
│                  Module (NgModule)                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │
│  │ Components  │  │  Services   │  │   Pipes     │  │
│  └─────────────┘  └─────────────┘  └─────────────┘  │
│  ┌─────────────┐  ┌─────────────┐                   │
│  │  Directives │  │   Guards    │                   │
│  └─────────────┘  └─────────────┘                   │
└─────────────────────────────────────────────────────┘
```

### Жизненный цикл компонента

```typescript
ngOnChanges()      // При изменении @Input
ngOnInit()         // После первого ngOnChanges
ngDoCheck()        // При каждом обнаружении изменений
ngAfterContentInit()   // После проекции контента
ngAfterContentChecked() // После проверки контента
ngAfterViewInit()      // После инициализации view
ngAfterViewChecked()   // После проверки view
ngOnDestroy()          // Перед уничтожением
```

---

## ⚡ Быстрый старт

```bash
# Установка Angular CLI
npm install -g @angular/cli

# Создание проекта
ng new my-app --routing --style=scss --strict
cd my-app

# Генерация
ng generate component users        # ng g c users
ng generate service users          # ng g s users
ng generate module users           # ng g m users
ng generate guard auth             # ng g g auth
ng generate interceptor logging    # ng g i logging

# Запуск
ng serve              # http://localhost:4200
ng serve --open       # Открыть в браузере
ng build              # Production build
ng build --watch      # Watch mode

# Тесты
ng test               # Unit tests
ng e2e                # E2E tests

# Обновление
ng update @angular/core @angular/cli
```

### Структура проекта

```
src/
├── app/
│   ├── core/              # Singleton сервисы, guards, interceptors
│   │   ├── guards/
│   │   ├── interceptors/
│   │   ├── services/
│   │   └── core.module.ts
│   ├── shared/            # Переиспользуемые компоненты, pipes, directives
│   │   ├── components/
│   │   ├── pipes/
│   │   ├── directives/
│   │   └── shared.module.ts
│   ├── features/          # Функциональные модули
│   │   ├── users/
│   │   │   ├── components/
│   │   │   ├── services/
│   │   │   ├── users-routing.module.ts
│   │   │   └── users.module.ts
│   │   └── auth/
│   ├── app-routing.module.ts
│   ├── app.component.ts
│   └── app.module.ts
├── assets/
├── environments/
└── styles/
```

---

## 🔧 Практические примеры

### Components

```typescript
// users.component.ts
import {
  Component,
  OnInit,
  OnDestroy,
  Input,
  Output,
  EventEmitter,
  ViewChild,
  ElementRef,
} from '@angular/core';
import { Subject } from 'rxjs';
import { takeUntil } from 'rxjs/operators';

@Component({
  selector: 'app-users',
  templateUrl: './users.component.html',
  styleUrls: ['./users.component.scss'],
  standalone: false,  // Или true для standalone components (Angular 14+)
})
export class UsersComponent implements OnInit, OnDestroy {
  // Input
  @Input() users: User[] = [];
  @Input() loading = false;
  
  // Output
  @Output() userSelected = new EventEmitter<User>();
  @Output() deleteRequested = new EventEmitter<number>();
  
  // ViewChild
  @ViewChild('searchInput') searchInput!: ElementRef;
  
  // Local state
  searchTerm = '';
  private destroy$ = new Subject<void>();
  
  constructor(private userService: UserService) {}
  
  ngOnInit(): void {
    this.userService.users$
      .pipe(takeUntil(this.destroy$))
      .subscribe(users => this.users = users);
  }
  
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
  
  onSelectUser(user: User): void {
    this.userSelected.emit(user);
  }
  
  onDeleteUser(id: number): void {
    this.deleteRequested.emit(id);
  }
}

// Template
/*
<div class="users">
  <input #searchInput [value]="searchTerm" (input)="searchTerm = $event.target.value" />
  
  @if (loading) {
    <app-loading />
  } @else {
    @for (user of users; track user.id) {
      <app-user-card
        [user]="user"
        (selected)="onSelectUser($event)"
        (delete)="onDeleteUser($event.id)"
      />
    } @empty {
      <p>No users found</p>
    }
  }
</div>
*/
```

### Directives

```typescript
// Structural directive
import {
  Directive,
  Input,
  TemplateRef,
  ViewContainerRef,
} from '@angular/core';

@Directive({
  selector: '[appUnless]',
  standalone: true,
})
export class UnlessDirective {
  @Input() set appUnless(condition: boolean) {
    if (!condition) {
      this.viewContainer.createEmbeddedView(this.templateRef);
    } else {
      this.viewContainer.clear();
    }
  }
  
  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef,
  ) {}
}

// Attribute directive
@Directive({
  selector: '[appHighlight]',
  standalone: true,
})
export class HighlightDirective {
  @Input('appHighlight') color = 'yellow';
  
  constructor(private el: ElementRef) {}
  
  @HostListener('mouseenter') onMouseEnter() {
    this.el.nativeElement.style.backgroundColor = this.color;
  }
  
  @HostListener('mouseleave') onMouseLeave() {
    this.el.nativeElement.style.backgroundColor = null;
  }
}

// Использование
/*
<div *appUnless="isLoggedIn">
  Please log in
</div>

<p appHighlight="lightblue">Hover me</p>
*/
```

### Pipes

```typescript
// Custom pipe
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'truncate',
  standalone: true,
})
export class TruncatePipe implements PipeTransform {
  transform(value: string, limit = 50, completeWords = false): string {
    if (!value || value.length <= limit) return value;
    
    if (completeWords) {
      return value.substring(0, value.lastIndexOf(' ', limit)) + '...';
    }
    
    return value.substring(0, limit) + '...';
  }
}

// Impure pipe (вызывается при каждом change detection)
@Pipe({
  name: 'sortBy',
  pure: false,
  standalone: true,
})
export class SortByPipe implements PipeTransform {
  transform<T>(array: T[], key: keyof T, direction: 'asc' | 'desc' = 'asc'): T[] {
    if (!array) return array;
    
    return [...array].sort((a, b) => {
      if (a[key] < b[key]) return direction === 'asc' ? -1 : 1;
      if (a[key] > b[key]) return direction === 'asc' ? 1 : -1;
      return 0;
    });
  }
}

// Использование
/*
{{ longText | truncate:100 }}
{{ items | sortBy:'name':'desc' | async }}
*/
```

### Services & Dependency Injection

```typescript
// Service
import { Injectable } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable, of } from 'rxjs';
import { catchError, tap } from 'rxjs/operators';

@Injectable({
  providedIn: 'root',  // Singleton во всём приложении
})
export class UserService {
  private apiUrl = '/api/users';
  
  constructor(private http: HttpClient) {}
  
  getUsers(): Observable<User[]> {
    return this.http.get<User[]>(this.apiUrl);
  }
  
  getUser(id: number): Observable<User> {
    return this.http.get<User>(`${this.apiUrl}/${id}`);
  }
  
  createUser(user: CreateUserDto): Observable<User> {
    return this.http.post<User>(this.apiUrl, user);
  }
  
  // Кэширование
  private usersCache$?: Observable<User[]>;
  
  getUsersCached(): Observable<User[]> {
    if (!this.usersCache$) {
      this.usersCache$ = this.getUsers().pipe(
        tap(() => this.usersCache$ = undefined),  // Сброс кэша при ошибке
        catchError(() => {
          this.usersCache$ = undefined;
          throw Error('Failed to fetch users');
        })
      );
    }
    return this.usersCache$;
  }
}

// Injection Token
export const API_URL = new InjectionToken<string>('API_URL');

// Использование
providers: [
  { provide: API_URL, useValue: 'https://api.example.com' }
]

// Factory provider
export const loggerFactory = (http: HttpClient) => {
  return new LoggerService(http);
};

providers: [
  {
    provide: LoggerService,
    useFactory: loggerFactory,
    deps: [HttpClient],
  }
]
```

### Routing

```typescript
// app-routing.module.ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';
import { AuthGuard } from './core/guards/auth.guard';

const routes: Routes = [
  { path: '', redirectTo: '/dashboard', pathMatch: 'full' },
  {
    path: 'dashboard',
    loadChildren: () => import('./features/dashboard/dashboard.module')
      .then(m => m.DashboardModule),
    canActivate: [AuthGuard],
  },
  {
    path: 'users',
    loadChildren: () => import('./features/users/users.module')
      .then(m => m.UsersModule),
  },
  {
    path: 'users/:id',
    loadComponent: () => import('./features/users/user-detail.component')
      .then(m => m.UserDetailComponent),
  },
  { path: '**', redirectTo: 'dashboard' },  // Wildcard route
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule],
})
export class AppRoutingModule {}

// Guards
@Injectable({ providedIn: 'root' })
export class AuthGuard implements CanActivate {
  constructor(private authService: AuthService, private router: Router) {}
  
  canActivate(route: ActivatedRouteSnapshot): boolean {
    if (this.authService.isAuthenticated()) {
      return true;
    }
    this.router.navigate(['/login']);
    return false;
  }
}

// Resolve guard
@Injectable({ providedIn: 'root' })
export class UserResolver implements Resolve<User> {
  constructor(private userService: UserService) {}
  
  resolve(route: ActivatedRouteSnapshot): Observable<User> {
    return this.userService.getUser(+route.params['id']);
  }
}

// Использование в компоненте
class UserDetailComponent implements OnInit {
  user!: User;
  
  constructor(private route: ActivatedRoute) {}
  
  ngOnInit(): void {
    this.user = this.route.snapshot.data['user'];
    // или
    this.route.data.subscribe(data => this.user = data['user']);
  }
}
```

### HTTP Client

```typescript
// Interceptor
@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const token = localStorage.getItem('token');
    
    const authReq = req.clone({
      headers: req.headers.set('Authorization', `Bearer ${token}`),
    });
    
    return next.handle(authReq).pipe(
      catchError((error: HttpErrorResponse) => {
        if (error.status === 401) {
          // Redirect to login
        }
        return throwError(() => error);
      })
    );
  }
}

// Error interceptor
@Injectable()
export class ErrorInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    return next.handle(req).pipe(
      catchError((error: HttpErrorResponse) => {
        let errorMsg = 'An error occurred';
        
        if (error.error instanceof ErrorEvent) {
          // Client error
          errorMsg = `Error: ${error.error.message}`;
        } else {
          // Server error
          errorMsg = `Error Code: ${error.status}\nMessage: ${error.message}`;
        }
        
        return throwError(() => new Error(errorMsg));
      })
    );
  }
}

// Регистрация
providers: [
  { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true },
  { provide: HTTP_INTERCEPTORS, useClass: ErrorInterceptor, multi: true },
]

// Использование
this.http.get<User[]>('/api/users')
  .pipe(
    catchError(this.handleError)
  )
  .subscribe(users => this.users = users);

private handleError(error: HttpErrorResponse) {
  console.error('Error:', error);
  return throwError(() => new Error('Something went wrong'));
}
```

### RxJS Patterns

```typescript
import { Observable, Subject, BehaviorSubject, combineLatest, forkJoin } from 'rxjs';
import { map, filter, switchMap, debounceTime, distinctUntilChanged, takeUntil } from 'rxjs/operators';

// Subject
private searchSubject = new Subject<string>();
searchResults$ = this.searchSubject.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(term => this.searchService.search(term))
);

onSearch(term: string): void {
  this.searchSubject.next(term);
}

// BehaviorSubject (текущее значение + новые)
private currentUserSubject = new BehaviorSubject<User | null>(null);
currentUser$ = this.currentUserSubject.asObservable();

login(user: User): void {
  this.currentUserSubject.next(user);
}

// combineLatest (комбинация последних значений)
filteredUsers$ = combineLatest([
  this.users$,
  this.filter$,
]).pipe(
  map(([users, filter]) => users.filter(u => u.name.includes(filter)))
);

// forkJoin (ожидание всех запросов)
loadDashboard() {
  return forkJoin({
    users: this.http.get<User[]>('/users'),
    posts: this.http.get<Post[]>('/posts'),
    comments: this.http.get<Comment[]>('/comments'),
  });
}

// Async pipe в template
/*
{{ currentUser$ | async | json }}

@for (user of users$ | async; track user.id) {
  <app-user [user]="user" />
}
*/
```

---

## 🎯 Best Practices

### ✅ Делать

```typescript
// 1. Standalone components (Angular 14+)
@Component({
  selector: 'app-user',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule],
  template: '...',
})

// 2. Signals (Angular 16+)
import { Component, signal, computed, effect } from '@angular/core';

@Component({...})
export class CounterComponent {
  count = signal(0);
  double = computed(() => this.count() * 2);
  
  constructor() {
    effect(() => console.log(`Count: ${this.count()}`));
  }
  
  increment() {
    this.count.update(c => c + 1);
  }
}

// 3. Unsubscribe с async pipe
/*
{{ data$ | async }}
*/

// 4. TrackBy в ngFor
/*
@for (item of items; track item.id) {
  <app-item [item]="item" />
}
*/

// 5. Lazy loading модулей
loadChildren: () => import('./module').then(m => m.Module)
```

### ❌ Не делать

```typescript
// 1. Избегать subscribe в компонентах
this.service.getData().subscribe(data => {  // ❌
  this.data = data;
});

this.data$ = this.service.getData();  // ✅ (async pipe в template)

// 2. Не создавать Observable в template
/*
{{ getData() | async }}  // ❌ вызывается каждый change detection
*/
data$ = this.service.getData();  // ✅
/*
{{ data$ | async }}
*/

// 3. Избегать any
data: any;  // ❌
data: User[];  // ✅

// 4. Не забывать unsubscribe
private destroy$ = new Subject<void>();

ngOnInit() {
  this.service.data$
    .pipe(takeUntil(this.destroy$))
    .subscribe();
}

ngOnDestroy() {
  this.destroy$.next();
  this.destroy$.complete();
}
```

---

## 🔗 Связанные заметки

- [[TypeScript-Cheatsheet]] — TypeScript основы
- [[React-Cheatsheet]] — React для сравнения
- [[RxJS-Basics]] — RxJS операторы

---

## 📝 Заметки

> [!TIP] Совет от 15 лет опыта
> 
> 1. **Standalone Components** — будущее Angular
> 2. **Signals** — новая реактивность (Angular 16+)
> 3. **Async Pipe** — автоматический unsubscribe
> 4. **Lazy Loading** — обязательно для больших приложений
> 5. **Angular DevTools** — для отладки

> [!INFO] Полезные библиотеки
> ```bash
> # UI
> npm install @angular/material
> npm install @ng-bootstrap/ng-bootstrap
> npm install primeNG
> 
> # State
> npm install @ngrx/store @ngrx/effects  # NgRx
> npm install @ngxs/store  # NgXS
> 
> # Forms
> npm install @angular/forms  # Встроенные
  
> # Utils
> npm install lodash-es
> npm install moment  # или date-fns
> ```
