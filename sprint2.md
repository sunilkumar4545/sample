# Spring Framework Curriculum - Sprint 2 (REST API & Spring Boot 3)

This document maps **Spring REST, RESTful Architecture, and Spring Boot 3** concepts directly to your **MediaConnect** project.

---

## 1️⃣ Overview of RESTful Architecture

### 💻 Code from Your Project
**File:** `Backend/src/main/java/com/mediaconnect/backend/controller/UserController.java`

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/me") // 1. Resource: User Profile. Verb: GET (Read)
    public ResponseEntity<UserDTO> getCurrentUser() { ... }

    @PutMapping("/me") // 2. Verb: PUT (Update Idempotent)
    public ResponseEntity<UserDTO> updateCurrentUser(@RequestBody UserDTO dto) { ... }
}
```

### 🔵 Technical Explanation
**REST (Representational State Transfer)** is an architectural style, not a protocol. It treats data as **Resources** identified by URIs.
*   **Stateless**: The server does not store client context between requests. Each request contains all necessary info (e.g., JWT Token).
*   **Uniform Interface**: Uses standard HTTP verbs (`GET`, `POST`, `PUT`, `DELETE`).
*   **Resource Representation**: Data is transferred in formats like JSON or XML (MediaConnect uses **JSON**).

### 🧐 Why is it used?
In `MediaConnect`, the frontend (Angular) and backend (Spring Boot) are separate applications. REST allows them to communicate regardless of language. The backend exposes data at `/api/users/me`, and the frontend consumes it.

### ⚙️ Internal Working
When a client hits `GET /api/users/me`:
1.  **DispatcherServlet** (Front Controller) receives the request.
2.  It uses **HandlerMapping** to find which Controller method matches the URI and HTTP Method.
3.  It invokes `getCurrentUser()`.
4.  The return value (`UserDTO`) is passed to a **HttpMessageConverter** (Jackson) which serializes the Java Object into a JSON string.

### 🙋‍♂️ Interview Q&A

**Q1: What is the difference between `@Controller` and `@RestController`?**
> **Answer:** `@RestController` is a convenience annotation that combines `@Controller` and `@ResponseBody`. It eliminates the need to annotate every request handling method with `@ResponseBody`.
> *   `@Controller`: Returns a View (HTML page).
> *   `@RestController`: Returns Data (JSON/XML).

**Q2: What is "Statelessness" in REST?**
> **Answer:** It means the server holds no session state. If a user sends request A then request B, request B cannot rely on memory from request A. This improves scalability as any server in a cluster can handle any request.

---

## 2️⃣ Introduction to Spring REST

### 💻 Code from Your Project
**File:** `Backend/src/main/java/com/mediaconnect/backend/controller/UserController.java`

```java
@PostMapping("/subscribe")
public ResponseEntity<?> subscribe(@RequestBody Map<String, String> payload) {
    if (plan == null) {
        return ResponseEntity.badRequest().body("Plan is required"); // 400 Bad Request
    }
    return ResponseEntity.ok(updatedUser); // 200 OK
}
```

### 🔵 Technical Explanation
Spring REST builds on the Servlet API to simplify HTTP handling.
*   **@RequestBody**: Deserializes the HTTP Request Body (JSON) into a Java Object.
*   **ResponseEntity**: A wrapper for the whole HTTP response: status code, headers, and body. It gives you full control over the HTTP response.
*   **Content Negotiation**: Spring determines the response format (JSON/XML) based on the `Accept` header.

### 🧐 Why is it used?
We need to send specific status codes. If a user tries to subscribe to an invalid plan, returning just a String is bad practice. usage of `ResponseEntity.badRequest()` explicitly tells the browser/client "This was your fault" (Status 400).

### ⚙️ Internal Working
Inside `subscribe(@RequestBody ...)`:
1.  Spring sees `@RequestBody`.
2.  It looks at the `Content-Type` header (e.g., `application/json`).
3.  It finds a compatible `HttpMessageConverter` (MappingJackson2HttpMessageConverter).
4.  Jackson's `ObjectMapper` reads the InputStream and maps fields to the `Map<String, String>`.

### 🙋‍♂️ Interview Q&A

**Q1: What is the role of `HttpMessageConverter`?**
> **Answer:** It is responsible for converting HTTP request bodies to Java objects and Java objects to HTTP response bodies. Default implementation uses Jackson for JSON.

**Q2: When should I use `ResponseEntity` vs returning the Object directly?**
> **Answer:**
> *   Return **Object** (e.g., `User`): When the status is always 200 OK.
> *   Return **ResponseEntity**: When you need to customize headers or return different status codes (e.g., 404 Not Found, 201 Created) based on logic.

---

## 3️⃣ Benefits of using Spring Boot for RESTful services

### 💻 Code from Your Project
**File:** `Backend/pom.xml`
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

### 🔵 Technical Explanation
Without Spring Boot, creating a REST API requires configuring `web.xml`, DispatcherServlet, Jackson beans, and a standalone Tomcat server.
Spring Boot provides:
1.  **Starter Dependencies**: `spring-boot-starter-web` includes everything needed.
2.  **Embedded Server**: Uses **Tomcat** inside the jar. No external server installation needed.
3.  **Auto-Configuration**: Automatically registers DispatcherServlet and Error Mappings.

### 🧐 Why is it used?
Ideally, setting up a web server takes hours. With the `starter-web` dependency in your `pom.xml`, `MediaconnectApplication.run()` launches a fully production-ready REST server on port 8081 in seconds.

### ⚙️ Internal Working
When `spring-boot-starter-web` is on the classpath:
1.  `DispatcherServletAutoConfiguration` registers the DispatcherServlet.
2.  `EmbeddedWebServerFactoryCustomizer` starts Tomcat on port 8080 (or 8081).
3.  `JacksonAutoConfiguration` configures the ObjectMapper.

### 🙋‍♂️ Interview Q&A

**Q1: How do you change the default port in Spring Boot?**
> **Answer:** By setting `server.port=8081` in `application.properties`.

**Q2: Can we use Jetty instead of Tomcat?**
> **Answer:** Yes. You exclude `spring-boot-starter-tomcat` from `spring-boot-starter-web` and add `spring-boot-starter-jetty`.

---

## 4️⃣ Setting up a Spring Boot project for REST

### 💻 Code from Your Project
**File:** `Backend/src/main/resources/application.properties`
```properties
server.port=8081
# The Context Path (Root URL)
# server.servlet.context-path=/api 
```

**File:** `Backend/src/main/java/com/mediaconnect/backend/MediaconnectApplication.java`
```java
@SpringBootApplication // @Configuration + @EnableAutoConfiguration + @ComponentScan
public class MediaconnectApplication {
    public static void main(String[] args) {
        SpringApplication.run(MediaconnectApplication.class, args);
    }
}
```

### 🔵 Technical Explanation
Setting up requires:
1.  **Project Initializer**: Normally via start.spring.io.
2.  **Main Class**: The `@SpringBootApplication` annotation is the distinct configuration point.
    *   `@EnableAutoConfiguration`: Tells Spring Boot to "guess" how you want to configure Spring based on the jar dependencies.
    *   `@ComponentScan`: Tells Spring to look for `@RestController`, `@Service`, etc., in the current package and sub-packages.

### 🧐 Why is it used?
Before Boot, "Component Scanning" had to be defined in XML. In your project, just having `MediaconnectApplication` at the root (`com.mediaconnect.backend`) ensures that `UserController` (in `.controller`) and `UserService` (in `.service`) are automatically found and wired up.

### ⚙️ Internal Working
`SpringApplication.run()`:
1.  Creates an appropriate `ApplicationContext` (AnnotationConfigServletWebServerApplicationContext).
2.  Performs a component scan.
3.  Starts the Embedded Tomcat Server.

### 🙋‍♂️ Interview Q&A

**Q1: What happens if I put `MediaconnectApplication` in a sub-package?**
> **Answer:** Spring Boot will fail to scan components outside that sub-package. It scans the current package and *descendants* by default. You would need to explicitly use `@ComponentScan(basePackages="...")` to fix it.

---

## 5️⃣ What's New in Spring Boot 3?

### 💻 Code from Your Project
**File:** `Backend/pom.xml`
```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.0</version> <!-- Boot 3.x! -->
</parent>
<properties>
    <java.version>17</java.version> <!-- Requires Java 17+ -->
