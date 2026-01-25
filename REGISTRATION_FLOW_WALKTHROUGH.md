# 🚀 User Registration Flow - Detailed Code Walkthrough

This document provides a comprehensive, beginner-friendly explanation of how a new user registers in the **MediaConnect** application. We will trace the journey of data from the moment a user types their name in the browser (Frontend) to the moment it is saved in the database (Backend).

---

## 🗺️ High-Level Overview

1.  **User** fills out the registration form in the browser.
2.  **Angular (Frontend)** captures this data and validates it.
3.  **AuthService (Frontend)** sends the data as a package (JSON) to the Backend API.
4.  **Controller (Backend)** receives the package.
5.  **Service (Backend)** checks if the user already exists, encrypts the password, and prepares the data.
6.  **Repository (Backend)** saves the data into the **Database**.
7.  **Database** confirms the save.
8.  **Backend** sends back a success response.
9.  **Frontend** receives the success and redirects the user to the login page.

---

## 🟢 Step 1: The User Interface (Frontend)

The journey begins in the browser.

### 1. `register.component.html`
**Location:** `Frontend/src/app/components/register/register.component.html`

This file defines **what the user sees**. It's the HTML form where users input their details.

**Key Logic:**
- **`[formGroup]="registerForm"`**: Binds the HTML form to the TypeScript logic (`registerForm`). This is "Reactive Forms" in Angular.
- **`formControlName="email"`**: Connects a specific input box (like Email) to the variable `email` in the TypeScript file.
- **`@if` blocks**: These are used to show error messages (e.g., "Password must be at least 6 characters") strictly if the user makes a mistake.
- **`(ngSubmit)="onSubmit()"`**: When the user clicks "Register", this tells Angular to run the `onSubmit()` function in the TypeScript file.

```html
<!-- Example Snippet -->
<form [formGroup]="registerForm" (ngSubmit)="onSubmit()">
    <input formControlName="email" placeholder="Enter your email">
    <button type="submit">Register</button>
</form>
```

### 2. `register.component.ts`
**Location:** `Frontend/src/app/components/register/register.component.ts`

This file is the **brain of the page**. It handles the data before sending it anywhere.

**Key Logic:**
- **`FormBuilder`**: Used to create the `registerForm`. It defines the rules (Validators).
    - `email: ['', [Validators.required, Validators.email]]`: Means email cannot be empty and must look like an email.
- **`onSubmit()`**: This function is called when the button is clicked.
    1. It checks `if (this.registerForm.valid)`.
    2. If valid, it calls `this.authService.register(this.registerForm.value)`.
    3. It subscribes to the result: `next` for success (redirect to login), `error` for failure (show error message).

### 3. `user.model.ts` (AuthRequest)
**Location:** `Frontend/src/app/models/user.model.ts`

This is a **contract**. It ensures the Frontend speaks the same language as the Backend.

```typescript
export interface AuthRequest {
    email: string;
    password: string;
    fullName?: string;
    genrePreferences?: string[];
}
```
This interface guarantees that the data sent to the backend has exactly these fields, which matches what the backend expects.

---

## 🟡 Step 2: Sending the Request (Frontend Service)

Now the data leaves the component and prepares to travel over the internet.

### 4. `auth.service.ts`
**Location:** `Frontend/src/app/services/auth.service.ts`

This is the **courier**. Its job is to talk to the Backend API.

**Key Logic:**
- **`HttpClient`**: An Angular tool used to make web requests.
- **`register(request: AuthRequest)`**:
    ```typescript
    register(request: AuthRequest): Observable<AuthResponse> {
        return this.http.post<AuthResponse>('http://localhost:8081/api/auth/register', request);
    }
    ```
    - It sends a **POST** request to `http://localhost:8081/api/auth/register`.
    - It sends the `request` object (user data) inside the body of the request.

---

## 🔵 Step 3: Receiving the Request (Backend Controller)

The data has traveled across the network and arrived at the Java Backend.

### 5. `AuthController.java`
**Location:** `Backend/.../controller/AuthController.java`

This is the **Receptionist**. It sits at the door (URL) and accepts requests.

**Key Logic:**
- **`@RestController`**: Tells Spring Boot this class handles web requests.
- **`@RequestMapping("/api/auth")`**: Acts as the base address.
- **`@PostMapping("/register")`**: Listens specifically for POST requests at `/register`.
- **`@RequestBody AuthRequest request`**: This is magic. It takes the JSON sent by Angular and converts it automatically into a Java object (`AuthRequest`).

