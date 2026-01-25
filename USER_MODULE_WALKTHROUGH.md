# 👤 User Module: Detailed Code Walkthrough

This document dissects the code for the User Profile module, explaining exactly how data flows and how specific lines of code implement the business logic.

---

# 🎨 PART 1: FRONTEND (Angular)

## 1. `profile.component.html` (The View)
**Location:** `Frontend/src/app/components/profile/profile.component.html`

This file handles the visual presentation.

```html
<!-- LOADING STATE: Spinner shows while 'loading' is true -->
@if (loading) {
    <div class="loading-state">
        <div class="spinner"></div>
    </div>
} 
@else if (user) {
    <!-- DISPLAY MODE: Shows Name -->
    <div class="detail-group">
        <label>Full Name</label>
        @if (isEditing) {
            <!-- EDIT MODE: Shows Input -->
            <input type="text" [(ngModel)]="editData.fullName" class="form-input">
        } @else {
            <div class="value-large">{{ user.fullName }}</div>
        }
    </div>

    <!-- SUBSCRIPTION BADGE LOGIC -->
    <div [ngClass]="user.subscriptionStatus === 'ACTIVE' ? 'status-active' : 'text-danger'">
        {{ user.subscriptionStatus || 'INACTIVE' }}
    </div>
}
```
**Logic Explanation:**
- **Control Flow (`@if`)**: We manage three states: Loading, View, and Edit.
- **Two-Way Binding (`[(ngModel)]`)**: When the user types in the input box, `editData.fullName` updates instantly in the TypeScript code.
- **Dynamic Styling (`[ngClass]`)**: The subscription status text turns green (`status-active`) or red (`text-danger`) based on the value of `user.subscriptionStatus`.

## 2. `profile.component.ts` (The Logic)
**Location:** `Frontend/src/app/components/profile/profile.component.ts`

**Initialization Code:**
```typescript
ngOnInit() {
    this.refreshUser(); // AUTOMATIC TRIGGER
}

refreshUser() {
    this.loading = true; // Start Spinner
    this.userService.getMe().subscribe({
        next: (data) => {
            this.user = data; // Save fetched data
            
            // SYNC NAVBAR: Update the global auth state regarding who is logged in
            if (data) {
                 this.authService.updateUser({ ...data } as any);
            }
            this.loading = false; // Stop Spinner
        }
    });
}
```

**Update (Save) Logic:**
```typescript
saveProfile() {
    const originalEmail = this.user?.email; // 1. Remember old email

    this.userService.updateMe(this.editData).subscribe({
        next: (updatedUser) => {
            // 2. CHECK: Did email change?
            if (updatedUser.email !== originalEmail) {
                this.responseMessage = 'Email updated! Redirecting to login...';
                // 3. SECURITY: Force Logout because Token is now invalid
                setTimeout(() => {
                    this.authService.logout();
                }, 2000);
                return;
            }

            // 4. NORMAL UPDATE: Just update the view
            this.user = updatedUser;
            this.isEditing = false;
        }
    });
}
```

## 3. `user.service.ts` (The Bridge)
**Location:** `Frontend/src/app/services/user.service.ts`

