# 🎓 MediaConnect: Authentication Interview Questions (Frontend & Backend)

This document contains **50+ detailed interview questions** based specifically on the **Registration and Login** modules of the MediaConnect project. The answers are tailored to the actual implementation used in the code.

---

# 🎨 frontend (Angular)

## **Basics & Components**

### 1. What is a "Standalone Component" in Angular, and why did we use it for `LoginComponent`?
**Answer:**
A standalone component (`standalone: true`) operates without needing to be declared in an `NgModule`. It manages its own dependencies directly via the `imports` array. In our `LoginComponent`, we imported `CommonModule`, `ReactiveFormsModule`, and `RouterModule` directly in the component metadata. This modern approach simplifies the project structure and makes components easier to reuse and test.

### 2. Explain the difference between `FormGroup` and `FormBuilder`. Which one is used in `RegisterComponent`?
**Answer:**
`FormGroup` is the actual container that holds the values and validation status of a group of form controls. `FormBuilder` is a helper service (factory) that makes creating `FormGroup` instances easier with less code.
In `RegisterComponent`, we inject `FormBuilder` (as `fb`) to create the `registerForm`. Instead of writing `new FormGroup({ email: new FormControl(...) })`, we simply write `this.fb.group({ email: [...] })`.

### 3. How does the `(ngSubmit)` event work in the login form?
**Answer:**
`(ngSubmit)` is an Angular event binding that intercepts the standard HTML form submission. When the user presses Enter or clicks "Login", instead of the browser reloading the page (default behavior), Angular catches the event and executes our custom `onSubmit()` method defined in `LoginComponent`.

### 4. What is the purpose of `formControlName` in the HTML template?
**Answer:**
`formControlName` is a directive that links a specific HTML input element (like the email textbox) to a specific `FormControl` instance defined inside the TypeScript `FormGroup`. It establishes a sync: when the user types in the box, the TypeScript variable updates, and if TypeScript changes the value, the specific input box updates.

### 5. Why do we access form controls using `loginForm.get('email')` instead of just `loginForm.email`?
**Answer:**
`loginForm` is a `FormGroup` object, but the controls are stored inside a map/dictionary structure. `loginForm.email` is not a property of the class. The `.get('email')` method safely retrieves the abstract control instance associated with the key 'email'. We use `.get()` to access properties like `.invalid`, `.touched`, and `.value`.

---

## **Validation & Logic**

### 6. Explain how the "Genres" checkboxes work in `RegisterComponent`. What is `FormArray`?
**Answer:**
We cannot use a simple `FormControl` for genres because a user can select multiple options. We use a `FormArray`, which manages a dynamic list of controls.
- When a user **checks** a box, we push a new `FormControl` with that value into the array.
- When they **uncheck** it, we search for the index of that value and remove it.
- We added a custom validator to ensure `array.length >= 2`.

### 7. What is the difference between `touched` and `dirty` in Angular forms?
**Answer:**
- **Touched**: True if the user has put the cursor inside the field and then clicked away (blur event).
- **Dirty**: True if the user has actually modified the value.
In our code (`register.component.html`), we use `control.invalid && control.touched` to show error messages. This prevents showing scary red errors before the user has even clicked on the field.

### 8. What does `markAllAsTouched()` do, and why do we call it in `onSubmit()`?
**Answer:**
If a user tries to submit an empty form without clicking any fields, the individual fields are still "untouched," so the error messages (which rely on `touched`) won't show up. `markAllAsTouched()` programmatically sets all fields to "touched" status, instantly triggering all visual validation errors so the user sees what is missing.

---

## **Services & HTTP**

### 9. Why is `AuthService` marked with `providedIn: 'root'`?
**Answer:**
This syntax (`@Injectable({ providedIn: 'root' })`) makes the service a **Singleton** available throughout the entire application. Angular creates only **one instance** of `AuthService`. This is crucial because we need to share the same state (like the current user or token) across multiple components (Login, Register, Navbar).

