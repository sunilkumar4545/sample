# 🏠 Home Module: Complete Code Walkthrough

This document explains exactly how the **Home Page** displays content. It covers fetching trending movies, search filtering, and personalized "Continue Watching" lists.

---

# 🎨 PART 1: FRONTEND (Angular)

## 1. `home.component.html` (The View)
**Location:** `Frontend/src/app/components/home/home.component.html`

This file organizes the layout into sections.

```html
<!-- SEARCH BAR & FILTERS -->
<select (change)="filterGenre($event)">
    <option *ngFor="let g of genres" [value]="g">{{g}}</option>
</select>

<!-- CONTENT SECTIONS -->
<div *ngIf="!isSearching">
    <!-- 1. Continue Watching (Only for Logged In Users) -->
    <div *ngIf="continueWatching.length > 0">
        <h4>Continue Watching</h4>
        <div *ngFor="let movie of continueWatching">
            <app-movie-card [movie]="movie"></app-movie-card>
        </div>
    </div>

    <!-- 2. Trending Section (Public) -->
    <h4>Trending Now</h4>
    <div *ngFor="let movie of trendingMovies">
         <app-movie-card [movie]="movie"></app-movie-card>
    </div>
</div>

<!-- SEARCH RESULTS (Replaces content if searching) -->
<div *ngIf="isSearching">
    <h4>Results for "{{searchQuery}}"</h4>
    <div *ngFor="let movie of searchResults">
        <app-movie-card [movie]="movie"></app-movie-card>
    </div>
</div>
```
**Logic:**
- **`*ngIf="!isSearching"`**: Shows the standard dashboard (Trending, Popular) by default.
- **`*ngIf="isSearching"`**: Swaps the view to show a grid of search results when the user types or filters.
- **`app-movie-card`**: A reusable child component that takes a `[movie]` object and renders the image and title.

## 2. `home.component.ts` (The Controller)
**Location:** `Frontend/src/app/components/home/home.component.ts`

**Initialization (`ngOnInit`):**
```typescript
ngOnInit(): void {
    // 1. Fetch Dropdown Options
    this.loadFilterOptions(); 
    
    // 2. Fetch Public Data (Trending/Popular)
    this.loadPublicData();

    // 3. Fetch Personal Data (Only if logged in)
    if (this.authService.isAuthenticated()) {
        this.loadUserData();
    }
}
```

**Loading Public Data:**
```typescript
loadPublicData() {
    this.movieService.getTrendingMovies().subscribe({
        next: (data) => {
            this.trendingMovies = data;
        }
    });
    // ... same for popularMovies ...
}
```

**Loading User Data (Continue Watching):**
```typescript
loadUserData() {
    const user = this.authService.currentUser();
    if (user) {
        // Calls a different service: WatchHistory
        this.watchHistoryService.getUserHistory(Number(user.userId)).subscribe({
            next: (data) => {
                // Map the raw history data to a format the Movie Card understands
                this.continueWatching = data.map((h: any) => ({
                    ...h.movie, // Spread movie details (title, image)
                    progress: h.progressSeconds // Add progress for the bar
                }));
            }
        });
    }
}
```

**Filter Logic:**
```typescript
filterGenre(event: any) {
    const genre = event.target.value;
    if (genre) {
        this.isSearching = true; // Switch View Mode
        this.searchQuery = `Genre: ${genre}`;
        // Call API
        this.movieService.filterByGenre(genre).subscribe(data => this.searchResults = data);
    }
}
```

## 3. `movie.service.ts` (The Bridge)
**Location:** `Frontend/src/app/services/movie.service.ts`

```typescript
@Injectable({ providedIn: 'root' })
export class MovieService {
    private apiUrl = 'http://localhost:8081/api/movies';

    getTrendingMovies(): Observable<Movie[]> {
        return this.http.get<Movie[]>(`${this.apiUrl}/trending`);
    }

    filterByGenre(genre: string): Observable<Movie[]> {
        return this.http.get<Movie[]>(`${this.apiUrl}/filter/genre?genre=${genre}`);
    }
    
    getAvailableGenres(): Observable<string[]> {
        return this.http.get<string[]>(`${this.apiUrl}/filters/genres`);
    }
}
```

