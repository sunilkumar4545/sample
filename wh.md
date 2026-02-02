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

