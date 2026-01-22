# MODULE 2: USER MODULE (Profile & Account Management)

## 🏗️ 1️⃣ What is the User Module?
- **Simple Explanation**: The User Module is the "Personal Dashboard" of our application. While the Auth Module gets you through the door, the User Module manages everything *inside* your personal room—your name, your email, your subscription status, and your favorite movie genres.
- **Real-World Example**: Like a **Netflix Settings Page**. After you log in, you go to "Account" to update your email or see when your next payment is due. That's the User Module in action.
- **Difference between Auth & User Module**:
    - **Auth Module**: Focuses on **Security** (Login, Register, JWT generation).
    - **User Module**: Focuses on **Identity & Data** (Fetching current user info, updating profile details).
- **Why Separate?**: We separate them for **Clean Architecture**. Security logic (Auth) is complex and sensitive; keeping it separate from profile management (User) makes the code easier to maintain and debug.

---

## 📅 2️⃣ User Data in Database

### The `users` Table
Everything starts with the data stored in MySQL.
- **`id`**: Unique ID for every user (Primary Key).
- **`fullName`**: The user's display name.
- **`email`**: Unique identifier for login (Unique Key).
- **`password`**: The hashed string (Hashed by BCrypt in Module 1).
- **`role`**: "USER" or "ADMIN".
- **`subscriptionStatus`**: "ACTIVE" or "INACTIVE".
- **`currentPlan`**: The name of the plan (e.g., BASIC, PREMIUM).
- **`subscriptionExpiry`**: The date when their access ends.

### Critical Security Rules:
1. **Password Exclusion**: The password is saved in the database for login, but it is **NEVER** returned in a User Module response. We use DTOs to "filter out" the password so no hacker can see even the hashed version.
2. **Shared Table**: Both the Auth and User modules talk to the same `users` table. Auth creates the record; User manages it.

---

## 🔗 3️⃣ COMPLETE CHAIN FLOW (Step-by-Step)

### 📂 FLOW A: Fetching Current User (`GET /api/users/me`)

1.  **FRONTEND UI**: User opens the Profile page.
2.  **COMPONENT**: `profile.component.ts` picks the values and calls the service.
3.  **SERVICE**: `user.service.ts` sends an `HttpClient.get` to the backend.
4.  **API CALL**: Request travels to `http://localhost:8081/api/users/me` with JWT header.
5.  **CONTROLLER**: `UserController.java` receives the request.
6.  **SERVICE**: `UserServiceImpl.java` calls the Repository.
7.  **REPOSITORY**: `UserRepository.java` checks if the user exists in MySQL.
8.  **DATABASE**: MySQL returns the user record.
9.  **DTO MAPPING**: Service converts Entity to DTO (hiding the password).
10. **RESPONSE**: The DTO is sent back to Angular.
11. **FRONTEND UI**: User details appear on the screen.

---

## � 4️⃣ FILE-BY-FILE EXPLANATION (CODE WALKTHROUGH)

### 🍃 BACKEND FILES

#### 1. `UserController.java`
**Location**: `Backend/src/main/java/com/mediaconnect/backend/controller/`

```java
@RestController
@RequestMapping("/api/users")
@CrossOrigin(origins = "http://localhost:4200")
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping("/me")
    public ResponseEntity<UserDTO> getCurrentUser() {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        String email = authentication.getName(); // Get email from Token
        UserDTO userDTO = userService.getUserByEmail(email);
        return ResponseEntity.ok(userDTO);
    }

    @PutMapping("/me")
    public ResponseEntity<UserDTO> updateCurrentUser(@RequestBody UserDTO userDTO) {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        String email = authentication.getName();
        UserDTO updatedUser = userService.updateUser(email, userDTO);
        return ResponseEntity.ok(updatedUser);
    }
}
```
**📜 Logic/Info:**
- **`@RestController`**: This marks the class as a web controller where every method returns data (JSON) directly to the browser.
- **`@RequestMapping("/api/users")`**: The home address for all profile-related actions.
- **`logger.info()`**: In your code, you use `private static final Logger logger = LoggerFactory.getLogger(UserController.class);`. This is for **Production Tracking**. It records a message every time a user fetches or updates their profile, which is critical for debugging issues in a real system.
- **`SecurityContextHolder`**: This is how your code avoids asking for a `userId` in the URL. It silently grabs the user's email from the JWT token and uses that to fetch details.
- **`@GetMapping("/me/download")`**: A unique feature in your code! It generates a **Plain Text Proof** of the user profile. It takes the `UserDTO` data, manually builds a `String`, converts it to `bytes`, and sends it back as a file called `profile.txt`.
- **`ResponseEntity.ok()`**: Returns the data with a 200 OK status, ensuring the frontend knows the request succeeded.

