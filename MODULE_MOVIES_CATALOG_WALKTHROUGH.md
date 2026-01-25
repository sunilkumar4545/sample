# 🎬 MODULE 4: Movies & Catalog - Complete Deep Dive

This document explains **exactly** how the Movie Catalog works in MediaConnect. We will trace how the application fetches thousands of movies, filters them by genre (like "Action"), and handles search queries.

---

## 🗺️ 1. What is the Catalog? (High-Level)

- **The Goal**: Display a library of movies to the user.
- **The Challenge**: We can't just "show everything" at once in a messy pile. We need organization (Trending, Popular) and tools to find things (Search, Filters).
- **The Data Flow**:
    1.  **Frontend** asks: "Give me the top 5 trending movies."
    2.  **Backend** asks Database: "Which 5 movies have the most views?"
    3.  **Database** counts the views and replies.
    4.  **Backend** sends the list to Frontend.
    5.  **Frontend** draws a "Card" for each movie.

---

## 📂 2. List of All Impacted Files

Here is the map of files we will explore.

### 🟢 Frontend (Angular)
| File Name | Location | Role |
|-----------|----------|------|
| **home.component.html** | `src/app/components/home/` | The **View**. Defines the grid layout, search bar, and dropdowns. |
| **home.component.ts** | `src/app/components/home/` | The **Brain**. Handles user clicks, stores the movie lists, and talks to the Service. |
| **movie.service.ts** | `src/app/services/` | The **Courier**. Sends HTTP requests to the Backend. |
| **movie-card.component.ts** | `src/app/components/movie-card/` | A **Reusable Component**. Draws *one* single movie. We use it 50 times on one page. |

### 🔵 Backend (Spring Boot)
| File Name | Location | Role |
|-----------|----------|------|
| **MovieController.java** | `controller` | The **Receptionist**. Receives requests like `/api/movies/trending`. |
| **MovieServiceImpl.java** | `service` | The **Worker**. Contains logic (e.g., splitting "Action,Drama" into specific genres). |
| **MovieRepository.java** | `repository` | The **Librarian**. Knows how to talk to the SQL Database. |
| **Movie.java** | `entity` | The **Definition**. Defines what a "Movie" is (id, title, genre, url). |

---

## � 3. PART 1: FRONTEND VISUALS & LOGIC

### 1. `home.component.html` (The Layout)
**Location:** `Frontend/src/app/components/home/home.component.html`

This file handles **User Interaction**.

**The Search & Filter Bar:**
```html
<div class="row">
    <!-- Search Box -->
    <input [(ngModel)]="searchQuery" (keyup.enter)="search()" placeholder="Search...">
    
    <!-- Genre Dropdown -->
    <select (change)="filterGenre($event)">
        <option value="">All Genres</option>
        <option *ngFor="let g of genres" [value]="g">{{ g }}</option>
    </select>
</div>
```
- **`[(ngModel)]`**: Two-way binding. If the user types in the box, the variable `searchQuery` updates instantly.
- **`(change)`**: An Event Listener. When the user picks a new option, it runs `filterGenre()`.
- **`*ngFor`**: The Loop. It creates an `<option>` for every genre in our list.

**The Display Logic (Switching Views):**
```html
<div *ngIf="!isSearching">
    <!-- DEFAULT VIEW: Show Trending & Popular -->
    <h3>Trending</h3>
    <div class="row">
        <div class="col" *ngFor="let movie of trendingMovies">
            <app-movie-card [movie]="movie"></app-movie-card>
        </div>
    </div>
</div>

<div *ngIf="isSearching">
    <!-- SEARCH VIEW: Show Results Only -->
    <h3>Results for "{{ searchQuery }}"</h3>
    <div class="row">
         <div class="col" *ngFor="let m of searchResults">
             <app-movie-card [movie]="m"></app-movie-card>
         </div>
    </div>
</div>
```
- **`*ngIf="!isSearching"`**: If the user is NOT searching, show the fancy Trending rows.
- **`*ngIf="isSearching"`**: If specific filters are active, hide the rows and show a big grid of results.

### 2. `home.component.ts` (The Logic)
**Location:** `Frontend/src/app/components/home/home.component.ts`

**Initialization (`ngOnInit`):**
```typescript
ngOnInit(): void {
    // 1. Get Filters (so the dropdown isn't empty)
    this.loadFilterOptions();

    // 2. Get Movies (so the screen isn't empty)
    this.loadPublicData();
}
```

**Handling Filter Clicks:**
```typescript
filterGenre(event: any) {
    const selectedGenre = event.target.value;

    if (selectedGenre) {
        // 1. Change Mode
        this.isSearching = true;

        // 2. Clear old text to avoid confusion
        this.searchQuery = `Genre: ${selectedGenre}`;

        // 3. Call Service
        this.movieService.filterByGenre(selectedGenre).subscribe(data => {
            this.searchResults = data;
        });
    } else {
        // If they selected "All Genres", reset everything
        this.clearSearch();
    }
}
```

---

## 📡 4. PART 2: THE API LAYER

### 3. `movie.service.ts` (The Bridge)
**Location:** `Frontend/src/app/services/movie.service.ts`

This file constructs the URL.

```typescript
// Get distinct genres for the dropdown
getAvailableGenres(): Observable<string[]> {
    return this.http.get<string[]>('http://localhost:8081/api/movies/filters/genres');
}

// Filter movies
filterByGenre(genre: string): Observable<Movie[]> {
    // Example URL: http://localhost:8081/api/movies/filter/genre?genre=Action
    return this.http.get<Movie[]>(`${this.apiUrl}/filter/genre?genre=${genre}`);
}
```