</properties>
```

### 🔵 Technical Explanation
Spring Boot 3 (released Nov 2022) is the current major version.
**Key Technical Changes:**
1.  **Jakarta EE 9/10**: The Big Shift. Package names changed from `javax.*` to `jakarta.*`.
    *   Old: `javax.persistence.Entity`
    *   New: `jakarta.persistence.Entity` (Your project uses this!).
2.  **Java 17 Baseline**: It *requires* Java 17 or higher. It will not run on Java 8 or 11.
3.  **Observability**: Native support for **Micrometer Tracing** (formerly Spring Cloud Sleuth).
4.  **Native Image Support**: Built-in support for compiling to GraalVM Native Images (starts in milliseconds, uses tiny memory).

### 🧐 Why is it used?
Your `pom.xml` shows version `3.3.0`. This ensures your project is future-proof, performant, and secure. You are using the latest `jakarta.persistence` annotations in your entities because of this update.

### ⚙️ Internal Working
*   **Jakarta Namespace**: The move to the Eclipse Foundation required rebranding from `javax`. All internal libraries (Hibernate, Tomcat) had to be updated to Jakarta-compatible versions.
*   **AOT Compilation**: For Native Images, Spring does "Ahead-of-Time" processing to identify beans and reflection usage at build time, rather than runtime.

### 🙋‍♂️ Interview Q&A

**Q1: What is the most breaking change in Spring Boot 3?**
> **Answer:** The migration from `javax.*` to `jakarta.*` packages for specifications like JPA, Validation, and Servlet API. Any code importing `javax.persistence` must be updated to `jakarta.persistence`.

**Q2: Why does Spring Boot 3 require Java 17?**
> **Answer:** To leverage modern language features like Records, Sealed Classes, Text Blocks, and the new Garbage Collectors (ZGC) that dramatically improve performance for cloud-native apps.

---

## 6️⃣ Building a Simple REST Controller

### 💻 Code from Your Project
**File:** `Backend/src/main/java/com/mediaconnect/backend/controller/MovieController.java`

```java
@RestController // 1. Defines this class as a REST Controller
@RequestMapping("/api/movies") // 2. Base URI for all methods
public class MovieController {

