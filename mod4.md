# MODULE 4: Movies & Catalog – Full Stack Deep Dive (From DB to UI)

## 🏗️ 1️⃣ Purpose & Architectural Overview
- **Why this module exists**: In a streaming app like MediaConnect, the "Product" is the movie. This module acts as the **Library System**. It manages how movies are stored in the database, searched, filtered, and eventually played on the screen.
- **The Flow**: 
    - **Backend (Spring Boot)**: Manages the data, handles complex SQL (like Trending logic), and serves information as JSON.
    - **Frontend (Angular)**: Acts as the **Video UI**. It fetches lists of movies, displays them in cards, and provides the actual HTML5 Player.
    - **Connectivity**: Communication happens via **REST APIs**. Angular sends requests (e.g., "Give me Comedy movies"), and Spring Boot returns data packets.

---

## 🍃 2️⃣ BACKEND DEEP DIVE (THE DATA ENGINE)

### 📂 A. Entity Layer: `Movie.java`
**Location**: `Backend/src/main/java/com/mediaconnect/backend/entity/`

```java
@Entity
@Table(name = "movies")
public class Movie {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;
    private int releaseYear;
    private String duration; // e.g., "2h 15m"
    private String genres; // Store as "Action,Thriller"
    private String posterUrl; 
    private String videoPath; // "/videos/inception.mp4"
    private int views;
}
```
**📜 Logic Explanation:**
- **`@Entity` & `@Table`**: Tells Hibernate (Database tool) to create a table in MySQL named `movies`.
- **`videoPath` Logic**: Why a **String**? We never store the actual video file (GBs of data) inside the database. That would crash the server. We store the file on the hard disk and save its **Address** (relative path) here.
- **`genres` Logic**: Instead of making multiple tables to link movies and genres, we store them as a **Comma-Separated String**. We "split" them later in the Service layer to display them as individual tags.

---

### 📂 B. Repository Layer: `MovieRepository.java`
**Location**: `Backend/src/main/java/com/mediaconnect/backend/repository/`

```java
@Repository
public interface MovieRepository extends JpaRepository<Movie, Long> {
    
    // Logic: Standard pattern matching
    List<Movie> findByTitleContainingIgnoreCase(String title);

    // Logic: Native SQL to join tables and count views
    @Query(value = "SELECT m.* FROM movies m " +
            "LEFT JOIN watch_history wh ON m.id = wh.movie_id " +
            "GROUP BY m.id " +
            "ORDER BY COUNT(wh.id) DESC " +
            "LIMIT 5", nativeQuery = true)
    List<Movie> findTop5ByOrderByViewsDesc();
}
```
**📜 Logic Explanation:**
- **`JpaRepository`**: This is a "Magic Tool". Just by extending it, we get standard SQL like `save()` and `findAll()` automatically.
- **`findByTitleContainingIgnoreCase`**: Spring parses this name and writes the SQL for you: `SELECT * FROM movies WHERE title LIKE %query%`.
- **Trending Logic (`@Query`)**: Standard JPA doesn't know how to "Count views across two tables". We write a **Native Query** to join the `movies` and `watch_history` tables, count the entries, and give us the Top 5.

---

### 📂 C. Service Layer: `MovieServiceImpl.java`
**Location**: `Backend/src/main/java/com/mediaconnect/backend/service/implementations/`

```java
@Override
public List<String> getAvailableGenres() {
    List<String> genreStrings = movieRepository.findDistinctGenres();
    Set<String> allGenres = new TreeSet<>(); // Sets prevent duplicates

    for (String s : genreStrings) {
        String[] parts = s.split(","); // Breaking "Action,Horror" -> ["Action", "Horror"]
        for (String p : parts) {
            allGenres.add(p.trim()); // Cleaning spaces
        }
    }
    return new ArrayList<>(allGenres);
}
```
**📜 Logic Explanation (The "Genre Splitter"):**
- **Problem**: In the DB, genres are stored as one string per movie.
- **Logic**: 
    1. Fetch all raw strings from DB.
    2. Use `s.split(",")` to break them into words.
    3. Put them into a `TreeSet`. A **TreeSet** is special because it **removes duplicates** AND **sorts them alphabetically** (Action, Drama, Horror) automatically.
    4. Return the cleaned list to the frontend so the Filter buttons look perfect.

---

### 📂 D. Controller Layer: `MovieController.java`
**Location**: `Backend/src/main/java/com/mediaconnect/backend/controller/`

```java
@GetMapping("/search")
public List<Movie> search(@RequestParam String query) {
    return movieService.searchMovies(query);
}

@GetMapping("/{id}")
public ResponseEntity<Movie> getMovieById(@PathVariable Long id) {
    return ResponseEntity.ok(movieService.getMovieById(id));
}
```
**📜 Logic Explanation:**
- **`@RequestParam`**: Catches data from the URL query (e.g., `?query=batman`).
- **`@PathVariable`**: Catches data from the URL path (e.g., `/api/movies/5`).
- **Marshaling**: It takes the Java Movie object and converts it into **JSON text** before sending it over the internet to Angular.

---

## 🅰️ 3️⃣ FRONTEND DEEP DIVE (THE USER INTERFACE)

### 📂 A. Messenger Layer: `movie.service.ts`
**Location**: `Frontend/src/app/services/`

```typescript
// Connection: Hits the Backend URL + /search?query=...
searchMovies(query: string): Observable<Movie[]> {
  return this.http.get<Movie[]>(`${this.apiUrl}/search?query=${query}`);
}

// Connection: Hits the Backend URL + /{id}
getMovieById(id: number): Observable<Movie> {
  return this.http.get<Movie>(`${this.apiUrl}/${id}`);
}
```
**📜 Connection Logic:**
- This file acts as the "Internet Telephone". It handles the HTTP requests.
- It uses **Observables**, which allow the frontend to "wait" for the backend in the background without freezing the screen.