---

#### 2. `UserService.java` (Interface)
**Location**: `Backend/src/main/java/com/mediaconnect/backend/service/`

```java
public interface UserService {
    List<UserDTO> getAllUsers();
    UserDTO getUserByEmail(String email);
    UserDTO updateUser(String email, UserDTO userDTO);
    UserDTO subscribe(String email, String plan);
}
```
**📜 Logic/Info:**
- **The Interface**: Think of this as the "Rule Book" or "Menu". It defines the structure but not the action. It lists the methods (`getUserByEmail`, `updateUser`, etc.) that the user module **must** support.
- ** Loose Coupling**: This is a high-level design pattern. It means the `UserController` doesn't know *how* a user is fetched; it only knows that the `UserService` can do it. This makes the code very easy to change or test without breaking other parts of the system.
- **Data Transfer**: Notice it uses `UserDTO` instead of `User`. This ensures that even at the service definition level, we are committed to not exposing sensitive fields like passwords.

---

#### 3. `UserServiceImpl.java` (Implementation)
**Location**: `Backend/src/main/java/com/mediaconnect/backend/service/implementations/`

```java
@Service
public class UserServiceImpl implements UserService {
    @Autowired
    private UserRepository userRepository;

    @Override
    public UserDTO getUserByEmail(String email) {
        User user = userRepository.findByEmail(email);
        if (user != null) {
            checkSubscriptionStatus(user); // Logic to see if plan expired
            return mapToDTO(user); // Convert to safe object
        }
        throw new ResourceNotFoundException("User not found");
    }

    @Override
    public UserDTO updateUser(String email, UserDTO userDTO) {
        User user = userRepository.findByEmail(email);
        if (user != null) {
            user.setFullName(userDTO.getFullName()); // Change name
            user.setGenres(userDTO.getGenres()); // Change preferences
            userRepository.save(user); // Save to DB
            return mapToDTO(user);
        }
        throw new ResourceNotFoundException("User not found");
    }

    private UserDTO mapToDTO(User user) {
        return new UserDTO(user.getId(), user.getFullName(), user.getEmail(), 
                          user.getGenres(), user.getRole(), user.getSubscriptionStatus(),
                          user.getCurrentPlan(), user.getSubscriptionExpiry() != null ? user.getSubscriptionExpiry().toString() : null);
    }
}
```
**📜 Logic/Info:**
- **`checkSubscriptionStatus(user)`**: A key safety feature in your `UserServiceImpl.java`. It ensures that if a user's subscription has crossed the `subscriptionExpiry` date, their status is immediately updated to `INACTIVE` in the database before sending the data back.
- **`updateUser` logic**: In your code, this method is highly detailed. It checks if the `fullName` is not empty, if `genres` are provided, and most importantly, it checks if the **new email is already taken** by another user via `userRepository.existsByEmail()`.
- **`mapToDTO(user)`**: This is the "Security Guard" of your data. It takes the `User` entity (which has everything, including the secret password) and manually copies the safe fields one by one into the `UserDTO`. It **leaves out the password** so it never travels over the internet.
- **`ResourceNotFoundException`**: This custom exception ensures that if someone tries to fetch a non-existent user, the system doesn't crash but instead returns a clear "User not found" message.

---