    @Autowired
    private MovieService movieService;

    @GetMapping // 3. Maps GET /api/movies
    public List<Movie> getAllMovies() {
        return movieService.getAllMovies();
    }

    @GetMapping("/{id}") // 4. Maps GET /api/movies/1
    public ResponseEntity<Movie> getMovieById(@PathVariable Long id) {
        return ResponseEntity.ok(movieService.getMovieById(id));
    }
}
```

### 🔵 Technical Explanation
A **REST Controller** is a specialized Bean in the Spring Context.
1.  **@RestController**: Tells Spring, "Load me into the Context, and write my return values directly to the HTTP Response body (as JSON)."
2.  **@RequestMapping**: Organizes your API. `/api/movies` acts as a prefix.
3.  **Methods**: Each public method mapped with `@Get/Post/Put/DeleteMapping` becomes an API Endpoint.

### 🧐 Why is it used?
This is the "Receptionist" of your application. `MovieController` doesn't know *how* to fetch movies from the DB (that's the Service's job). It only knows how to receive a Web Request and return a JSON Response. separation of concerns!

### ⚙️ Internal Working
At startup:
1.  **Scanning**: Spring scans classes with `@RestController`.
2.  **Registration**: It registers `RequestMappingHandlerMapping` beans.
3.  **Mapping**: It creates a map: `GET /api/movies` -> `getAllMovies()`.
    When a request comes, DispatcherServlet looks up this map to find the right method.

### 🙋‍♂️ Interview Q&A

**Q1: Can I use `@Controller` instead of `@RestController`?**
> **Answer:** Yes, but `@Controller` defaults to returning a View Name (like `index.html` for Thymeleaf/JSP). To return JSON, you would have to add `@ResponseBody` to *every single method*. `@RestController` includes `@ResponseBody` automatically.

**Q2: What happens if two methods have the same URI and Method?**
> **Answer:** Spring Boot will fail to start! It throws an `IllegalStateException` saying there is an "Ambiguous mapping", because it wouldn't know which method to call.

---

## 7️⃣ Request and Response Handling

### 💻 Code from Your Project
**File:** `Backend/src/main/java/com/mediaconnect/backend/controller/MovieController.java` & `UserController.java`

```java
// 1. Path Variable (Dynamic URI)
@GetMapping("/{id}") 
public ResponseEntity<Movie> getById(@PathVariable Long id) { ... }

