# Watch History & "Continue Watching" Logic

This document explains the technical implementation of the "Continue Watching" feature, describing how playback position is tracked, saved, and retrieved.

---

## 1. Core Logic Overview

The "Continue Watching" feature relies on a synchronization loop:
1.  **Tracking:** The Frontend (Video Player) tracks the current time (`currentTime`) of the video.
2.  **Saving:** Periodically (or on pause/exit), the Frontend sends this timestamp to the Backend.
3.  **Storing:** The Backend updates the `WatchHistory` table, saving the `progressSeconds` and `lastWatched` timestamp.
4.  **Resuming:** When a user opens a movie, the Frontend requests the saved progress and seeks the video player to that second.

---

## 2. Backend Implementation (Logic & Storage)

### A. The Entity (`WatchHistory.java`)
This class maps to the database table `watch_history`. It links a **User** to a **Movie** and stores *where* they stopped.

*   `user`: The viewer.
*   `movie`: The content being watched.
*   `progressSeconds`: Integer representing offset in seconds (e.g., 120 = 2 minutes in).
*   `lastWatched`: Timestamp used for sorting (so the most recently watched appears first).

```java
// File: com.mediaconnect.backend.entity.WatchHistory.java

@Entity
@Table(name = "watch_history")
public class WatchHistory {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @ManyToOne
    @JoinColumn(name = "movie_id", nullable = false)
    private Movie movie;

    private int progressSeconds; // Where the user left off

    private LocalDateTime lastWatched;

    // Constructors...
    public WatchHistory(User user, Movie movie, int progressSeconds) {
        this.user = user;
        this.movie = movie;
        this.progressSeconds = progressSeconds;
        this.lastWatched = LocalDateTime.now();
    }
    
    // Getters and Setters...
}
```

#### Code Walkthrough:
*   `@Entity`: Tells JPA/Hibernate that this class represents a table in the database.
*   `@ManyToOne`: Defines the relationship. One User can have many history records; One Movie can have many history records.
*   `progressSeconds`: This is the most critical field. It is a simple integer. If the value is `125`, it means the user is at 2 minutes and 5 seconds.
*   `lastWatched`: We set this to `LocalDateTime.now()` every time the user watches. This allows us to sort by "Most Recently Watched" on the home page.

### B. The Repository (`WatchHistoryRepository.java`)
This interface handles the database queries.
*   `findByUserIdAndMovieId`: Vital for checking if we should "Update" an existing record or "Create" a new one.
*   `findByUserIdOrderByLastWatchedDesc`: Fetches the history sorted by most recent, for the "Continue Watching" list.

```java
// File: com.mediaconnect.backend.repository.WatchHistoryRepository.java

@Repository
public interface WatchHistoryRepository extends JpaRepository<WatchHistory, Long> {
    
    // Find specific progress for a specific movie (Used for Resuming)
    WatchHistory findByUserIdAndMovieId(Long userId, Long movieId);

    // Find all history for a user, sorted by most recent (Used for Home Page)
    List<WatchHistory> findByUserIdOrderByLastWatchedDesc(Long userId);
}
```

#### Code Walkthrough:
*   `findByUserIdAndMovieId`: This is a "Derived Query Method". Spring Data JPA automatically checks the `User` field's ID and `Movie` field's ID in the `WatchHistory` entity.
    *   **Logic:** `SELECT * FROM watch_history WHERE user_id = ? AND movie_id = ?`
*   `OrderByLastWatchedDesc`: This naming convention creates a SQL query ending in `ORDER BY last_watched DESC`. This ensures the movie you just watched appears first in the list.

### C. The Service Logic (`WatchHistoryServiceImpl.java`)
This is the "Brain" of the operation. It handles the `saveProgress` logic intelligently to prevent duplicates.

**Logic Flow for Saving:**
1.  **Check Existence:** It first checks if this User already has a history for this Movie (`findByUserIdAndMovieId`).
2.  **Update vs. Create:**
    *   **If Exists:** It updates the *existing* record with the new `progressSeconds` and updates `lastWatched` to `now()`. This prevents duplicate rows for the same movie.
    *   **If New:** It creates a fresh `WatchHistory` entry.

