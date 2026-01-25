# 📚 Master List of Technical Topics & Concepts
## Project: MediaConnect

This document details EVERY single technical concept, framework feature, and coding pattern used in this project. It serves as a comprehensive study guide for Frontend and Backend.

---

# 🎨 PART 1: FRONTEND (Angular 17+)

## 1. Core Architecture
*   **Standalone Components**: The project uses `standalone: true` for all components, removing the need for `NgModule`.
*   **Component-Based Architecture**: Separation of concerns into View (`.html`), Logic (`.ts`), and Style (`.css`).
*   **Dependency Injection (DI)**: Injecting services (`HttpClient`, `AuthService`, `Router`) into constructors.
*   **Single Page Application (SPA)**: The app loads once; pages change dynamically without refreshing the browser.

## 2. Signals & State Management (Modern Angular)
*   **Signals (`signal`)**: Used for managing local state (e.g., `selectedPlan = signal<SubscriptionPlan>(null)`). Mutable reactive values.
*   **Computed Signals (`computed`)**: Deriving state automatically (e.g., `cardNumberError = computed(() => ...)`). Values update only when dependencies change.
*   **RxJS Integration (`toSignal`)**: Converting Observables to Signals for easier template usage.
*   **Change Detection**: Using `ChangeDetectorRef` manually in some async operations to force UI updates.

## 3. RxJS (Reactive Extensions for JavaScript)
*   **Observables**: Handling asynchronous data streams (HTTP responses).
*   **BehaviorSubject**: Holding the "Current User" state so new subscribers get the data immediately.
*   **Operators**:
    *   `map`: Transforming data (e.g., converting backend Date string to JS Date object).
    *   `tap`: Performing side effects (e.g., saving Token to `localStorage` without changing the data).
    *   `switchMap`: Chaining requests (e.g., "Get ID" -> "Then Fetch Movie").
    *   `catchError`: Handling HTTP failures gracefully.
*   **Subscription Management**: Subscribing in `.ts` files to trigger requests.

## 4. Routing & Navigation
*   **`Router` Service**: Programmatic navigation (`this.router.navigate(['/home'])`).
*   **`RouterLink`**: HTML-based navigation (`<a routerLink="/login">`).
*   **`ActivatedRoute`**: Reading URL parameters (e.g., getting `id` from `/movie/123`).
*   **Route Guards**: Protecting pages (e.g., "Only Admin can see this page").
*   **Route Parameters**: Dynamic paths defined in `app.routes.ts`.

## 5. Forms & Validation
*   **Reactive Forms (`ReactiveFormsModule`)**:
    *   **`FormGroup`**: Grouping multiple inputs (Email + Password).
    *   **`FormControl`**: Managing a single input field.
    *   **`FormBuilder`**: Syntactic sugar to create forms easily.
*   **Template-Driven Forms (`FormsModule`)**:
    *   **`[(ngModel)]`**: Two-way data binding for simple inputs (Search Bar).
*   **Validators**:
    *   Built-in: `required`, `email`, `minLength`.
    *   Custom Logic: Checking if Password matches Confirm Password (in TypeScript).

## 6. HTTP & API Interaction
*   **`HttpClient`**: Making `GET`, `POST`, `PUT`, `DELETE` requests.
*   **Interceptors**: Intercepting every request to add the JWT Token to the header automatically.
*   **Generic Types**: Typing responses for safety (`http.get<Movie[]>(...)`).
*   **Error Handling**: Global or local error catching.

## 7. HTML & Directives
*   **Control Flow (Angular 17)**:
    *   `@if`, `@else`: Conditional rendering.
    *   `@for`: Looping through lists efficiently.
*   **Legacy Directives**: `*ngIf`, `*ngFor` (still present in some parts).
*   **Attribute Binding**: `[disabled]`, `[class.active]`, `[src]`.
*   **Event Binding**: `(click)`, `(ngSubmit)`, `(change)`, `(keyup.enter)`.

