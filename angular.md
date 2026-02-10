# Frontend Angular Concepts: Comprehensive Analysis

This document provides a detailed breakdown of all Angular concepts used in the `Frontend` project (MediaConnect), mapped directly to the files where they appear.

## 1. Project Architecture & Configuration

The application is built using **Angular 17+ (Standalone Architecture)**, moving away from `ngModules` in favor of standalone components and provider functions.

### Core Configuration Files
- **`src/app/app.ts`**: Root Component.
  - **Concept**: `standalone: true`. It imports `RouterOutlet`, `RouterModule`, and `CommonModule` directly. This is the entry point for the component tree.
- **`src/app/app.config.ts`**: Application Configuration.
  - **Concept**: `ApplicationConfig`.
  - **Providers**:
    - `provideRouter(routes)`: Sets up routing.
    - `provideHttpClient(withInterceptors([jwtInterceptor]))`: Configures HTTP client with interceptors globally.
- **`src/app/app.routes.ts`**: Route Definitions.
  - **Concept**: `Routes` array.
  - **Features**:
    - **Redirects**: `{ path: '', redirectTo: 'login', pathMatch: 'full' }`.
    - **Guarded Routes**: `canActivate: [AuthGuard]`.
    - **Parameterized Routes**: `path: 'movie/:id'`.
    - **Component Mapping**: Maps URLs to standalone components (e.g., `LoginComponent`, `HomeComponent`).

---

## 2. Components & Their Concepts

The application uses **Standalone Components** throughout. Below is a breakdown of concepts per component.

### Authentication Components
- **`LoginComponent`** (`src/app/components/login/login.component.ts`):
  - **Reactive Forms**: Uses `FormGroup`, `FormControl` (via `FormBuilder`), and `Validators`.
  - **Dependency Injection**: Injects `AuthService`, `Router`, `FormBuilder`.
  - **Navigation**: Uses `this.router.navigate(['/path'])` after successful login.
  - **Template Logic**:
    - Control Flow: `@if` for showing error messages.
    - Binding: `[formGroup]`, `(ngSubmit)`, `[class.invalid]`.

- **`RegisterComponent`** (`src/app/components/register/register.component.ts`):
  - **Form Validation**: Complex validators (e.g., matching passwords, likely implemented manually or via validators).
  - **HTTP Interaction**: Calls `AuthService.register()`.

### Feature Components
- **`ManageSubscriptionComponent`** (`src/app/components/manage-subscription/manage-subscription.component.ts`):
  - **Signals (Modern Angular)**:
    - **State**: `selectedPlan = signal<SubscriptionPlan | null>(null)`.
    - **Computed Values**: `cardNumberError = computed(() => ...)` derived from other signals.
    - **RxJS Interop**: `userSignal = toSignal(this.authService.currentUser$)` converts an Observable to a Signal.
  - **Change Detection**: `ChangeDetectionStrategy.OnPush` for performance.
  - **Manual Two-Way Binding**: `[ngModel]="cardNumber()"` and `(ngModelChange)="cardNumber.set($event)"` to update signals from template.

- **`MovieCardComponent`** (`src/app/components/movie-card/movie-card.component.ts`):
  - **Input Property**: `@Input() movie!: Movie;` allows parent components (`HomeComponent`, `AdminMoviesComponent`) to pass data into it.
  - **Presentation Component**: Focuses on displaying data ("Dumb Component").

- **`HomeComponent`** (`src/app/components/home/home.component.ts`):
  - **Lifecycle Hooks**: `ngOnInit()` to fetch movies on load.
  - **Structural Directives**: `@for` loop in HTML to render lists of `app-movie-card`.
  - **Service Calls**: Calls `MovieService.getAllMovies()`.

- **`ProfileComponent`** (`src/app/components/profile/profile.component.ts`):
  - **User Data Display**: Fetches and displays user details.
  - **Protected Resource**: Accessed only if logged in.

- **`AdminDashboardComponent`, `AddMovieComponent`, `EditMovieComponent`**:
  - **Admin Logic**: Specific components protected by role checks (implemented in guards/services).
  - **CRUD Operations**: `AddMovie` and `EditMovie` likely use Forms to Create/Update movie data via `MovieService`.

---

## 3. Services & State Management

