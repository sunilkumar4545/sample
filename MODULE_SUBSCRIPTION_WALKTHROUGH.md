# MODULE: Subscription & Plans - Beginner Walkthrough

This guide explains how the Subscription and Plans system works in the MediaConnect application. It covers everything from viewing plans to making a payment and downloading an invoice.

---

## 📂 1. List of All Impacted Files

Here are all the files involved in this module and **why** they exist.

### 🟢 Frontend (Angular)
| File Name | Location | Why it exists? |
|-----------|----------|----------------|
| **subscription.service.ts** | `src/app/services/` | Contains methods to talk to the Backend (fetch plans, subscribe). |
| **manage-subscription.component.ts** | `src/app/components/` | The page where users view plans and pay. |
| **manage-subscription.component.html** | `src/app/components/` | The HTML design for the plans page. |
| **movie-detail.component.ts** | `src/app/components/` | Checks if the user is allowed to watch a movie. |
| **profile.component.ts** | `src/app/components/` | Shows subscription status and downloads the invoice. |
| **user.service.ts** | `src/app/services/` | Helper service to fetch user details and download files. |

### 🔵 Backend (Spring Boot)
| File Name | Location | Why it exists? |
|-----------|----------|----------------|
| **SubscriptionPlan.java** | `entity` | Defines what a "Plan" looks like in the database (ID, name, price). |
| **User.java** | `entity` | Defines the "User" and stores their subscription status. |
| **SubscriptionPlanRepository.java** | `repository` | Performs database operations (Save, Find, Delete) for plans. |
| **SubscriptionPlanService.java** | `service` | Interface defining the business logic methods. |
| **SubscriptionPlanServiceImpl.java** | `service/impl` | Actual logic implementation (fetching plans without streams). |
| **SubscriptionPlanController.java** | `controller` | Handles HTTP requests for Plans (GET /api/plans). |
| **UserController.java** | `controller` | Handles User-specific actions like "Subscribe" and "Download Invoice". |

---

## 📖 2. User Side – Viewing Plans

When a user visits the "Plans" page, the frontend asks the backend for a list of available plans.

### 1️⃣ Frontend: Asking for Plans
**File:** `subscription.service.ts`
```typescript
// This method sends a GET request to the backend
getPlans(): Observable<SubscriptionPlan[]> {
  // Call the backend API at localhost:8081/api/plans
  return this.http.get<SubscriptionPlan[]>('http://localhost:8081/api/plans');
}
```

### 2️⃣ Backend: Fetching from Database
**File:** `SubscriptionPlanController.java`
```java
@GetMapping
public ResponseEntity<List<SubscriptionPlanResponse>> getAllPlans() {
    // 1. Ask the service for all plans
    List<SubscriptionPlanResponse> plans = planService.getAllPlans();
    
    // 2. Return the list with 200 OK status
    return ResponseEntity.ok(plans);
}
```

**File:** `SubscriptionPlanServiceImpl.java` (Logic)
```java
@Override
public List<SubscriptionPlanResponse> getAllPlans() {
    // 1. Get raw data from database
    List<SubscriptionPlan> plans = repository.findAll();
    
    // 2. Create an empty list for the response
    List<SubscriptionPlanResponse> responseList = new ArrayList<>();

    // 3. Loop through each plan (No Streams!)
    for (SubscriptionPlan plan : plans) {
        // Convert Entity -> Response Object
        responseList.add(mapToResponse(plan));
    }
    
    return responseList;
}
```

### 📝 Logic Explanation
1.  **Frontend** sends a request: `GET /api/plans`.
2.  **Controller** receives it and calls the **Service**.
3.  **Service** uses the **Repository** to get all rows from the database table.
4.  **Service** loops through the data, converting it to a safe format (DTO) to send back.
5.  **Frontend** receives the JSON list and displays it as cards.

---

## 🔒 3. Movie Access Rule (Blocking Non-Subscribers)

This is the critical security check. If a user clicks a movie but hasn't paid, we stop them.

### 🚫 The Block Logic
**File:** `movie-detail.component.ts`

```typescript
loadMovieDetails(movieId: number) {
    // 1. First, check who the user is
    this.userService.getMe().subscribe(userDTO => {
        
        // 2. Check the status field
        if (userDTO.subscriptionStatus !== 'ACTIVE') {
            
            // 🛑 STOP! Redirect to Plans page
            alert("You must subscribe to watch movies!");
            this.router.navigate(['/plans']);
            return;
        }

        // ✅ ALLOW! User is active, so fetch the movie
        this.fetchMovieContent(movieId);
    });
}
```

### 📝 Logic Explanation
1.  The user tries to open a movie (e.g., Movie ID 101).
2.  The component **immediately** calls `userService.getMe()` to get the latest user details.
3.  It checks `userDTO.subscriptionStatus`.
    *   If **INACTIVE**: The user is kicked to the `/plans` page.
    *   If **ACTIVE**: The code proceeds to load the video player.

---

## 💳 4. Payment Flow (Mock Payment)

Since we don't have a real bank connection, we "mock" (simulate) the payment.

