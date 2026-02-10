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

# Angular Interview Questions Master List (60+ Questions)

This comprehensive guide covers both **concepts used in your MediaConnect project** and **general advanced Angular topics** that interviewers frequently ask.

---

## 🏗️ Section 1: Core Architecture & Standalone Components
*(Concepts used: `standalone: true`, `app.config.ts`, `main.ts`)*

**1. What is the difference between Standalone Components and NgModule-based components?**
*   **Answer**: Standalone components (`standalone: true`) manage their own dependencies via an `imports` array directly in the `@Component` decorator. They don't need to be declared in an `NgModule`. This reduces boilerplate and makes lazy loading easier.
*   **Demo**:
    ```typescript
    @Component({
      standalone: true,
      imports: [CommonModule],
      template: `<h1>Hello</h1>`
    })
    class HelloComponent {}
    ```

**2. How do you bootstrap an application in Angular 17+ without `AppModule`?**
*   **Answer**: We use the `bootstrapApplication` function in `main.ts`, passing the root component (e.g., `App`) and a configuration object (`ApplicationConfig`) that contains providers.
*   **Ref**: Your `app.config.ts`.

**3. What is the purpose of `CommonModule` in Angular?**
*   **Answer**: It exports standard directives and pipes like `ngIf`, `ngFor`, `ngClass`, `DatePipe`, etc. In standalone components, you must import it manually if you rely on these legacy directives (though new syntax `@if` doesn't need it).

**4. What is a Decorator in Angular?**
*   **Answer**: A function that attaches metadata to a class. Key decorators are `@Component` (views), `@Injectable` (services), `@Directive` (behavior), and `@Pipe` (transformation).

**5. What is `schemas: [CUSTOM_ELEMENTS_SCHEMA]`?**
*   **Answer**: It allows Angular to ignore unrecognized custom HTML elements during compilation. We avoided this in your project to ensure type safety.

**6. Explain the `angular.json` file.**
*   **Answer**: It is the configuration file for the Angular CLI workspace, defining build settings, assets paths, global styles (like your `index.css`), and environment configurations.

**7. What is View Encapsulation?**
*   **Answer**: It determines if component styles can affect other parts of the app.
    *   `Emulated` (Default): Styles are scoped to the component.
    *   `ShadowDom`: Uses native Shadow DOM.
    *   `None`: Styles are global.

**8. What is Binding in Angular? List the types.**
*   **Answer**:
    *   Interpolation: `{{ value }}`
    *   Property Binding: `[property]="value"`
    *   Event Binding: `(event)="handler()"`
    *   Two-Way Binding: `[(ngModel)]="value"`

**9. What is the difference between Constructor Injection vs `inject()`?**
*   **Answer**: Constructor injection declares deps in the `constructor(private srv: Service)`. The `inject(Service)` function is a newer, functional way to get dependencies that can be used in field initializers or functions without a constructor.
*   **Ref**: Your `ManageSubscriptionComponent` uses `inject()`.

**10. What is AOT (Ahead-of-Time) vs JIT (Just-in-Time) compilation?**
*   **Answer**:
    *   **AOT**: Compiles HTML/TS to JS *during the build phase* (faster render, smaller bundle). Default in prod.
    *   **JIT**: Compiles in the *browser* at runtime (slower).

---

## ⚡ Section 2: Signals & State Management
*(Concepts used: `signal()`, `computed()`, `BehaviorSubject`)*

**11. What are Angular Signals?**
*   **Answer**: Signals are a reactive primitive that holds a value and notifies consumers (like templates) when it changes. They enable fine-grained reactivity.
*   **Demo**: `count = signal(0);` → access with `count()`.

**12. Benefits of Signals over RxJS BehaviorSubject?**
*   **Answer**: Signals are synchronous and glitch-free. They don't require subscription management (no `.subscribe()` or `.unsubscribe()`), making them easier to use for view state.

**13. What is a `computed` signal?**
*   **Answer**: A read-only signal that derives its value from other signals. It updates automatically only when its dependencies change.
*   **Ref**: Your `cardNumberError = computed(...)`.

**14. What is `toSignal()`?**
*   **Answer**: An interoperability function that converts an RxJS Observable into a Signal.
*   **Ref**: Your `userSignal = toSignal(authService.currentUser$)`.

**15. What is `ChangeDetectionStrategy.OnPush`?**
*   **Answer**: It tells Angular to skip checking this component unless an `@Input` changes by reference or a Signal updates. It improves performance massively.

**16. What is Zone.js?**
*   **Answer**: A library that patches browser async events (clicks, timeouts) to tell Angular when to run change detection. (Note: Signals aim to eventually make Zone.js optional).

**17. RxJS `BehaviorSubject` vs `Subject`?**
*   **Answer**: `BehaviorSubject` requires an initial value and always emits the current value to new subscribers. `Subject` has no initial value and only emits subsequent values.
*   **Ref**: Your `AuthService` uses `BehaviorSubject` so new components immediately know if the user is logged in.

**18. What is the AsyncPipe (`| async`)?**
*   **Answer**: A pipe that subscribes to an Observable/Promise in the template and automatically unsubscribes when the component is destroyed.
    *   *Example*: `<div *ngIf="user$ | async as user">`.

---

## 📋 Section 3: Forms (Reactive & Template Driven)
*(Concepts used: `FormGroup`, `Validators`, `ngModel`)*

**19. Reactive Forms vs Template-Driven Forms?**
*   **Answer**:
    *   **Reactive**: explicit logic in component class (`FormGroup`), better testing/scaling. (Used in Login).
    *   **Template-Driven**: logic in HTML (`ngModel`), simpler for small forms. (Used in Subscription).

**20. functions of `FormBuilder`?**
*   **Answer**: A helper service to create `FormGroup` and `FormControl` instances with less boilerplate code.

**21. What are common built-in Validators?**
*   **Answer**: `Validators.required`, `Validators.email`, `Validators.minLength()`, `Validators.pattern()`.

**22. How to implement Custom Validators?**
*   **Answer**: Create a function that receives a `AbstractControl` and returns `ValidationErrors | null`.
    *   *Code*:
        ```typescript
        static noSpace(control: AbstractControl) {
            return control.value.includes(' ') ? { hasSpace: true } : null;
        }
        ```

**23. What are `.status` and `.valid` properties?**
*   **Answer**: Properties of a form control indicating its validation state (VALID, INVALID, PENDING, DISABLED).

**24. What is `markAllAsTouched()`?**
*   **Answer**: It triggers validation messages on all fields, commonly used when a user clicks "Submit" without filling the form.

---

## 🚦 Section 4: Routing & Guards
*(Concepts used: `canActivate`, `Router`, `routes`)*

**25. How do you define routes in Angular?**
*   **Answer**: By creating an array of `Route` objects (`{ path: 'home', component: HomeComponent }`) and passing it to `provideRouter`.

**26. What are the different types of Route Guards?**
*   **Answer**:
    *   `Include CanActivate`: Checks if we can visit a route.
    *   `CanDeactivate`: Checks if we can leave a route (e.g., Unsaved changes).
    *   `CanMatch`: Decides if a route definition matches.
    *   `Resolve`: Fetches data before route activation.

**27. What is Lazy Loading?**
*   **Answer**: Using `loadComponent` or `loadChildren` to fetch component code only when the user visits that route, reducing initial bundle size.
    *   *Example*: `{ path: 'admin', loadComponent: () => import('./admin.cmp').then(m => m.AdminCmp) }`.

**28. Difference between `routerLink` and `router.navigate()`?**
*   **Answer**: `routerLink` is a directive for HTML (`<a routerLink="/home">`), while `router.navigate()` is for programmatic navigation in TS (`this.router.navigate(['/home'])`).

**29. Application is showing 404 on refresh in production. Why?**
*   **Answer**: Angular adds routes on the client side. The server must be configured to fallback to `index.html` for unknown paths so Angular can handle the routing.

---

## 🌐 Section 5: HTTP & Interceptors
*(Concepts used: `HttpClient`, `InterceptorFn`, `Blob`)*

**30. What is an HTTP Interceptor?**
*   **Answer**: Middleware that intercepts requests/responses. Used for adding Auth tokens, logging, or global error handling.

**31. How to Handle Errors in HTTP?**
*   **Answer**: Use the `.pipe(catchError(error => ...))` operator in RxJS or the `error:` callback in `.subscribe()`.

**32. What is a `Blob` in HTTP context?**
*   **Answer**: A binary large object. We set `responseType: 'blob'` to download files (PDFs, Images) directly from an API.

**33. What is the difference between `mergeMap` and `switchMap`?**
*   **Answer**:
    *   `mergeMap`: Runs all inner observables in parallel.
    *   `switchMap`: Cancels the previous inner observable if a new value arrives (great for search typeaheads).

**34. Why do we need `subscribe()`?**
*   **Answer**: Observables are "lazy". The HTTP request is NOT sent until you call `.subscribe()`.

---

## 🛠️ Section 6: Lifecycle Hooks
*(Concepts used: `ngOnInit`)*

**35. List the Lifecycle Hooks in order.**
*   **Answer**:
    1.  `ngOnChanges` (Input updates)
    2.  `ngOnInit` (Initialization)
    3.  `ngDoCheck` (Custom change detection)
    4.  `ngAfterContentInit`
    5.  `ngAfterContentChecked`
    6.  `ngAfterViewInit` (View rendered)
    7.  `ngAfterViewChecked`
    8.  `ngOnDestroy` (Cleanup)

**36. `ngOnInit` vs Constructor?**
*   **Answer**: Constructor is for JS class initialization (DI). `ngOnInit` is for Angular initialization (fetching data like `getMovies()`) after inputs are bound.

**37. When to use `ngOnDestroy`?**
*   **Answer**: To unsubscribe from Observables, detach event listeners, or stop timers/intervals to prevent memory leaks.

---

## 🚀 Section 7: Advanced & Miscellaneous

**38. What is Content Projection (`ng-content`)?**
*   **Answer**: It allows you to insert HTML content from a parent into a designated slot in the child component.
    *   *Usage*: `<app-card> <h1>Title</h1> </app-card>` -> Child uses `<ng-content></ng-content>` to display `<h1>`.

**39. What is `ViewChild`?**
*   **Answer**: A decorator to access a child component, directive, or DOM element inside the parent component's class.

**40. What is a Pipe (`|`)?**
*   **Answer**: A class to transform data in the template (e.g., `{{ date | date:'short' }}`).
    *   *Pure Pipe*: Only re-runs if input reference changes (fast).
    *   *Impure Pipe*: Re-runs on every detection cycle (resource heavy).

**41. Types of Directives?**
*   **Answer**:
    1.  **Component**: Directive with a template.
    2.  **Structural**: Changes DOM layout (`@if`, `@for`).
    3.  **Attribute**: Changes appearance/behavior (`ngClass`, `ngStyle`).

**42. What is `track` in the new `@for` loop?**
*   **Answer**: It replaces `trackBy`. It provides a unique key (like `id`) for items in a list so Angular can optimize DOM updates when the list changes.

**43. What is Hydration (SSR)?**
*   **Answer**: In Angular 16+, hydration allows the client to "take over" server-rendered HTML without flickering or re-rendering the DOM from scratch.

**44. What is Deferrable Views (`@defer`)?**
*   **Answer**: A block (`@defer`) that allows you to lazy load the content inside it based on triggers (e.g., `@defer (on viewport)`) without complex routing.

**45. How do you optimize an Angular App?**
*   **Answer**:
    1.  Use `OnPush` change detection.
    2.  Lazy load routes.
    3.  Use `@defer` for heavy components.
    4.  Optimize images.
    5.  Prevent memory leaks (unsubscribe).

**46. What is the standard way to communicate between components?**
*   **Answer**:
    1.  Parent → Child: `@Input`.
    2.  Child → Parent: `@Output` / `EventEmitter`.
    3.  Sibling/Any: Shared Service (RxJS/Signals).

**47. What is `environment.ts`?**
*   **Answer**: A file to store config variables (API URLs, keys) that change between Dev and Prod builds.

**48. Why use TypeScript with Angular?**
*   **Answer**: Strong typing prevents runtime errors, provides better IDE support (Intellisense), and makes refactoring safer.

**49. What is `HostListener`?**
*   **Answer**: A decorator that declares a DOM event to listen directly on the component's host element (e.g., listening for window scroll).

**50. What is `HostBinding`?**
*   **Answer**: Allows you to set properties of the host element (like adding a class) from within the component class.

---

## 🔍 Section 8: Rapid Fire / Scenario Questions

**51. User reports the UI freezes when typing. Probable cause?**
*   **Answer**: Likely performing heavy calculation on every keystroke in a default Change Detection cycle or an impure pipe. Fix: Use `OnPush` or `debounceTime` in RxJS.

**52. How to prevent a user from leaving a form with unsaved changes?**
*   **Answer**: Use a `CanDeactivate` guard that checks the `isDirty` state of the form.

**53. How to pass data between unlinked components (e.g., Header and Settings)?**
*   **Answer**: Use a Shared Service with a `BehaviorSubject` or `Signal`.

**54. Diff between `ng-template` and `ng-container`?**
*   **Answer**:
    *   `ng-template`: Defines a template that is NOT rendered by default (used in `else` blocks).
    *   `ng-container`: A logical container that doesn't put an element in the DOM (good for grouping `@if` or loops).

**55. How to handle multiple HTTP calls that must finish before loading the page?**
*   **Answer**: Use `forkJoin([obs1$, obs2$])`.

**56. How to cancel an ongoing HTTP request?**
*   **Answer**: Unsubscribe from the observable.

**57. What is a Resolver?**
*   **Answer**: A script that fetches data *before* the route component is initialized, ensuring data is ready when the page loads.

**58. Explain `ChangeDetectorRef.detectChanges()`?**
*   **Answer**: Manually triggers a change detection cycle. Used when Angular doesn't auto-detect a change (e.g., updates inside a third-party library or outside Zone.js).
*   **Ref**: You used this in `ProfileComponent`.

**59. What is the difference between `package.json` and `package-lock.json`?**
*   **Answer**: `package.json` lists version ranges. `package-lock.json` locks the *exact* versions installed to ensure consistency across teams.

**60. How to debug an Angular app?**
*   **Answer**: Angular DevTools (Chrome Extension), `debugger;` statement, `console.log`, and the Network Tab.


## 1. The Authentication Flow (Step-by-Step)

The authentication process involves four key files working in harmony:
1.  **`LoginComponent`** (The UI Trigger)
2.  **`AuthService`** (The Logic & State)
3.  **`JwtInterceptor`** (The HTTP Middleware)
4.  **`AuthGuard`** (The Route Protector)

### Step 1: User Log In (`LoginComponent`)
*   **File**: `src/app/components/login/login.component.ts`
*   **Action**: User enters credentials and clicks "Login".
*   **Code Flow**:
    1.  The form triggers `onSubmit()`.
    2.  It calls `this.authService.login(formData)`.
    3.  If successful, it navigates to `/home` (or `/admin-dashboard`).

### Step 2: Storing the Token (`AuthService`)
*   **File**: `src/app/services/auth.service.ts`
*   **Action**: The service communicates with the backend and saves the session.
*   **Key Concept**: Uses the RxJS `tap` operator to perform side effects (saving data) without changing the response.
*   **Code Explanation**:
    ```typescript
    login(request): Observable<AuthResponse> {
        return http.post(url, request).pipe(
            tap(response => {
                // 1. Save Token to LocalStorage (Persistence)
                localStorage.setItem('token', response.token);
                
                // 2. Update Global State (BehaviorSubject)
                this.currentUserSubject.next(response);
            })
        );
    }
    ```

### Step 3: Protecting Routes (`AuthGuard`)
*   **File**: `src/app/guards/auth.guard.ts`
*   **Action**: Before the Router loads `/home`, it checks this guard.
*   **Logic**:
    1.  Calls `authService.isAuthenticated()`.
    2.  If `true`: Allows navigation.
    3.  If `false`: Redirects to `/login`.

---

## 2. The HTTP Request Flow (Interceptor)

Once logged in, every time the `HomeComponent` asks for movies, the **`JwtInterceptor`** silently does the heavy lifting.

**File**: `src/app/interceptors/jwt.interceptor.ts`

### How it works:
The interceptor sits between your Service and the Backend.

1.  **Intercept**: Catches the outgoing request.
2.  **Inject**: Gets the `AuthService` to retrieve the current token.
3.  **Clone & Modify**: Creates a copy of the request and adds the header:
    `Authorization: Bearer eyJhbGciOi...`
4.  **Forward**: Sends the modified request to the backend.

**Visual Flow:**
`MovieService.getMovies()` --> **INTERCEPTOR** (Adds Token) --> **API** (Server)

**Why is this important?**
Without this, you would have to manually add `headers: { Authorization: ... }` in *every single* service method (User, Movies, Subscription, etc.). The interceptor keeps your code DRY (Don't Repeat Yourself).

---

## 3. Data Fetching Example (`HomeComponent`)

**File**: `src/app/components/home/home.component.ts`

Now that Auth is set up, here is how data is actually loaded:

1.  **Initialization**: `ngOnInit()` runs when the component loads.
2.  **Trigger**: Calls `this.movieService.getAllMovies()`.
3.  **Reflect**:
    *   The `MovieService` returns an `Observable`.
    *   The component `.subscribe()`s to it.
    *   The **Interceptor** automatically adds the token.
    *   The **Backend** validates the token and returns the list of movies.
    *   The component updates its `movies` array.
    *   The `@for` loop in HTML detects the change and renders the `app-movie-card` elements.

---

## 4. Interview Questions on this Flow

**Q: Why do we use `BehaviorSubject` in `AuthService`?**
*   **A**: `BehaviorSubject` always holds the *current* value. This means if a component (like the Navbar) initializes *after* login, it can immediately grab the current user without waiting for a new event or reading from LocalStorage manually.

**Q: What happens if the Token expires?**
*   **A**: The backend will likely return a `401 Unauthorized` or `403 Forbidden` error. We typically handle this in the `subscribe({ error: ... })` block or use a global Error Interceptor to catch 401s and redirect to the Login page automatically.

**Q: Why do we clone the request in the Interceptor?**
*   **A**: HTTP Requests in Angular are **immutable** (cannot be changed). To add a header, we *must* create a clone of the original request with the new options.

**Q: Explain the difference between `localStorage` and `sessionStorage`.**
*   **A**:
    *   `localStorage`: Data persists even after the browser is closed and reopened (used in this project).
    *   `sessionStorage`: Data is lost when the browser tab is closed.
