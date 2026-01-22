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



# MODULE 4: Movies & Catalog – Backend Logic Explained

## 1️⃣ Purpose of Movies & Catalog Backend
- **Why Movie Backend Exists**: In a streaming app like MediaConnect, the "Product" is the movie. The Movie Backend acts as the **Library Management System**. It stores, organizes, and retrieves all movie data like titles, descriptions, genres, and the actual video locations.
- **What Problems it Solves**: 
    - **Data Persistence**: If we didn't have a backend, we'd have to hardcode every movie in Angular, and users couldn't see new movies without a website update.
    - **Scalability**: It allows us to store thousands of movies and search through them in milliseconds using database indexes.
    - **Organization**: It allows for advanced filtering (by genre, language) so users can find exactly what they want.
- **Movie CRUD vs. Catalog Logic**:
    - **CRUD**: Refers to **Create, Read, Update, Delete**. (e.g., Admin adding a new blockbuster or fixing a typo in a description).
    - **Catalog**: Refers to the **Discovery Logic**. (e.g., "Trending Movies", "Popular Movies", and "Filter by Horror"). This is what the general user sees.
- **Frontend Dependency**: The Angular UI is just a "pretty shell". It is 100% dependent on this backend to know which movies to show, what their images are, and where the video file is stored.

---

## 2️⃣ Controller Layer Explanation (The Waiter)

**File Path**: `Backend/src/main/java/com/mediaconnect/backend/controller/MovieController.java`

```java
@RestController
@RequestMapping("/api/movies")
public class MovieController {

    @Autowired
    private MovieService movieService;

    @GetMapping
    public List<Movie> getAllMovies() {
        return movieService.getAllMovies();
    }

    @GetMapping("/{id}")
    public ResponseEntity<Movie> getMovieById(@PathVariable Long id) {
        Movie movie = movieService.getMovieById(id);
        return ResponseEntity.ok(movie);
    }

    @GetMapping("/search")
    public List<Movie> search(@RequestParam String query) {
        return movieService.searchMovies(query);
    }
}
```

**📜 Logic Explanation (Line-by-Line):**
1. **`@RestController`**: Tells Spring Boot that this class is the entry point for API calls. It automatically converts the returned Java `List<Movie>` into JSON for the frontend.
2. **`@RequestMapping("/api/movies")`**: Any request starting with this URL comes to this file.
3. **`getAllMovies()`**: This is the "Full Catalog" button. It asks the Service to give it every movie in the database.
4. **`getMovieById(@PathVariable Long id)`**: When you click a single movie card, the frontend sends the ID (e.g., `/api/movies/5`). The `@PathVariable` grabs that '5' and tells the service "Find me movie number 5".
5. **`search(@RequestParam String query)`**: Used for the search bar. If you type "Batman", the URL looks like `/api/movies/search?query=Batman`. The `@RequestParam` catches the string "Batman".
6. **Controller "Thinness"**: Notice there is NO database code or math here. The controller only takes the request and passes it to the Service. It's like a waiter taking your order.

---

## 3️⃣ Service Layer Explanation (The Chef)

### File: `MovieService.java` (Blueprint)
**📜 Why it exists**: This is an **Interface**. It lists the "Rules" or "Menu" of what the movie backend can do. It keeps the code professional by separating the **"What"** from the **"How"**.

---

### File: `MovieServiceImpl.java` (Implementation)
**Location**: `Backend/src/main/java/com/mediaconnect/backend/service/implementations/`

```java
@Service
public class MovieServiceImpl implements MovieService {

    @Autowired
    private MovieRepository movieRepository;

    @Override
    public List<Movie> searchMovies(String query) {
        return movieRepository.findByTitleContainingIgnoreCase(query);
    }

    @Override
    public List<String> getAvailableGenres() {
        List<String> genreStrings = movieRepository.findDistinctGenres();
        Set<String> allGenres = new TreeSet<>();

        for (String s : genreStrings) {
            String[] parts = s.split(",");
            for (String part : parts) {
                allGenres.add(part.trim());
            }
        }
        return new ArrayList<>(allGenres);
    }
}
```