Services use the **`@Injectable({ providedIn: 'root' })`** pattern, making them singletons available throughout the app.

### `AuthService` (`src/app/services/auth.service.ts`)
- **State Management (RxJS)**:
  - Uses `BehaviorSubject` (`currentUserSubject`) to hold the current user state.
  - Exposes `currentUser$` as an `Observable` for components to subscribe to.
- **Persistence**: Usage of `localStorage` to persist JWT tokens and User data across refreshes.
- **Navigation**: Programmatic navigation based on Role (`ADMIN` vs `USER`).
- **Helper Methods**: `isLoggedIn()`, `logout()`, `getToken()`.

### Feature Services
- **`MovieService`** (`src/app/services/movie.service.ts`): Handles API calls for Movies (CRUD).
- **`SubscriptionService`** (`src/app/services/subscription.service.ts`): Fetches Plans and handles subscription payments.
- **`UserService`** (`src/app/services/user.service.ts`): Fetches User profile data.
- **`WatchHistoryService`** (`src/app/services/watch-history.service.ts`): Manages user viewing history.

---

## 4. Routing Guards & Interceptors

### `AuthGuard` (`src/app/guards/auth.guard.ts`)
- **Concept**: `CanActivate` interface.
- **Logic**: Checks `authService.isAuthenticated()`. If false, redirects to `/login`.
- **Usage**: Applied to sensitive routes in `app.routes.ts`.

### `JwtInterceptor` (`src/app/interceptors/jwt.interceptor.ts`)
- **Concept**: `HttpInterceptorFn` (Functional Interceptor).
- **Logic**:
  - Intercepts every outgoing HTTP request.
  - Injects `AuthService` to retrieve the Token.
  - Clones the request and adds the `Authorization: Bearer <token>` header if a token exists.
- **Configuration**: Registered in `app.config.ts`.

---

## 5. Template Syntax & Directives

### Control Flow (New Syntax)
- **`@if` ... `@else`**: Used extensively (e.g., `LoginComponent`, `ManageSubscriptionComponent`) to conditionally render errors or UI states.
- **`@for` ... `track`**: Used for rendering lists (e.g., `HomeComponent` movie lists, `ManageSubscriptionComponent` plans).

### Binding Syntax
- **Property Binding `[]`**:
  - `[formGroup]`, `[disabled]`, `[class.invalid]`, `[routerLink]`.
  - `[class.selected]`: Conditional class application.
- **Event Binding `()`**:
  - `(ngSubmit)`, `(click)`, `(ngModelChange)`.
- **Interpolation `{{ }}`**: Displaying dynamic text (e.g., `{{ user.fullName }}`).

### Built-in Directives
- **`ngClass`**: Dynamic styling based on expressions.
- **`routerLink`**: Navigation without page reload.

---

## 6. HTTP Client Pattern

### Concepts
- **`HttpClient`**: Injected into services.
- **`Observable`**: Return type of all HTTP methods.
- **`.subscribe()`**: Components subscribe to observables to trigger requests and handle responses.
- **Error Handling**: `error: (err) => ...` callback in subscriptions.
- **Pipe / Tap**: Used in `AuthService` to side-effect (save token) without altering the stream.

---

## 7. Angular Features Summary Table

| Feature | Usage Strategy | Key Files |
| :--- | :--- | :--- |
| **Component Type** | Standalone Component | All Components |
| **Form Handling** | Reactive Forms | `LoginComponent`, `RegisterComponent` |
| **State Management** | RxJS BehaviorSubject + Signals | `AuthService` (RxJS), `ManageSubscriptionComponent` (Signals) |
| **Routing** | Component Router | `app.routes.ts` |
| **Security** | Route Guards + Interceptors | `AuthGuard`, `JwtInterceptor` |
| **Change Detection** | Default + OnPush | `ManageSubscriptionComponent` (OnPush) |
| **Styling** | Component-Scoped CSS | All `.css` files |

---

## 8. Understanding Imports

This section explains the purpose of the common `import` statements found in the codebase.