---

## ⚙️ 5. PART 3: BACKEND LOGIC (Where the Magic Happens)

### 4. `MovieController.java` (The Receptionist)
**Location:** `Backend/.../controller/MovieController.java`

Handles the incoming HTTP requests.

```typescript
@GetMapping("/filter/genre")
public List<Movie> filterByGenre(@RequestParam String genre) {
    // Received "Action". Passing to Service...
    return movieService.filterByGenre(genre);
}

@GetMapping("/filters/genres")
public List<String> getAvailableGenres() {
    return movieService.getAvailableGenres();
}
```
- **`@RequestParam`**: Takes `?genre=Action` from the URL and puts it into the `String genre` variable.

### 5. `MovieServiceImpl.java` (The Logic)
**Location:** `Backend/.../service/implementations/MovieServiceImpl.java`

**Topic: Generating the Genre List**
The database stores genres as loose strings like:
- Movie A: "Action, Comedy"
- Movie B: "Drama"
- Movie C: "Action"

We need a clean list: `["Action", "Comedy", "Drama"]`.

```java
@Override
public List<String> getAvailableGenres() {
    // 1. Get all raw strings from DB
    List<String> rawGenres = movieRepository.findDistinctGenres();
    // rawGenres = ["Action, Comedy", "Drama", "Action"]

    List<String> cleanList = new ArrayList<>();

    // 2. Process each string
    for (String raw : rawGenres) {
        // Split by comma
        String[] parts = raw.split(","); 
        
        for (String part : parts) {
            String clean = part.trim(); // Remove spaces
            
            // Avoid duplicates
            if (!cleanList.contains(clean)) {
                cleanList.add(clean);
            }
        }
    }

    // 3. Sort Alphabetically
    Collections.sort(cleanList);
    return cleanList;
}
```
- **Why this logic?** Because our database is simple (stores "Action,Comedy" in one column) instead of complex (Many-to-Many relationships). This Java code simplifies the data for the Frontend.

---

## 🗄️ 6. PART 4: DATABASE ACCESS

### 6. `MovieRepository.java` (The Librarian)
**Location:** `Backend/.../repository/MovieRepository.java`

We use **Spring Data JPA**. We declare the method, and Spring writes the SQL.

```java
// Logic: "Find movies where the title has this text, ignore UPPER/lower case"
List<Movie> findByTitleContainingIgnoreCase(String query);

// Logic: "Find movies where genre column contains this word"
List<Movie> findByGenresContainingIgnoreCase(String genre);

// Logic: "Select specific column"
@Query("SELECT DISTINCT m.genres FROM Movie m")
List<String> findDistinctGenres();
```

**How `findByGenresContainingIgnoreCase("Action")` works:**
- It generates SQL like: `SELECT * FROM movies WHERE UPPER(genres) LIKE UPPER('%Action%')`.
- This matches "Action", "Action, Comedy", and "Sci-Fi, Action".

---

## 🔄 7. COMPLETE FLOW SUMMARY

Let's trace **"I want to see Action movies"**.

1.  **User**: Clicks the "Genre" dropdown and selects "Action".
2.  **Frontend (`home.component.ts`)**:
    - Detects change event.
    - Sets `isSearching = true` (Hides Trending).
    - Calls `movieService.filterByGenre('Action')`.
3.  **Service (`movie.service.ts`)**:
    - Sends `GET /api/movies/filter/genre?genre=Action`.
4.  **Controller (`MovieController.java`)**:
    - Receives request. Calls `movieService.filterByGenre("Action")`.
5.  **Backend Service (`MovieImpl`)**:
    - Calls `movieRepository.findByGenresContainingIgnoreCase("Action")`.
6.  **Repository**:
    - Runs SQL: `SELECT * FROM movies WHERE genres LIKE '%Action%'`.
    - Returns list of 10 movies.
7.  **Return Trip**:
    - Repository -> Service -> Controller -> Frontend.
8.  **Frontend**:
    - `this.searchResults` = [Movie 1, Movie 2...].
    - `*ngFor` updates the HTML.
    - **User** sees the grid of Action movies.

---

## � 8. TECHNICAL GLOSSARY & CONCEPTS

### Frontend Concepts
- **`*ngFor`**: Angular's "For Loop". Used to repeat HTML elements for each item in a list.
- **`[(ngModel)]`**: "Two-way Binding". Links a variable (TS) to an Input (HTML). If one changes, the other changes.
- **`@Input` (in `MovieCard`)**: Allows a parent component (Home) to pass data *down* to a child component (Card).
- **`ChangeDetectorRef`**: Sometimes Angular gets lazy and doesn't notice data changed. We use this to yell "Update the screen now!".

### Backend Concepts
- **`@RequestParam`**: Extracts data from the URL query String (after the `?`).
- **`ContainingIgnoreCase`**: A Spring Data Keyword. It handles the SQL `LIKE %...%` wildcard and case insensitivity logic automatically.
- **`Collections.sort()`**: Standard Java utility to sort a list. Used to make the Genre dropdown A-Z.
- **`List<String>` vs `String[]`**:
    - `String[]` (Array) is fixed size. Good for splitting strings (`split(",")`).
    - `List<String>` (ArrayList) is dynamic. Good for building the final result.