### 10. How does `HttpClient.post<AuthResponse>()` leverage TypeScript generics?
**Answer:**
By specifying `<AuthResponse>` (the generic type), we tell TypeScript that the JSON response from the backend will match the structure of our `AuthResponse` interface. This gives us compile-time safety and autocomplete. If we try to access `response.xyz`, TypeScript will throw an error because `xyz` doesn't exist on `AuthResponse`.

### 11. What is the purpose of the `.pipe(tap(...))` operator in the `login` method?
**Answer:**
`tap` is an RxJS operator used for "side effects." It allows us to peek at the data stream without modifying it. In `AuthService.login()`, we use `tap` to intercept the successful response and save the JWT token to `localStorage` **before** the data is passed back to the component. This ensures the session is saved automatically upon success.

### 12. Explain the flow of an `Observable` when we call `subscribe()`.
**Answer:**
An `Observable` is lazy; it does nothing until subscribed to.
1. The component calls `.subscribe()`.
2. The HTTP request is essentially "fired."
3. If the backend responds with 200 OK, the `next` callback runs (success logic).
4. If the backend responds with 4xx/5xx, the `error` callback runs (failure logic).
5. The stream completes.

---

## **RxJS & State Management**

### 13. What is a `BehaviorSubject` and why do we use it for `currentUser`?
**Answer:**
A `BehaviorSubject` is a special type of Observable that holds a "current value." Unlike a regular Subject, when a new component subscribes to it, it immediately receives the last stored value. We use it to store the `currentUser` state so that components like the **Navbar** can immediately know if a user is logged in (and show "Logout" instead of "Login") without waiting for a new event.

### 14. What are the `next` and `error` properties in the subscribe object?
**Answer:**
They are callback functions defined in the Observer pattern.
- **`next`**: Handles the data when it arrives successfully.
- **`error`**: Handles any exceptions or HTTP error statuses returned by the server. We use this to display specific error messages like "Email already exists" to the user.

---

## **Routing & Guards**

### 15. What is the role of `Interceptor` in the login flow?
**Answer:**
The `JwtInterceptor` intercepts every outgoing HTTP request. It checks if a token exists in `localStorage`. If it does, it clones the request and adds the `Authorization: Bearer <token>` header. This automates the authentication process so we don't have to manually add the header in every single service call.

### 16. How does `AuthGuard` protect a route?
**Answer:**
`AuthGuard` implements the `CanActivate` interface. Before the Router navigates to a protected page (like `/home`), it runs `canActivate()`.
- It checks `authService.isAuthenticated()`.
- If true, it returns `true` (navigation proceeds).
- If false, it redirects to `/login` and returns `false` (navigation blocked).

---

# ⚙️ BACKEND (Spring Boot)

## **Controller Layer**

### 17. What is the difference between `@Controller` and `@RestController`?
**Answer:**
`@Controller` is used for traditional MVC where the return value is often a View (JSP/HTML). `@RestController` is a specialized version for REST APIs. It combines `@Controller` and `@ResponseBody`. This means every method automatically serializes its return object into JSON format in the HTTP response body, which is exactly what our Angular frontend expects.

### 18. What does `@RequestBody` do in the `register` method?
**Answer:**
It tells Spring to look at the HTTP Request Body (payload), take the JSON string (e.g., `{"email": "..."}`), and deserialize (convert) it into the Java POJO class `AuthRequest`. Without this annotation, the object `request` would be null or empty.

### 19. Why do we return `ResponseEntity<AuthResponse>` instead of just `AuthResponse`?
**Answer:**
`ResponseEntity` gives us full control over the HTTP response. While returning `AuthResponse` simply sends the data with status 200, `ResponseEntity` allows us to customize the **Status Code** (e.g., 201 Created, 400 Bad Request) and **Headers**. It is a best practice for building robust APIs.

---

## **Service Layer**