### Core Angular Imports
- **`import { Component } from '@angular/core';`**: The fundamental decorator used to define a class as an Angular component. It allows specifying the `selector`, `templateUrl`, `styleUrls`, and `standalone` properties.
- **`import { Injectable } from '@angular/core';`**: Marks a class as available to be provided and injected as a dependency. Used in all services (e.g., `AuthService`).
- **`import { inject } from '@angular/core';`**: A modern, functional way to inject dependencies without constructor arguments. Used in `ManageSubscriptionComponent` and Interceptors.
- **`import { signal, computed } from '@angular/core';`**: Imports the Signals API, used for fine-grained reactivity in `ManageSubscriptionComponent`.
- **`import { OnInit } from '@angular/core';`**: Lifecycle hook interface. Forces the class to implement `ngOnInit()`, which runs after the component initializes.

### Common Angular Modules
- **`import { CommonModule } from '@angular/common';`**: Exports all the basic Angular directives and pipes, such as `ngIf` (legacy), `ngFor` (legacy), `ngClass`, `DatePipe`, etc. In standalone components, this is often imported to access these utilities.
- **`import { HttpClient } from '@angular/common/http';`**: The service used to perform HTTP requests. It returns Observables.
- **`import { HttpInterceptorFn } from '@angular/common/http';`**: Type definition for writing functional HTTP interceptors (like `jwtInterceptor`).

### Routing Imports
- **`import { Router, RouterModule } from '@angular/router';`**: 
  - `Router`: The service used to navigate programmatically (e.g., `this.router.navigate(['/home'])`).
  - `RouterModule`: Used in the component's `imports` array to enable `routerLink` in the HTML.
- **`import { ActivatedRoute } from '@angular/router';`**: Used to access route parameters (e.g., `movie/:id` in `MovieDetailComponent`).
- **`import { Routes } from '@angular/router';`**: Type definition for the route configuration array in `app.routes.ts`.

### Forms Imports
- **`import { FormBuilder, FormGroup, Validators } from '@angular/forms';`**:
  - `FormBuilder`: A service helper to create form controls easily.
  - `FormGroup`: A collection of form controls (used in `LoginComponent`).
  - `Validators`: Built-in validation functions (e.g., `.required`, `.email`).
- **`import { ReactiveFormsModule, FormsModule } from '@angular/forms';`**: 
  - `ReactiveFormsModule`: Needed for `[formGroup]` syntax.
  - `FormsModule`: Needed for `[(ngModel)]` syntax (used in subscription forms).

### RxJS Imports (Reactive Library)
- **`import { Observable, BehaviorSubject } from 'rxjs';`**:
  - `Observable`: Represents a stream of data over time (e.g., HTTP responses).
  - `BehaviorSubject`: A type of Observable that holds a current value (used in `AuthService` state).
- **`import { tap } from 'rxjs/operators';`**: operator used to perform side effects (like logging or saving to localStorage) without modifying the data stream.

---

## 9. Complete List of Angular Concepts Used in This Project

Below is a quick-reference list of every Angular-specific concept, keyword, or decorator found in the source code.

### Decorators
- **`@Component`**: Defines a component.
- **`@Injectable`**: Defines a service.
- **`@Input`**: Allows data to flow *into* a component (used in `MovieCardComponent`).
- **`@Output`**: Allows data to flow *out* of a component (imported but not actively used in the searched files).

### Directives & Template Syntax
- **`@if` / `@else`**: Control flow (Conditionally show elements).
- **`@for`**: Control flow (Loop through lists).
- **`[property]`**: Property binding (one-way, component to view).
- **`(event)`**: Event binding (view to component).
- **`[(ngModel)]`**: Two-way data binding.
- **`ngClass`**: Dynamically add/remove CSS classes.
- **`routerLink`**: Navigate to a new route.

### Architecture & DI
- **`standalone: true`**: Defines a component as standalone (no NgModule required).
- **`imports: []`**: Array in `@Component` to bring in dependencies like `CommonModule`.
- **`providers: []`**: Function in `app.config.ts` to register global services.
- **`inject()`**: Functional dependency injection.

### Routing
- **`Routes`**: Array defining the application's navigation paths.
- **`RouterOutlet`**: The placeholder where the routed component is rendered.
- **`CanActivate`**: Interface for Route Guards.

### Forms
- **`ReactiveFormsModule`**: Module for model-driven forms.
- **`FormGroup` / `FormControl`**: Classes representing form state.
- **`Validators`**: Static methods for validation rules.

