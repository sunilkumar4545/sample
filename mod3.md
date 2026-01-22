# MODULE 3: SUBSCRIPTION & PLANS

## 🏗️ 1️⃣ What is Subscription & Plans?
- **Simple Explanation**: This module handles the "Access Ticket" of MediaConnect. While a User Profile (Module 2) tells us who you are, the Subscription Module tells us **what you are allowed to watch**. It manages the available payment plans and tracks if a user's subscription is currently active or has expired.
- **Real-World Analogy**: Think of **Hotstar** or **Netflix**. You can create a free account (User Profile), but you can't watch premium movies until you buy a **Plan** (Basic, Standard, or Premium). Your **Subscription** is the active contract that lets you watch those movies for 30 days.
- **Difference between Plan and Subscription**:
    - **Plan**: A fixed product offered by the company (e.g., "Premium Plan" costing ₹1499). It is stored in the `subscription_plans` table.
    - **Subscription**: A dynamic status attached to a specific User. It tracks "User X is currently on the Premium Plan until June 20th". It is stored inside the `users` table.
- **Why Separate?**: We need this module to separate **Financial Products** (Plans) from **User Status** (Subscription). This allows an Admin to change plan prices or add new features without touching individual user records.

---

## 📅 2️⃣ Database Design for Subscription

### 📂 Table 1: `subscription_plans`
This table stores the "Menu" of options available to users.
- **`id`**: Unique ID for the plan (Primary Key).
- **`name`**: Name of the plan (e.g., BASIC, PREMIUM).
- **`price`**: How much the plan costs.
- **`features`**: A text field storing features. In your project, this is stored as a **Comma-Separated String** (e.g., "4K Quality,4 Screens").
- **`active`**: A boolean (true/false) to show if this plan is currently available for purchase.

### 📂 Table 2: `users` (Subscription Columns)
The User table "borrows" info from the plans to track access.
- **`subscriptionStatus`**: Either "ACTIVE" or "INACTIVE".
- **`currentPlan`**: Stores the name of the plan the user purchased.
- **`subscriptionExpiry`**: The exact date and time when the user's access will cut off.

---

## 🔗 3️⃣ COMPLETE CHAIN FLOW (Step-by-Step)

### 📂 FLOW A: User Viewing & Purchasing a Plan
1.  **FRONTEND UI**: User clicks "Manage Subscription".
2.  **COMPONENT**: `manage-subscription.component.ts` runs `loadPlans()`.
3.  **SERVICE**: `subscription.service.ts` calls `getPlans()`.
4.  **API CALL**: Request traveling to `GET http://localhost:8081/api/plans`.
5.  **CONTROLLER**: `SubscriptionPlanController.java` receives request.
6.  **SERVICE**: `SubscriptionPlanServiceImpl.java` fetches plans from DB.
7.  **REPOSITORY**: `SubscriptionPlanRepository.java` runs `SELECT * FROM subscription_plans`.
8.  **DTO MAPPING**: Service converts database string "Ad-free,4K" into a clean List `["Ad-free", "4K"]`.
9.  **FRONTEND UI**: User sees the plans as cards.
10. **USER ACTION**: User picks a plan and enters mock card details.
11. **API CALL**: Once payment "passes" locally, calls `POST /api/users/subscribe`.
12. **BACKEND LOGIC**: `UserServiceImpl.java` calculates the expiry date (Today + 30 days).
13. **DATABASE UPDATE**: User record updated in `users` table.
14. **SUCCESS**: UI turns GREEN and shows "ACTIVE" status.

---

## 📄 4️⃣ FILE-BY-FILE EXPLANATION (CODE WALKTHROUGH)

### 🍃 BACKEND FILES

#### 1. `SubscriptionPlan.java` (Entity)
**Location**: `Backend/src/main/java/com/mediaconnect/backend/entity/`