### 20. Why do we code to an interface (`AuthService`) and not the class (`AuthServiceImpl`)?
**Answer:**
This is the principle of "Loose Coupling" and "Abstraction." By injecting the interface in the controller, we can easily swap the implementation later (e.g., for testing mock services) without changing the controller code. It also separates the "What" (interface contract) from the "How" (implementation details).

### 21. Explain the role of `AuthenticationManager` in the login logic.
**Answer:**
`AuthenticationManager` is the core container for authentication in Spring Security. When we call `.authenticate()`, it delegates the check to the configured `AuthenticationProvider`. It handles the entire lifecycle: retrieving the user (via `UserDetailsService`), checking the password (via `PasswordEncoder`), and either returning a fully authenticated token or throwing `BadCredentialsException`.

### 22. What happens if `userRepository.existsByEmail(...)` returns true during registration?
**Answer:**
We throw a `RuntimeException("Email already exists!")`. We act immediately to protect data integrity. This exception bubbles up and is eventually caught by our `GlobalExceptionHandler`, which formats it into a user-friendly JSON error response (409 Conflict or 500 Internal Error) for the frontend.

---

## **Security & Cryptography**

### 23. Why do we use `BCryptPasswordEncoder`? Why not just save the password as plain text?
**Answer:**
Storing plain text passwords is a massive security risk. If the DB is compromised, all user accounts are stolen. `BCrypt` is a hashing algorithm. It is "one-way" (you cannot decode it back to the password) and includes a "salt" (random data) to defend against dictionary/rainbow table attacks. `PasswordEncoder.matches()` compares the raw input with the stored hash to verify logic.

### 24. What is a JWT (JSON Web Token)? What are its three parts?
**Answer:**
JWT is a compact, URL-safe means of representing claims to be transferred between two parties.
1.  **Header**: Algorithm and token type.
2.  **Payload**: The data (Claims) like subject (email), issue date, expiry, and role.
3.  **Signature**: Verify that the sender is who it says it is and to ensure that the message wasn't changed along the way. Generated using the Secret Key.

### 25. What is the "Secret Key" in `JwtUtil` and why must it be kept secret?
**Answer:**
The Secret Key (`HMAC`) is used to sign the token. If an attacker gets this key, they can generate their own valid tokens for *any* user ("forging a signature"). In our code, it's a hardcoded string or environment variable. The backend uses it to verify that the token received was indeed created by us and hasn't been tampered with.

### 26. Explain how `JwtFilter` works. Why does it extend `OncePerRequestFilter`?
**Answer:**
`JwtFilter` is a custom filter in the Spring Security chain. Extending `OncePerRequestFilter` guarantees it creates logic exactly once per HTTP request. Its job is to:
1.  Look for the `Authorization` header.
2.  Check for `Bearer ` prefix.
3.  Extract and validate the token.
4.  If valid, manually set the `Authentication` object in the `SecurityContext`. This "logs the user in" for the duration of that specific request.

### 27. What is `auth.requestMatchers(...).permitAll()` doing in `SecurityConfig`?
**Answer:**
It configures the authorization rules. `.permitAll()` publicly exposes specific endpoints. We use it for `/api/auth/**` (Login/Register) because a user *cannot* have a token before they log in. If we didn't do this, no one could ever register or log in.

---

## **Database & JPA**

### 28. How does `userRepository.findByEmail(email)` work if we didn't write the SQL query?
**Answer:**
This is a "Derived Query Method" feature of Spring Data JPA. Spring analyzes the method name `findByEmail`. It knows `find` means SELECT, `By` is the WHERE clause, and `Email` is the property of the User entity. It dynamically generates the JPQL/SQL: `SELECT u FROM User u WHERE u.email = ?1` at runtime.

### 29. What annotations make the `User` class a database entity?
**Answer:**
-   `@Entity`: Marks it as a JPA entity.
-   `@Table(name="users")`: Maps it to the `users` table.
-   `@Id`: key.
-   `@GeneratedValue`: Strategy for auto-incrementing IDs.