// 2. Query Parameter (Filtering)
// URL: /api/movies/search?query=Avatar
@GetMapping("/search") 
public List<Movie> search(@RequestParam String query) { ... }

// 3. Request Body (JSON Data) and Custom Response
@PostMapping("/subscribe")
public ResponseEntity<?> subscribe(@RequestBody Map<String, String> payload) {
    if (!isValid(payload)) {
        // Custom 400 Bad Request
        return ResponseEntity.badRequest().body("Invalid Plan");
    }
    // Custom 200 OK
    return ResponseEntity.ok(updatedUser);
}
```

### 🔵 Technical Explanation
*   **@PathVariable**: Extracts values from the URI path itself. Used when the value is *part of the resource identity* (like an ID).
*   **@RequestParam**: Extracts values from the URL query string (`?key=value`). Used for *sorting, filtering, or pagination*.
*   **@RequestBody**: Maps the JSON body to a Java Object (DTO).
*   **ResponseEntity<T>**: A generic container that lets you set the **Status Code** (200, 404, 500), **Headers**, and **Body**.

### 🧐 Why is it used?
*   **Path Variables** make URLs clean and SEO friendly (`/movies/1` vs `/movies?id=1`).
*   **Query Params** allow optional filtering without changing the resource structure.
*   **ResponseEntity** is crucial for error cases. Usage of `return null` is lazy; returning `ResponseEntity.notFound().build()` is professional (Status 404).

### ⚙️ Internal Working
*   **Argument Resolvers**: Spring has specific resolvers (`PathVariableMethodArgumentResolver`, `RequestParamMethodArgumentResolver`) that parse the `HttpServletRequest`, extract the string values, convert them to the target type (Request -> String "1" -> Long 1), and pass them to your method.

### 🙋‍♂️ Interview Q&A

**Q1: What is the difference between `@PathVariable` and `@RequestParam`?**
> **Answer:**
> *   `@PathVariable`: Used for identifying a specific resource (`/users/12`). It is embedded in the URL path.
> *   `@RequestParam`: Used for filtering or sorting logic (`/users?role=admin`). It is appended to the end of the URL.

**Q2: If I send a JSON field `user_name` but my Java field is `userName`, will it map?**
> **Answer:** By default, No. Jackson expects exact verification. You either need to rename the JSON to `userName` or use `@JsonProperty("user_name")` on the Java field to map it correctly.

---

## 8️⃣ Exception Handling in REST

### 💻 Code from Your Project
**File:** `Backend/src/main/java/com/mediaconnect/backend/exception/GlobalExceptionHandler.java`

```java
@RestControllerAdvice // 1. AOP: Advisors for Controllers
public class GlobalExceptionHandler {

    // 2. Handles specific Java Exception
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(404, ex.getMessage());
        return new ResponseEntity<>(error, HttpStatus.NOT_FOUND); // Return 404
    }

    // 3. Fallback for everything else
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGlobal(Exception ex) {
        // ... Log error ...
        return new ResponseEntity<>(error, HttpStatus.INTERNAL_SERVER_ERROR); // Return 500
    }
}
```

### 🔵 Technical Explanation
**Global Exception Handling** using `@ControllerAdvice` allows you to separate error handling logic from successful business logic.
*   **@RestControllerAdvice**: Combines `@ControllerAdvice` + `@ResponseBody`. It intercepts exceptions thrown by *any* controller.
*   **@ExceptionHandler**: specific method that runs when a specific Exception class is thrown.

### 🧐 Why is it used?
Without this, if your `UserService` throws `ResourceNotFoundException`, the user would see a chaotic 500 Internal Server Error trace. With this, they see a clean JSON: `{"status": 404, "message": "User not found"}`. It centralizes error logic, avoiding `try-catch` blocks in every controller method.

### ⚙️ Internal Working
1.  **AOP (Aspect Oriented Programming)**: When a Controller method throws an exception, the DispatcherServlet catches it.
2.  It checks the `ExceptionHandlerExceptionResolver`.
3.  It finds the `@RestControllerAdvice` bean and looks for a matching `@ExceptionHandler` method for that specific Exception type.
4.  It executes that method and returns *its* response instead.

### 🙋‍♂️ Interview Q&A

**Q1: What is the benefit of `@RestControllerAdvice` over `try-catch`?**
> **Answer:**
> 1.  **Cleaner Code**: Controllers focus only on the "Happy Path".
> 2.  **Consistency**: Error responses (structure, naming) are identical across the entire API.
> 3.  **Maintainability**: You change error logic in one place, not 50 places.

**Q2: Can I handle exceptions locally in just one Controller?**
> **Answer:** Yes, by defining an `@ExceptionHandler` method inside that specific Controller class. However, it won't apply to other controllers.

---

## 9️⃣ RESTful Resource Representation & DTOs

### 💻 Standard Industry Code (DTO Pattern)
**Why not use Entities directly?**
If you expose `@Entity User`, you might accidentally expose the `password` field or creating an infinite loop with `List<Order>`.

**The Solution: Data Transfer Object (DTO)**
```java
public class UserDTO {
    // 1. Decoupling: Field names can differ from DB
    @JsonProperty("user_full_name") 
    private String fullName;