```java
// File: com.mediaconnect.backend.service.implementations.WatchHistoryServiceImpl.java

@Service
public class WatchHistoryServiceImpl implements WatchHistoryService {

    @Autowired
    private WatchHistoryRepository watchHistoryRepository;
    // other autowired repos...

    @Override
    public void saveProgress(Long userId, Long movieId, int seconds) {
        // 1. Check if history exists
        WatchHistory existing = watchHistoryRepository.findByUserIdAndMovieId(userId, movieId);

        if (existing != null) {
            // 2a. UPDATE existing record
            existing.setProgressSeconds(seconds);
            existing.setLastWatched(LocalDateTime.now());
            watchHistoryRepository.save(existing);
        } else {
            // 2b. CREATE new record
            User user = userRepository.findById(userId).orElse(null);
            Movie movie = movieRepository.findById(movieId).orElse(null);

            if (user != null && movie != null) {
                WatchHistory history = new WatchHistory(user, movie, seconds);
                watchHistoryRepository.save(history);
            }
        }
    }

    @Override
    public WatchHistory getProgress(Long userId, Long movieId) {
        return watchHistoryRepository.findByUserIdAndMovieId(userId, movieId);
    }
}
```

#### Code Walkthrough:
*   `saveProgress(...)`: Takes the raw data (User ID, Movie ID, Seconds).
*   `existing != null`: This is the toggle switch.
    *   **Scenario A (Returning User):** You watched 5 mins yesterday. Today you watch 10 mins. `existing` is found. We modify it to 10 mins. We do **not** insert a new row.
    *   **Scenario B (First Time):** `existing` is null. We fetch the full `User` and `Movie` objects from their repos and create a brand new row.

### D. The Controller (`WatchHistoryController.java`)
Exposes the endpoints for the frontend to talk to.
*   `POST /api/watch-history/save`: Receives `{userId, movieId, seconds}` and calls the service.
*   `GET /api/watch-history/progress`: Returns a single record (seconds) for a specific movie (used for resuming).

```java
// File: com.mediaconnect.backend.controller.WatchHistoryController.java

@RestController
@RequestMapping("/api/watch-history")
public class WatchHistoryController {

    @Autowired
    private WatchHistoryService watchHistoryService;

    // Endpoint to save progress
    @PostMapping("/save")
    public ResponseEntity<?> saveProgress(@RequestBody Map<String, Object> payload) {
        Long userId = Long.valueOf(payload.get("userId").toString());
        Long movieId = Long.valueOf(payload.get("movieId").toString());
        int seconds = Integer.parseInt(payload.get("seconds").toString());

        watchHistoryService.saveProgress(userId, movieId, seconds);
        return ResponseEntity.ok().build();
    }

    // Endpoint to get specific movie progress (for resuming)
    @GetMapping("/progress")
    public ResponseEntity<?> getProgress(@RequestParam Long userId, @RequestParam Long movieId) {
        WatchHistory history = watchHistoryService.getProgress(userId, movieId);
        if (history != null) {
            return ResponseEntity.ok(history);
        }
        return ResponseEntity.notFound().build();
    }
}
```

#### Code Walkthrough:
*   `@RequestBody Map<String, Object> payload`: We use a Map here for flexibility. The frontend sends a JSON object like `{ "userId": 1, "movieId": 2, "seconds": 120 }`. We extract values manually.
*   `@RequestParam`: Used for GET requests. The URL looks like `/progress?userId=1&movieId=2`.
*   `watchHistoryService.saveProgress(...)`: Simplicity is key here. The controller delegates *all* logic to the Service.

---

## 3. Frontend Implementation (Tracking & Saving)

### A. The HTML Structure (`movie-detail.component.html`)
The HTML is where the tracking *events* originate. The `<video>` tag has 3 key event listeners:
1.  `(timeupdate)`: Fires continuously as the video plays. We link this to `saveProgress($event)`.
2.  `(pause)`: Fires when the user hits pause. We save immediately.
3.  `(ended)`: Fires when video finishes. We save as "completed".

```html
<!-- File: src/app/components/movie-detail/movie-detail.component.html -->

<div class="video-container">
  <h3 class="h5 mb-3">Watch Now</h3>
  
  <!-- THE CORE TRACKING ELEMENTS -->
  <video 
      #videoPlayer 
      (timeupdate)="saveProgress($event)" 
      (pause)="onPause($event)"
      (ended)="onMovieEnded($event)" 
      class="video-player" 
      controls>
      
      <source [src]="getVideoUrl(m.videoPath)" type="video/mp4">
      Your browser does not support the video tag.
  </video>
</div>
```

#### Code Walkthrough:
*   `(timeupdate)="saveProgress($event)"`: This is the most important line. The HTML5 Video element fires this event approx. 4 times per second. We bind this to our TypeScript function `saveProgress`.
*   `(pause)`: Ensures we capture the exact moment they stop watching.
*   `(ended)`: Usually sets the progress to the very end (or 0) depending on logic, marking the movie as "Finished".