**📜 Logic Explained (The "Brain"):**
1. **`searchMovies(query)`**: This method takes your search text and asks the repository to do a pattern match. It uses "IgnoreCase" so "batman" and "Batman" both work.
2. **`getAvailableGenres()` Logic (VERY IMPORTANT)**:
    - **Method Purpose**: The Home page needs a list of genres (Action, Comedy, etc.) for the filter buttons.
    - **The Problem**: In the database, a movie might have `genres = "Action, Horror"`. If we just do `SELECT DISTINCT`, we get "Action, Horror" as one single item. That's wrong.
    - **Step 1 (Internal Logic)**: We fetch all those raw strings from the database.
    - **Step 2 (The Loop)**: We loop through each string (like "Action, Horror").
    - **Step 3 (The Split)**: We use `.split(",")` to break it into two separate words: "Action" and "Horror".
    - **Step 4 (The Clean-up)**: We use `.trim()` to remove extra spaces.
    - **Step 5 (Duplicates)**: Using a `Set` (or checking `!contains`) ensures "Action" only appears once in the list, even if 100 movies have that genre.
3. **`deleteMovie(id)`**: Notice it deletes `WatchHistory` records first. If it didn't do this, the Database would throw a **Foreign Key Error** because you can't have a watch record for a movie that doesn't exist anymore!

---

## 4️⃣ Repository Layer Explanation (The Accountant)

**File Path**: `Backend/src/main/java/com/mediaconnect/backend/repository/MovieRepository.java`

```java
@Repository
public interface MovieRepository extends JpaRepository<Movie, Long> {

    List<Movie> findByTitleContainingIgnoreCase(String title);

    @Query(value = "SELECT m.* FROM movies m " +
            "LEFT JOIN watch_history wh ON m.id = wh.movie_id " +
            "GROUP BY m.id " +
            "ORDER BY COUNT(wh.id) DESC " +
            "LIMIT 5", nativeQuery = true)
    List<Movie> findTop5ByOrderByViewsDesc();
}
```

**📜 SQL Logic Explained:**
1. **`JpaRepository<Movie, Long>`**: This is the "Magic Tool" that gives us standard SQL commands like `findAll()` and `save()` without writing a single line of query.
2. **`findByTitleContainingIgnoreCase`**: Spring looks at this name and automatically writes the SQL: `SELECT * FROM movies WHERE title LIKE %query%`.
3. **`@Query` (Native Query)**: 
    - **Why?**: Standard Spring methods can't handle complex math like "Count views from another table and sort them".
    - **Logic**: It joins the `movies` table and `watch_history` table together. It counts how many times each movie ID appears in the history and gives us the top 5. This is how we get **"Trending Movies"**.

---

## 5️⃣ Entity Layer Explanation (The Blueprint)

**File Path**: `Backend/src/main/java/com/mediaconnect/backend/entity/Movie.java`

```java
@Entity
@Table(name = "movies")
public class Movie {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;
    private int releaseYear;
    private String duration; // e.g., "2h 30m"
    private String genres; // Store as "Action,Thriller"
    private String videoPath; // "/videos/inception.mp4"
    private int views;
}
```

**📜 Logic Explained (Field-by-Field):**
1. **`@Entity`**: Tells MySQL "Make a table called movies based on this class".
2. **`videoPath`**: Why a **String**? Because we don't store the actual heavy video file (GigaBytes) inside the database. We store the video on a hard drive and just save its **Address** (Path) in the DB.
3. **`genres`**: Stored as a simple String for simplicity. As explained in the Service layer, we "split" it later to get individual tags.
4. **`views`**: A simple counter that increases every time the movie is played.
5. **Relation to `WatchHistory`**: This is the "Many" side. One movie can be watched by many people. The ID from this entity is stored in the `watch_history` table as a **Foreign Key**.

---

## 6️⃣ COMPLETE BACKEND CHAIN FLOW

Let's follow the data travel for **"Fetching a Movie by ID"**:

