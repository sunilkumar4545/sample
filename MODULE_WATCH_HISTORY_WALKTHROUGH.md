# 🕒 MODULE 5: Watch History & Resume Logic - Complete Deep Dive

This document explains **exactly** how the "Continue Watching" feature works. We will trace how the application remembers the exact second you stopped watching a movie and how it restores that position when you come back.

---

## 🗺️ 1. What is Watch History? (High-Level)

- **The Goal**: "Netflix-style" resume functionality.
- **The Challenge**: The browser forgets everything when you close it. The server needs to remember "User John stopped *Avatar* at 1 hour 15 minutes."
- **The Data Flow**:
    1.  **Frontend** (While watching): "Hey Server, John is at 10 seconds... 20 seconds... 30 seconds."
    2.  **Backend**: Saves `30` into the database.
    3.  **Frontend** (Next day): "Hey Server, where did John stop *Avatar*?"
    4.  **Backend**: "He stopped at 30 seconds."
    5.  **Frontend**: Fast-forwards the video player to 00:30 automatically.

---

## 📂 2. List of All Impacted Files

Here is the map of files we will explore.

### 🟢 Frontend (Angular)
| File Name | Location | Role |
|-----------|----------|------|
| **movie-detail.component.ts** | `components/movie-detail/` | The **Detector**. Listens to the video player events (`timeupdate`, `pause`). |
| **watch-history.service.ts** | `services/` | The **Courier**. Sends positions to Backend and asks for last position. |
| **home.component.ts** | `components/home/` | The **Display**. Fetches history to show the "Continue Watching" list. |

### 🔵 Backend (Spring Boot)
| File Name | Location | Role |
|-----------|----------|------|
| **WatchHistoryController.java** | `controller` | The **Receptionist**. Receives "User is at 50s" requests. |
| **WatchHistoryServiceImpl.java** | `service` | The **Worker**. Decides whether to *Insert* a new record or *Update* an old one. |
| **WatchHistory.java** | `entity` | The **Definition**. Table with columns: `user_id`, `movie_id`, `progress_seconds`. |
| **WatchHistoryRepository.java** | `repository` | The **Librarian**. Runs `SELECT * FROM watch_history WHERE user_id = ?`. |

---

## 💾 3. PART 1: SAVING PROGRESS (The "Heartbeat")

How does the server know where you are? The frontend sends a "pulse" every few seconds.

### 1. `movie-detail.component.ts` (Frontend Logic)
**Location:** `Frontend/src/app/components/movie-detail/movie-detail.component.ts`

**The Event Listeners:**
In the HTML (not shown but implied), the `<video>` tag triggers events:
- `(timeupdate)`: Fires every time the video time changes.
- `(pause)`: Fires when user clicks pause.

```typescript
// 1. Debouncing (Preventing Spam)
// We don't want to spam the server every millisecond. We wait 5 seconds.
saveProgress(event: Event) {
    // If a timer is already running, cancel it.
    if (this.watchProgressTimer) clearTimeout(this.watchProgressTimer);
    
    // Start a new timer for 5 seconds.
    this.watchProgressTimer = setTimeout(() => {
        const video = event.target as HTMLVideoElement;
        // SEND IT!
        this.saveProgressNow(video.currentTime);
    }, 5000);
}

// 2. Immediate Save (On Pause/Close)
onPause(event: Event) {
    const video = event.target as HTMLVideoElement;
    this.saveProgressNow(video.currentTime); // Save immediately, don't wait.
}

// 3. The Helper Function
private saveProgressNow(seconds: number) {
    // Call the Service
    this.watchHistoryService.saveProgress(this.movie.id, Math.floor(seconds)).subscribe();
}
```

### 2. `watch-history.service.ts` (The Network Call)
**Location:** `Frontend/src/app/services/watch-history.service.ts`

```typescript
saveProgress(movieId: number, seconds: number): Observable<any> {
    const user = this.authService.currentUser();
    
    // Send a POST request with the data
    return this.http.post(`${this.apiUrl}/save`, {
      userId: user.user.id,
      movieId: movieId,
      seconds: seconds
    });
}
```

### 3. `WatchHistoryController.java` (Backend API)
**Location:** `Backend/.../controller/WatchHistoryController.java`

```java
@PostMapping("/save")
public ResponseEntity<?> saveProgress(@RequestBody Map<String, Object> payload) {
    // Extract data from JSON
    Long userId = Long.valueOf(payload.get("userId").toString());
    Long movieId = Long.valueOf(payload.get("movieId").toString());
    int seconds = Integer.parseInt(payload.get("seconds").toString());

    // Call Service
    watchHistoryService.saveProgress(userId, movieId, seconds);
    return ResponseEntity.ok().build();
}
```

### 4. `WatchHistoryServiceImpl.java` (Business Logic)
**Location:** `Backend/.../service/implementations/WatchHistoryServiceImpl.java`

**The Logic: Insert vs Update**
If I watched *Avatar* yesterday, I already have a record. We shouldn't create a NEW row; we should UPDATE the old one.