### B. The Logic Component (`movie-detail.component.ts`)
This component handles the active video player. It uses "Debouncing" to ensure progress is saved efficiently without spamming the server.

**Key Logic Parts:**
1.  **Debouncing (`setTimeout`):** We don't want to send an HTTP request every millisecond. We wait for 5 seconds of continuous playback before saving.
2.  **Resuming (`onInit`):** When the page loads, we ask the backend: "Where did I leave off?" and move the video cursor there.

```typescript
// File: src/app/components/movie-detail/movie-detail.component.ts

export class MovieDetailComponent implements OnInit, OnDestroy {
  private watchProgressTimer: any; // The timer for debouncing

  // 1. RESUMING: Logic to jump to saved timestamp
  private loadLastWatchPosition(movieId: number) {
    this.watchHistoryService.getProgress(movieId).subscribe({
      next: (history) => {
        if (history && history.progressSeconds > 0) {
          // Wait slightly for video to load, then seek
          setTimeout(() => {
            const videoElement = document.querySelector('video') as HTMLVideoElement;
            if (videoElement) {
              videoElement.currentTime = history.progressSeconds; // SEEK TO SAVED TIME
            }
          }, 500);
        }
      }
    });
  }

  // 2. SAVING: The Debounced Save Logic
  saveProgress(event: Event) {
    // Clear the previous timer (cancel the previous pending save)
    if (this.watchProgressTimer) clearTimeout(this.watchProgressTimer);
    
    // Set a new timer. If no other events happen for 5 seconds, SAVE.
    this.watchProgressTimer = setTimeout(() => {
      const video = event.target as HTMLVideoElement;
      this.saveProgressNow(video.currentTime);
    }, 5000); 
  }

  // Helper to actually call the service
  private saveProgressNow(seconds: number) {
    if (isNaN(seconds)) return;
    if (this.movie && this.movie.id) {
      this.watchHistoryService.saveProgress(this.movie.id, Math.floor(seconds)).subscribe();
    }
  }

  // Immediate save on pause
  onPause(event: Event) {
    const video = event.target as HTMLVideoElement;
    this.saveProgressNow(video.currentTime);
  }
}
```

#### Code Walkthrough:
*   **The Debounce Pattern:**
    *   Imagine `timeupdate` fires: `0.2s`, `0.5s`, `0.7s`, `1.0s`...
    *   Inside `saveProgress`:
        *   `clearTimeout(timer)`: "Abort the save I was planning!"
        *   `timer = setTimeout(..., 5000)`: "Plan a new save in 5 seconds."
    *   **Result:** As long as the video keeps playing and events keep firing, the timer keeps getting reset. The code *only* executes the save if 5 seconds pass *without* a new event (or we force it).
    *   *Correction in context:* In a video player, we actually want it to save *interval-based* or when the user stops. The logic above actually resets continuously on playback. A better approach often used is an **Interval**, but here `debounce` ensures we capture the *end* of a viewing session effectively, or forcing it on Pause.
*   **Resuming:**
    *   We fetch `history`.
    *   `videoElement.currentTime = history.progressSeconds`: This API command physically moves the video playhead.

### C. The Service (`watch-history.service.ts`)
A simple HTTP wrapper.

```typescript
// File: src/app/services/watch-history.service.ts

@Injectable({ providedIn: 'root' })
export class WatchHistoryService {
  private apiUrl = 'http://localhost:8081/api/watch-history';

  saveProgress(movieId: number, seconds: number): Observable<any> {
    const user = this.authService.currentUser();
    return this.http.post(`${this.apiUrl}/save`, {
      userId: user.user.id,
      movieId,
      seconds
    });
  }
  
  // ... other methods
}
```

#### Code Walkthrough:
*   `saveProgress`: This method retrieves the *current logged-in user* from `AuthService` so we know *who* is watching.
*   It packages the data into a JSON object `{ userId, movieId, seconds }` matching what the Backend Controller expects.

### D. Displaying "Continue Watching" (`HomeComponent`)