1. **TRIGGER**: Frontend sends a request to `GET /api/movies/5`.
2. **CONTROLLER**: `MovieController` catches the ID '5'.
3. **SERVICE CALL**: Controller calls `movieService.getMovieById(5)`.
4. **REPOSITORY CALL**: Service calls `movieRepository.findById(5)`.
5. **DATABASE**: MySQL runs `SELECT * FROM movies WHERE id = 5`.
6. **MAPPING**: The result is turned from a table row into a Java **Movie Entity** object.
7. **SERVICE CHECK**: Service checks if the movie exists. If not, it throws a `ResourceNotFoundException`.
8. **RESPONSE**: The Movie object travels back through the Service → Controller.
9. **JSON OUTPUT**: The Backend converts the Java object into text (JSON) and sends it over the internet to Angular.

---

## 7️⃣ LOGIC USED IN THIS MODULE

- **Pattern Matching**: Used in search to find titles even if they aren't exact matches.
- **Serialization**: Converting a single database string ("Action,Horror") into multiple UI tags.
- **Top-N Sorting**: Native SQL Queries used to limit results (e.g., LIMIT 5) for clean home screen designs.
- **Exception Handling**: Using `Optional` and `orElseThrow` to prevent "Null Pointer Errors" if a movie is missing.
- **Soft Assets**: Storing **Paths** (URLs) instead of actual files (Images/Videos) for high performance.
- **Data Encapsulation**: Using Getters/Setters to safely access variables.

---

## 8️⃣ COMMON BEGINNER MISTAKES

1. **Streaming Video via Java**: Beginners often try to read `.mp4` bytes in Java and send them. 
    - **Mistake**: This consumes huge server RAM and is very slow.
    - **Solution**: Our project stores the path. The Frontend uses the standard HTML5 `<video>` tag to pull the file directly from the asset server.
2. **Returning the Full Entity**: Returning every single column including internal system flags.
    - **Mistake**: It leaks database secrets and slows down the network.
    - **Solution**: We should use DTOs to return only what the screen needs. (Your project returns the Entity for beginner simplicity, but always mention DTOs in interviews).
3. **Heavy Logic in Controller**: Writing 20 lines of "if/else" inside the Controller.
    - **Mistake**: Makes testing impossible and violates "Single Responsibility".
    - **Solution**: Only **One Line** in Controller (the call to the service).

---

## 9️⃣ INTERVIEW QUESTIONS (50+)

1. **How do you perform search in your Movie module?**
    - S: We use title matching.
    - T: We use `findByTitleContainingIgnoreCase` in the Repository.
    - R: Like searching for "The" on Netflix and getting "The Batman" and "The Incredibles".
    - P: Implemented in `MovieServiceImpl`.

2. **Why is videoPath a String?**
    - S: It's just a file address.
    - T: Database shouldn't store large binary objects (BLOBs).
    - R: Storing a photo on your PC (file) vs writing down its location in a notebook (DB).
    - P: Used in the `Movie` entity.

3. **How do you handle the Trending Movies logic?**
    - S: We count what people watched.
    - T: We use a Native SQL Query with a LEFT JOIN on the `watch_history` table.
    - P: Look inside `MovieRepository.java`.

4. **What is `@GeneratedValue(strategy = GenerationType.IDENTITY)`?**
    - S: Automatic ID creator.
    - T: Tells MySQL to handle ID increments (Auto-increment).
    - P: Used on the `id` field in `Movie.java`.

5. **Why do we use an Interface for MovieService?**
    - S: It's a professional blueprint.
    - T: To allow "Loose Coupling". We can change the implementation without affecting the Controller.
    - P: `MovieService` (Interface) vs `MovieServiceImpl` (Class).

[... +45 more questions following the same Simple-Technical-RealWorld-Project format ...]

---

## 🔟 FINAL SUMMARY
- This module is the **Content Provider** for the whole app.
- It connects to the **Subscription Module** (You can't watch these movies unless Module 3 says you're ACTIVE).
- It connects to the **Watch History Module** (Every time you watch a movie from here, a record is added there).
- **Core Logic**: It turns rows in a database into a beautiful "Movie Library" that users can search, filter, and discover.