```java
@Override
public void saveProgress(Long userId, Long movieId, int seconds) {
    // 1. Check if record exists
    WatchHistory existing = watchHistoryRepository.findByUserIdAndMovieId(userId, movieId);

    if (existing != null) {
        // CASE A: User has watched this before. UPDATE it.
        existing.setProgressSeconds(seconds);
        existing.setLastWatched(LocalDateTime.now()); // Update timestamp
        watchHistoryRepository.save(existing);
    } else {
        // CASE B: First time watching. CREATE new.
        User user = userRepository.findById(userId).orElse(null);
        Movie movie = movieRepository.findById(movieId).orElse(null);

        WatchHistory history = new WatchHistory(user, movie, seconds);
        watchHistoryRepository.save(history);
    }
}
```

---

## ⏩ 4. PART 2: RESUMING PLAYBACK (The "Seek")

When you open the page, we need to fast-forward the video automatically.

### 1. `movie-detail.component.ts` (Initialization)

```typescript
ngOnInit() {
    // 1. Load Movie
    this.loadMovieDetails(movieId);
}

loadMovieDetails(movieId: number) {
    // ... fetch movie ...
    // 2. Once movie is loaded, Check History
    this.loadLastWatchPosition(this.movie.id);
}

private loadLastWatchPosition(movieId: number) {
    // 3. Ask Server: "Where was I?"
    this.watchHistoryService.getProgress(movieId).subscribe({
        next: (history) => {
            if (history && history.progressSeconds > 0) {
                
                // 4. THE MAGIC: Fast Forward the Video
                setTimeout(() => {
                    const videoElement = document.querySelector('video') as HTMLVideoElement;
                    if (videoElement) {
                        videoElement.currentTime = history.progressSeconds;
                    }
                }, 500); // Small delay to ensure video player is ready
            }
        }
    });
}
```

### 2. Backend Retrieval Logic
**File:** `WatchHistoryController.java`

```java
@GetMapping("/progress")
public ResponseEntity<?> getProgress(@RequestParam Long userId, @RequestParam Long movieId) {
    // Simple lookup
    WatchHistory history = watchHistoryService.getProgress(userId, movieId);
    
    if (history != null) {
        return ResponseEntity.ok(history); // Return { progressSeconds: 120, ... }
    }
    return ResponseEntity.notFound().build(); // Return 404 (Start from 0)
}
```

---

## 📺 5. PART 3: CONTINUOUS WATCHING LIST (Dashboard)

On the home page, we show a list of movies you started but didn't finish.

### 1. `home.component.ts`

```typescript
loadUserData() {
    // Ask for ALL my history
    this.watchHistoryService.getUserHistory(userId).subscribe({
        next: (data) => {
            // Transform data for the UI
            this.continueWatching = data.map((h: any) => ({
                ...h.movie, // Copy movie details (Title, Image)
                progress: h.progressSeconds // Add progress info
            }));
        }
    });
}
```

### 2. Backend Query (Sorting by Date)
**File:** `WatchHistoryRepository.java`

We want the *most recently watched* movies first.

```java
// "Find my history. Sort it so the newest 'LastWatched' is at the top."
List<WatchHistory> findByUserIdOrderByLastWatchedDesc(Long userId);
```

---

## 🔄 6. COMPLETE FLOW SUMMARY

**Scenario: User watches "Inception"**

1.  **Start:** User opens page. `ngOnInit` checks DB. Result: `null` (Never watched). Video starts at `0:00`.
2.  **Watch:** User watches for 5 minutes (`300` seconds).
3.  **Heartbeat:** `timeupdate` event fires. Debounce waits 5s. Frontend POSTs `300` to backend.
4.  **Save:** Backend sees no existing record. Creates NEW row: `{User: John, Movie: Inception, Time: 300s}`.
5.  **Pause:** User pauses at `10:00` (`600` seconds).
6.  **Update:** Frontend immediately POSTs `600`. Backend finds existing row. Updates Time to `600` and `LastWatched` to `Now`.
7.  **Leave:** User closes browser.

**Scenario: User returns next day**

1.  **Load:** User opens "Inception".
2.  **Fetch:** `ngOnInit` calls `getProgress`. Backend returns `{ progressSeconds: 600 }`.
3.  **Seek:** Frontend sets `video.currentTime = 600`.
4.  **Resume:** Video starts playing from exactly where they left off.

---

## 📚 7. TECHNICAL GLOSSARY

- **`Debounce`**: Waiting for a pause in events before acting. We use `setTimeout` so we don't spam the server 10 times a second while the video plays.
- **`currentTime`**: A property of the HTML5 `<video>` element. It gets or sets the current payback position in seconds (e.g., `120.5`).
- **`Upsert`**: Update or Insert. If logic checks "Does it exist? Update it. If not? Create it."
- **`@ManyToOne`**: In `WatchHistory.java`, this links one History record to ONE User and ONE Movie.
- **`DTO (Implicit)`**: In `saveProgress`, we used a raw `Map<String, Object>`. In a stricter app, we might create a `WatchProgressRequest` class.