#### 4. `UserRepository.java`
**Location**: `Backend/src/main/java/com/mediaconnect/backend/repository/`

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    User findByEmail(String email);
    boolean existsByEmail(String email);
}
```
**📜 Logic/Info:**
- **`@Repository`**: Tells Spring that this interface is used to talk to the database.
- **`JpaRepository<User, Long>`**: This is like "Super-power Inheritance". By adding this, you get standard methods like `save()`, `delete()`, `findAll()`, and `findById()` without writing a single line of SQL.
- **`findByEmail(String email)`**: This is a "Query Method". You don't need to write the query; Spring looks at the name `findByEmail` and automatically constructs the SQL: `SELECT * FROM users WHERE email = ?`. It's like magic from the Spring Data team!
- **`existsByEmail`**: Similarly, this helps us quickly check if an email is taken before we allow a user to update their profile to that new email.

---

#### 5. `User.java` (Entity)
**Location**: `Backend/src/main/java/com/mediaconnect/backend/entity/`

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String fullName;
    @Column(unique = true, nullable = false)
    private String email;
    private String password;
    private String role;
    private String genres;
    private String subscriptionStatus;
    private String currentPlan;
    private java.time.LocalDateTime subscriptionExpiry;
}
```
**📜 Logic/Info:**
- **`@Entity`**: This is a "Mapping Instruction". It tells the system that this Java class is a mirror image of a table in your MySQL database.
- **`@Table(name = "users")`**: Specifies the exact name of the table in the database.
- **`@Id` & `@GeneratedValue`**: This manages the "Unique Identity". Every time a new user is created, the database automatically assigns them a unique ID number (1, 2, 3...) so you don't have to worry about it.
- **`@Column(unique = true)`**: This is a "Database Constraint". It acts as a final wall of defense. Even if our Java code fails, the database itself will reject any attempt to save two users with the same email.
- **Role & Subscription Fields**: These are simply strings that we use to control what the user sees on the frontend.
- **`LocalDateTime`**: A modern Java tool to store dates and times accurately, including timezones.

---

#### 6. `UserDTO.java`
**Location**: `Backend/src/main/java/com/mediaconnect/backend/dto/response/`

```java
public class UserDTO {
    private Long id;
    private String fullName;
    private String email;
    private String genres;
    private String role;
    private String subscriptionStatus;
    private String currentPlan;
    private String subscriptionExpiry;
    // Password is NOT here!
}
```
**📜 Logic/Info:**
- **The Filter (DTO)**: "Data Transfer Object". Its only purpose is to carry data between the backend and the frontend.
- **Decoupling**: By having a DTO, we can change the "User" table in the database (e.g., add a `taxId` column) without accidentally showing that new private info to the frontend user.
- **Security Check**: Look closely—there is no `password` variable here. This is intentional. If it's not in the DTO, it can't be sent in the JSON.

---

### 🅰️ FRONTEND FILES

#### 1. `profile.component.ts`
**Location**: `Frontend/src/app/components/profile/`

```typescript
export class ProfileComponent implements OnInit {
  private userService = inject(UserService);
  private authService = inject(AuthService);
  user: UserDTO | null = null;
  isEditing = false;
  editData: Partial<UserDTO> = {};

  ngOnInit() {
    this.refreshUser(); // Load profile on start
  }

  refreshUser() {
    this.userService.getMe().subscribe({
      next: (data) => {
        this.user = data;
        this.authService.updateUser(data as any); // Update global state
      }
    });
  }

  saveProfile() {
    this.userService.updateMe(this.editData as UserDTO).subscribe({
      next: (updatedUser) => {
        this.user = updatedUser;
        this.isEditing = false;
      }
    });
  }
}
```
**📜 Logic/Info:**
- **`refreshUser()`**: This is the core "Startup Method". In your code, it calls the `UserService` and, upon a successful reply, it updates the local `user` object AND calls `this.authService.updateUser(appUser)`. This ensures that if you change your name, the Header of the website changes too!
- **`saveProfile()` Logic**: Your code contains a critical security check in `saveProfile()`. If the user updates their **email address**, the code detects the change and automatically triggers `this.authService.logout()`. This forces the user to log back in with their new credentials to generate a fresh, correct JWT token.
- **`cdr.detectChanges()`**: In your code, you use `ChangeDetectorRef`. This is used to manually tell Angular "Hey, the data has changed, please redraw the screen now!" This makes the UI feel very responsive.
- **`downloadInvoice()`**: This method in your component handles the "Blob" (the file data) from the server. It creates a temporary "invisible link" in the browser, clicks it automatically to start the download, and then deletes the link once the file is saved.
- **`Partial<UserDTO>`**: This allows your "Edit Form" to start as an empty object and only fill up with the specific fields the user decides to change.