    // 2. Security: Hide sensitive data (password is missing)
    private String email;

    // 3. Validation: Ensure data integrity at the entry gate
    @NotNull(message = "Email cannot be null")
    @Email(message = "Invalid email format")
    private String email;
}
```

### 🔵 Technical Explanation
**DTO (Data Transfer Object)** is a design pattern used to transfer data between subsystems (e.g., Backend to Frontend).
*   **Decoupling**: The API schema (`UserDTO`) is independent of the Database Schema (`UserEntity`). You can refactor the DB without breaking the API.
*   **Serialization**: Annotations like `@JsonProperty` (Jackson) control how Java Objects become JSON. `private String name` -> `{"name": "..."}`.
*   **Versioning**: As APIs evolve, you may need `v1/UserDTO` and `v2/UserDTO`.

### 🧐 Customizing JSON (Jackson)
*   `@JsonProperty("name")`: Renames a field in the JSON output.
*   `@JsonIgnore`: Completely omits a sensitive field (like `password` or `internalFlags`) from the JSON.
*   `@JsonInclude(JsonInclude.Include.NON_NULL)`: Skips fields that are `null` to save bandwidth.

### ❓ API Versioning Strategies
1.  **URI Versioning** (Most Common): `/api/v1/users` vs `/api/v2/users`. Easy to see/cache.
2.  **Request Parameter**: `/api/users?version=1`.
3.  **Header Versioning**: `Accept: application/vnd.company.app-v1+json`. "Purest" REST but harder to test in browser.

### 🙋‍♂️ Interview Q&A

**Q1: Why should I never return an Entity from a Controller?**
> **Answer:**
> 1.  **Security**: You might leak sensitive columns (password, salt).
> 2.  **Performance**: Entities often have Lazy Loaded relationships. Serializing them can trigger "N+1 Queries" or `LazyInitializationException`.
> 3.  **Coupling**: Changing a DB column name would break the Frontend.

**Q2: How do you map Entity to DTO?**
> **Answer:**
> *   **Manual**: Simple getters/setters (Good for small projects).
> *   **ModelMapper / MapStruct**: Libraries that automate mapping based on field names (Standard in Enterprise).

---

## 🔟 RESTful CRUD & Optimistic Locking

### 💻 Standard Industry Code (Robust CRUD)

**Handling Concurrent Updates (The "Lost Update" Problem)**
Imagine two admins, Alice and Bob, open the same Movie to edit.
1. Alice reads Title="Avatar".
2. Bob reads Title="Avatar".
3. Alice saves Title="Avatar 2".
4. Bob saves Title="Avatar Remastered".
*Result:* Alice's work is overwritten silently.

**The Solution: Optimistic Locking (`@Version`)**
```java
@Entity
public class Movie {
    @Id
    private Long id;
    