```typescript
@Injectable({ providedIn: 'root' })
export class UserService {
    private apiUrl = 'http://localhost:8081/api/users';

    // GET Request: "Who am I?"
    getMe(): Observable<UserDTO> {
        return this.http.get<UserDTO>(`${this.apiUrl}/me`);
    }

    // PUT Request: "Update me"
    updateMe(user: UserDTO): Observable<UserDTO> {
        return this.http.put<UserDTO>(`${this.apiUrl}/me`, user);
    }
}
```
**Note:** We do not pass an ID (like `/users/1`). The backend identifies the user strictly via the JWT Token to prevent IDOR attacks (accessing someone else's data).

---

# ⚙️ PART 2: BACKEND (Spring Boot)

## 4. `UserController.java` (The Gatekeeper)
**Location:** `Backend/.../controller/UserController.java`

```java
@GetMapping("/me")
public ResponseEntity<UserDTO> getCurrentUser() {
    // 1. SECURITY CONTEXT: Who is this?
    // JwtFilter already ran and put the UserDetails here.
    Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
    String email = authentication.getName(); 

    // 2. DELEGATE: Ask Service to get data for THIS email
    UserDTO userDTO = userService.getUserByEmail(email);
    
    return ResponseEntity.ok(userDTO);
}
```
**Logic Explanation:**
- **Statelessness:** The method relies entirely on `SecurityContextHolder`. It assumes the request passed the `JwtFilter`.

## 5. `UserServiceImpl.java` (The Business Logic)
**Location:** `Backend/.../service/implementations/UserServiceImpl.java`

**Fetch Method (`getUserByEmail`):**
```java
@Override
public UserDTO getUserByEmail(String email) {
    // 1. DB Lookup
    User user = userRepository.findByEmail(email);
    
    if (user != null) {
        // 2. LAZY EXPIRATION CHECK
        // "Is your plan expired NOW?"
        checkSubscriptionStatus(user);
        
        // 3. MAP TO DTO (Hide Password)
        return mapToDTO(user);
    }
    // 4. ERROR HANDLING
    throw new ResourceNotFoundException("User not found with email: " + email);
}
```

**Subscription Check Logic:**
```java
private void checkSubscriptionStatus(User user) {
    // Check if status is ACTIVE *AND* expiry date is in the past
    if ("ACTIVE".equals(user.getSubscriptionStatus()) &&
        user.getSubscriptionExpiry().isBefore(LocalDateTime.now())) {
            
        // EXPIRE THEM!
        user.setSubscriptionStatus("INACTIVE");
        userRepository.save(user); // Commit change to DB
    }
}
```

**Update Method (`updateUser`):**
```java
@Override
public UserDTO updateUser(String email, UserDTO userDTO) {
    User user = userRepository.findByEmail(email);

    // LOGIC: Email Change
    if (!userDTO.getEmail().equals(user.getEmail())) {
        // Validation: Is new email taken?
        if (userRepository.existsByEmail(userDTO.getEmail())) {
            throw new RuntimeException("Email already in use");
        }
        user.setEmail(userDTO.getEmail());
    }

    // LOGIC: Name Change
    if (userDTO.getFullName() != null) {
        user.setFullName(userDTO.getFullName());
    }

    userRepository.save(user); // Save to DB
    return mapToDTO(user);
}
```

## 6. `UserDTO.java` (The Data Carrier)
**Location:** `Backend/.../dto/response/UserDTO.java`

```java
public class UserDTO {
    private Long id;
    private String fullName;
    private String email;
    private String role; 
    // Notice: NO PASSWORD field here.
    // This protects sensitive data.
}
```

## 7. `GlobalExceptionHandler.java` (The Safety Net)
**Location:** `Backend/.../exception/GlobalExceptionHandler.java`

If `UserServiceImpl` throws `ResourceNotFoundException`:

```java
@ExceptionHandler(ResourceNotFoundException.class)
public ResponseEntity<ErrorResponse> handleResourceNotFound(ResourceNotFoundException ex) {
    // Return nice JSON { status: 404, message: "User not found..." }
    ErrorResponse error = new ErrorResponse( HttpStatus.NOT_FOUND.value(), ex.getMessage());
    return new ResponseEntity<>(error, HttpStatus.NOT_FOUND);
}
```

---

# 📝 FLOW SUMMARY

1.  **Trigger:** `profile.component.ts` calls `userService.getMe()`.
2.  **Request:** HTTP GET with JWT Token (`Authorization: Bearer ...`).
3.  **Security:** `JwtFilter` validates token -> sets `SecurityContext`.
4.  **Controller:** `UserController` asks `SecurityContext` for email.
5.  **Service:** `UserService` fetches `User` entity from DB.
6.  **Logic:** `UserService` checks and updates subscription expiry (Lazy Check).
7.  **Protection:** DTO mapper removes password.
8.  **Response:** JSON UserDTO sent to frontend.
9.  **View:** HTML displays the data.

---

# ⚡ COMMUNICATION LOGIC DEEP DIVE (How it REALLY works)

This section explains the invisible handshake between the Frontend and Backend that makes the code work.

### 1️⃣ The Handshake (Frontend -> Backend)
The `UserService` in Angular makes a seemingly simple request:
`this.http.get('api/users/me')`

**BUT** underneath the hood, the **Interceptor** intercepts this call before it leaves the browser:
1.  **Interceptor (`jwt.interceptor.ts`)** wakes up.
2.  It checks `localStorage` for a `token`.
3.  **Found:** `eyJhbGciOiJIUz...`
4.  **Action:** It stamps the request with `Authorization: Bearer eyJhbGci...`.
5.  **Send:** Now the request travels over the internet.

### 2️⃣ The Gatekeeper (Backend Entry)
The request reaches the Spring Boot server (Tomcat).
1.  Before any Controller code runs, the **Filter Chain** activates.
2.  **`JwtFilter.class`** inspects the header.
3.  **Validation:** It uses `JwtUtil` to mathematically verify the signature using the Secret Key.
    - *Is it expired?* No.
    - *Is the signature valid?* Yes.
    - *Who is it?* "john@email.com"
4.  **Context Setting:** `SecurityContextHolder.getContext().setAuthentication(...)`
    - This is like putting a name tag on the request thread.

### 3️⃣ The Logic Execution (Controller & Service)
Now the `UserController.getCurrentUser()` actually runs.
1.  It doesn't look at the request body or URL ID.
2.  It looks at the "Name Tag" (`SecurityContextHolder`).
3.  **Fetch:** `userService.getUserByEmail("john@email.com")`.
4.  **DB Query:** `SELECT * FROM users WHERE email = 'john@email.com'`.
5.  **Return:** The Database Entity is converted to a DTO (stripped of password) and sent back as JSON.

### 4️⃣ The Response (Backend -> Frontend)
Angular receives the JSON:
`{ "fullName": "John Doe", "email": "john@email.com", ... }`

The Component subscribes to this data and updates the HTML variables (`this.user = data`), which instantly renders on screen.