### HTTP & Async
- **`HttpClient`**: Service for making REST calls.
- **`HttpInterceptorFn`**: Functional interceptor for Modifying requests.
- **`Observable`**: Stream of async events.
- **`subscribe()`**: Executing the Observable.

### Modern Angular Features
- **`signal()`**: Create a reactive state primitive.
- **`computed()`**: Create a derived signal.
- **`toSignal()`**: Convert an Observable to a Signal.
- **`ChangeDetectionStrategy.OnPush`**: Performance optimization strategy.



# Angular Concepts Interview Preparation Guide

This guide covers core Angular concepts used in the **MediaConnect** project. For each concept, we provide a simple explanation, how it is actually used in your project, a basic code example, and a "Perfect Interview Answer".

---

## 1. Decorators

### `@Component`
*   **Explanation**: The most basic decorator that marks a class as an Angular component and provides metadata like the selector, template, and styles.
*   **In MediaConnect**: Used in every component file (e.g., `LoginComponent`, `HomeComponent`). It defines `standalone: true` to avoid using NgModules.
*   **Simple Example**:
    ```typescript
    @Component({
      selector: 'app-hello',
      template: '<h1>Hello World</h1>',
      standalone: true
    })
    export class HelloComponent {}
    ```
*   **🗣️ Interview Answer**: "The `@Component` decorator identifies a class as an Angular component. It allows us to configure the component's selector, HTML template, styles, and dependencies (via the `imports` array in standalone components). effectively turning a TypeScript class into a UI element."

### `@Injectable`
*   **Explanation**: Marks a class as a service that can be injected into other parts of the application.
*   **In MediaConnect**: Used in `AuthService`, `MovieService`, etc., with `providedIn: 'root'` to make them global singletons.
*   **Simple Example**:
    ```typescript
    @Injectable({ providedIn: 'root' })
    export class DataService { ... }
    ```
*   **🗣️ Interview Answer**: "`@Injectable` is used to define a service class. When we pass `{ providedIn: 'root' }`, Angular creates a single instance (singleton) of the service and makes it available throughout the application, which is efficient for sharing state and logic."

### `@Input`
*   **Explanation**: Allows a parent component to pass data *down* to a child component.
*   **In MediaConnect**: Used in `MovieCardComponent` (`@Input() movie!: Movie;`) to receive movie details from the `HomeComponent`.
*   **Simple Example**:
    ```typescript
    // Child
    @Input() title: string = '';
    // Parent HTML
    <app-child [title]="'Welcome'"></app-child>
    ```
*   **🗣️ Interview Answer**: "`@Input` is a decorator used for Parent-to-Child communication. It marks a property in the child component as settable from the parent's template using property binding syntax `[property]=value`."

### `@Output`
*   **Explanation**: Allows a child component to send data/events *up* to a parent component.
*   **In MediaConnect**: Imported but not heavily used, but typically used for button clicks or form submissions inside child components.
*   **Simple Example**:
    ```typescript
    // Child
    @Output() notify = new EventEmitter<string>();
    send() { this.notify.emit('Message'); }
    // Parent HTML
    <app-child (notify)="handleEvent($event)"></app-child>
    ```
*   **🗣️ Interview Answer**: "`@Output` is used for Child-to-Parent communication. It uses an `EventEmitter` to dispatch custom events that the parent can listen to using event binding syntax `(event)=handler()`."

---

## 2. Directives & Template Syntax

### Control Flow (`@if` / `@else`, `@for`)
*   **Explanation**: The new built-in block syntax for structural directives in Angular 17+.
*   **In MediaConnect**: 
    *   `@if`: Used in `LoginComponent` to show validation errors (`@if(loginForm.get('email')?.invalid)`).
    *   `@for`: Used in `HomeComponent` to list movies (`@for(movie of movies; track movie.id)`).
*   **Simple Example**:
    ```html
    @if (isAdmin) {
      <button>Delete</button>
    } @else {
      <p>View Only</p>
    }
    
    @for (item of items; track item.id) {
      <li>{{ item.name }}</li>
    }
    ```