### 30. Why is `subscriptionStatus` defaulted to `INACTIVE` in `AuthServiceImpl`?
**Answer:**
This is specific business logic. A new user has registered but hasn't paid yet. We explicitly set the state to `INACTIVE` in the backend to ensure no user accidentally gets free access by default. It guarantees a consistent initial state.

---

## **Advanced / Scenario Questions**

### 31. Trace the exact path of a data packet during Login.
**Answer:**
`Component` -> `Service` -> `HTTP POST` -> `Filter Chain` -> `Controller` -> `AuthService` -> `AuthManager` -> `UserDetailsService` -> `Repository` -> `Database`. (And back).

### 32. If a user gets a 403 Forbidden error, what are the likely causes?
**Answer:**
1.  **Token Issues**: The token is expired or invalid (signature mismatch).
2.  **Role Issues**: The user is authenticated (valid token) but tried to access an admin-only endpoint (`/api/admin`) while having only `ROLE_USER`.
3.  **Missing Token**: The frontend failed to attach the header.

### 33. Why do we store the token in `localStorage` and not `sessionStorage`?
**Answer:**
`sessionStorage` is cleared when the tab is closed. `localStorage` persists even after the browser is closed and reopened. For a better UX (User Experience), we usually want the user to stay logged in when they return to the site tomorrow, so `localStorage` is preferred here.

### 34. How would you implement "Logout" in this stateless architecture?
**Answer:**
Since the server is stateless commit doesn't "know" who is logged in, we cannot "delete" the session on the server.
**Frontend Side**: We simply delete the token from `localStorage` (`removeItem`). The user "forgets" their badge.
**Backend Side (Advanced)**: For higher security, we could implement a "Blacklist" (Redis cache) of tokens that have been logged out but haven't expired yet, and check against it in the `JwtFilter`.

### 35. Why use DTOs (`AuthRequest`, `UserDTO`) instead of the `User` Entity?
**Answer:**
-   **Security**: The `User` entity contains the password hash. We never want to send that to the frontend via JSON. DTOs allow us to filter fields.
-   **Decoupling**: The database structure might change (e.g., column splitting), but the API contract (DTO) can stay the same, preventing breaking changes for the frontend.

### 36. What is the `UserDetailsService` interface and why did we implement `CustomUserDetailsService`?
**Answer:**
Spring Security needs a way to fetch users, but it doesn't know about our specific `UserRepository` or database. `UserDetailsService` is the standard interface Spring uses to ask "Give me the user data for this username." We implement it to bridge the gap: our implementation calls `userRepository.findByEmail()` and converts our `User` entity into a Spring Security `UserDetails` object.

### 37. What happens if the JWT expires while the user is using the app?
**Answer:**
1.  The frontend sends a request with the expired token.
2.  The backend `JwtUtil` checks the expiry date and sees it's in the past.
3.  `validateToken` returns `false`.
4.  `JwtFilter` does **not** set the authentication in the context.
5.  Spring Security denies the request (403 Forbidden).
6.  ( Ideally) The frontend interceptor catches this 403, clears the storage, and redirects the user to the Login page.

### 38. Why is `CorsConfiguration` needed in `SecurityConfig`?
**Answer:**
Browsers block requests from one domain (Frontend: localhost:4200) to another (Backend: localhost:8081) by default for security. We must explicitly allow traffic from `http://localhost:4200` and allow methods like `POST`, `GET`, etc., in the backend configuration to tell the browser "It's safe, let them talk."

### 39. What is Dependency Injection and where is it literally used in `AuthController`?
**Answer:**
Dependency Injection is a design pattern where an object receives other objects that it depends on. In `AuthController`, we have `@Autowired private AuthService authService;`. Spring scans the project, finds the `AuthService` bean (created via `@Service`), and literally "injects" it into the controller variable. We don't write `authService = new AuthServiceImpl()`.