## 8. Styling & UI
*   **Bootstrap 5**: Grid system (`row`, `col`), Cards, Buttons, and Utilities.
*   **Scoped CSS**: Styles defined in `component.css` apply ONLY to that component.
*   **DOM Manipulation**: Accessing video elements directly for "Resume Watch" functionality.

---

# ⚙️ PART 2: BACKEND (Spring Boot & Java)

## 1. Spring Core & Architecture
*   **Inversion of Control (IoC)**: Spring manages object creation.
*   **Dependency Injection (`@Autowired`)**: Wiring dependencies (Service inside Controller).
*   **Layered Architecture**:
    *   **Controller**: Handling HTTP requests.
    *   **Service**: Business logic.
    *   **Repository**: Database interaction.
    *   **Entity**: Database structure.

## 2. REST API Development
*   **`@RestController`**: Defining the class as an API handler.
*   **HTTP Mappings**: `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`.
*   **Request Data**:
    *   `@RequestBody`: JSON -> Java Object.
    *   `@PathVariable`: URL path -> Variable (`/movie/{id}`).
    *   `@RequestParam`: Query string -> Variable (`?query=action`).
*   **`ResponseEntity<?>`**: A wrapper to control HTTP Status Codes (200 OK, 400 Bad Request) and Body.

## 3. Database & JPA (Java Persistence API)
*   **Entities (`@Entity`)**: Mapping Java classes to SQL tables.
*   **Annotations**: `@Id`, `@GeneratedValue`, `@Table`, `@Column`, `@ManyToOne`.
*   **Spring Data JPA Repository**:
    *   **CRUD**: `save()`, `findById()`, `deleteById()`, `findAll()` provided automatically.
    *   **Derived Queries**: `findByEmail()`, `existsByName()`.
    *   **Custom JPQL**: `@Query("SELECT m FROM Movie m WHERE...")`.
    *   **Native Queries**: `@Query(value="SELECT * FROM...", nativeQuery=true)` for complex SQL logic.

## 4. Security & Authentication (Spring Security)
*   **JWT (JSON Web Token)**: Stateless authentication mechanism.
*   **Filters (`OncePerRequestFilter`)**: Custom logic that runs before every request to check the token.
*   **Password Encoding (`BCrypt`)**: Hashing passwords before saving to DB.
*   **AuthenticationManager**: Spring's built-in manager to verify credentials.
*   **Security Context**: Storing the current user's details in `SecurityContextHolder` for the duration of the request.
*   **CORS (Cross-Origin Resource Sharing)**: Allowing Frontend (localhost:4200) to talk to Backend (localhost:8081).

## 5. Logic & Data Processing
*   **DTO Pattern (Data Transfer Object)**:
    *   Why? To hide sensitive data (password) and structure responses.
    *   Example: `UserDTO` (safe) vs `User` (database entity).
*   **String Manipulation**: Splitting comma-separated strings (Genres, Features).
*   **Date/Time API**: `LocalDateTime` for subscription expiry calculations.

## 6. Exception Handling
*   **Global Exception Handler (`@ControllerAdvice`)**: Central place to catch errors.
*   **Custom Exceptions**: `ResourceNotFoundException`, `BadCredentialsException`.
*   **Standardized Error Response**: Returning a clean JSON (`timestamp`, `message`, `status`) instead of a stack trace.

## 7. File Handling
*   **Byte Array Generation**: Creating text files (Invoices) in memory.
*   **HTTP Headers**: Setting `Content-Disposition` to trigger browser downloads.
*   **MIME Types**: `MediaType.TEXT_PLAIN`.

## 8. Testing (JUnit 5 & Mockito)
*   **Unit Tests**: Testing individual methods in isolation.
*   **Mocking (`@Mock`, `@InjectMocks`)**: Faking the database or dependencies to test logic without a real DB connection.
*   **Assertions**: `assertEquals`, `assertNotNull`, `assertThrows`.