```java
@PostMapping("/register")
public ResponseEntity<AuthResponse> register(@RequestBody AuthRequest request) {
    // Passes the work to the Manager (Service)
    AuthResponse response = authService.register(request);
    return ResponseEntity.ok(response);
}
```

### 6. `AuthRequest.java`
**Location:** `Backend/.../dto/request/AuthRequest.java`

Identical to the frontend model. It is a simple class (POJO) to hold the incoming data (email, password, fullName, genres).

---

## 🟣 Step 4: Business Logic (Backend Service)

The Receptionist passes the data to the **Manager** (Service) to do the actual work.

### 7. `AuthService.java` (Interface) & `AuthServiceImpl.java` (Implementation)
**Location:** `Backend/.../service/implementations/AuthServiceImpl.java`

This is the **Brain** of the backend. It contains the core rules.

**Detailed Logic Flow in `register()` method:**

1.  **Check for Duplicates**:
    ```java
    boolean exists = userRepository.existsByEmail(request.getEmail());
    if (exists) { throw new RuntimeException("Email already exists..."); }
    ```
    - Before doing anything, it checks if this email is already taken to prevent duplicates.

2.  **Create User Entity**:
    - It creates a new `User` object.
    - It maps fields from `AuthRequest` to `User`.

3.  **Password Hashing (Crucial Security)**:
    ```java
    user.setPassword(passwordEncoder.encode(request.getPassword()));
    ```
    - **Never save plain passwords!** It uses `PasswordEncoder` (likely BCrypt) to scramble the password so it's unreadable in the database.

4.  **Set Defaults**:
    - Sets Role to `"USER"`.
    - Sets Subscription to `"INACTIVE"`.

5.  **Save to DB**:
    ```java
    User savedUser = userRepository.save(user);
    ```
    - Calls the repository to store the final user object.

---

## 🟠 Step 5: Database Interaction

Now we actually write to the hard drive (Database).

### 8. `User.java` (Entity)
**Location:** `Backend/.../entity/User.java`

This maps the Java class to the SQL Database setup.
- **`@Entity`**: Tells Hibernate this class represents a database table.
- **`@Table(name = "users")`**: The table name in MySQL will be `users`.
- **`@Id`**: The Primary Key.

### 9. `UserRepository.java`
**Location:** `Backend/.../repository/UserRepository.java`

This is the **Data Access Layer**.
- It extends `JpaRepository<User, Long>`.
- By doing this, we get methods like `save()`, `findByEmail()`, `existsByEmail()` for **FREE**. We don't need to write manual SQL queries (e.g., `INSERT INTO...`). Spring Data JPA does it for us.

### 10. `application.properties`
**Location:** `Backend/src/main/resources/application.properties`

Configuration file connecting the backend to the database.
- `spring.datasource.url`: Where the DB lives (localhost:3306).
- `spring.jpa.hibernate.ddl-auto=update`: Automatically creates/updates the table structure based on your `User.java` file.

---

## 🔴 Step 6: Handling Errors (When things go wrong)

What if the email already exists?

### 11. `GlobalExceptionHandler.java`
**Location:** `Backend/.../exception/GlobalExceptionHandler.java`

This is the **Safety Net**.

**How it works:**
1. If `AuthServiceImpl` throws `new RuntimeException("Email exists")`...
2. The Controller doesn't crash. Instead, this **Handler** catches the exception.
3. **`@ExceptionHandler(Exception.class)`**: Catches the error.
4. It creates a nice `ErrorResponse` (JSON) with a status code (e.g., 500 or 400) and the error message.
5. Sends this clean JSON back to the Frontend so the user sees "Email already exists" instead of a chaotic server crash log.

### 12. `ErrorResponse.java` & `ResourceNotFoundException.java`
- Helper classes to structure the error message nicely (timestamp, status, message).

---

## 🟢 Step 7: The Response (Back to Frontend)

If everything goes well:

### 13. `AuthResponse.java` (Backend) -> `auth-response.model.ts` (Frontend)

1. `AuthServiceImpl` returns an `AuthResponse` object containing the user's ID, role, and details.
2. The Controller sends this as JSON with `200 OK` status.
3. **Frontend (`register.component.ts`)** receives it in the `.subscribe({ next: () => ... })` block.
4. The user is redirected to the Login page.

---

## 📝 Summary Flowchart

`User Input` ➡️ `register.component.html`
  ⬇️
`register.component.ts` (Validation)
  ⬇️
`auth.service.ts` (HTTP POST)
  ⬇️
