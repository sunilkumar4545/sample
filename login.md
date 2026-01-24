# LOGIN USER – COMPLETE WALKTHROUGH

This document explains **exactly** how a user logs in to the MediaConnect application, tracing the data flow from the Frontend UI to the Backend Database and back again.

--------------------------------------------------
## 1️⃣ What is Login? (Very Simple)

- **Login** is the process of verifying a user's identity.
- After **Registration**, the system knows the user exists. **Login** proves that the person trying to access the account is actually the owner.
- **Real-world example:** Registration is like getting a library card. Login is like showing that card and your ID every time you want to borrow a book.

**Why is it needed?**
The web is "stateless." The server forgets you immediately after every request. Login gives you a "Digital ID Badge" (JWT Token) that you show with every future request so the server remembers who you are.

--------------------------------------------------
## 2️⃣ FRONTEND – UI & FORM LOGIC

### `login.component.html`
**Location:** `Frontend/src/app/components/login/login.component.html`

This is the visual part of the login page.

- **Form Binding (`[formGroup]="loginForm"`)**: Connects the HTML to the TypeScript logic.
- **Inputs**:
  - `formControlName="email"`: Binds the input box to the `email` variable.
  - `formControlName="password"`: Binds the input box to the `password` variable.
- **Validation Messages**: Uses `@if` blocks to show errors (e.g., "Email is required") only if the user touches the field and creates an error.
- **Submit Button**: `<button type="submit">` triggers the `(ngSubmit)="onSubmit()"` event.

```html
<!-- Simplified Snippet -->
<form [formGroup]="loginForm" (ngSubmit)="onSubmit()">
    <input formControlName="email">
    <input formControlName="password">
    <button type="submit">Login</button>
</form>
```

### `login.component.ts`
**Location:** `Frontend/src/app/components/login/login.component.ts`

This is the brain of the login page.

1.  **`loginForm` Initialization**:
    - Created in the `constructor` using `FormBuilder`.
    - Rules: `email` is required and must look like an email. `password` must be at least 6 characters.

2.  **`onSubmit()` Method**:
    - This creates the trigger action.
    - **Step 1**: Checks `if (this.loginForm.valid)`.
    - **Step 2**: Sets `isLoading = true` to disable the button.
    - **Step 3**: Calls `this.authService.login(this.loginForm.value)`.
    - **Step 4**: Subscribes to the result.
        - **Success (`next`)**: Redirects user to `/home` after 1 second.
        - **Failure (`error`)**: Shows an error message like "Invalid email or password."

--------------------------------------------------
## 3️⃣ FRONTEND SERVICE CALL

### `auth.service.ts`
**Location:** `Frontend/src/app/services/auth.service.ts`

The service acts as the **bridge** between the UI and the Backend. The component doesn't know *how* to talk to the server; it just asks the service to do it.

