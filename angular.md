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