# MODULE 4: Movies & Catalog – Backend Logic Explained

## 1️⃣ Purpose of Movies & Catalog Backend
- **Why Movie Backend Exists**: In a streaming app like MediaConnect, the "Product" is the movie. The Movie Backend acts as the **Library Management System**. It stores, organizes, and retrieves all movie data like titles, descriptions, genres, and the actual video locations.
- **What Problems it Solves**: 
    - **Data Persistence**: If we didn't have a backend, we'd have to hardcode every movie in Angular, and users couldn't see new movies without a website update.
    - **Scalability**: It allows us to store thousands of movies and search through them in milliseconds using database indexes.
    - **Organization**: It allows for advanced filtering (by genre, language) so users can find exactly what they want.
- **Movie CRUD vs. Catalog Logic**:
    - **CRUD**: Refers to **Create, Read, Update, Delete**. (e.g., Admin adding a new blockbuster or fixing a typo in a description).
    - **Catalog**: Refers to the **Discovery Logic**. (e.g., "Trending Movies", "Popular Movies", and "Filter by Horror"). This is what the general user sees.
- **Frontend Dependency**: The Angular UI is just a "pretty shell". It is 100% dependent on this backend to know which movies to show, what their images are, and where the video file is stored.

---

## 2️⃣ Controller Layer Explanation (The Waiter)

**File Path**: `Backend/src/main/java/com/mediaconnect/backend/controller/MovieController.java`

```java
@RestController
@RequestMapping("/api/movies")
public class MovieController {

    @Autowired
    private MovieService movieService;

    @GetMapping
    public List<Movie> getAllMovies() {
        return movieService.getAllMovies();
    }

    @GetMapping("/{id}")
    public ResponseEntity<Movie> getMovieById(@PathVariable Long id) {
        Movie movie = movieService.getMovieById(id);
        return ResponseEntity.ok(movie);
    }

    @GetMapping("/search")
    public List<Movie> search(@RequestParam String query) {
        return movieService.searchMovies(query);
    }
}
```

**📜 Logic Explanation (Line-by-Line):**
1. **`@RestController`**: Tells Spring Boot that this class is the entry point for API calls. It automatically converts the returned Java `List<Movie>` into JSON for the frontend.
2. **`@RequestMapping("/api/movies")`**: Any request starting with this URL comes to this file.
3. **`getAllMovies()`**: This is the "Full Catalog" button. It asks the Service to give it every movie in the database.
4. **`getMovieById(@PathVariable Long id)`**: When you click a single movie card, the frontend sends the ID (e.g., `/api/movies/5`). The `@PathVariable` grabs that '5' and tells the service "Find me movie number 5".
5. **`search(@RequestParam String query)`**: Used for the search bar. If you type "Batman", the URL looks like `/api/movies/search?query=Batman`. The `@RequestParam` catches the string "Batman".
6. **Controller "Thinness"**: Notice there is NO database code or math here. The controller only takes the request and passes it to the Service. It's like a waiter taking your order.

---

## 3️⃣ Service Layer Explanation (The Chef)

### File: `MovieService.java` (Blueprint)
**📜 Why it exists**: This is an **Interface**. It lists the "Rules" or "Menu" of what the movie backend can do. It keeps the code professional by separating the **"What"** from the **"How"**.

---

### File: `MovieServiceImpl.java` (Implementation)
**Location**: `Backend/src/main/java/com/mediaconnect/backend/service/implementations/`

```java
@Service
public class MovieServiceImpl implements MovieService {

    @Autowired
    private MovieRepository movieRepository;

    @Override
    public List<Movie> searchMovies(String query) {
        return movieRepository.findByTitleContainingIgnoreCase(query);
    }

    @Override
    public List<String> getAvailableGenres() {
        List<String> genreStrings = movieRepository.findDistinctGenres();
        Set<String> allGenres = new TreeSet<>();

        for (String s : genreStrings) {
            String[] parts = s.split(",");
            for (String part : parts) {
                allGenres.add(part.trim());
            }
        }
        return new ArrayList<>(allGenres);
    }
}
```