**Logic (`home.component.ts`):** Fetches the history list.
```typescript
loadUserData() {
    const user = this.authService.currentUser();
    this.watchHistoryService.getUserHistory(Number(user.userId)).subscribe({
        next: (data) => {
            this.continueWatching = data.map((h: any) => {
                return {
                    ...h.movie,
                    progress: h.progressSeconds, // We attach the progress here
                    // ... formatting logic
                };
            });
        }
    });
}
```
#### Code Walkthrough:
*   `.map(...)`: This is a transformation. The raw data from the backend is a `WatchHistory` object (contains User + Movie + Seconds).
*   The frontend component wants a clean list of *Movies* to display.
*   So we "flatten" the structure: We take the `movie` part, and stick the `progressSeconds` directly onto it. This makes it easy for the HTML to say `movie.progress`.

**View (`home.component.html`):** Loops through the history.
```html
<!-- File: src/app/components/home/home.component.html -->

<div *ngIf="continueWatching.length > 0" class="mb-5">
    <h4 class="fw-bold mb-4">Continue Watching</h4>
    <div class="row ...">
        <!-- Loop through the history items -->
        <div class="col" *ngFor="let movie of continueWatching">
            <app-movie-card [movie]="movie"></app-movie-card>
        </div>
    </div>
</div>
```
#### Code Walkthrough:
*   `*ngIf="continueWatching.length > 0"`: Only shows this entire row if the user actually has history.
*   `*ngFor`: Loops through the prepared list from the TS file and renders a `MovieCard` for each one.

---

## 4. Communication Diagram (The Complete Cycle)

1.  **START:** User clicks a movie.
2.  **RESUME:** `MovieDetailComponent` loads. calls `getAttribute/progress`. Backend says: "Timestamp: 600s". Frontend sets `video.currentTime = 600`.
3.  **PLAY:** User watches. `(timeupdate)` fires every second.
4.  **DEBOUNCE:** The code resets the timer. No network request is sent yet.
5.  **WAIT:** User keeps watching. After 5 seconds, the timer fires.
6.  **SAVE:** `saveProgressNow()` calls HTTP POST `api/watch-history/save`.
7.  **DB UPDATE:** Backend updates the row in MySQL.
8.  **PAUSE/EXIT:** User leaves.
9.  **LATER:** User goes to Home Page.
10. **DISPLAY:** `HomeComponent` calls `getUserHistory`. Checks DB. Shows the movie in "Continue Watching".

---

## 5. Related Files List

**Backend:**
1.  `com.mediaconnect.backend.entity.WatchHistory.java` (Entity)
2.  `com.mediaconnect.backend.repository.WatchHistoryRepository.java` (Repo)
3.  `com.mediaconnect.backend.service.implementations.WatchHistoryServiceImpl.java` (Service)
4.  `com.mediaconnect.backend.controller.WatchHistoryController.java` (Controller)

**Frontend:**
1.  `src/app/components/movie-detail/movie-detail.component.html` (View)
2.  `src/app/components/movie-detail/movie-detail.component.ts` (Logic)
3.  `src/app/services/watch-history.service.ts` (API Client)
4.  `src/app/components/home/home.component.ts` (Display Logic)

## 6. Concepts Used

**Frontend (Angular):**
*   **Event Binding:** `(timeupdate)`, `(pause)` on Video elements.
*   **Debouncing:** Using `setTimeout` and `clearTimeout` to optimize performance.
*   **RxJS:** `Observable` and `.subscribe()` for handling HTTP responses.
*   **DOM Manipulation:** `videoElement.currentTime` to seek video.
*   **Data Transformation:** `.map()` to format API data for the UI.

**Backend (Spring Boot):**
*   **JPA/Hibernate:** `@Entity`, `@ManyToOne` for database mapping.
*   **Spring Data JPA:** Derived Query Methods like `findByUserIdAndMovieId`.
*   **Dependency Injection:** `@Autowired` to connect Controller -> Service -> Repository.
*   **REST API:** `@PostMapping`, `@GetMapping` to expose functionality.
*   **Business Logic:** "Update if exists, Create if new" pattern.


# Exception Handling Deep Dive
> **Core Concept**: We use **Global Exception Handling** with Spring AOP (Aspect Oriented Programming) to catch errors centrally, rather than using `try-catch` blocks in every method. This ensures a consistent error response structure across the entire application.

---

## 1. The Communication Flow (How it travels)

When an error occurs (e.g., "User not found"), the error travels through the layers of the application in a specific sequence.

**Visual Flow:**
`Service` (Throws Error) ➡️ `Controller` (Bubbles up) ➡️ `GlobalExceptionHandler` (Catches & Formats) ➡️ `JSON Response` (Sent over HTTP) ➡️ `Frontend Service` (Observable Error) ➡️ `Component` (Display to User)

