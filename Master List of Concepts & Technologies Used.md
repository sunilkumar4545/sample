# 📚 Master List of Concepts & Technologies Used
> **Project**: MediaConnect  
> **Purpose**: A comprehensive list of every single technical concept used in the Backend and Frontend, mapped to the specific modules where they are implemented.

---

# 🛠️ PART 1: BACKEND CONCEPTS (Spring Boot)

## 1. Core Spring Framework
| Concept | Description | Modules Used In |
|:---|:---|:---|
| **Inversion of Control (IoC)** | Spring manages object creation (Beans) instead of using `new Class()`. | **All Modules** (Global) |
| **Dependency Injection (DI)** | Injecting Services into Controllers using `@Autowired` or Constructor Injection. | **All Modules** (e.g., `UserController` needs `UserService`) |
| **Aspect-Oriented Programming (AOP)** | Separating "Cross-Cutting Concerns" like Error Handling from logic. | **Exception Handling Module** (`GlobalExceptionHandler`) |
| **DTO Pattern** | Using Data Transfer Objects (`UserDTO`, `AuthRequest`) to hide Entity internals. | **Auth**, **User**, **Subscription** |

## 2. Spring Web (REST API)
| Concept | Description | Modules Used In |
|:---|:---|:---|
| **REST Controllers** | Classes annotated with `@RestController` to handle HTTP requests. | **All Modules** (`AuthController`, `MovieController`) |
| **Request Mapping** | `@GetMapping`, `@PostMapping`, `@PutMapping` to map to URL paths. | **All Modules** |
| **Path Variables** | `@PathVariable` to extract IDs from URLs (e.g., `/movies/{id}`). | **Movies** (`getMovieById`), **Watch History** (`getUserHistory`) |
| **Request Body** | `@RequestBody` to map JSON to Java Objects. | **Auth** (`register`, `login`), **Subscription** (`subscribe`) |
| **Response Entity** | `ResponseEntity<?>` to fully control Status Codes (200, 404) and Headers. | **All Modules** |

## 3. Spring Data JPA (Database)
| Concept | Description | Modules Used In |
|:---|:---|:---|
| **JPA Entities** | Java classes mapped to DB tables (`@Entity`, `@Table`). | **All Modules** (`User`, `Movie`, `WatchHistory`, `SubscriptionPlan`) |
| **Derived Query Methods** | Auto-generating SQL from method names (e.g., `findByEmail`). | **Auth** (`existsByEmail`), **User** (`findByEmail`), **Watch History** (`findByUserId...`) |
| **Custom JPQL Queries** | Writing SQL-like queries on Objects (`@Query`). | **Analytics** (`findEngagementAnalytics`), **Watch History** (Top 5 Sort) |
| **Native Queries** | Writing raw SQL for complex joins. | **Movies** (Trending Logic), **Analytics** |
| **Pagination** | Using `Pageable` and `PageRequest` to fetch data in chunks. | **Movies Catalog** (Filtering & Listing) |
| **One-to-Many / Many-to-One** | Entity relationships (User has many History items). | **Watch History**, **User**, **Movies** |

## 4. Spring Security (Authentication)
| Concept | Description | Modules Used In |
|:---|:---|:---|
| **JWT (JSON Web Token)** | Stateless authentication using signed tokens. | **Auth Module** (Login/Register) |
| **Security Filter Chain** | Configuring which URLs are public vs private. | **Auth Module** (`SecurityConfig`: `/api/auth` is public) |
| **OncePerRequestFilter** | Custom `JwtFilter` to check token on every request. | **Auth Module** (Global Security) |
| **BCrypt Hashing** | One-way hashing of passwords before saving. | **Auth Module** (Registration) |
| **Context Holder** | `SecurityContextHolder` to store generic user details per request. | **User Module** (`getMe`), **Subscription** (Subscribe) |
| **CORS Configuration** | Allowing Frontend (localhost:4200) to talk to Backend. | **Auth Module** (`CorsConfigurationSource`) |

## 5. Advanced Logic
| Concept | Description | Modules Used In |
|:---|:---|:---|
| **File Byte Generation** | Generating `.txt` invoices in memory from Strings. | **Subscription Module** (Download Invoice) |
| **Debouncing (Logic)** | Backend logic optimized to handle frequent updates. | **Watch History** ("Upsert" Logic) |
| **Scheduled Tasks** | (Optional) Logic for expiring subscriptions. | **Subscription Module** (Conceptually) |

