# Comprehensive Explanation of Login, JWT, and Security Flow

This document provides a detailed, line-by-line explanation of the authentication and security implementation in the MediaConnect application. It covers how the Frontend (Angular) and Backend (Spring Boot) communicate, how JWTs work, and the role of every file involved.

---

## 1. Key Terminology & Concepts

Before diving into the code, let's understand the core concepts and confusing terms.

### **Authentication vs. Authorization**
*   **Authentication**: Verifying *who* you are (e.g., checking username and password).
*   **Authorization**: Verifying *what* you are allowed to do (e.g., can you access the admin dashboard?).

### **Encryption vs. Hashing vs. Encoding**
*   **Encoding (Base64)**: Converting data into a different format (e.g., binary to text) so it can be safely transmitted. **It is NOT secure**. It can be easily reversed (decoded). JWTs are *encoded*, not encrypted.
*   **Hashing (BCrypt)**: A one-way process that transforms data (like a password) into a fixed-length string setup of characters. You **cannot** convert the hash back to the original password. This is why we store *hashed* passwords in the database. When you login, we hash your input and compare it to the stored hash.
*   **Encryption**: Transforming data using a key (secret) so only someone with the key can read it. You *can* decrypt it back to original form.

### **JWT (JSON Web Token)**
A token that proves your identity. It consists of three parts separated by dots (`.`):
1.  **Header**: Algorithm used (e.g., HS256).
2.  **Payload**: Data (e.g., user email, role, expiration time).
3.  **Signature**: A hash of the Header + Payload + Secret Key. This ensures the token hasn't been tampered with.

*   **Secret Key**: A private string (like a password) known ONLY to the backend. It is used to *sign* the token. If a hacker changes the payload (e.g., changes role to ADMIN), the signature won't match, and the backend will reject it.

### **AuthenticationManager**
The specific interface in Spring Security that actually performs the check: "Do these credentials match?" It coordinates the process by calling the `UserDetailsService` to get the user data and the `PasswordEncoder` to check the password.

---

## 2. Backend File Explanations (Step-by-Step)

### A. The Foundation: `User.java` (Entity)
**File**: `Backend/src/main/java/com/mediaconnect/backend/entity/User.java`
**Purpose**: Represents the database table `users`.

*   **Key Fields**:
    *   `email` (Unique): The username for login.
    *   `password`: Stores the **BCrypt hashed** password, NOT the plain text.
    *   `role`: Stores permissions (e.g., "USER", "ADMIN").

### B. Accessing Data: `UserRepository.java`
**File**: `Backend/src/main/java/com/mediaconnect/backend/repository/UserRepository.java`
**Purpose**: Allows the application to talk to the database.

*   **Line 9**: `User findByEmail(String email);`
    *   **Explanation**: Spring Data JPA automatically generates the SQL query `SELECT * FROM users WHERE email = ?`. This is used during login to find the user.

### C. The Brains: `CustomUserDetailsService.java`
**File**: `Backend/src/main/java/com/mediaconnect/backend/security/CustomUserDetailsService.java`
**Purpose**: Bridges your database (`User` entity) with Spring Security. Spring Security needs to know about users, but it doesn't know your specific `User` class. This service acts as a translator.

*   **Line 24**: `loadUserByUsername(String email)`
    *   This is the standard method Spring Security calls when it needs to verify a user.
*   **Line 26**: `userRepository.findByEmail(email)`
    *   Fetches your custom user from the DB.
*   **Line 40**: `return new org.springframework.security.core.userdetails.User(...)`
    *   **Crucial Step**: Converts your `com.mediaconnect.backend.entity.User` into a `org.springframework.security.core.userdetails.User`. This "Spring User" object is what the security framework understands.

### D. The Toolset: `JwtUtil.java`
**File**: `Backend/src/main/java/com/mediaconnect/backend/security/JwtUtil.java`
**Purpose**: Handles all JWT operations: creation, validation, and reading.

*   **Line 22**: `SECRET = "..."`
    *   This is your **Secret Key**. It's used to sign tokens. **Role**: Like the wax seal on a letter. If the seal is broken, you know someone tampered with it.
*   **Line 25**: `generateToken(String email)`
    *   Called after successful login to create the token string given to the frontend.
*   **Line 32-38**: Building the token.
    *   `setSubject(email)`: Puts the email inside the token.
    *   `setIssuedAt`, `setExpiration`: Sets validity (10 hours here).
    *   `signWith(SECRET_KEY)`: Uses your secret key to lock the token.
*   **Line 46**: `validateToken(String token, String email)`
    *   Checks two things:
        1.  Does the email in the token match the user attempting access?
        2.  Is the expiration date in the future?

### E. The Gatekeeper: `JwtFilter.java`
**File**: `Backend/src/main/java/com/mediaconnect/backend/security/JwtFilter.java`
**Purpose**: An interceptor that sits in front of every request. It checks if a valid JWT is present.

*   **Line 30**: `doFilterInternal(...)`
    *   Runs for every request.
*   **Line 34**: `request.getHeader("Authorization")`
    *   Looks for the header `Authorization: Bearer <token>`.
*   **Line 42**: `jwtUtil.extractUsername(jwt)`
    *   Decodes the token to find out *who* is claiming to be calling.