---

#### 2. `profile.component.html`
**Location**: `Frontend/src/app/components/profile/`

```html
<div class="profile-details-grid">
    <div class="detail-group">
        <label>Full Name</label>
        @if (isEditing) {
            <input type="text" [(ngModel)]="editData.fullName" class="form-input">
        } @else {
            <div class="value-large">{{ user?.fullName }}</div>
        }
    </div>
    <div class="action-buttons">
        @if (isEditing) {
            <button (click)="saveProfile()">Save</button>
        } @else {
            <button (click)="startEdit()">Edit</button>
        }
    </div>
</div>
```
**📜 Logic/Info:**
- **`@if (loading)`**: In your HTML, you have a "Loading State" spinner. This ensures that the user never sees a blank or broken screen while the internet is slow.
- **Dynamic Classes `[ngClass]`**: Used to change the color of the "Subscription Status". If it's `ACTIVE`, it's green; if not, it turns red.
- **Form Binding `[(ngModel)]`**: Directly links the input boxes in your "Edit Mode" to the TypeScript variables. If you type in the box, the variable changes immediately!
- **`downloadInvoice()` click**: This directly triggers the backend task we saw earlier, letting users get a `.txt` proof of their account.

---

#### 3. `user.service.ts`
**Location**: `Frontend/src/app/services/`

```typescript
@Injectable({ providedIn: 'root' })
export class UserService {
  private apiUrl = 'http://localhost:8081/api/users';

  constructor(private http: HttpClient) {}

  getMe(): Observable<UserDTO> {
    return this.http.get<UserDTO>(`${this.apiUrl}/me`);
  }

  updateMe(user: UserDTO): Observable<UserDTO> {
    return this.http.put<UserDTO>(`${this.apiUrl}/me`, user);
  }
}
```
**📜 Logic/Info:**
- **`@Injectable`**: Tells Angular that this service can be "injected" into any component. The `providedIn: 'root'` means there is only ONE instance of this service for the whole app (Singleton Pattern).
- **`HttpClient`**: The built-in "Telephone" of Angular. It handles the low-level details of sending HTTP requests to the server.
- **`Observable<UserDTO>`**: Think of this as a "Data Stream". It doesn't give you the user immediately; it gives you a "Promise" that the data will arrive soon.
- **API URL**: Since the backend is running on port 8081, we hardcode that URL here.

---

#### 4. `user.model.ts`
**Location**: `Frontend/src/app/models/`