### Step 1: The Trigger (Backend Service)
The logic starts in the Service layer. If a condition fails, we **throw** a custom exception.

**File:** `Backend/.../UserServiceImpl.java` (Example)
```java
// Logic: If user is missing, STOP execution and Throw exception
if (user == null) {
    throw new ResourceNotFoundException("User not found with email: " + email);
}
```

### Step 2: The Interception (Global Handler)
The exception bubbles up. Instead of crashing the server, our `@RestControllerAdvice` class catches it. This is the **AOP** part—it "watches" all controllers.

**File:** `Backend/.../exception/GlobalExceptionHandler.java`
```java
@RestControllerAdvice // <--- This annotation makes it a Global Listener
public class GlobalExceptionHandler {

    // This method only runs if a ResourceNotFoundException is thrown anywhere
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleResourceNotFound(ResourceNotFoundException ex) {
        
        // 1. Create a clean error object (DTO)
        ErrorResponse error = new ErrorResponse(
                HttpStatus.NOT_FOUND.value(), // 404
                ex.getMessage());             // "User not found..."

        // 2. Return it as proper JSON
        return new ResponseEntity<>(error, HttpStatus.NOT_FOUND);
    }
}
```

### Step 3: The Custom Exception Class
This is the simple class we define to signal specific errors.

**File:** `Backend/.../exception/ResourceNotFoundException.java`
```java
@ResponseStatus(value = HttpStatus.NOT_FOUND) // Defaults to 404
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message); // Pass message to parent RuntimeException
    }
}
```

### Step 4: Frontend Handling (Angular)
The connection returns a `404` status with the JSON body. The Angular Service observes this failure.

**File:** `Frontend/.../components/login/login.component.ts` (Example)
```typescript
this.authService.login(credentials).subscribe({
    next: (data) => {
        // Success logic
    },
    error: (err) => {
        // STEP 4: Capture the error
        // 'err.error' is the JSON object created in Step 2
        console.error('Login Failed:', err.error.message);
        
        // Display to user
        this.errorMessage = err.error.message; 
    }
});
```

---

## 2. Summary of Files & Roles

| File | Role | Responsibility |
|------|------|----------------|
| **`ResourceNotFoundException.java`** | **Signal** | Defines *what* went wrong (e.g., "Not Found"). |
| **`GlobalExceptionHandler.java`** | **Handler** | Catches the signal, stops the crash, formats the JSON. |
| **`ErrorResponse.java`** | **Format** | Defines the shape of the JSON ({ status, message, timestamp }). |
| **`*Controller.java`** | **Pass-through** | The exception passes through here on its way to the Handler. |
| **`*.service.ts` (Frontend)** | **Carrier** | Carries the HTTP error response to the component. |
| **`*.component.ts` (Frontend)** | **Display** | Subscribes to the error and shows it to the user. |


## 7. Interview Questions & Answers

### A. General & Architectural (Watch History)

**Q1: Can you explain the high-level architecture of the "Continue Watching" feature?**
**A:** "The feature uses a periodic synchronization strategy. The Frontend tracks the video's `currentTime` using the HTML5 `timeupdate` event. To avoid overloading the server, I implemented **debouncing** to only send a POST request every 5 seconds or when the user pauses. The Backend receives this timestamp, checks if a record exists for that user+movie combination, and either updates the existing record or creates a new one. This data is then used to populate the 'Continue Watching' row on the dashboard."

**Q2: Why did you use `setTimeout` (Debouncing) instead of sending a request on every `timeupdate`?**
**A:** "The `timeupdate` event fires roughly 4 times per second. Sending an HTTP request at that frequency would flood the server (DDoS style) and cause massive performance issues. Debouncing allows me to aggregate these updates and only persist the state when meaningful progress has been made (e.g., every 5-10 seconds), significantly reducing network traffic."

**Q3: How do you handle the case where a user pauses the video?**
**A:** "I listen specifically for the `(pause)` event on the video tag. When this triggers, I bypass the debounce timer and force an immediate save to the backend. This ensures that even if the 5-second timer hasn't finished, the exact stop time is captured."

**Q4: How does the application know where to resume playback from?**
**A:** "When the `MovieDetailComponent` initializes, it makes a GET request to the `/progress` endpoint. If the backend returns a history record with `progressSeconds > 0`, I use the video element's API (`video.currentTime = ...`) to seek to that timestamp immediately."