### 40. Explain the `GlobalExceptionHandler` and `ErrorResponse` logic.
**Answer:**
Instead of scattering try-catch blocks everywhere, we use AOP (Aspect Oriented Programming). The `@RestControllerAdvice` class watches all controllers.
If `AuthService` throws `RuntimeException("Bad Creds")`:
1.  Execution jumps out of the controller.
2.  `GlobalExceptionHandler` catches it.
3.  It wraps the error message into a neat `ErrorResponse` JSON object.
4.  It returns a clean HTTP response (e.g., 500) to the client.

---

## **Coding & Syntax specific**

### 41. Write the JUnit test signature to test the `login` method in AuthService.
**Answer:**
```java
@Test
void testLogin_Success() {
    // 1. Mock Repository & AuthManager
    // 2. Call authService.login(request)
    // 3. Assert Response is not null
    // 4. Assert Token is present
}
```

### 42. In `JwtUtil`, how is the expiry time calculated?
**Answer:**
`new Date(System.currentTimeMillis() + 1000 * 60 * 60 * 10)`
- `System.currentTimeMillis()`: current time in ms.
- `1000 * 60 * 60 * 10`: Converts 10 hours into milliseconds.
- Adds them together to get the future timestamp.

### 43. Why is `loginForm` declared as `FormGroup` type and not `any`?
**Answer:**
Using strict typing (`FormGroup`) allows TypeScript to protect us. It gives us autocomplete for methods like `.get()`, `.valid`, `.value`. Using `any` defeats the purpose of TypeScript and leads to runtime errors that are hard to debug.

### 44. What does the `@if` block in the HTML template do?
**Answer:**
It's the new Angular Control Flow syntax (v17+). It conditionally renders the DOM content. It's more performant and readable than the older `*ngIf` structural directive, though they serve the same purpose.

### 45. Why do we map `User` to `UserDTO` in `AuthServiceImpl`?
**Answer:**
The `User` entity has sensitive data (`password`, `version`, audit logs) and recursive relationships potentially. The logic `mapToDTO` manually copies only the safe fields (`id`, `email`, `role`, `genres`) to a vanilla Java object to ensure the API output is clean, safe, and minimal.

---

## **Scenario-Based Troubleshooting**

### 46. "I clicked register, but nothing happened." How do you debug?
**Answer:**
1.  Check Browser Console (F12) for JS errors (Frontend logic fail?).
2.  Check Network Tab: Was the request sent? Is it pending? what is the status? (400? 500?).
3.  Check Backend Console Logs: Did the request reach the controller? Are there exceptions in the logs?

### 47. "I logged in, but when I refresh, I am logged out." Why?
**Answer:**
You likely forgot to implement the logic to **retrieve** the user from `localStorage` on app initialization. In `AuthService`, the constructor should check `localStorage.getItem('currentUser')` and load it into the `currentUser` BehaviorSubject so the app remembers the state.

### 48. "The backend says 401 Unauthorized even though I sent the token."
**Answer:**
1.  Is the token format correct? (`Bearer <token_string>` - space is crucial).
2.  Is the token expired? (Check payload expiry).
3.  Is the Secret Key identical in generation and validation?
4.  Did the `JwtFilter` run? (Debug logs in filter).

### 49. API returns "CORS Error". What is missing?
**Answer:**
The Backend `SecurityConfig` is likely missing the explicit CORS configuration allowing `localhost:4200`. Alternatively, the Browser is pre-flighting (OPTIONS request) and the server is rejecting the OPTIONS method.

### 50. Why does `userRepository` extend `JpaRepository<User, Long>`?
**Answer:**
-   `User`: The Entity type it manages.
-   `Long`: The type of the Primary Key (`@Id`).
This generic definition allows Spring to generate strongly-typed methods like `findById(Long id)` automatically.