`AuthController.java` (Receives Request)
  ⬇️
`AuthServiceImpl.java` (Logic: Hash Pwd, Check Email)
  ⬇️
`UserRepository.java` (Save)
  ⬇️
`MySQL Database` (Stored!)


# 🧠 Complete Concepts & Technical Glossary: Registration Module

This document provides an exhaustive list of every single technical concept, pattern, and keyword—from the absolute basics to advanced architecture—used in the registration flow.

---

# 🎨 frontend (Angular & HTML/CSS)

## 1. Component Architecture
- **Component `@Component`**: The basic building block of Angular. It combines logic (`.ts`), view (`.html`), and style (`.css`).
  - **`selector`**: Defines the custom HTML tag (e.g., `<app-register>`).
  - **`standalone: true`**: A modern Angular 17+ feature that removes the need for `NgModules` (app.module.ts). Components manage their own dependencies.
  - **`templateUrl` & `styleUrls`**: Links the component to its external files.

## 2. Dependency Injection (DI)
- **Constructor Injection**: Passing dependencies into the `constructor()`.
  - *Example*: `constructor(private fb: FormBuilder)` tells Angular: "I need the FormBuilder tool, please find it and give it to me."
- **Singleton Services**: Services like `AuthService` are created once and shared across the entire app (`providedIn: 'root'`).

## 3. Reactive Forms (The Form Engine)
- **`ReactiveFormsModule`**: The module required to use model-driven forms.
- **`FormBuilder`**: A helper service that makes creating form groups easier and cleaner than using `new FormGroup(...)`.
- **`FormGroup`**: Represents the entire form. It tracks the collective value and validity of all its children.
- **`FormControl`**: Tracks the value and validation status of an individual input field (e.g., just the "email" box).
- **`FormArray`**: An alternative to `FormGroup` for managing a list of controls. Used here for the checkboxes (Genres), allowing dynamic addition/removal of `FormControl`s.
- **`Validators`**:
  - **`Validators.required`**: Ensures the field is not empty.
  - **`Validators.email`**: Uses a regex pattern to ensure the text looks like an email.
  - **`Validators.minLength(n)`**: Enforces a minimum character count.
- **AbstractControl**: The base class for `FormGroup`, `FormControl`, and `FormArray`.
- **`markAllAsTouched()`**: A utility method used on submit to trigger validation error messages visually even if the user didn't click inside the fields.
- **`formControlName`**: The directive connecting the HTML input to the TypeScript `FormControl`.

## 4. Angular Directives (HTML Superpowers)
- **Structural Directives (`*` or `@`)**:
  - **`@if` (Control Flow)**: New Angular 17 syntax to conditionally render HTML. Replaces `*ngIf`.
  - **`*ngFor`**: Iterates over a list (arrays) to generate repeated HTML elements (used for generating genre checkboxes).
- **Attribute Directives (`[]`)**:
  - **Property Binding (`[property]="value"`)**: One-way data binding from TS to HTML. Example: `[disabled]="isLoading"` locks the button when `isLoading` is true.
  - **Class Binding (`[class.name]="condition"`)**: Adds/removes a CSS class dynamically. Example: `[class.invalid]` adds red borders on error.
  - **`[ngClass]`**: Allows adding multiple dynamic classes at once based on conditions.

## 5. Event Binding (`()`)
- **`(click)` / `(ngSubmit)`**: Listens for user actions. `(ngSubmit)` prevents the default browser refresh and calls our function `onSubmit()`.
- **`(change)`**: Listens for value changes (used on checkboxes to detect when a genre is ticked/unticked).

## 6. Asynchronous Programming (RxJS)
- **Observables**: A stream of data that arrives over time. The HTTP request returns an Observable.
- **`subscribe()`**: The trigger. An Observable does nothing until you subscribe. It initiates the HTTP request.
- **Observer Object (`{ next, error }`)**: The callback functions inside subscribe. behavior for "Happy Path" (`next`) and "Error Path" (`error`).
- **`BehaviorSubject`**: A special type of Observable that holds a value (current user state). Unlike a regular Subject, it can tell new subscribers the *last* known value immediately. (Used in `AuthService` to keep track of login state).
- **Pipe & Tap (`.pipe(tap(...))`)**: `pipe` chains operators. `tap` allows us to perform side effects (like saving to localStorage) without changing the data stream.

## 7. HTTP Communication
- **`HttpClient`**: Angular's simplified API for making XMLHttpRequests.
- **HTTP Verbs**: `POST` (used for Register/send data), `GET`, `PUT`, `DELETE`.
- **Generic Types (`<T>`)**: `http.post<AuthResponse>(...)` tells TypeScript: "Expect the answer to be in the shape of an `AuthResponse` object."