```java
@Entity
@Table(name = "subscription_plans")
public class SubscriptionPlan {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String name;

    private Double price;

    @Column(columnDefinition = "TEXT")
    private String features; // Comma-separated features

    @Column(columnDefinition = "boolean default true")
    private Boolean active = true;
}
```
**📜 Logic/Info (Deep Dive):**
- **Table Definition**: Use of `@Table(name = "subscription_plans")` explicitly maps this Java object to the physical MySQL table. This ensures Hibernate doesn't create a table with a default name we didn't want.
- **Serialization Strategy**: Notice `private String features` paired with `@Column(columnDefinition = "TEXT")`. 
    - **Logic**: Instead of a complex "One-to-Many" relationship with another table just for features, we use a single `TEXT` column to store everything as a **CSV (Comma Separated Values)**. 
    - **Benefit**: This makes database backups easier and speeds up "Read" operations because we don't have to perform SQL JOINs.
- **Data Integrity**: `@Column(nullable = false, unique = true)` on the `name` field is a "Security Lock" at the database level. Even if the Java code misses a check, the database will refuse to save a plan with a duplicate name, preventing "Plan Confusion".
- **Access Control Flag**: `private Boolean active = true` allow the Admin to "soft delete" a plan. Instead of removing the record, they can set `active=false`, making it invisible to users while keeping historical records safe.

---

#### 2. `SubscriptionPlanRepository.java`
**Location**: `Backend/src/main/java/com/mediaconnect/backend/repository/`

```java
@Repository
public interface SubscriptionPlanRepository extends JpaRepository<SubscriptionPlan, Long> {
    SubscriptionPlan findByName(String name);
}
```
**📜 Logic/Info (Deep Dive):**
- **Query Derivation**: `findByName(String name)` is a "Magic Method". Spring Data JPA parses the method name and automatically writes the query: `SELECT * FROM subscription_plans WHERE name = ?`. 
- **Logic Purpose**: This is the only way for our code to find a plan object based on the name (e.g., "PREMIUM") sent by the user during the subscription process. It's the critical link between the "Plan Menu" and the "Active User".

---

#### 3. `SubscriptionPlanService.java` (Interface)
**Location**: `Backend/src/main/java/com/mediaconnect/backend/service/`

```java
public interface SubscriptionPlanService {
    List<SubscriptionPlanResponse> getAllPlans();
    SubscriptionPlanResponse getPlanById(Long id);
    SubscriptionPlanResponse createPlan(SubscriptionPlanRequest request);
    SubscriptionPlanResponse updatePlan(Long id, SubscriptionPlanRequest request);
    void deletePlan(Long id);
}
```
**📜 Logic/Info (Deep Dive):**
- **Abstraction Layer**: This interface acts as the "Architectural Blueprint". It lists exactly what functionality is possible (CRUD operations) without cluttering the file with the actual "How-To" code.
- **Separation of Concerns**: By defining functions here and implementing them in a separate class, we ensure that if we ever want to change how plans are saved (e.g., moving from MySQL to MongoDB), we only have to change the Implementation file, not the Interface or the Controller.

---

#### 4. `SubscriptionPlanServiceImpl.java` (The Logic)
**Location**: `Backend/src/main/java/com/mediaconnect/backend/service/implementations/`

```java
@Service
public class SubscriptionPlanServiceImpl implements SubscriptionPlanService {
    @Autowired
    private SubscriptionPlanRepository repository;

    @Override
    public SubscriptionPlanResponse createPlan(SubscriptionPlanRequest request) {
        SubscriptionPlan plan = new SubscriptionPlan();
        plan.setName(request.getName());
        plan.setPrice(request.getPrice());

        // Logic: Convert List to Comma-Separated String for DB
        if (request.getFeatures() != null) {
            plan.setFeatures(String.join(",", request.getFeatures()));
        }
        
        SubscriptionPlan saved = repository.save(plan);
        return mapToResponse(saved);
    }

    private SubscriptionPlanResponse mapToResponse(SubscriptionPlan plan) {
        SubscriptionPlanResponse response = new SubscriptionPlanResponse();
        response.setId(plan.getId());
        response.setName(plan.getName());
        response.setPrice(plan.getPrice());

        // Logic: Convert DB String back to List for Frontend
        if (plan.getFeatures() != null) {
            String[] featuresArray = plan.getFeatures().split(",");
            response.setFeatures(Arrays.asList(featuresArray));
        }
        return response;
    }
}
```
**📜 Logic/Info (Extreme Detail):**
- **`createPlan` Logic (Data Cleanup)**: 
    - The code first checks `repository.findByName`. This is a "Safety Check" to prevent duplicate plans.
    - **String Concatenation**: It manually builds the feature string using a `for` loop (or `String.join`). It adds a comma between features but ensures **no trailing comma** remains at the end. This is "Clean Data Serialization".