*   **🗣️ Interview Answer**: "This is the new Control Flow syntax introduced in Angular 17. It replaces `*ngIf` and `*ngFor` with a cleaner, more performant syntax built directly into the template engine, removing the need to import `CommonModule` for basic logic."

### `[property]` (Property Binding)
*   **Explanation**: One-way binding from the Component (Logic) to the View (Template).
*   **In MediaConnect**: `[disabled]="isLoading"` in `LoginComponent` disables the button when loading.
*   **Simple Example**: `<img [src]="userImage" />`
*   **🗣️ Interview Answer**: "Property binding, denoted by square brackets `[]`, allows us to set an element's property dynamically based on a variable in our component class."

### `(event)` (Event Binding)
*   **Explanation**: One-way binding from the View (Template) to the Component (Logic) triggered by user actions.
*   **In MediaConnect**: `(ngSubmit)="onSubmit()"` in forms, `(click)="logout()"` in generic buttons.
*   **Simple Example**: `<button (click)="saveData()">Save</button>`
*   **🗣️ Interview Answer**: "Event binding, denoted by parentheses `()`, allows us to listen for DOM events like clicks or keyups and execute a method in our component class."

### `[(ngModel)]` (Two-Way Binding)
*   **Explanation**: Syncs data in both directions: View updates Model, Model updates View. Requires `FormsModule`.
*   **In MediaConnect**: Used in `ManageSubscriptionComponent` for the credit card inputs (`[(ngModel)]="cardNumber"`).
*   **Simple Example**: `<input [(ngModel)]="username" />`
*   **🗣️ Interview Answer**: "Two-way binding combines property binding and event listener. It's commonly used in template-driven forms to keep an input field and a typescript variable in perfect sync."

### `ngClass`
*   **Explanation**: Adds or removes CSS classes dynamically based on conditions.
*   **In MediaConnect**: `<div [ngClass]="responseType">` in `LoginComponent` to style alerts as red (error) or green (success).
*   **Simple Example**: `<div [ngClass]="{ 'active': isActive }">Menu Item</div>`
*   **🗣️ Interview Answer**: "`ngClass` is a directive that allows us to conditionally apply CSS classes. We can pass it a string, an array, or an object where keys are class names and values are boolean conditions."

---

## 3. Architecture & Dependency Injection

### `standalone: true`
*   **Explanation**: Allows a component to manage its own dependencies without needing an `NgModule`.
*   **In MediaConnect**: All components (e.g., `app.ts`) use this. They import `CommonModule`, `RouterOutlet` directly in the `@Component` metadata.
*   **🗣️ Interview Answer**: "Standalone components simplify Angular architecture by removing the need for `NgModules`. A component specifies exactly what imports it needs, making dependencies explicit and tree-shakable."

### `inject()`
*   **Explanation**: A function to inject dependencies into a class, field, or function, serving as an alternative to Constructor Injection.
*   **In MediaConnect**: Used in `ManageSubscriptionComponent`: `authService = inject(AuthService);`.
*   **🗣️ Interview Answer**: "`inject()` allows us to inject dependencies without declaring them in the constructor parameters. It's especially useful for functional guards, interceptors, and cleaner class fields."

---

## 4. Routing

### `Routes` & `RouterOutlet`
*   **Explanation**: `Routes` defines the map of URL paths to components. `RouterOutlet` is the placeholder where the matched component is rendered.
*   **In MediaConnect**: Defined in `app.routes.ts`. `<router-outlet>` is in `app.html` to swap between Login, Home, etc.
*   **🗣️ Interview Answer**: "Angular's Router maps URLs to components. We define a `Routes` array, and the router dynamically loads the correct component into the `<router-outlet>` placeholder based on the current browser URL."

### `CanActivate` (AuthGuard)
*   **Explanation**: Interface for a Guard that decides if a route can be activated (viewed).
*   **In MediaConnect**: `AuthGuard` checks `isAuthenticated()`. If false, it redirects to Login.
*   **Simple Example**:
    ```typescript
    canActivate() {
      return this.authService.isLoggedIn() ? true : this.router.parseUrl('/login');
    }
    ```
*   **🗣️ Interview Answer**: "`CanActivate` is a route guard interface. It runs logic before a route is loaded, allowing us to restrict access to secure pages based on authentication state or permissions."

---

## 5. Forms