## 8. Routing & Navigation
- **`Router` Service**: The programmatic way to move users. `this.router.navigate(['/path'])`.
- **`routerLink`**: The HTML way to move users. `<a routerLink="/login">`. It intercepts clicks to prevent full page reloads.

## 9. Browser Storage
- **`localStorage`**: A persistent key-value store in the browser. We save the Token and User object here so they stay logged in even if they refresh the page.

## 10. CSS/Styling Concepts
- **Scoped Styles**: Styles defined in `register.component.css` ONLY apply to this component. They don't leak out to the rest of the app.
- **Flexbox**: Used in `.auth-container` to center the card (`display: flex`, `justify-content: center`).
- **Responsive Design**: Making the layout adapt to screen sizes (implied by container classes).

---

# ⚙️ Backend (Java Spring Boot)

## 1. Spring Framework Core (IoC & DI)
- **Inversion of Control (IoC)**: Spring manages the creation of objects (Beans), not us.
- **Dependency Injection (DI)**: The process where Spring gives objects their dependencies.
- **Annotations**:
  - **`@Component`**: Generic bean.
  - **`@Service`**: Specialized bean for business logic.
  - **`@Repository`**: Specialized bean for database access (catches DB exceptions).
  - **`@Controller` / `@RestController`**: Specialized bean for web handling.
  - **`@Autowired`**: The annotation that requests a dependency.

## 2. REST API Architecture
- **Statelessness**: The server doesn't remember previous requests. Each request stands alone (hence the need for sending tokens every time).
- **Mappings**:
  - **`@RequestMapping`**: Base URL path (`/api/auth`).
  - **`@PostMapping`**: Semantic mapping for creation actions.
- **Serialization / Deserialization (Jackson)**: The library Spring uses to convert Java Objects <-> JSON string. `User` object -> JSON.
- **`ResponseEntity`**: A wrapper for the entire HTTP response. It lets us control the Body, Headers, and Status Code (200, 404, 500).

## 3. Data Persistence (JPA & Hibernate)
- **ORM (Object-Relational Mapping)**: Mapping Java Classes to SQL Tables logic.
- **Entities (`@Entity`)**:
  - **`@Id`**: Primary Key.
  - **`@GeneratedValue`**: Auto-incrementing logic (`1, 2, 3...`).
  - **`@Column`**: Customize column details (`unique = true`, `nullable = false`).
- **Repository Pattern**: Abstraction layer over data access. `JpaRepository` gives us standard CRUD operations (Create, Read, Update, Delete) for free.
- **JPQL / Derived Queries**: Methods like `existsByEmail(String email)` are parsed by Spring Data to generate the SQL query: `SELECT COUNT(u) > 0 FROM User u WHERE u.email = ?`.

## 4. Security & Cryptography
- **Hashing**: One-way transformation of data. `123456` -> `$2a$10$f3...`. Unlike encryption, it cannot be decrypted.
- **BCrypt**: The specific hashing algorithm used by `PasswordEncoder`. It incorporates "Salting" (adding random data) to protect against Rainbow Table attacks.
- **Authentication vs. Authorization**:
  - This flow handles **Authentication** (Who are you? -> Registration/Login).
  - **Authorization** (What can you do? -> ROLE_USER vs ROLE_ADMIN) is set here (`user.setRole("USER")`).

## 5. Exception Handling
- **AOP (Aspect-Oriented Programming)**: The concept behind `@RestControllerAdvice`. It allows "cross-cutting concerns" (like error handling) to be separated from business logic.
- **Global Handling**: Centralizing all error logic in one place (`GlobalExceptionHandler`) rather than `try-catch` blocks in every controller.

## 6. Java Language Features
- **Interfaces**: `AuthService` is an interface. This promotes loose coupling. The controller talks to the *interface*, not the specific implementation (`AuthServiceImpl`).
- **Generics**: `ResponseEntity<AuthResponse>` ensures type safety at compile time.
- **Streams & Lambdas**: (Likely used in mapper logic) concise functional programming style.
- **POJOs (Plain Old Java Objects)**: `AuthRequest`, `UserDTO`. Simple classes holding detailed data with getters/setters.

## 7. Logging
- **SLF4J (Simple Logging Facade for Java)**: An abstraction.
- **Levels**: usage of `INFO` (normal flow), `WARN` (something weird), `ERROR` (crash).