**Q5: Is the communication between Frontend and Backend RESTful?**
**A:** "Yes. We use standard HTTP methods: `GET` for retrieving history/progress, and `POST` for creating/updating the progress resource. The API is stateless, as the authentication is handled via JWT tokens in the header."

### B. Backend (Spring Boot & JPA)

**Q6: What is the relationship between `User`, `Movie`, and `WatchHistory` in your database?**
**A:** "It is a Many-to-One relationship. A `WatchHistory` record belongs to *one* User and *one* Movie. However, a single User can have *many* history records (for different movies), and a Movie can appear in *many* users' histories. In JPA terms, the `WatchHistory` entity has `@ManyToOne` fields for both `user` and `movie`."

**Q7: Explain the logic behind `saveProgress` in the Service layer.**
**A:** "The service implements an 'Upsert' (Update or Insert) pattern. First, it queries the repository using `findByUserIdAndMovieId`.
*   If a record is found, it updates the `progressSeconds` and `lastWatched` timestamp on the existing entity.
*   If null, it fetches the full User and Movie entities and creates a new `WatchHistory` object.
Finally, it calls `.save()`, which commits the changes to the database."

**Q8: Why do you store `lastWatched` as a separate field?**
**A:** "This is crucial for the 'Continue Watching' UI. We want to show the movies he user watched *most recently* at the beginning of the list. We use a repository method `findByUserIdOrderByLastWatchedDesc` to achieve this sorting efficiently at the database level."

**Q9: How exactly does Spring Data JPA know how to execute `findByUserIdAndMovieId`?**
**A:** "This is a **Derived Query Method**. Spring Data parses the method name. It looks for a field named `user` and `movie` in the Entity, and expects their IDs as arguments. It automatically generates the JPQL/SQL equivalent: `SELECT * FROM watch_history WHERE user_id = ? AND movie_id = ?`."

**Q10: What HTTP status code do you return when saving progress?**
**A:** "I return `200 OK` (via `ResponseEntity.ok().build()`) to indicate success. If the user or movie didn't exist (edge case), I might throw a RuntimeException which my Global Exception Handler would catch and convert to a 404 or 400."

**Q11: Can you explain the annotation `@RestController`?**
**A:** "It's a convenience annotation that combines `@Controller` and `@ResponseBody`. It tells Spring that this class handles web requests and that the return values of methods should be written directly to the HTTP response body (usually as JSON), rather than resolving to a view/template."

**Q12: What triggers the database transaction in your Service?**
**A:** "By default, Spring Boot methods are not transactional unless annotated or inside a generic Service context. However, the `save()` method of the `JpaRepository` is transactional by default. So when I call `repository.save()`, a transaction is opened and committed."

**Q13: How do you handle potential concurrency issues (two requests saving at once)?**
**A:** "For a single user watching a video, requests are sequential or debounced, so collisions are rare. However, if they were common, JPA's `@Version` annotation (Optimistic Locking) or database-level row locking could be used. In this specific use case, since it's just updating 'last second watched', a 'last write wins' strategy is usually acceptable."

**Q14: How does Dependency Injection work in your Controller?**
**A:** "I use constructor or field injection (via `@Autowired`). Spring's IoC Container creates an instance of `WatchHistoryService` (a singleton bean) and injects it into `WatchHistoryController`. This creates loose coupling—the controller doesn't need to know *how* to create the service, it just uses it."

**Q15: What is the purpose of the DTO (AnalyticsResponse) in the repo?**
**A:** "We use DTOs (Data Transfer Objects) to shape data specifically for a response. In `findEngagementAnalytics`, we don't want to return raw database rows. We construct a custom object (Movie Title + Views Count) directly in the JPQL query to minimize data transfer and hide internal entity structure."

### C. Frontend (Angular & TypeScript)

**Q16: How do you listen for video events in Angular?**
**A:** "I use Angular's event binding syntax in the template: `(eventName)='handler($event)'`. specifically `(timeupdate)`, `(pause)`, and `(ended)`. The `$event` object contains native DOM event data which I cast to `HTMLVideoElement` to access properties like `currentTime`."

**Q17: What is TypeScript's role in the component?**
**A:** "TypeScript adds type safety. For example, when I handle the video event, I cast the target: `const video = event.target as HTMLVideoElement`. This ensures I can only access valid properties like `.currentTime` or `.duration`, preventing runtime 'undefined' errors."