*   **Line 51**: `jwtUtil.validateToken(...)`
    *   Asks `JwtUtil` if the token is valid.
*   **Line 52-55**: `SecurityContextHolder.getContext().setAuthentication(...)`
    *   **The "Magic" Line**: This tells Spring Security "This user is valid, let them pass." without this, the user is still considered anonymous, even with a valid token.

### F. The Rules: `SecurityConfig.java`
**File**: `Backend/src/main/java/com/mediaconnect/backend/security/SecurityConfig.java`
**Purpose**: Defines the security policy for the entire application.

*   **Line 70**: `new BCryptPasswordEncoder()`
    *   Tells Spring to use the **BCrypt** hashing algorithm.
*   **Line 38-52**: `securityFilterChain` definition.
    *   `.csrf(disable)`: We don't need CSRF for Stateless APIs (JWT).
    *   `.requestMatchers("/api/auth/**").permitAll()`: **Open Door**. Allows anyone to access Login and Register URLs without a token.
    *   `.anyRequest().authenticated()`: **Closed Door**. All other URLs require a valid token.
*   **Line 55**: `.addFilterBefore(jwtFilter, ...)`
    *   Inserts your `JwtFilter` into the security chain. It says "Check for a JWT *before* you try to process standard username/password login".

### G. The Logic: `AuthServiceImpl.java`
**File**: `Backend/src/main/java/com/mediaconnect/backend/service/implementations/AuthServiceImpl.java`
**Purpose**: Handles the business logic for Login and Register.

*   **Line 85**: `authenticationManager.authenticate(...)`
    *   **The Test**: Takes the raw email and password from the user request. It hashes the incoming password and compares it to the hashed password in the DB (found by `CustomUserDetailsService`). If they don't match, it throws an error.
*   **Line 99**: `jwtUtil.generateToken(...)`
    *   If authentication passed (line 85 didn't throw error), we generate the "Pass" (JWT).

### H. Initial Setup: `DataInitializer.java`
**File**: `Backend/src/main/java/com/mediaconnect/backend/config/DataInitializer.java`
**Purpose**: Creates a default admin user on startup.

*   **Line 35**: `passwordEncoder.encode("admin123")`
    *   **Important**: Usually, you manually create users via Register. Here, we must manually *hash* the password before saving, because we are sidestepping the `AuthService`.

---

## 3. Frontend File Explanations

### I. Adding the Pass: `jwt.interceptor.ts`
**File**: `Frontend/src/app/interceptors/jwt.interceptor.ts`
**Purpose**: Automatically attaches the JWT to every outgoing HTTP request.

*   **Line 12**: `Authorization: Bearer ${token}`
    *   Adds the standard HTTP header. This corresponds exactly to what `JwtFilter.java` reads in the backend (Line 34).

### J. Protecting Pages: `auth.guard.ts`
**File**: `Frontend/src/app/guards/auth.guard.ts`
**Purpose**: Prevents users from seeing pages (like Dashboard) if they aren't logged in.

*   **Line 12**: `authService.isAuthenticated()`
    *   Checks if a token exists in local storage.
*   **Line 15**: `router.navigate(['/login'])`
    *   If no token, kicks the user back to the login page.

---

## 4. The Complete Communication Flow

### Scenario 1: User Login
(How a user gets a token)

1.  **User Action**: User enters email "messi@goat.com" and password "password123" on the Angular Login Page.
2.  **Frontend**: Sends `POST /api/auth/login` with JSON `{email, password}`.
3.  **Backend (SecurityConfig)**: Sees `/api/auth/login`. This URL is `permitAll()`, so it lets the request through without checking for a token.
4.  **Backend (AuthServiceImpl)**: `login()` method is called.
    *   Calls `authenticationManager.authenticate()`.
    *   `AuthenticationManager` asks `CustomUserDetailsService` to `loadUserByUsername("messi@goat.com")`.
    *   It retrieves the user from DB (hashed password is e.g., `$2a$10$XyZ...`).
    *   It takes the user inputs "password123", **hashes it**, and compares it to `$2a$10$XyZ...`.
    *   **Result**: Match!
5.  **Backend (AuthServiceImpl)**: Calls `jwtUtil.generateToken("messi@goat.com")`.
    *   Result: `eyJh...` (The JWT string).
6.  **Backend**: Sends `AuthResponse` containing the JWT back to Angular.
7.  **Frontend**: Saves the JWT (usually in `localStorage`).

### Scenario 2: Accessing a Protected Resource
(How the token is used)

1.  **User Action**: User clicks "View Plans" (`/plans`).
2.  **Frontend (Angular)**: Needs to fetch plan data from `GET /api/plans`.
3.  **Frontend (JwtInterceptor)**: Wakes up before the request leaves.
    *   Reads token from storage.
    *   Adds header: `Authorization: Bearer eyJh...`.
    *   Sends request.
4.  **Backend (JwtFilter)**: Intercepts the request *before* it reaches the controller.
    *   Reads the Header.
    *   Calls `jwtUtil.validateToken()`.
    *   Extracts email "messi@goat.com".
    *   Checks signature (was this token made by us?) and expiration.
    *   **Success**: It tells Spring Security "This is Messi, he is authenticated."
5.  **Backend (Controller)**: The request finally reaches the `PlansController`.
    *   The controller returns the list of plans.
6.  **Frontend**: Displays the plans.