**The `login()` Method:**
```typescript
login(request: AuthRequest): Observable<AuthResponse> {
  return this.http.post<AuthResponse>('http://localhost:8081/api/auth/login', request)
    .pipe(
      tap(response => this.handleAuthSuccess(response))
    );
}
```
- **API URL**: Targets `http://localhost:8081/api/auth/login`.
- **Method**: `POST` (because we are sending sensitive data).
- **`tap(...)`**: A "side effect." Before returning the data to the component, it runs `handleAuthSuccess` to save the token (we'll see this in Section 10).

--------------------------------------------------
## 4️⃣ HTTP REQUEST DETAILS

When you click "Login," the browser sends a digital package to the server.

- **URL**: `http://localhost:8081/api/auth/login`
- **Method**: `POST`
- **Headers**: `Content-Type: application/json`
- **Body (JSON)**:
  ```json
  {
    "email": "john@example.com",
    "password": "secretpassword"
  }
  ```

--------------------------------------------------
## 5️⃣ BACKEND CONTROLLER LOGIC

### `AuthController.java`
**Location:** `Backend/.../controller/AuthController.java`

The **Controller** is the receptionist. It receives the request and passes it to the specialist (Service).

- **`@PostMapping("/login")`**: Listens for the login request.
- **`@RequestBody AuthRequest request`**: Takes the JSON above and converts it into a Java Object automatically.
- **Action**: Calls `authService.login(request)`.

```java
@PostMapping("/login")
public ResponseEntity<AuthResponse> login(@RequestBody AuthRequest request) {
    AuthResponse response = authService.login(request);
    return ResponseEntity.ok(response);
}
```

--------------------------------------------------
## 6️⃣ BACKEND SERVICE LOGIC (MOST IMPORTANT)

### `AuthService.java`
**Location:** `Backend/.../service/AuthService.java`
- An **Interface** that defines the contract: `AuthResponse login(AuthRequest request);`.

### `AuthServiceImpl.java`
**Location:** `Backend/.../service/implementations/AuthServiceImpl.java`

This is where the **Real Work** happens.

**Detailed Logic Flow (`login` method):**

1.  **Authentication Manager**:
    ```java
    authenticationManager.authenticate(
        new UsernamePasswordAuthenticationToken(request.getEmail(), request.getPassword())
    );
    ```
    - This is a Spring Security magic box. It takes the raw email/password.
    - It secretly calls `CustomUserDetailsService` (see next section) to find the user and check the password hash.
    - If the password is wrong, it **THROWS AN EXCEPTION** immediately (`BadCredentialsException`). The code stops here.

2.  **Fetch User (If Auth Successful)**:
    ```java
    User user = userRepository.findByEmail(request.getEmail());
    ```
    - Since authentication passed, we retrieve the full user details from the DB to put into the response.

3.  **Generate Token**:
    ```java
    String token = jwtUtil.generateToken(user.getEmail());
    ```
    - Calls `JwtUtil` to create the digital ID badge.

4.  **Return Response**:
    - Returns a new `AuthResponse` containing the `token` and user details.

--------------------------------------------------
## 7️⃣ DATABASE & ENTITY LOGIC

### `UserRepository.java`
**Location:** `Backend/.../repository/UserRepository.java`

- **`findByEmail(String email)`**: Uses Spring Data JPA. It translates this Java method into a SQL query:
  `SELECT * FROM users WHERE email = ?`

### `User.java`
**Location:** `Backend/.../entity/User.java`

- **Fields**: stored in the table.
- **Password**: Stored as a **BCrypt Hash** (e.g., `$2a$10$Xy...`), not plain text. This is why we need the `AuthenticationManager` to check it—we can't just do `if (password == dbPassword)`.

--------------------------------------------------
## 8️⃣ JWT CREATION & VALIDATION FLOW (VERY IMPORTANT)

### Login Phase: `JwtUtil.java`
**Location:** `Backend/.../security/JwtUtil.java`

1.  **`generateToken(email)`**:
    - Creates a map of **Claims** (data inside the token).
    - **Subject**: The user's email.
    - **IssuedAt**: Current time.
    - **Expiration**: Current time + 10 hours.
    - **Signature**: Signs the token using a secret key (`HMAC SHA256`). This ensures no one can fake the token.

### Subsequent Requests Phase: `JwtFilter.java`
**Location:** `Backend/.../security/JwtFilter.java`

This runs **before** every request (except login/register).

1.  **Check Header**: Looks for `Authorization: Bearer <token>`.
2.  **Extract Email**: Decodes the token to get the email.
3.  **Validation**: Checks `jwtUtil.validateToken()`.
    - Is the signature valid?
    - Is the token expired?
4.  **Set Context**: If valid, it tells Spring Security: "This user is logged in as [Email] with [Role]."

--------------------------------------------------
## 9️⃣ EXCEPTION HANDLING FLOW (FAILURE CASES)

What if the password is wrong?

1.  `AuthenticationManager` in `AuthServiceImpl` fails validation.
2.  It throws `BadCredentialsException`.
3.  **`GlobalExceptionHandler.java`** catches this exception.
4.  It creates an `ErrorResponse` Object:
    - **Status**: 401 (Unauthorized) or 500.
    - **Message**: "Bad credentials" or "Invalid email or password."
5.  Sends this JSON back to the Frontend.

The Frontend `login.component.ts` sees this in the `error:` block and displays "Invalid email or password."

--------------------------------------------------
## 🔟 RESPONSE FLOW (SUCCESS CASE)

If everything works, the backend sends an `AuthResponse` JSON:

```json
{
  "token": "eyJhGciOiJIUzI1Ni...",
  "role": "USER",
  "userId": 1,
  "user": { ... }
}
```

### Back in the Frontend `auth.service.ts`:
The `handleAuthSuccess()` method runs:

1.  **Store Token**: `localStorage.setItem('token', response.token);`
    - Saves the digital badge in the browser's persistent storage.
2.  **Store User**: `localStorage.setItem('currentUser', ...);`
3.  **Update State**: Notifies the rest of the app (via `BehaviorSubject`) that "User is now logged in."

### **Final Step: Redirect**
The component waits 1 second and then:
`this.router.navigate(['/home']);`

--------------------------------------------------
## 1️⃣1️⃣ COMPLETE CHAIN FLOW SUMMARY

1.  **User** types email/password and clicks "Login".
2.  **Frontend** creates a JSON object and POSTs it to `/api/auth/login`.
3.  **Backend Controller** receives it and calls the Service.
4.  **Backend Service** calls `AuthenticationManager`.
5.  **Auth Manager** checks the DB, hashes the input password, and compares it with the stored hash.
6.  **If Valid**: Service generates a **JWT Token**.
7.  **Backend** returns the Token + User Info.
8.  **Frontend** saves the Token in `localStorage`.
9.  **Frontend** navigates to the Home Page.
10. **Future Requests**: The `JwtInterceptor` automatically attaches the Token to every request, proving who the user is.


# 🔐 Complete Code Walkthrough: User Login Flow

This document provides a line-by-line explanation of the code used for Logging In. We will trace how data travels from the HTML Input all the way to the Database and back, explaining the specific logic in YOUR project files.

---

# 🎨 PART 1: FRONTEND (Angular)

We start where the user interacts with the application.

## 1. `login.component.html` (The View)
**Location:** `Frontend/src/app/components/login/login.component.html`

This file creates the visual form.

**Key Code Explanation:**
```html
<form [formGroup]="loginForm" (ngSubmit)="onSubmit()">
    <!-- Email Input -->
    <input formControlName="email">
    <!-- Password Input -->
    <input formControlName="password">
    <button type="submit">Login</button>
</form>
```
- **`[formGroup]="loginForm"`**: Tells Angular "This form is controlled by the variable `loginForm` in the TypeScript file."
- **`formControlName="email"`**: Connects this specific input box to the `email` field inside `loginForm`.
- **`(ngSubmit)="onSubmit()"`**: When the button is clicked, do NOT refresh the page. Instead, run the `onSubmit()` function.

## 2. `login.component.ts` (The Logic)
**Location:** `Frontend/src/app/components/login/login.component.ts`

This handles the user interaction logic.

**Initialization (Constructor):**
```typescript
this.loginForm = this.fb.group({
  email: ['', [Validators.required, Validators.email]],
  password: ['', [Validators.required, Validators.minLength(6)]]
});
```
- We use `FormBuilder` to define the rules. The email MUST be present and valid. The password MUST be at least 6 chars.

**The Trigger (`onSubmit`):**
```typescript
onSubmit() {
    if (this.loginForm.valid) {
        // 1. Call the Service
        this.authService.login(this.loginForm.value).subscribe({
            next: () => {
                // 2. Success! Navigate to Home
                this.router.navigate(['/home']);
            },
            error: (err) => {
                // 3. Failure! Show error message
                this.responseMessage = 'Invalid email or password.';
            }
        });
    }
}
```
- The component **does not know** about HTTP or Databases. It just asks `AuthService` to "try logging in."

## 3. `auth.service.ts` (The Courier)
**Location:** `Frontend/src/app/services/auth.service.ts`

This service transports data to the backend.

```typescript
login(request: AuthRequest): Observable<AuthResponse> {
    // Sends POST request to Backend
    return this.http.post<AuthResponse>('http://localhost:8081/api/auth/login', request)
        .pipe(
            // "Tap" lets us look at the response without changing it
            tap(response => {
                // SAVE THE TOKEN! This is crucial.
                localStorage.setItem('token', response.token);
                // Save user details
                localStorage.setItem('currentUser', JSON.stringify(response));
            })
        );
}
```
- **`http.post`**: Sends the JSON package (`{email: "...", password: "..."}`) to the server.
- **`localStorage.setItem('token', ...)`**: Stores the "Digital ID Badge" (JWT) in the browser's memory so the user stays logged in.

---

# ⚙️ PART 2: BACKEND (Spring Boot)

Now the data arrives at the server.

## 4. `AuthController.java` (The Gatekeeper)
**Location:** `Backend/src/main/java/com/mediaconnect/backend/controller/AuthController.java`

```java
@PostMapping("/login")
public ResponseEntity<AuthResponse> login(@RequestBody AuthRequest request) {
    // Pass the work to the Service
    AuthResponse response = authService.login(request);
    return ResponseEntity.ok(response);
}
```
- **`@RequestBody`**: Takes the JSON from Angular and turns it into a Java Object (`AuthRequest`).
- It delegates the complex logic to `authService`. It does NOT do any validation itself.

## 5. `AuthServiceImpl.java` (The Brain)
**Location:** `Backend/src/main/java/com/mediaconnect/backend/service/implementations/AuthServiceImpl.java`

This is the most critical file.

```java
public AuthResponse login(AuthRequest request) {
    // 1. AUTHENTICATE
    authenticationManager.authenticate(
        new UsernamePasswordAuthenticationToken(request.getEmail(), request.getPassword())
    );
    
    // 2. FETCH USER (Only happens if step 1 succeeds)
    User user = userRepository.findByEmail(request.getEmail());

    // 3. GENERATE TOKEN
    String token = jwtUtil.generateToken(user.getEmail());

    // 4. RETURN RESPONSE
    return new AuthResponse(token, user.getRole(), user.getId(), mapToDTO(user));
}
```
- **`authenticationManager.authenticate(...)`**: This single line does a LOT.
    - It finds the user in the DB.
    - It hashes the input password (e.g., `123456` -> `$2a$10$...`).
    - It checks if the hashed input matches the hashed password in the DB.
    - If they DON'T match, it **throws an exception** immediately.
- **`jwtUtil.generateToken`**: Creates the encrypted string containing the user's identity.

## 6. `JwtUtil.java` (The Badge Maker)
**Location:** `Backend/src/main/java/com/mediaconnect/backend/security/JwtUtil.java`

This file handles the cryptography for the token.

```java
public String generateToken(String email) {
    return Jwts.builder()
            .setSubject(email) // Who is this for?
            .setIssuedAt(new Date()) // When was it made?
            .setExpiration(new Date(System.currentTimeMillis() + 10 hour)) // When does it die?
            .signWith(SECRET_KEY) // Lock it with our secret key
            .compact();
}
```
- It uses a **SECRET KEY**. Only the server knows this key. This prevents users from faking their own tokens.

## 7. `SecurityConfig.java` (The Rules)
**Location:** `Backend/src/main/java/com/mediaconnect/backend/security/SecurityConfig.java`

This file configures Spring Security.

```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/auth/**").permitAll() // Allow Login/Register without token
    .anyRequest().authenticated() // Block EVERYTHING else unless they have a token
)
```
- It tells Spring: "Let anyone access the login page. But for everything else (like watching movies), they MUST show a valid token."

## 8. `GlobalExceptionHandler.java` (The Safety Net)
**Location:** `Backend/src/main/java/com/mediaconnect/backend/exception/GlobalExceptionHandler.java`

If the password is wrong, `AuthServiceImpl` (via AuthManager) throws `BadCredentialsException`. This file catches it.

```java
@ExceptionHandler(Exception.class)
public ResponseEntity<ErrorResponse> handleGlobalException(Exception ex) {
    // Returns a nice JSON message to Angular instead of a 500 server crash
    return new ResponseEntity<>(error, HttpStatus.INTERNAL_SERVER_ERROR);
}
```

---

# 🔄 PART 3: CONTINUOUS AUTHENTICATION (JWT Filter)

Once logged in, how does the server know who we are for the NEXT request?

## 9. `JwtFilter.java` (The ID Checker)
**Location:** `Backend/src/main/java/com/mediaconnect/backend/security/JwtFilter.java`

This code runs **before every single request**.

```java
protected void doFilterInternal(...) {
    // 1. Get the Authorization Header
    String authHeader = request.getHeader("Authorization");

    // 2. Check if it starts with "Bearer "
    if (authHeader != null && authHeader.startsWith("Bearer ")) {
        String token = authHeader.substring(7);
        // 3. Extract Email from Token
        String email = jwtUtil.extractUsername(token);

        // 4. Validate Token
        if (jwtUtil.validateToken(token, email)) {
            // 5. Tell Spring "This user is valid!"
            SecurityContextHolder.getContext().setAuthentication(...);
        }
    }
    chain.doFilter(request, response);
}
```
- It intercepts requests.
- It "unlocks" the door if the token is valid.
- If the token is fake or expired, it does nothing (user remains unauthenticated), and `SecurityConfig` blocks them.

---

# 📊 DATA TRANSFER OBJECTS (DTOs)

These simple classes communicate data between Frontend and Backend.

**`AuthRequest`**:
- Input: `{ email, password }`

**`AuthResponse`**:
- Output: `{ token, role, userId, userDTO }`
- We use this so we don't send sensitive info (like password hash) back to the UI.

**`UserDTO`**:
- A safe version of the User object for the frontend to display (Name, Genre Preferences, etc).

---

# 📝 LOGIC SUMMARY

1. **User Input**: Angular Form collects Email + Password.
2. **Transport**: Value sent via POST to `/api/auth/login`.
3. **Guard**: Controller receives it.
4. **Logic**: Service asks AuthManager "Is this password correct for this email?".
5. **DB Access**: AuthManager checks `users` table via Repository.
6. **Decision**:
   - **Match?**: Generate JWT Token -> Return 200 OK with Token.
   - **No Match?**: Throw Exception -> Return 401/500 Error.
7. **Storage**: Frontend saves Token in `localStorage`.
8. **Future**: Frontend attaches Token to headers; Backend `JwtFilter` checks it.