### 1️⃣ Frontend: Sending the Payment
**File:** `manage-subscription.component.ts`
```typescript
processPayment() {
    // 1. Get the selected plan name (e.g., "Premium")
    const planName = this.selectedPlan.name;

    // 2. Send it to the backend
    this.subscriptionService.subscribe(planName).subscribe({
        next: (updatedUser) => {
            // 3. Payment Success! Show message
            this.successMessage = "Payment Successful! You are now subscribed.";
            
            // 4. Update the stored user info so the app knows they are active
            this.authService.updateUser(updatedUser);
        },
        error: (err) => alert("Payment failed")
    });
}
```

### 2️⃣ Backend: Processing the Subscription
**File:** `UserController.java`

```java
@PostMapping("/subscribe")
public ResponseEntity<?> subscribe(@RequestBody Map<String, String> payload) {
    // 1. Get the current logged-in user's email
    String email = SecurityContextHolder.getContext().getAuthentication().getName();
    
    // 2. Get the plan name from the request
    String planName = payload.get("plan");

    // 3. Call the service to update logic
    UserDTO updatedUser = userService.subscribe(email, planName);

    return ResponseEntity.ok(updatedUser);
}
```

**File:** `UserServiceImpl.java` (Logic implementation)
```java
public UserDTO subscribe(String email, String planName) {
    // 1. Find the user in the database
    User user = userRepository.findByEmail(email);

    // 2. Set Status to ACTIVE
    user.setSubscriptionStatus("ACTIVE");
    user.setCurrentPlan(planName);

    // 3. Calculate Expiry Date (Today + 30 Days)
    LocalDateTime today = LocalDateTime.now();
    LocalDateTime expiryDate = today.plusDays(30);
    user.setSubscriptionExpiry(expiryDate);

    // 4. Save updates to Database
    User savedUser = userRepository.save(user);

    // 5. Return updated info
    return mapToDTO(savedUser);
}
```

### 📝 Logic Explanation
1.  **User** clicks "Pay".
2.  **Frontend** sends the plan name to `POST /api/users/subscribe`.
3.  **Backend** finds the user and:
    *   Changes status to `ACTIVE`.
    *   Sets `currentPlan` to the selected plan.
    *   Calculates `expiryDate` = Now + 30 Days.
4.  **Database** updates the user row.
5.  **Frontend** receives the new user object and updates the UI immediately.

---

## 👤 5. Profile & Invoice Download (Text File)

The user can view their subscription logic and download a simple text invoice.

### 1️⃣ Invoice Generation Logic
**File:** `UserController.java`

```java
@GetMapping("/me/download")
public ResponseEntity<byte[]> downloadProfile() {
    // 1. Get current user
    String email = SecurityContextHolder.getContext().getAuthentication().getName();
    UserDTO user = userService.getUserByEmail(email);

    // 2. Create the file content (String)
    String fileContent = "--- MEDIA CONNECT INVOICE ---\n" +
            "User Name: " + user.getFullName() + "\n" +
            "Email: " + user.getEmail() + "\n" +
            "Plan: " + user.getCurrentPlan() + "\n" +
            "Status: " + user.getSubscriptionStatus() + "\n" +
            "Expires On: " + user.getSubscriptionExpiry() + "\n";

    // 3. Convert String to Bytes
    byte[] data = fileContent.getBytes();

    // 4. Return as a downloadable file
    return ResponseEntity.ok()
            // This header tells the browser to download a file named 'profile.txt'
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=profile.txt")
            .contentType(MediaType.TEXT_PLAIN) // Data type is Plain Text
            .body(data);
}
```

### 2️⃣ Frontend: Triggering Download
**File:** `profile.component.ts`

```typescript
downloadInvoice() {
    this.userService.downloadProfile().subscribe((fileData: Blob) => {
        // 1. Create a virtual URL for the file data
        const url = window.URL.createObjectURL(fileData);
        
        // 2. Create a hidden <a> tag
        const link = document.createElement('a');
        link.href = url;
        link.download = 'invoice.txt'; // The name of the downloaded file
        
        // 3. "Click" the link automatically
        link.click();
        
        // 4. Clean up
        window.URL.revokeObjectURL(url);
    });
}
```

### 📝 Logic Explanation
1.  Backend creates a simple **Java String** with all the details.
2.  It converts that String into **bytes**.
3.  It adds a special header (`CONTENT_DISPOSITION`) that tells the browser "Do not open this, Save it".
4.  Frontend receives the **Blob** (Binary Large Object).
5.  Frontend hacks a hidden link click to force the browser to start the download dialog.

---

## 🔄 6. Complete Flow Summary

Here is the step-by-step lifecycle of a user interacting with this module:

1.  **Discovery**: User clicks a Movie Card 🎬.
2.  **Check**: App checks `subscriptionStatus`. It is `INACTIVE` ❌.
3.  **Redirect**: User is sent to `/plans` page 🔄.
4.  **Selection**: User sees plans (fetched from DB) and selects "Premium" ✅.
5.  **Payment**: User clicks "Pay". Frontend calls Backend 📡.
6.  **Update**: Backend sets `ACTIVE`, adds +30 days, saves to DB 💾.
7.  **Access**: User is redirected back. Now `subscriptionStatus` is `ACTIVE` ✅.
8.  **Action**: User can now watch the movie 🍿.
9.  **Record**: User goes to Profile and clicks "Download Invoice".
10. **Delivery**: A `.txt` file with payment details is downloaded to their PC 📄.