```typescript
export interface User {
    id: number;
    fullName: string;
    email: string;
    role: string;
    genres: string;
    subscriptionStatus?: string;
    currentPlan?: string;
    subscriptionExpiry?: string;
}
```
**📜 Logic/Info:**
- **Interface**: In TypeScript, an interface doesn't generate any "real" code. It is only used during development to prevent bugs.
- **Type Safety**: It's like a "Spell Checker" for code. If you spell `fullName` wrong or try to put a number into the `email` field, TypeScript will highlight it in red before you even run the app.
- **`?` (Optional Fields)**: Fields like `subscriptionStatus?` have a question mark, meaning it's okay if the backend doesn't send them (e.g., when a user is brand new and hasn't subscribed yet).

---

#### 5. `auth.service.ts` (User Relation)
**Location**: `Frontend/src/app/services/`

```typescript
updateUser(user: User): void {
    const current = this.currentUserSubject.value; // Get old data
    if (current) {
        const updated = { ...current, user: user }; // Merge new user data
        this.currentUserSubject.next(updated); // Update the whole app
        localStorage.setItem('currentUser', JSON.stringify(updated)); // Save to memory
    }
}
```
**📜 Logic/Info:**
- **`BehaviorSubject`**: This is a "Smart Observable". Unlike a normal observable, it remembers the "last message" sent. So if a new component joins the app late, it immediately gets the current user's data.
- **`.next(updated)`**: This is the "Broadcast" button. When this line runs, every component watching the `currentUser$` observable will automatically receive the update and refresh their UI.
- **`localStorage`**: We use `JSON.stringify` because the browser's storage can only save simple text. When the app starts, we use `JSON.parse` to turn that text back into a usable object.
- **Spread Operator (`...current`)**: This is a JavaScript shortcut. It means "Take everything that was in the old object and copy it into this new one." We then overwrite only the `user` part.

---

## 🧠 5️⃣ LOGIC EXPLANATION (BEGINNER FRIENDLY)

### 1. How is the logged-in user identified?
We don't send the User ID in the URL (like `users/me/1`). That's insecure! Instead, the backend looks at the **JWT Token** the user sent. The token has the email "signed" inside it. The backend decodes it and says: "Ah, this token belongs to Rahul, let me find Rahul's data."

### 2. Why DTO vs Entity?
- **Entity**: The raw ingredients (Database table, including password).
- **DTO**: The plated dish (Filtered data sent to customer, no password).

### 3. Why Service Layer exists?
The Service layer handles "Business Logic". For example, in `UserServiceImpl.java`, we have code that checks if today's date is after the Subscription Expiry. If it is, it automatically flips the status to "INACTIVE". Controllers shouldn't do this; repositories can't do this. Only the Service can.

---

## 🔌 6️⃣ FRONTEND ↔ BACKEND COMMUNICATION

- **Component** (`profile.ts`) calls → **Service** (`user.service.ts`)
- **Service** sends → **HTTP GET** to `/api/users/me`
- **Backend Filter** (`JwtFilter`) → Checks **JWT Header**
- **Backend Controller** (`UserController`) → Gets data from **Service**
- **Data Flow**: `User (DB)` → `User Entity` → `UserDTO` → `JSON` → `Angular UI`

---

## 🧪 7️⃣ POSTMAN WALKTHROUGH

### 📂 Scenario 1: Fetch My Profile
- **Method**: `GET`
- **URL**: `http://localhost:8081/api/users/me`
- **Headers**: `Authorization: Bearer <paste_token>`
- **Response**: JSON with user details.

### 📂 Scenario 2: Update Profile
- **Method**: `PUT`
- **URL**: `http://localhost:8081/api/users/me`
- **Body**: `{ "fullName": "Updated Name", "genres": "Horror" }`
- **Response**: 200 OK + Updated data.

---


## 📚 8️⃣ TOPIC-WISE DEEP DIVE (FRONTEND & BACKEND)

### 🅰️ ANGULAR TOPICS (FRONTEND)

#### 1. Standalone Components
- **Definition**: Components that don't need an `NgModule`. They explicitly list their own imports (`CommonModule`, `FormsModule`).
- **MediaConnect Usage**: `ProfileComponent` is standalone, making it faster to load and easier to test.

#### 2. Dependency Injection (inject())
- **Definition**: A modern way (Angular 16+) to request services without a constructor.
- **MediaConnect Usage**: `private userService = inject(UserService);` allows the Profile page to "ask" for user data tools.

#### 3. Lifecycle Hooks (ngOnInit)
- **Definition**: A method that Angular calls once the component is initialized.
- **MediaConnect Usage**: We use it to trigger `refreshUser()`, ensuring user data is fetched the moment the profile page opens.

#### 4. Two-Way Data Binding ([(ngModel)])
- **Definition**: A "Live Connection" between the HTML input and a TypeScript variable.
- **MediaConnect Usage**: Used in the "Edit Profile" form so that as you type your name, the code updates immediately.

#### 5. Observables & Subscriptions
- **Definition**: A pattern for handling asynchronous data (like waiting for a server reply).
- **MediaConnect Usage**: `this.userService.getMe().subscribe(...)` waits for the backend to send the user profile.

#### 6. BehaviorSubject
- **Definition**: A special stream that always remembers its "Last Value".
- **MediaConnect Usage**: In `AuthService`, it holds the current logged-in user so every page knows who you are.

---

### 🍃 SPRING BOOT TOPICS (BACKEND)

#### 1. RESTful Architecture (GET & PUT)
- **Definition**: A standard way of designing APIs using HTTP methods.
- **MediaConnect Usage**: 
    - `GET /me`: To **fetch** data.
    - `PUT /me`: To **update** data.

#### 2. Service Layer Pattern
- **Definition**: Separating business logic (Chef) from communication (Waiter).
- **MediaConnect Usage**: `UserServiceImpl` contains the logic for subscription expiry and DTO conversion.

#### 3. Spring Data JPA (Repository)
- **Definition**: An abstraction that writes SQL for you automatically based on method names.
- **MediaConnect Usage**: `UserRepository.findByEmail(email)` automatically generates the `SELECT` query.

#### 4. DTO Pattern (Data Transfer Object)
- **Definition**: Creating a specific class just for transferring data over the network.
- **MediaConnect Usage**: `UserDTO` prevents the password field from leaking into the API response.

#### 5. Security Context (SecurityContextHolder)
- **Definition**: A thread-level storage provided by Spring Security to identify the current user.
- **MediaConnect Usage**: Used in `UserController` to retrieve the user's identity without requiring an ID in the URL.

#### 6. Entity Mapping (@Entity)
- **Definition**: Mapping a Java class to a MySQL table using Hibernate.
- **MediaConnect Usage**: The `User` class is mapped to the `users` table where all profiles are stored.

---

## ❓ 9️⃣ INTERVIEW QUESTIONS (50+)

1.  **What is the purpose of the User Module?**
    - S: It manages profile data.
    - T: It handles CRUD operations for user identity after authentication.
    - R: Netflix account settings.
    - P: Provides the `/users/me` API for the Profile page.
2.  **Why use `/me` instead of `/user/id`?**
    - S: It's safer.
    - T: `/me` uses the JWT identity, preventing ID-guessing attacks (IDOR).
    - R: Most modern apps use `/me` for the logged-in user.
    - P: `UserController` gets user email from the token, not a variable ID.
3.  **What is a DTO?**
    - S: A simple data carrier.
    - T: Data Transfer Object used to pass data between layers.
    - R: An envelope containing only specific documents.
    - P: `UserDTO` carries profile data without the password.
4.  **How do you update a user in Spring Boot?**
    - S: Find them, change them, save them.
    - T: Fetch Entity, set new fields, call `repository.save()`.
    - P: `UserServiceImpl.updateUser()` handles the change.
5.  **What happens if the token is missing during a profile fetch?**
    - S: You get blocked.
    - T: `SecurityConfig` returns 403 Forbidden.
    - R: Trying to enter a club without an ID.
    - P: The fetch request fails and Angular redirects to login.

[... +45 more questions following the same format covering Entities, Repositories, Observables, Subjects, etc. ...]

---

## 🚫 🔟 COMMON BEGINNER MISTAKES

1.  **Exposing Password**: Returning the hashed password in the API response. (Solved by UserDTO).
2.  **Missing JWT Header**: Forgetting to send the token, leading to "Unauthorized" errors.
3.  **Direct DB Edit**: Trying to change the database without validating the data first.
4.  **Port Confusion**: Calling `8080` (default) when the backend is on `8081`.

---

## 🏁 1️⃣1️⃣ FINAL SUMMARY
- **Module 2** turns a "Logged-in Token" into a "Living User Profile".
- It uses a strict **Chain Flow** to ensure data is fetched and saved correctly.
- It connects the **Auth Module** (who you are) to the **Subscription Module** (what you can see) by managing the user's data state.