    @Version // The Magic Annotation
    private Integer version; 
}
```

**Controller Handling:**
```java
@PutMapping("/{id}")
public ResponseEntity<?> update(@RequestBody MovieDTO dto) {
    try {
        service.update(dto);
        return ResponseEntity.ok().build();
    } catch (ObjectOptimisticLockingFailureException e) {
        // 409 Conflict
        return ResponseEntity.status(HttpStatus.CONFLICT).body("Data was modified by someone else. Please refresh.");
    }
}
```

### 🔵 Technical Explanation
**CRUD (Create, Read, Update, Delete)** relies on proper HTTP Verb usage:
*   `POST` (Create): **Not Idempotent**. Sending it twice creates two records.
*   `PUT` (Update): **Idempotent**. Replaces the target resource entirely.
*   `PATCH` (Partial Update): **Not Idempotent** (usually). Updates specific fields.

**Optimistic Locking**: Instead of identifying the row by ID (`WHERE id=1`), Hibernate identifies it by ID + Version (`WHERE id=1 AND version=5`).
*   If the version in DB is 5, it succeeds and increments to 6.
*   If someone else bumped it to 6 already, the query (`WHERE id=1 AND version=5`) finds 0 rows. Hibernate throws `OptimisticLockException`.

### 🧐 Input Validation (`@Valid`)
Never trust the client.
*   Add **Jakarta Validation** annotations (`@NotNull`, `@Size`, `@Pattern`) to your DTO fields.
*   Add `@Valid` to your Controller method: `public void create(@Valid @RequestBody UserDTO dto)`.
*   If validation fails, Spring throws `MethodArgumentNotValidException` (400 Bad Request).

### 🙋‍♂️ Interview Q&A

**Q1: What is the difference between PUT and PATCH?**
> **Answer:**
> *   **PUT**: Replaces the *entire* resource. If you send `{ "name": "New" }` and forget the "email" field, email becomes null.
> *   **PATCH**: Updates *only* the fields sent. Useful for huge resources.

**Q2: Explain Optimistic vs Pessimistic Locking.**
> **Answer:**
> *   **Pessimistic**: Locks the database row (`SELECT ... FOR UPDATE`) as soon as you read it. No one else can read/write until you finish. Safe but slow.
> *   **Optimistic**: Allows everyone to read/write, but checks for conflicts *at the very end* using a version number. Better for web apps (Stateless).

---

## 🧠 Quick Recap (Sharpen Your Memory)

| Topic | 🔵 Technical Concept (The Keyword) |
| :--- | :--- |
| **Architectural Style** | **REST** (Representational State Transfer) - NOT a protocol. |
| **Resources** | Identification via **URI**, Manipulation vs **Verbs**. |
| **Spring REST** | `@RestController`, `@RequestBody`, `ResponseEntity`, `Content Negotiation`. |
| **Boot Auto-Config** | **DispatcherServlet**, **Embedded Tomcat**, **Jackson**. |
| **Request Handling** | `@PathVariable` (Identity), `@RequestParam` (Filter), `@RequestBody` (Payload). |
| **Response Handling** | `ResponseEntity` (Status, Headers, Body). |
| **Exception Handling** | `@RestControllerAdvice`, `@ExceptionHandler` (AOP Pattern). |
| **DTO Pattern** | **Decoupling**, **Security**, **@JsonProperty**, **MapStruct**. |
| **Concurrency** | **Optimistic Locking** (`@Version`), **409 Conflict**. |
| **Validation** | `@Valid`, `@NotNull`, **MethodArgumentNotValidException**. |
| **Boot 3 Changes** | **Jakarta EE** (new pkg), **Java 17** (Records), **Native Images**. |

---

## 📚 Ultimate REST Annotation Cheat Sheet

### 🌐 Web & Controller Annotations
**Source Library:** `spring-web`

| Annotation | Use Case | "Senior" Explanation |
| :--- | :--- | :--- |
| `@RestController` | REST API Class | "Meta-annotation for `@Controller` + `@ResponseBody`. Returns data, not views." |
| `@GetMapping` | Read Operation | "Maps HTTP GET. Should be idempotent and safe." |
| `@PostMapping` | Create Operation | "Maps HTTP POST. NOT idempotent. Used for creating resources." |
| `@PutMapping` | Update Operation | "Maps HTTP PUT. Full resource update. Idempotent." |
| `@PatchMapping` | Partial Update | "Maps HTTP PATCH. Updates only specific fields. Not always idempotent." |
| `@DeleteMapping`| Delete Operation | "Maps HTTP DELETE. Idempotent." |
| `@PathVariable` | URI Template | "Extracts values from the URI path (`/users/{id}`). Mandatory by default." |
| `@RequestParam` | Query Params | "Extracts values from URL query string (`/users?page=1`). Optional by default." |
| `@RequestBody` | JSON Payload | "Triggers Jackson to deserialize JSON into a Java POJO." |
| `@ResponseStatus`| Status Code | "Declaratively sets the HTTP Status code (e.g., `@ResponseStatus(HttpStatus.CREATED)`)." |
| `@CrossOrigin` | CORS Config | "Directly on the controller/method to allow calling from a different domain/port." |