### `ReactiveFormsModule`, `FormGroup`, `Validators`
*   **Explanation**: A technique to build forms where the logic is defined in the component class (explicit), offering better scalability than template-driven forms.
*   **In MediaConnect**: `LoginComponent` uses `FormBuilder` to create a `FormGroup` with `Validators.required` and `Validators.email`.
*   **Simple Example**:
    ```typescript
    loginForm = new FormGroup({
      email: new FormControl('', Validators.required)
    });
    ```
*   **🗣️ Interview Answer**: "Reactive Forms allow us to manage form state and validation programmatically in the component class. This provides immense control, easier testing, and better handling of complex validation logic compared to template-driven forms."

---

## 6. HTTP & Async

### `HttpClient` & `Observable`
*   **Explanation**: `HttpClient` is Angular's API for making HTTP requests. It returns `Observables` (streams of data).
*   **In MediaConnect**: `AuthService` uses `http.post<AuthResponse>(url, body)` to login users.
*   **🗣️ Interview Answer**: "`HttpClient` is Angular's mechanism for communicating with backends. It returns `Observables`, which are powerful streams that allow us to cancel requests, modify data with operators, and handle errors gracefully."

### `HttpInterceptorFn`
*   **Explanation**: A function that sits in the middle of HTTP requests/responses to transform them globally.
*   **In MediaConnect**: `jwtInterceptor` automatically attaches the generic `Authorization: Bearer token` header to every single API call.
*   **🗣️ Interview Answer**: "Interceptors act as middleware for HTTP requests. I use them to attach Authentication tokens to headers globally so I don't have to repeat that logic in every single service method."

---

## 7. Modern Angular Features

### Signals (`signal`, `computed`)
*   **Explanation**: A new reactive primitive in Angular that allows fine-grained tracking of state changes. When a signal changes, Angular knows exactly where to update the UI without traversing the whole tree.
*   **In MediaConnect**: `ManageSubscriptionComponent` uses:
    *   `cardNumber = signal('')`: Stores the input value.
    *   `cardNumberError = computed(...)`: Automatically recalculates validation error whenever `cardNumber` changes.
*   **Simple Example**:
    ```typescript
    count = signal(0);
    double = computed(() => this.count() * 2);
    increment() { this.count.set(this.count() + 1); }
    ```
*   **🗣️ Interview Answer**: "Signals are Angular's new system for fine-grained reactivity. Unlike RxJS subjects, Signals hold a current value and automatically notify dependency tracking contexts (like template views) when they change, leading to more efficient rendering."

### `ChangeDetectionStrategy.OnPush`
*   **Explanation**: Tells Angular to ONLY check this component for changes if an Input changes or a manual event/signal fires. It ignores global timers or unrelated events.
*   **In MediaConnect**: Used in `ManageSubscriptionComponent` to optimize performance since it uses Signals.
*   **🗣️ Interview Answer**: "`OnPush` strategy improves performance by detaching the component from the default change detection cycle. Angular will only re-render the component if its `@Input` references change or if a Signal inside it updates, rather than checking it on every single browser event."
## 8. File Handling & Browser APIs (From Profile Download)

### `Blob` & `window.URL.createObjectURL`
*   **Explanation**: `Blob` means "Binary Large Object". It represents raw data like PDFs or Images. `createObjectURL` is a standard browser API that creates a temporary unique URL for a Blob kept in memory.
*   **In MediaConnect**: Used in `ProfileComponent` for `downloadInvoice()`:
    1.  `UserService` calls API with `{ responseType: 'blob' }`.
    2.  `ProfileComponent` calls `window.URL.createObjectURL(blob)` to get a URL.
    3.  A hidden `<a>` tag is created, clicked programmatically, and then removed.
*   **Simple Example**:
    ```typescript
    const url = window.URL.createObjectURL(myBlob);
    const link = document.createElement('a'); 
    link.href = url;
    link.download = 'invoice.txt';
    link.click();
    ```
*   **🗣️ Interview Answer**: "To handle file downloads in Angular, we fetch the file as a Blob. Then, we use the browser's native `URL.createObjectURL` API to generate a temporary link, create a hidden anchor element, trigger a click event on it to start the download, and finally revoke the object URL to prevent memory leaks."

