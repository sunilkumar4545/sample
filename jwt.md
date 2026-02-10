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


# JWT Filter Logic: A Deep Dive

This document provides a line-by-line explanation of the `doFilterInternal` method in `JwtFilter.java`. This method is the "Gatekeeper" of your backend, intercepting every single request to check for a valid ID card (the Token).

---

## 🏛️ The Big Picture

Think of `JwtFilter` as the security guard at the entrance of a building.
1.  **Request Arrives**: Someone tries to enter (`GET /api/movies`).
2.  **Check Header**: The guard asks, "Do you have your ID badge?" (`Authorization: Bearer <token>`).
3.  **Validate**: The guard scans the badge. Is it fake? Is it expired?
4.  **Grant Access**: Ifvalid, the guard updates the **Security Log** (`SecurityContextHolder`) saying "User ABC is inside".
5.  **Proceed**: The user is allowed to go to the front desk (`Controller`).

---

## 🔍 Line-by-Line Code Walkthrough

```java
@Override
protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
        throws ServletException, IOException {
```
*   **Concept**: `OncePerRequestFilter`. This ensures this logic runs exactly once for every API call.
*   **Arguments**:
    *   `request`: The incoming HTTP packet (Headers, Body, URL).
    *   `response`: The outgoing reply we are building.
    *   `chain`: The list of other filters to run after this one (like CORS, Logging).

### Step 1: Extract the Header

```java
final String authHeader = request.getHeader("Authorization");

String email = null;
String jwt = null;
```
*   **Logic**: We look for a header named `Authorization`.
*   **Example Header**: `Authorization: Bearer eyJhbGciOiJIUzI1Ni...`

### Step 2: Parse the Token

```java
if (authHeader != null && authHeader.startsWith("Bearer ")) {
    jwt = authHeader.substring(7); // Remove "Bearer " prefix
    try {
        email = jwtUtil.extractUsername(jwt);
    } catch (Exception e) {
        // Token extraction failed
    }
}
```
*   **Why `substring(7)`?**: "Bearer " is exactly 7 characters (6 letters + 1 space). We want only the messy string part after it.
*   **Action**: We ask `jwtUtil` to blindly read the email inside the token. Note: We haven't verified if the signature is valid yet, we just want to know *who* claims to be knocking.

### Step 3: Check Authentication Status

```java
if (email != null && SecurityContextHolder.getContext().getAuthentication() == null) {
```
*   **Condition A**: `email != null`: We successfully found a token.
*   **Condition B**: `SecurityContextHolder...authentication == null`: The user is NOT already logged in for this request.
    *   *Why?* Since we are stateless, every request starts as "Not Logged In". We must prove it every time.

### Step 4: Load User from Database

```java
    UserDetails userDetails = this.userDetailsService.loadUserByUsername(email);
```
*   **Action**: We look up the full user record (Password, Role/Authorities) from the MySQL database using the email we found in the token.
*   **Variable**: `userDetails` sits here, waiting to be compared.

### Step 5: The Crucial Validation

```java
    if (jwtUtil.validateToken(jwt, userDetails.getUsername())) {
```
*   **Action**: We pass the **Raw Token** (`jwt`) and the **Database Username** to `jwtUtil.validateToken()`.
*   **Checks Performed**:
    1.  **Expiration**: Is `Date.now() < Token.Expiration`?
    2.  **Signature**: Does the token's mathematical signature match our Server's Secret Key? (Prevents tampering).
    3.  **User Match**: Does the token belong to the user we just loaded?

### Step 6: Create the "Ticket" (Authentication Token)

```java
        UsernamePasswordAuthenticationToken authToken = new UsernamePasswordAuthenticationToken(
                userDetails, null, userDetails.getAuthorities());
                
        authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
```
*   **Concept**: `UsernamePasswordAuthenticationToken`.
*   **Explanation**: This is NOT the JWT. This is an internal Spring Security object that acts as a "Authenticated Session Ticket" for the duration of just this one request.
*   **Parameters**:
    1.  `userDetails`: The user object (Who they are).
    2.  `null`: Credentials (we erase the password for safety).
    3.  `authorities`: The roles (e.g., `ROLE_ADMIN`) so `@PreAuthorize` can work.

### Step 7: Stamp the Ticket

```java
        SecurityContextHolder.getContext().setAuthentication(authToken);
        logger.info("JWT token validated successfully for user: {}", email);
```
*   **Crucial Step**: We place the `authToken` into the global `SecurityContext`.
*   **Result**: From line 56 onwards, anywhere in the app (Controllers, Services) calls `SecurityContextHolder.getContext().getAuthentication()`, it will find this User. **The user is now officially logged in.**

### Step 8: Allow Passage

```java
chain.doFilter(request, response);
```
*   **Action**: We are done. We pass the request to the next filter. Eventually, it reaches `MovieController`.

---

## 📝 Simple Real-World Analogy

Imagine checking into a Flight:

1.  **Header Check**: You walk to TSA and show your Boarding Pass (Token).
2.  **Extract Info**: The agent reads "John Doe" (Email) from the pass.
3.  **DB Check**: The agent looks up "John Doe" in the Flight Database (`loadUserByUsername`).
4.  **Validation**:
    *   Is the flight today? (Expiration).
    *   Is the pass printed on valid paper? (Signature).
    *   Does the face match the ID? (User Match).
5.  **Context**: The agent stamps "Security Cleared" on your pass (`setAuthentication`).
6.  **Filter Chain**: You proceed to the Gate (`Controller`).

---

## 🗣️ Interview Explanation

"In the `doFilterInternal` method, I first extract the JWT from the `Authorization` header by stripping the 'Bearer ' prefix. I decode the username (email) from the token. If a username exists and the user isn't already authenticated in the current context, I fetch the user's details from the database.

Then, I perform a critical validation step: checking if the JWT signature is valid and if it matches the user details. If valid, I create a `UsernamePasswordAuthenticationToken`—Spring's internal representation of a logged-in user—populated with their authorities (Roles). Finally, I ensure to set this token in the `SecurityContextHolder`, effectively logging the user in for the duration of that request, before passing control up the chain."