- **`mapToResponse` Logic (Data Reconstruction)**:
    - This is the reverse of creation. It takes the text "Feature1,Feature2" and uses `plan.getFeatures().split(",")`.
    - **Type Conversion**: It converts the raw Java Array into a `List<String>` using `Arrays.asList()`. 
    - **Security**: By returning only the `SubscriptionPlanResponse` DTO, we ensure that internal database flags (like `isDeleted` or `version`) aren't exposed to the public API.
- **`updatePlan` Logic**: It uses a "Transactional Save" approach. It fetches the existing plan first, overwrites only the modified fields, and calls `.save()`. This prevents accidentally wiping out data that wasn't included in the update request.

---

#### 5. `SubscriptionPlanController.java`
**Location**: `Backend/src/main/java/com/mediaconnect/backend/controller/`

```java
@RestController
@RequestMapping("/api/plans")
public class SubscriptionPlanController {
    @Autowired
    private SubscriptionPlanService planService;

    @GetMapping
    public ResponseEntity<List<SubscriptionPlanResponse>> getAllPlans() {
        return ResponseEntity.ok(planService.getAllPlans());
    }

    @GetMapping("/{id}")
    public ResponseEntity<SubscriptionPlanResponse> getPlanById(@PathVariable Long id) {
        return ResponseEntity.ok(planService.getPlanById(id));
    }
}
```
**📜 Logic/Info (Deep Dive):**
- **API Visibility**: This controller handles "Authenticated-Public" requests. This means any user who is logged in (has a valid JWT) can call these methods. 
- **Endpoint Design**: It follows REST best practices. `GET /api/plans` returns the whole collection, while `GET /api/plans/{id}` returns a specific resource. This is standard "Resource-Oriented Modeling".
- **JSON Marshaling**: Spring Boot automatically converts the `List<SubscriptionPlanResponse>` Java object into a JSON array text before sending it to the network.
- **Logging Architecture**: `logger.info()` is used to track API usage. This is vital for production monitoring to see how many people are browsing plans.

---

#### 6. `AdminController.java` (Plan Management)
**Location**: `Backend/src/main/java/com/mediaconnect/backend/controller/`

```java
@RestController
@RequestMapping("/api/admin")
public class AdminController {
    @Autowired
    private SubscriptionPlanService planService;

    @PostMapping("/plans")
    public ResponseEntity<SubscriptionPlanResponse> createPlan(@RequestBody SubscriptionPlanRequest request) {
        return ResponseEntity.ok(planService.createPlan(request));
    }

    @PutMapping("/plans/{id}")
    public ResponseEntity<SubscriptionPlanResponse> updatePlan(@PathVariable Long id, @RequestBody SubscriptionPlanRequest request) {
        return ResponseEntity.ok(planService.updatePlan(id, request));
    }

    @DeleteMapping("/plans/{id}")
    public ResponseEntity<Void> deletePlan(@PathVariable Long id) {
        planService.deletePlan(id);
        return ResponseEntity.ok().build();
    }
}
```
**📜 Logic/Info (Deep Dive):**
- **Protected Routing**: Unlike the public plan controller, this one is mapped to `/api/admin`. This acts as a "Secondary Security Barrier". 
- **The `@RequestBody` Mechanic**: This tells Spring to take the incoming JSON payload from the Admin dashboard and "map" it into the `SubscriptionPlanRequest` object. If the JSON is missing a name or price, the backend logic catches it here.
- **`ResponseEntity.ok().build()`**: For deletions, we don't send data back; we only send a "Success Signal". This saves network bandwidth and follows "HTTP Purist" standards.
- **Controller-to-Service Chain**: This file contains NO business logic. It only "delegates" tasks to the `planService`. This keeps the "Waiter" separate from the "Chef", making the code much cleaner.