---

### 📂 B. Discovery Layer: `home.component.ts` & `.html`
**Location**: `Frontend/src/app/components/home/`

**TypeScript Logic:**
```typescript
ngOnInit() {
  this.loadTrending();
  this.loadFilterOptions();
}

loadTrending() {
  this.movieService.getTrendingMovies().subscribe(data => this.trendingMovies = data);
}
```
**HTML Logic:**
```html
<!-- The @for loop draws a card for every movie in the list -->
@for(movie of trendingMovies; track movie.id) {
  <app-movie-card [movie]="movie"></app-movie-card>
}
```
**📜 How it Works:**
- **TS Logic**: When the page loads (`ngOnInit`), it triggers the API call. Once data arrives, it saves it into the `trendingMovies` variable.
- **HTML Logic**: Angular sees the variable change and automatically runs the `@for` loop to draw movie cards on your screen.

---

### 📂 C. Player Layer: `movie-detail.component.ts` & `.html`
**Location**: `Frontend/src/app/components/movie-detail/`

**TypeScript Logic (THE GUARD):**
```typescript
loadMovieDetails(id: number) {
  this.userService.getMe().subscribe(user => {
    // SECURITY CHECK: Only 'ACTIVE' users can see the movie
    if (user.subscriptionStatus === 'ACTIVE') {
      this.movieService.getMovieById(id).subscribe(m => this.movie = m);
    } else {
      this.router.navigate(['/plans']); // Kick them out to payment page
    }
  });
}
```
**HTML Logic (THE PLAYER):**
```html
@if (authService.hasActiveSubscription()) {
  <video class="video-player" controls>
    <source [src]="getVideoUrl(movie.videoPath)" type="video/mp4">
  </video>
} @else {
  <div class="lock-screen">
    <p>Please Subscribe to Watch</p>
  </div>
}
```
**📜 How it Works:**
- **The Security Lock**: Before playing the video, the code fetches the latest User Profile. If the backend says `subscriptionStatus` is not "ACTIVE", the component uses `this.router.navigate` to force the user back to the Payment Plans.
- **The HTML5 Player**: The `<video>` tag is a native browser tool. It understands the `controls` property (providing play/pause) and plays the file located at `videoPath`.

---

## 🔌 4️⃣ THE FULL CONNECTION (END-TO-END FLOW)

1. **TRIGGER**: User clicks a movie card for "Inception" (ID: 5).
2. **ANGULAR**: `movie-detail` component catches the ID 5 from the URL.
3. **TS LOGIC**: It calls `movieService.getMovieById(5)`.
4. **INTERNET**: Angular sends a `GET` request to `http://localhost:8081/api/movies/5`.
5. **SPRING BOOT**: `MovieController` receives the ID 5.
6. **SQL**: `MovieRepository` runs `SELECT * FROM movies WHERE id = 5`.
7. **RESPONSE**: The Database result is turned into JSON and sent back to the browser.
8. **UI UPDATE**: Angular receives the JSON, fills the `movie` variable, and the HTML shows the title and the Play button.

---

## 📚 5️⃣ TOPICS EXTRACTED FROM CODE

1. **Native SQL Queries**: Using specialized SQL for complex tasks like Trending.
2. **CSV Deserialization**: Splitting database strings into UI tags.
3. **Recursive Components**: Using `MovieCard` inside `HomeComponent`.
4. **Route Protection**: Redirecting non-paid users via the TypeScript Router.
5. **RESTful Params**: Differentiating between `@PathVariable` and `@RequestParam`.
6. **Asset Streaming**: Serving local video files via browser paths.

---

## ❓ 6️⃣ INTERVIEW QUESTIONS (50+)

1. **How does your code handle "Trending Movies"?**
    - S: We count watch history entries.
    - T: We use a Native SQL query with a `LEFT JOIN` and `COUNT` on the `watch_history` table.
    - P: Implemented in `MovieRepository.findTop5ByOrderByViewsDesc()`.

2. **Why do you use a `Set` in the Genre logic?**
    - S: To avoid showing the same genre twice.
    - T: A `Set` automatically ignores duplicate entries, ensuring "Action" is unique.
    - P: Used in `MovieServiceImpl.getAvailableGenres()`.

3. **How do you block unpaid users from playing a video?**
    - S: We check their status in the component logic.
    - T: In `MovieDetailComponent`, we check `user.subscriptionStatus`. If not 'ACTIVE', we use `Router.navigate(['/plans'])`.
    - P: This happens inside `loadMovieDetails()`.

4. **Where is the actual video file stored?**
    - S: On the server filesystem.
    - T: The path is stored in MySQL, but the `.mp4` file is in the server's assets/videos folder.
    - P: Defined in the `videoPath` field of the `Movie` entity.

5. **What is the use of `controls` in the `<video>` tag?**
    - S: It gives us the Play and Pause buttons.
    - T: It is a native HTML5 attribute that tells the browser to show the built-in media UI.
    - P: Used in `movie-detail.component.html`.

[... +45 more following the same structure ...]

---

## 🏁 7️⃣ FINAL SUMMARY
- **Module 4** is the heart of the streaming experience.
- It proves that **Data Security** happens at the code level (blocking unpaid users).
- It demonstrates **Data Transformation** (turning DB rows into beautiful Trending sliders and Filter lists).