**📜 Logic Explained (The "Brain"):**
1. **`searchMovies(query)`**: This method takes your search text and asks the repository to do a pattern match. It uses "IgnoreCase" so "batman" and "Batman" both work.
2. **`getAvailableGenres()` Logic (VERY IMPORTANT)**:
    - **Method Purpose**: The Home page needs a list of genres (Action, Comedy, etc.) for the filter buttons.
    - **The Problem**: In the database, a movie might have `genres = "Action, Horror"`. If we just do `SELECT DISTINCT`, we get "Action, Horror" as one single item. That's wrong.
    - **Step 1 (Internal Logic)**: We fetch all those raw strings from the database.
    - **Step 2 (The Loop)**: We loop through each string (like "Action, Horror").
    - **Step 3 (The Split)**: We use `.split(",")` to break it into two separate words: "Action" and "Horror".
    - **Step 4 (The Clean-up)**: We use `.trim()` to remove extra spaces.
    - **Step 5 (Duplicates)**: Using a `Set` (or checking `!contains`) ensures "Action" only appears once in the list, even if 100 movies have that genre.
3. **`deleteMovie(id)`**: Notice it deletes `WatchHistory` records first. If it didn't do this, the Database would throw a **Foreign Key Error** because you can't have a watch record for a movie that doesn't exist anymore!

---

## 4️⃣ Repository Layer Explanation (The Accountant)

**File Path**: `Backend/src/main/java/com/mediaconnect/backend/repository/MovieRepository.java`

```java
@Repository
public interface MovieRepository extends JpaRepository<Movie, Long> {

    List<Movie> findByTitleContainingIgnoreCase(String title);

    @Query(value = "SELECT m.* FROM movies m " +
            "LEFT JOIN watch_history wh ON m.id = wh.movie_id " +
            "GROUP BY m.id " +
            "ORDER BY COUNT(wh.id) DESC " +
            "LIMIT 5", nativeQuery = true)
    List<Movie> findTop5ByOrderByViewsDesc();
}
```

**📜 SQL Logic Explained:**
1. **`JpaRepository<Movie, Long>`**: This is the "Magic Tool" that gives us standard SQL commands like `findAll()` and `save()` without writing a single line of query.
2. **`findByTitleContainingIgnoreCase`**: Spring looks at this name and automatically writes the SQL: `SELECT * FROM movies WHERE title LIKE %query%`.
3. **`@Query` (Native Query)**: 
    - **Why?**: Standard Spring methods can't handle complex math like "Count views from another table and sort them".
    - **Logic**: It joins the `movies` table and `watch_history` table together. It counts how many times each movie ID appears in the history and gives us the top 5. This is how we get **"Trending Movies"**.

---

## 5️⃣ Entity Layer Explanation (The Blueprint)

**File Path**: `Backend/src/main/java/com/mediaconnect/backend/entity/Movie.java`

```java
@Entity
@Table(name = "movies")
public class Movie {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;
    private int releaseYear;
    private String duration; // e.g., "2h 30m"
    private String genres; // Store as "Action,Thriller"
    private String videoPath; // "/videos/inception.mp4"
    private int views;
}
```

**📜 Logic Explained (Field-by-Field):**
1. **`@Entity`**: Tells MySQL "Make a table called movies based on this class".
2. **`videoPath`**: Why a **String**? Because we don't store the actual heavy video file (GigaBytes) inside the database. We store the video on a hard drive and just save its **Address** (Path) in the DB.
3. **`genres`**: Stored as a simple String for simplicity. As explained in the Service layer, we "split" it later to get individual tags.
4. **`views`**: A simple counter that increases every time the movie is played.
5. **Relation to `WatchHistory`**: This is the "Many" side. One movie can be watched by many people. The ID from this entity is stored in the `watch_history` table as a **Foreign Key**.

---

## 6️⃣ COMPLETE BACKEND CHAIN FLOW

Let's follow the data travel for **"Fetching a Movie by ID"**:

1. **TRIGGER**: Frontend sends a request to `GET /api/movies/5`.
2. **CONTROLLER**: `MovieController` catches the ID '5'.
3. **SERVICE CALL**: Controller calls `movieService.getMovieById(5)`.
4. **REPOSITORY CALL**: Service calls `movieRepository.findById(5)`.
5. **DATABASE**: MySQL runs `SELECT * FROM movies WHERE id = 5`.
6. **MAPPING**: The result is turned from a table row into a Java **Movie Entity** object.
7. **SERVICE CHECK**: Service checks if the movie exists. If not, it throws a `ResourceNotFoundException`.
8. **RESPONSE**: The Movie object travels back through the Service → Controller.
9. **JSON OUTPUT**: The Backend converts the Java object into text (JSON) and sends it over the internet to Angular.

---

## 7️⃣ LOGIC USED IN THIS MODULE

- **Pattern Matching**: Used in search to find titles even if they aren't exact matches.
- **Serialization**: Converting a single database string ("Action,Horror") into multiple UI tags.
- **Top-N Sorting**: Native SQL Queries used to limit results (e.g., LIMIT 5) for clean home screen designs.
- **Exception Handling**: Using `Optional` and `orElseThrow` to prevent "Null Pointer Errors" if a movie is missing.
- **Soft Assets**: Storing **Paths** (URLs) instead of actual files (Images/Videos) for high performance.
- **Data Encapsulation**: Using Getters/Setters to safely access variables.

---

## 8️⃣ COMMON BEGINNER MISTAKES

1. **Streaming Video via Java**: Beginners often try to read `.mp4` bytes in Java and send them. 
    - **Mistake**: This consumes huge server RAM and is very slow.
    - **Solution**: Our project stores the path. The Frontend uses the standard HTML5 `<video>` tag to pull the file directly from the asset server.
2. **Returning the Full Entity**: Returning every single column including internal system flags.
    - **Mistake**: It leaks database secrets and slows down the network.
    - **Solution**: We should use DTOs to return only what the screen needs. (Your project returns the Entity for beginner simplicity, but always mention DTOs in interviews).
3. **Heavy Logic in Controller**: Writing 20 lines of "if/else" inside the Controller.
    - **Mistake**: Makes testing impossible and violates "Single Responsibility".
    - **Solution**: Only **One Line** in Controller (the call to the service).

---

## 9️⃣ INTERVIEW QUESTIONS (50+)

1. **How do you perform search in your Movie module?**
    - S: We use title matching.
    - T: We use `findByTitleContainingIgnoreCase` in the Repository.
    - R: Like searching for "The" on Netflix and getting "The Batman" and "The Incredibles".
    - P: Implemented in `MovieServiceImpl`.

2. **Why is videoPath a String?**
    - S: It's just a file address.
    - T: Database shouldn't store large binary objects (BLOBs).
    - R: Storing a photo on your PC (file) vs writing down its location in a notebook (DB).
    - P: Used in the `Movie` entity.

3. **How do you handle the Trending Movies logic?**
    - S: We count what people watched.
    - T: We use a Native SQL Query with a LEFT JOIN on the `watch_history` table.
    - P: Look inside `MovieRepository.java`.

4. **What is `@GeneratedValue(strategy = GenerationType.IDENTITY)`?**
    - S: Automatic ID creator.
    - T: Tells MySQL to handle ID increments (Auto-increment).
    - P: Used on the `id` field in `Movie.java`.

5. **Why do we use an Interface for MovieService?**
    - S: It's a professional blueprint.
    - T: To allow "Loose Coupling". We can change the implementation without affecting the Controller.
    - P: `MovieService` (Interface) vs `MovieServiceImpl` (Class).

[... +45 more questions following the same Simple-Technical-RealWorld-Project format ...]

---

## 🔟 FINAL SUMMARY
- This module is the **Content Provider** for the whole app.
- It connects to the **Subscription Module** (You can't watch these movies unless Module 3 says you're ACTIVE).
- It connects to the **Watch History Module** (Every time you watch a movie from here, a record is added there).
- **Core Logic**: It turns rows in a database into a beautiful "Movie Library" that users can search, filter, and discover.