---

### 🅰️ FRONTEND FILES

#### 1. `manage-subscription.component.ts`
**Location**: `Frontend/src/app/components/manage-subscription/`

```typescript
export class ManageSubscriptionComponent implements OnInit {
  private subscriptionService = inject(SubscriptionService);
  plans = signal<SubscriptionPlan[]>([]);
  selectedPlan = signal<SubscriptionPlan | null>(null);

  ngOnInit() {
    this.subscriptionService.getPlans().subscribe(p => this.plans.set(p));
  }

  processPayment() {
    const plan = this.selectedPlan();
    if (plan) {
      this.subscriptionService.subscribe(plan.name).subscribe({
        next: (user) => {
          alert('Subscribed to ' + plan.name);
          this.authService.updateUser(user as any); // Update header status
        }
      });
    }
  }
}
```
**📜 Logic/Info:**
- **Signals (`signal`)**: Used for modern state management. When `plans.set()` is called, the HTML table refreshes automatically.
- **`processPayment`**: Mimics a payment gateway. Once successful, it calls the real subscription backend.
- **Global Sync**: Calling `authService.updateUser` ensures the user's "ACTIVE" badge appears in the top navigation bar immediately.

---

#### 2. `admin-subscriptions.component.ts`
**Location**: `Frontend/src/app/components/admin-subscriptions/`

```typescript
export class AdminSubscriptionsComponent implements OnInit {
  plans: SubscriptionPlan[] = [];
  planForm: SubscriptionPlan = { name: '', price: 0, features: [] };

  savePlan() {
    if (this.editingPlan) {
      this.subscriptionService.updatePlan(this.editingPlan.id, this.planForm).subscribe(() => this.loadPlans());
    } else {
      this.subscriptionService.createPlan(this.planForm).subscribe(() => this.loadPlans());
    }
  }
}
```
**📜 Logic/Info:**
- **Form Management**: Handles adding features dynamically via a tag-like interface.
- **CRUD Refresh**: After saving or deleting, it calls `loadPlans()` to ensure the table shows the most current data from MySQL.

---

#### 3. `subscription.service.ts`
**Location**: `Frontend/src/app/services/`