---

# 🎨 PART 2: FRONTEND CONCEPTS (Angular 17+)

## 1. Core Angular
| Concept | Description | Modules Used In |
|:---|:---|:---|
| **Standalone Components** | Components that don't need `NgModule` (`standalone: true`). | **All Components** (`LoginComponent`, `HomeComponent`) |
| **Dependency Injection** | Injecting `HttpClient`, `Router`, `AuthService`. | **All Components** |
| **Lifecycle Hooks** | `ngOnInit` (Load data), `ngOnDestroy` (Cleanup timers). | **Movies** (Load Details), **Watch History** (Clear Timer) |
| **Control Flow (@if, @for)** | New Angular syntax for template logic. | **Home** (Movie Grid), **Details** (Show Video), **Plans** (List) |

## 2. RxJS & Reactivity
| Concept | Description | Modules Used In |
|:---|:---|:---|
| **Observables** | Handling async HTTP responses. | **All Service Calls** |
| **BehaviorSubject** | Holding "Current User" state for immediate access. | **Auth Module** (Navbar "Login/Logout" button state) |
| **Subscription (.subscribe)** | Triggering the request and handling response. | **All Components** |
| **Tap Operator** | Side effects (Saving to localStorage) without changing data. | **Auth Module** (Login) |
| **Map Operator** | Transforming complex Backend Objects to simple UI Objects. | **Watch History** (Flattening history list for Home) |
| **Debouncing (Manual)** | Using `setTimeout` to delay API calls during rapid events. | **Watch History** (Video Time Tracking) |

## 3. Forms & Validation
| Concept | Description | Modules Used In |
|:---|:---|:---|
| **Reactive Forms** | `FormGroup`, `FormControl`, `FormBuilder`. | **Auth Module** (Login, Register) |
| **Form Arrays** | Dynamic lists of controls (Checkboxes). | **Auth Module** (Register: Selecting Genres) |
| **Validation** | `Validators.required`, `Validators.email`. | **Auth Module** (Input checks) |
| **Custom State** | Checking `touched` vs `dirty` to show errors. | **Auth Module** (Error messages) |

## 4. Routing & Guards
| Concept | Description | Modules Used In |
|:---|:---|:---|
| **Route Guards** | `CanActivate` to block non-logged-in users. | **All Protected Pages** (`/home`, `/profile`) |
| **Interceptors** | `JwtInterceptor` adds "Bearer Token" to requests. | **Global** (Auth logic) |
| **ActivatedRoute** | Reading ID from URL (`/movie/5`). | **Movie Details** |

## 5. Media & DOM
| Concept | Description | Modules Used In |
|:---|:---|:---|
| **Video Events** | Listening to `(timeupdate)`, `(pause)`, `(ended)`. | **Watch History** (Player Synchronization) |
| **Blob Handling** | Receiving binary data and creating "Download Links". | **Subscription** (Invoice Download) |
| **DOM Direct Access** | Using `document.querySelector` to seek video. | **Watch History** (Resuming playback position) |

---

# 📦 PART 3: MODULE-BY-MODULE BREAKDOWN

Here is which module uses which specific set of concepts:

### 1. **Authentication Module**
*   **Backend**: `BCrypt`, `JWT`, `SecurityContext`, `Repository`, `Derived Queries`, `DTOs`.
*   **Frontend**: `Reactive Forms`, `FormArray`, `BehaviorSubject`, `Tap`, `Route Guards`.

### 2. **User & Profile Module**
*   **Backend**: `Derived Queries` (findByEmail), `File Generation` (Invoice), `JPA Updates`.
*   **Frontend**: `Blob Handling` (Downloads), `Observables`.

### 3. **Subscription Module**
*   **Backend**: `Date Calculation` (Expiry), `State Management` (Active/Inactive), `Complex Logic`.
*   **Frontend**: `Conditional Rendering` (@if active), `Navigation` (Redirects).

### 4. **Movies Catalog Module**
*   **Backend**: `Native Queries` (Trending), `Pagination` (Limit), `Search Logic`.
*   **Frontend**: `Control Flow` (@for loops), `Input Binding` (Search Bar).

### 5. **Watch History (Continue Watching) Module**
*   **Backend**: `Upsert Logic` (Update if exists, else Create), `Sorting` (OrderByLastWatched).
*   **Frontend**: `Event Binding` (Video), `Debouncing` (Performance), `DOM Manipulation` (Seeking).