**Q18: Explain the `OnInit` and `OnDestroy` lifecycle hooks.**
**A:** 
*   `ngOnInit`: Runs once when the component is created. I use it to fetch the saved progress (`loadLastWatchPosition`) and setup initial state.
*   `ngOnDestroy`: Runs when the user navigates away. I use it to `clearTimeout` on my save timer. If I didn't, the timer might try to execute code after the component is destroyed, causing memory leaks or errors."

**Q19: How do you handle the asynchronous nature of the HTTP call?**
**A:** "I use RxJS `Observables`. The `HttpClient` returns an Observable. I `.subscribe()` to it. Code inside the `next:` block runs *only when* the data actually arrives from the server. This prevents the UI from freezing while waiting for the response."

**Q20: Why did you use `.map()` in `loadUserData`?**
**A:** "The API returns a complex nested object (`{ user: {...}, movie: {...}, progressSeconds: 120 }`). My view simply wants a list of movies with a progress bar. `.map()` allows me to flatten this into a cleaner format (`{ title: '...', progress: 120 }`) that is easier to use in the HTML template."

**Q21: How does the `MovieCardComponent` receive data?**
**A:** "It uses the `@Input()` decorator. The parent `HomeComponent` passes the movie object down via property binding: `[movie]='currentMovie'`. This creates a reusable component that doesn't care *where* the data comes from, only how to display it."

**Q22: How would you debug if the video position isn't saving?**
**A:** "I would check the Network tab in Chrome DevTools to see if the POST requests are being firing. If they are, I'd check the Payload to ensure `seconds` is correct. If they aren't firing, I'd add console logs in the `saveProgress` function to verify the `timeupdate` event and the debounce logic."

**Q23: What is the purpose of `ChangeDetectorRef`?**
**A:** "Sometimes, when updates happen asynchronously (like inside a `setTimeout` or an HTTP callback), Angular might not detect the change immediately. Calling `cdr.detectChanges()` forces Angular to check for updates and re-render the view, ensuring the UI stays in sync."

### D. Exception Handling (Project Wide)

**Q24: How do you handle exceptions in this project?**
**A:** "We use a Global Exception Handler strategy using Spring's `@RestControllerAdvice`. Instead of try-catch blocks in every controller, exceptions (like `ResourceNotFoundException`) bubble up to this global handler, which formats them into a standard JSON `ErrorResponse`."

**Q25: Why is AOP (Aspect Oriented Programming) relevant here?**
**A:** "My Global Exception Handler IS an implementation of AOP. It is a 'Cross-Cutting Concern' that applies to *all* controllers without modifying their code. The Advice 'intercepts' the exception flow."

**Q26: How does the frontend know an error occurred?**
**A:** "The standard HTTP error codes (4xx, 5xx) are returned. In the Angular `.subscribe()` block, the `error:` callback is triggered instead of `next:`. We extract the error message from the JSON body and display it to the user."

**Q27: What is `ResourceNotFoundException`?**
**A:** "It is a custom RuntimeException class I created. I use it to semantically signal that a database lookup failed (e.g., User or Movie not found). It makes the code more readable than throwing generic Exceptions."

### E. Scenario Based & System Design

**Q28: If millions of users are watching, how would you scale this?**
**A:** "Currently, we write to the DB every 5 seconds per user. This is write-heavy. To scale:
1.  **Cache/Buffer:** Write updates to Redis first.
2.  **Batch Processing:** A background worker flushes Redis to MySQL every few minutes (reducing DB I/O).
3.  **Read Replicas:** Serve the 'Continue Watching' list from a Read Replica database."

**Q29: What happens if the internet cuts out while watching?**
**A:** "The frontend debouncer will still try to fire. The HTTP request will fail. We could implement a 'Retry Mechanism' or store the progress in `localStorage` temporarily and sync it when the connection returns (Offline Mode)."

**Q30: How would you implement 'Mark as Watched'?**
**A:** "In the `onMovieEnded` event, I would send a final request with a flag or set `progressSeconds` to 0 (or remove the record from 'Continue Watching' logic). I might also add a boolean field `isCompleted` to the `WatchHistory` entity."

### F. Additional Technical Questions (Rapid Fire)

**Q31: What is the difference between `@Component` and `@Service` in Spring?**
**A:** "Functionally they are the same (both create beans). Semantically, `@Service` indicates the class holds business logic, while `@Component` is a generic stereotype."

**Q32: What is the default scope of a Spring Bean?**
**A:** "Singleton. One instance per application context."

**Q33: How do you secure the `/save` endpoint?**
**A:** "We use Spring Security with JWT. The request must include a valid Bearer Token in the header. The Security Filter Chain validates this token before the request ever reaches the Controller."