---

# ⚙️ PART 2: BACKEND (Spring Boot)

## 4. `MovieController.java` (The Entry Point)
**Location:** `Backend/.../controller/MovieController.java`

```java
@RestController
@RequestMapping("/api/movies")
public class MovieController {

    @GetMapping("/trending")
    public List<Movie> getTrending() {
        return movieService.getTrendingMovies();
    }

    @GetMapping("/filter/genre")
    public List<Movie> filterByGenre(@RequestParam String genre) {
        return movieService.filterByGenre(genre);
    }
}
```

## 5. `MovieServiceImpl.java` (Business Logic)
**Location:** `Backend/.../service/implementations/MovieServiceImpl.java`

**Trending Logic:**
```java
@Override
public List<Movie> getTrendingMovies() {
    // Delegates to a specialized Repository Query
    return movieRepository.findTop5ByOrderByViewsDesc();
}
```

**Genre Logic (Complex):**
```java
@Override
public List<String> getAvailableGenres() {
    // 1. Get raw strings like "Action, Comedy" from DB
    List<String> genreStrings = movieRepository.findDistinctGenres();
    
    // 2. Java Logic to Split and Clean
    List<String> allGenres = new ArrayList<>();
    for (String gStr : genreStrings) {
        String[] parts = gStr.split(","); // "Action, Comedy" -> ["Action", " Comedy"]
        for (String p : parts) {
            String clean = p.trim(); // " Comedy" -> "Comedy"
            if (!allGenres.contains(clean)) {
                allGenres.add(clean);
            }
        }
    }
    Collections.sort(allGenres); // A-Z Sort
    return allGenres;
}
```

## 6. `MovieRepository.java` (Database Access)
**Location:** `Backend/.../repository/MovieRepository.java`

**Advanced SQL Queries:**

```java
@Repository
public interface MovieRepository extends JpaRepository<Movie, Long> {

    // 1. Search Logic (Case Insensitive)
    List<Movie> findByTitleContainingIgnoreCase(String title);

    // 2. Genre Filter
    List<Movie> findByGenresContainingIgnoreCase(String genre);

    // 3. Trending (Smart Query)
    // Counts how many times a movie appears in 'watch_history' table
    @Query(value = "SELECT m.* FROM movies m " +
            "LEFT JOIN watch_history wh ON m.id = wh.movie_id " +
            "GROUP BY m.id " +
            "ORDER BY COUNT(wh.id) DESC " +
            "LIMIT 5", nativeQuery = true)
    List<Movie> findTop5ByOrderByViewsDesc();
}
```
**Explanation of Trending Query:**
- It doesn't just look at a "views" column. It joins with the `watch_history` table to count *actual* user interactions. This creates a "Real-Time Trending" algorithm.

---

# 📝 FLOW SUMMARY

**Scenario: User Opens Home Page**

1.  **Frontend (`ngOnInit`)**:
    - Calls `getTrendingMovies()` -> Backend calculates top 5 watched movies.
    - Calls `getAvailableGenres()` -> Backend splits comma-separated strings to find unique genres.
    - If Logged In -> Calls `watchHistoryService.getUserHistory()`.

2.  **Display (`HTML`)**:
    - `trendingMovies` array fills the "Trending" row.
    - `continueWatching` array fills the top row with progress bars.
    - `genres` array populates the `<select>` dropdown.

**Scenario: User Filters by "Action"**

1.  **Dropdown Change**: `filterGenre('Action')` is triggered.
2.  **State Change**: `isSearching = true`. Main dashboard is hidden. Search Grid is shown.
3.  **API Call**: `GET /api/movies/filter/genre?genre=Action`.
4.  **Backend**: `movieRepository.findByGenresContainingIgnoreCase("Action")` runs `SELECT * FROM movies WHERE genres LIKE '%Action%'`.
5.  **Result**: List of Action movies displayed in the grid.