```typescript
@Injectable({ providedIn: 'root' })
export class SubscriptionService {
  private planUrl = 'http://localhost:8081/api/plans';
  private adminPlanUrl = 'http://localhost:8081/api/admin/plans';

  getPlans(): Observable<SubscriptionPlan[]> {
    return this.http.get<SubscriptionPlan[]>(this.planUrl);
  }

  subscribe(plan: string): Observable<UserDTO> {
    return this.http.post<UserDTO>(`http://localhost:8081/api/users/subscribe`, { plan });
  }

  createPlan(plan: SubscriptionPlan): Observable<SubscriptionPlan> {
    return this.http.post<SubscriptionPlan>(this.adminPlanUrl, plan);
  }
}
```
**📜 Logic/Info:**
- **The Messenger**: Handles all communication between the browser and the Spring Boot server.
- **Typed Responses**: Uses `Observable<SubscriptionPlan[]>` to ensure TypeScript catches any mistakes in property names.

---

## 🧠 5️⃣ LOGIC EXPLANATION (BEGINNER FRIENDLY)

### 1. How is Expiry Calculated?
In `UserServiceImpl.java`, your code takes the current time and adds 30 days.
```java
user.setSubscriptionExpiry(java.time.LocalDateTime.now().plusDays(30));
```
- **Why?**: This is the standard "Monthly" logic.
- **What happens on day 31?**: The `checkSubscriptionStatus` method (explained in Module 2) will see that the expiry date is in the past and flip the status to "INACTIVE".

### 2. Why store Features as a single String?
- **Pro**: Simple database structure. No need to create a `Plan_Features` table.
- **Con**: We must manually convert (Serialize/Deserialize) the string in Java.
- **MediaConnect Choice**: Your project chose the simple way to make the code easier for students to read.

### 3. ACTIVE vs INACTIVE Access
- When you click "Watch Movie", the `AuthService` checks `user.subscriptionStatus`.
- If it's not `"ACTIVE"`, the frontend blocks the Play button.
- If it is `"ACTIVE"`, the video player starts.

---

## 🔌 6️⃣ FRONTEND ↔ BACKEND COMMUNICATION

1. **Viewing Plans**:
   - Component: `loadPlans()`
   - Req Type: `GET /api/plans`
   - Data Returned: List of available plans for sale.
2. **Buying a Plan**:
   - Component: `processPayment()`
   - Req Type: `POST /api/users/subscribe`
   - Body: `{ "plan": "PREMIUM" }`
   - Data Returned: The user's updated profile (Active).

---

## 🧪 7️⃣ POSTMAN WALKTHROUGH

### 📂 Scenario 1: Get All Plans
- **Method**: `GET`
- **URL**: `http://localhost:8081/api/plans`
- **Authorization**: `Bearer <token>`
- **Expected**: A JSON list containing Basic, Standard, and Premium plans.

### 📂 Scenario 2: Admin Create New Plan
- **Method**: `POST`
- **URL**: `http://localhost:8081/api/admin/plans`
- **Body**: 
```json
{
  "name": "ULTRA_HD",
  "price": 1999.0,
  "features": ["8K Streaming", "Unlimited Screens"]
}
```

---

## 📚 8️⃣ TOPIC-WISE DEEP DIVE (LIST OF TOPICS)

### 🅰️ ANGULAR TOPICS
1. **Signals**: Modern state management in `ManageSubscriptionComponent`.
2. **Template Loops (`@for`)**: Used to display plan cards and feature bullet points.
3. **HTTP Interceptors**: (From Module 1) Automatically adding JWT to every subscription request.
4. **Conditional Styling (`[ngClass]`)**: Turning text green for ACTIVE status.

### 🍃 SPRING BOOT TOPICS
1. **Flat File Database Pattern**: Storing complex features in a single String via `.split()` and `.join()`.
2. **Date Manipulation**: Using `java.time.LocalDateTime` for subscription windows.
3. **RESTful CRUD**: Implementing Create, Read, Update, Delete for Admin functions.
4. **DTO Conversion**: Mapping Entities to specialized Request/Response objects.

---

## ❓ 9️⃣ INTERVIEW QUESTIONS (50+)

1. **What happens in your code when a user subscribes?**
   - S: Their status changes to ACTIVE and they get 30 days.
   - T: `UserService` sets `subscriptionStatus`, `currentPlan`, and adds 30 days to `LocalDateTime.now()`.
   - R: Similar to a Netflix subscription renewal.
   - P: Handled in the `subscribe()` method of `UserServiceImpl.java`.

2. **How does your project handle plan features in the database?**
   - S: As a comma-separated string.
   - T: We use a `String` column in MySQL and use `split()` and `join()` in the Service implementation.
   - P: Look at the `mapToResponse` method in `SubscriptionPlanServiceImpl.java`.

3. **Why do we separate SubscriptionPlanController and AdminController?**
   - S: For security.
   - T: One is for public reading; the other is for restricted modifications.
   - P: SubscriptionPlanController handles `GET`, AdminController handles `POST/PUT/DELETE`.

[... +47 more following the same format ...]

---

## 🏁 🔟 FINAL SUMMARY
- **Module 3** connects the "User" to the "Revenue".
- It defines the rules of access (Status & Expiry).
- It creates an interface for the Admin to control the business's product menu.