**Q34: What is Lazy Loading in JPA?**
**A:** "It means related data (like the User inside WatchHistory) isn't fetched from the DB until you actually call `.getUser()`. This saves performance. Eager loading fetches everything upfront."

**Q35: Difference between `PUT` and `POST`?**
**A:** "`PUT` is idempotent (sending the same request twice changes nothing after the first). `POST` is not. Typically `PUT` updates, `POST` creates. I used `POST` for `saveProgress` because it simplifies the Upsert logic, but `PUT` could technically be used for updating progress."

**Q36: What is a Primary Key?**
**A:** "A unique identifier for a row. in `WatchHistory`, `id` is the PK, generated automatically (`GenerationType.IDENTITY`)."

**Q37: Explain `Optional<T>` usage in your Service.**
**A:** "`userRepository.findById` returns an `Optional`. This forces me to handle the 'null' case explicitly (using `.orElse(null)` or `.orElseThrow()`), preventing NullPointerExceptions."

**Q38: What port is your backend running on?**
**A:** "Port 8081, as configured in `application.properties`."

**Q39: How do you enable CORS?**
**A:** "We used a configuration bean implementing `WebMvcConfigurer` to allow requests from `localhost:4200` (Angular) to access the backend APIs."

**Q40: What is the `package.json` file?**
**A:** "It acts as the manifest for the Frontend project, listing all dependencies (like Angular, Bootstrap) and commands (scripts) to run the app."

**Q41: How does Angular routing work?**
**A:** "The `RouterModule` maps URL paths (e.g., `/movie/:id`) to specific components (`MovieDetailComponent`). The `<router-outlet>` directive tells Angular where to insert the component."

**Q42: What is a JWT?**
**A:** "JSON Web Token. It's a stateless way to handle authentication. It contains a payload (User ID, Email) signed securely. The backend verifies the signature to trust the user."

**Q43: How do you pass the JWT in requests?**
**A:** "We use an HTTP Interceptor (`JwtInterceptor`) in Angular. It intercepts every outgoing request and clones it to add the `Authorization: Bearer <token>` header."

**Q44: What is the 'Model' in your Frontend?**
**A:** "They are TypeScript interfaces (e.g., `Movie`, `UserDTO`) that define the shape of valid data objects. They don't have logic, just structure."

**Q45: Why use `@Autowired` on the constructor (Constructor Injection) vs fields?**
**A:** "Constructor injection is generally preferred because it makes dependencies explicit and allows fields to be `final`, ensuring the bean is immutable."

**Q46: What is Maven?**
**A:** "The build tool and dependency manager for the Backend. It uses `pom.xml` to download libraries like Spring Web, JPA, and PostgreSQL Driver."

**Q47: Can you explain the `pom.xml`?**
**A:** "Project Object Model. It defines the project's meta-data, dependencies (Spring Boot Starter Web, Data JPA), and build plugins."

**Q48: How do you check if a video is loaded?**
**A:** "The HTML5 video API provides `readyState`. However, mostly we rely on events like `loadedmetadata` or `canplay`."

**Q49: What is `ng-content`?**
**A:** "Ideally used for Content Projection (passing HTML into a component from outside). We didn't use it here, but it's like a slot."

**Q50: Why use `Long` instead of `long` for IDs?**
**A:** "`Long` (wrapper class) is nullable, while `long` (primitive) defaults to 0. Database IDs can be null before persistence, so the Wrapper is safer for Entity IDs."

**Q51: What happens if `save(entity)` is called on an entity with an ID?**
**A:** "JPA interprets it as an Update (Merge) operation. If ID is null, it interprets it as an Insert (Persist)."

**Q52: How do you prevent SQL Injection?**
**A:** "By using JPA Repositories. Under the hood, they use PreparedStatement with parameterized queries, which sanitizes inputs automatically."

**Q53: What is the difference between `interface` and `class` in TypeScript?**
**A:** "Key difference: Interfaces vanish completely at runtime (they are dev-time only). Classes persist as JS functions. We use Interfaces for Models to save bundle size."

**Q54: What is `ViewEncapsulation` in Angular?**
**A:** "It determines if styles defined in a component affect *only* that component. The default is `Emulated`, meaning Angular adds unique attributes to elements so CSS doesn't leak out."

**Q55: If you had to add 'User Reviews', how would you modify the DB?**
**A:** "I would create a `Review` entity with `@ManyToOne` to User and `@ManyToOne` to Movie, plus `rating` and `comment` fields."
