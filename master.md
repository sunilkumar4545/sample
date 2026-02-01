# SPRINT 2: Developer Foundations & Interview Guide

This document explains core software engineering principles, the framework we use, and our build tools. It connects theoretical concepts directly to the **MediaConnect** codebase for interview preparation.

---

## 1. SOLID Principles of Object-Oriented Programming

SOLID is an acronym for five design principles intended to make software designs more understandable, flexible, and maintainable.

### 1️⃣ Single Responsibility Principle (SRP)
**Concept**: A class should have one, and only one, reason to change. It should handle a single part of the functionality.

**In MediaConnect**:
- **Explanation**: We separate concerns distinctively. We do not put business logic in Controllers, and we do not put database queries in Services.
- **Project Structure**:
    - **`UserController.java`**: Responsible ONLY for handling HTTP requests (GET/POST), parsing inputs, and sending HTTP responses. It delegates logic.
    - **`UserServiceImpl.java`**: Responsible ONLY for business logic (updating profiles, checking subscriptions).
    - **`UserRepository.java`**: Responsible ONLY for database interaction.
- **Interview Answer**: "In our project, we follow SRP by strictly separating layers. For example, `UserController` only handles web traffic, while `UserServiceImpl` contains the actual business rules. If I need to change how we validate a subscription, I only touch the Service, not the Controller."

### 2️⃣ Open-Closed Principle (OCP)
**Concept**: Objects or entities should be open for extension but closed for modification.

**In MediaConnect**:
- **Explanation**: We use Interfaces to allow behavior to be extended without modifying existing code.
- **Project Concept**: `UserService` is an interface. Currently, we have `UserServiceImpl`. If we wanted to add a `TestUserServiceImpl` for testing or a `PremiumUserServiceImpl`, we could do so without changing the `UserController` code that relies on the interface.
- **Code Snippet**:
    ```java
    // START: Interface definition (Open for extension)
    public interface UserService {
        UserDTO subscribe(String email, String plan);
    }
    
    // START: Implementation (One extension)
    @Service
    public class UserServiceImpl implements UserService { ... }
    ```

### 3️⃣ Liskov Substitution Principle (LSP)
**Concept**: Subtypes must be substitutable for their base types without breaking the application.

**In MediaConnect**:
- **Explanation**: Any class that implements `UserService` must behave correctly when used by `UserController`. We ensure our implementation (`UserServiceImpl`) returns the expected `UserDTO` and handles errors gracefully (throwing custom exceptions like `ResourceNotFoundException` rather than crashing), fulfilling the contract expected by the Controller.

### 4️⃣ Interface Segregation Principle (ISP)
**Concept**: Clients should not be forced to depend on interfaces they do not use.

**In MediaConnect**:
- **Explanation**: Instead of one massive `GeneralService` interface, we split them into specific ones:
    - `UserService`: User management.
    - `MovieService`: Movie catalog.
    - `AuthService`: Authentication.
- **Why**: This prevents the `MovieController` from knowing about unrelated methods like `subscribe()`.

### 5️⃣ Dependency Inversion Principle (DIP)
**Concept**: High-level modules should not depend on low-level modules. Both should depend on abstractions (interfaces).

**In MediaConnect**:
- **Explanation**: We use **Dependency Injection**. The `UserController` does not use `new UserServiceImpl()`. Instead, it asks Spring to provide *some* implementation of `UserService`.
- **Code Snippet**:
    ```java
    @RestController
    public class UserController {
        @Autowired // Spring injects the dependency here
        private UserService userService; // Depends on Interface (Abstraction), not Class
    }
    ```

---

## 2. Spring Framework Overview

### What is Spring?
Spring is a powerful, lightweight framework for Java heavily used for building Enterprise-level applications. It simplifies development by managing object lifecycles and interactions.

### Inversion of Control (IoC) & Dependency Injection (DI)
- **IoC**: Instead of you creating objects (`new User()`), you give that control to the Spring Framework (the Container).
- **DI**: The Container "injects" the necessary dependencies into your objects at runtime.

**In MediaConnect**:
- **Annotation**: `@Autowired` is the key here.
- **How it works**:
    1.  **Instantiation**: Spring detects `@Service` on `UserServiceImpl` and creates an instance (Bean) in its memory (IoC).
    2.  **Injection**: When Spring creates `UserController`, it sees it needs `UserService`. It looks in its memory, finds the `UserServiceImpl` bean, and injects it (DI).

### Spring Modules Used

1.  **Spring Core**: The heart of the framework handling IoC and DI.
2.  **Spring MVC (Web)**:
    - **Usage**: `spring-boot-starter-web` dependency in `pom.xml`.
    - **Files**: `UserController.java`, `MovieController.java`.
    - **Concept**: Handles REST API requests (`@GetMapping`, `@PostMapping`).
3.  **Spring Data JPA (ORM)**:
    - **Usage**: `spring-boot-starter-data-jpa`.
    - **Files**: `UserRepository.java`.
    - **Concept**: Maps Java objects (`User` entity) to Database tables (`users` table). Eliminates boilerplate SQL code.
4.  **Spring Security**:
    - **Usage**: `spring-boot-starter-security`.
    - **Files**: `JwtFilter.java`, `SecurityConfig.java`.
    - **Concept**: Handles Authentication (Login) and Authorization (Role checks).
5.  **Spring AOP (Aspect Oriented Programming)**:
    - **Usage**: While we may not write custom Aspects, Spring uses this internally for things like `@Transactional` (rolling back DB on error) and `@RestControllerAdvice` (global exception handling).

### Benefits of Spring
1.  **Lightweight & Fast**: POJO-based implementation.
2.  **Boilerplate Reduction**: No manual transaction management or connection pooling code.
3.  **Testability**: Easy to mock dependencies for Unit Testing.

---

## 3. Maven Build Tool

### Introduction
Maven is a build automation tool used primarily for Java projects. It manages dependencies (libraries), builds the code, and packages it.

### The `pom.xml` File
**Project Object Model**: This is the configuration file for Maven. It tells Maven what libraries our project needs.

**Key Dependencies in MediaConnect**:
1.  **`spring-boot-starter-web`**:
    - **Why**: Gives us a Tomcat web server and libraries to build REST APIs.
2.  **`spring-boot-starter-data-jpa`**:
    - **Why**: Provides Hibernate and JPA for database connection.
3.  **`mysql-connector-j`**:
    - **Why**: Examples the Java application how to talk to a MySQL database specifically.
4.  **`jjwt-api`, `jjwt-impl`**:
    - **Why**: Used for creating and verifying JSON Web Tokens (JWT) for secure login.
5.  **`springdoc-openapi...`**:
    - **Why**: Generates the Swagger UI page for API testing.

---

## 4. IoC Container & Bean Configuration (Deep Dive)

### The IoC Container
**Analogy**: Think of the IoC Container as a "Factory" or "Hotel User". You don't build the furniture in your room; you request a room, and the hotel (Spring) provides it fully furnished.
**Technical**: The `ApplicationContext` is the sophisticated IoC container we use. It is a superset of the simpler `BeanFactory`. In MediaConnect, `SpringApplication.run()` boots up this container.

### Configuring Beans

#### 1. Annotation-Based Configuration (Mainly used in MediaConnect)
This is the modern way. We just tag our classes, and Spring finds them.
-   **`@Component`**: Generic Spring managed component.
-   **`@Service`**: Specialized component for Business Logic (e.g., `UserServiceImpl`).
-   **`@Repository`**: Specialized component for Data Access (e.g., `UserRepository`).
-   **`@Controller`**: Specialized component for Web Requests (e.g., `UserController`).

**In Project**:
```java
@Service // Tells Spring: "Manage this class!"
public class UserServiceImpl implements UserService { ... }
```

#### 2. Java-Based Configuration (`@Configuration` & `@Bean`)
Used when we need to configure third-party classes or complex setups that aren't just "one class".
**In MediaConnect**: See **`SecurityConfig.java`**.
```java
@Configuration
public class SecurityConfig {
    @Bean // Explicitly telling Spring to create this object
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```
*Why?* We cannot add `@Component` to the source code of `BCryptPasswordEncoder` (it's a library), so we create it manually via `@Bean`.

#### 3. XML Configuration
**Status**: Legacy. **Not used** in MediaConnect.
*Interview Note*: "We use completely Annotation and Java-configuration based setup, avoiding verbose XML files."

---

## 5. Dependency Injection (DI) Types

### 1. Field Injection (Used Heavily)
Easiest to write, but sometimes criticized for making testing harder (without reflection).
**In MediaConnect**:
```java
@Autowired
private UserRepository userRepository;
```
*Where?* `UserServiceImpl.java`, `JwtFilter.java`.

### 2. Constructor Injection (Best Practice)
Injects dependencies via the constructor. Ensures the object cannot be created without them.
**In MediaConnect**:
```java
// Inside SecurityConfig.java
public SecurityConfig(JwtFilter jwtFilter, CustomUserDetailsService userDetailsService) {
    this.jwtFilter = jwtFilter;
    this.userDetailsService = userDetailsService;
}
```

### 3. Setter Injection
Dependencies are set via public setter methods. Useful for optional dependencies.
*Status*: Not explicitly used in our core logic, but supported.

### Autowiring & Qualifiers
-   **`@Autowired`**: "Spring, find a bean that matches this type and give it to me."
-   **`@Qualifier`**: "Spring, I have two implementations of `UserService` (e.g., `RegularUserService`, `PremiumUserService`), please give me the 'Premium' one."
    -   *Project Note*: We currently have only one implementation (`UserServiceImpl`), so we don't need `@Qualifier`. If we added another, we would fail startup without it.

---

## 6. Aspect-Oriented Programming (AOP)

### Concepts
AOP allows us to separate "Cross-Cutting Concerns" (things that affect the whole app, like logging, security, error handling) from the business logic.

-   **Aspect**: The modularization of a concern (e.g., "Logging" or "Error Handling").
-   **Advice**: The actual action to be taken (e.g., "Log this message" or "Return this JSON error").
-   **Pointcut**: A predicate that matches *where* advice should be applied (e.g., "All methods in Controller package").

### In MediaConnect Project
We use AOP primarily for **Global Exception Handling**.

**File**: `GlobalExceptionHandler.java`
-   **The Aspect**: `@RestControllerAdvice` (A specialized Aspect for web controllers).
-   **The Pointcut**: "All Controllers in the application".
-   **The Advice**: `@ExceptionHandler(ResourceNotFoundException.class)`.

**Code Snippet**:
```java
@RestControllerAdvice // Aspect targeting all Controllers
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class) // Pointcut (When this error happens...)
    public ResponseEntity<ErrorResponse> handleResourceNotFound(ResourceNotFoundException ex) {
        // Advice (Do this...)
        ErrorResponse error = new ErrorResponse(404, ex.getMessage());
        return new ResponseEntity<>(error, HttpStatus.NOT_FOUND);
    }
}
```

*Benefit*: We don't write try-catch blocks in every single Controller method. The Aspect catches mistakes globally.

---

## 7. MVC and ORM (Model-View-Controller)

### Architecture Flow
1.  **Request**: Enters via **Controller** (`UserController`).
2.  **Processing**: Controller calls **Service** for logic.
3.  **Data**: Service uses **Repository** (ORM) to specific Data.
4.  **Response**: Controller returns **UserDTO** (JSON).

### Layers in MediaConnect
-   **Controller Layer**: `UserController.java`
    -   Handles `@GetMapping`, `@PostMapping`. Validates inputs.
-   **Model Layer (Entity)**: `User.java`
    -   Represents the Database Table.
-   **View Layer**: **Implicit**. Since we are a REST API, our "View" is JSON data consumed by the Angular Frontend.
    -   *Technically, `UserDTO` acts as the View Model.*

### ORM (Object Relational Mapping)
We use **Hibernate** (via Spring Data JPA).
*Concept*: Instead of writing complex SQL queries like `SELECT * FROM users WHERE email = ?`, we write Java methods.
**File**: `UserRepository.java`
```java
public interface UserRepository extends JpaRepository<User, Long> {
    User findByEmail(String email); // Hibernate generates the SQL automatically!
}
```

---

## 8. Spring Boot & Auto-Configuration

### Why Spring Boot?
Spring Framework is powerful but requires huge setup. Spring Boot is "Opinionated Spring" - it makes decisions for you so you can start coding fast.

### Auto-Configuration
Spring Boot looks at your `pom.xml` (classpath) and configures beans automatically.
**In MediaConnect**:
-   We added `spring-boot-starter-web` -> Boot automatically configures internal **Tomcat Server**.
-   We added `spring-boot-starter-data-jpa` + `mysql-connector` -> Boot automatically attempts to connect to a DB using generic settings.
-   **We Override**: We provide specifics in `application.properties` (URL, Username, Password), and Boot uses those values to configure the DataSource.

**File**: `MediaconnectApplication.java`
```java
@SpringBootApplication // (Includes @Configuration, @EnableAutoConfiguration, @ComponentScan)
public class MediaconnectApplication {
    public static void main(String[] args) {
        SpringApplication.run(MediaconnectApplication.class, args);
    }
}
```

---

## 9. Reactive Programming (Conceptual)

### Overview
*Note: MediaConnect uses the standard "Servlet Stack" (Blocking I/O), not Reactive.*

**Reactive Web Framework (Spring WebFlux)** is designed for high concurrency with a small number of threads. It uses **Non-blocking I/O**.
-   **Mono**: Expected 0 or 1 result (like `Optional`).
-   **Flux**: Expected 0 to N results (like `Stream` or `List`).

**Why we don't use it**:
-   Our application relies on `MySQL` (JDBC), which is traditionally blocking.
-   Reactive stacks require everything to be non-blocking (DB drivers, etc.).
-   For standard CRUD apps (like ours), the complexity of Reactive is unnecessary.

---

## 10. Frontend & Backend Interactions (Visual Guide)

### Flow 1: Subscription Logic
**Scenario**: User upgrades to "Premium".

```text
[Frontend: ProfileComponent]
       |
       | (Angular Form)
   User selects "Premium" -> Clicks "Pay"
       |
  [UserService.ts] 
       | http.post('/api/users/subscribe')
       | + Header: { "Authorization": "Bearer xyMz..." }
       v
---------------------------------------------------------
                     INTERNET
---------------------------------------------------------
       v
[Backend: Port 8081]
       |
  [JwtFilter.java] -> Checks "Bearer xyMz..."
       |           -> Valid? Set SecurityContext. Proceed.
       |           -> Invalid? Return 403 Forbidden.
       |
  [UserController.java] -> subscribe(email, plan)
       |
  [UserServiceImpl.java] (Business Logic)
       | 1. userRepo.findByEmail(email)
       | 2. user.setPlan("Premium")
       | 3. user.setExpiry(Now + 30 Days)
       | 4. userRepo.save(user)
       |
  [Database] -> UPDATE users SET plan='Premium'...
       |
  [Response] -> 200 OK { "currentPlan": "Premium", ... } (JSON)
       v
[Frontend] -> Shows "Subscription Successful!"
```

### Flow 2: Form Handling & Validation
**Frontend**:
-   **Visual**: Red border on input fields if invalid.
-   **Logic**: `FormGroup.valid` checks.
-   **File**: `login.component.ts`.

**Backend**:
-   **Logic**: Data integrity checks.
-   **Example**: `UserServiceImpl.updateUser` checks if `email` is already taken.
-   **Exception**: If taken, throws `RuntimeException`.
-   **Response**: `GlobalExceptionHandler` converts this to `500/400 Error`.
-   **Result**: Frontend sees error, displays "Email already in use".

### Best Ways to Remember for Interview
1.  **Map Files**: Always link a concept to a file you know.
    -   *Dependency Injection* -> `UserController` (Where `@Autowired` is).
    -   *AOP* -> `GlobalExceptionHandler`.
    -   *Configuration* -> `SecurityConfig`.
2.  **Tell a Story**: "First the request hits the Controller, then it asks the Service for help, which asks the Repository to get data."
3.  **Keywords**:
    -   **IoC**: "Spring manages the objects."
    -   **DI**: "Spring gives me the objects."
    -   **AOP**: "Handling errors in one place."


# SPRINT 3: JPA & Data Persistence Guide
This document serves as both a learning module and an interview preparation guide for **Spring Data JPA**. Each topic explains the concept, its usage in our **MediaConnect** project, and provides technical depth for interviews.

---

## 1. Overview of Spring Data JPA
### 📘 Topic Explanation
- **Simple Concept**: Imagine you want to talk to a database (MySQL). Normally, you need to learn SQL (`SELECT * FROM users`). Spring Data JPA is a "Translator" that lets you talk in Java code (`userRepository.findAll()`), and it automatically writes the SQL for you.
- **Technical**: Spring Data JPA is an abstraction layer on top of **JPA (Java Persistence API)**. It reduces boilerplate code by providing standard interfaces (like `JpaRepository`) for CRUD operations. It uses **Hibernate** as the underlying ORM provider to interact with the DB.
- **Problem it Solves**: Eliminates repetitive SQL and JDBC code (opening connections, handling result sets).

### 🏗️ Project Usage
- **Module**: Entire Backend Data Layer.
- **Why**: We need to save Users, Movies, and Watch History. Instead of writing 100 lines of SQL, we just extend `JpaRepository`.

### 🗂️ File Mapping
- **Entity**: `User.java`, `Movie.java`
- **Repository**: `UserRepository.java`, `MovieRepository.java`

### 💻 Code Snippet
```java
// We just create an interface, and Spring implements the logic!
public interface UserRepository extends JpaRepository<User, Long> {
    // Spring automatically knows this means: SELECT * FROM users WHERE email = ?
    User findByEmail(String email); 
}
```

---

## 2. Relationship between JPA and Spring Data JPA
### 📘 Topic Explanation
- **Simple**: JPA is the "Rule Book" (Specification). Spring Data JPA is the "Helper Robot" that reads the book and does the work for you. Hibernate is the "Engine" inside the robot.
- **Technical**:
    - **JPA**: Standard API for ORM in Java (defines annotations like `@Entity`, `@Id`).
    - **Hibernate**: The most popular implementation of JPA.
    - **Spring Data JPA**: A framework that adds extra magic (Repositories) on top of JPA/Hibernate.

### 🏗️ Project Usage
- **Evidence**: We use JPA annotations (`@Entity`) in `User.java` but run them using Spring's Repository interface.

### 🧠 Interview Tip
> "JPA is the specification (Interface), Hibernate is the implementation (Class), and Spring Data JPA is the utility layer that simplifies using them."

---

## 3. Configuring Database Connection (application.properties)
### 📘 Topic Explanation
- **Simple**: How does the app know *which* database to connect to? Use a config file.
- **Technical**: We define the **JDBC URL**, **Username**, **Password**, and **Driver** in `application.properties`. Spring Boot's "Auto-Configuration" reads this and sets up the `DataSource` bean.

### 🗂️ File Mapping
- **File**: `src/main/resources/application.properties`

### 💻 Code Snippet
```properties
# Connect to MySQL on Localhost
spring.datasource.url=jdbc:mysql://localhost:3306/mediaconnect?createDatabaseIfNotExist=true
spring.datasource.username=root
spring.datasource.password=root
# ORM Settings
spring.jpa.hibernate.ddl-auto=update  
spring.jpa.show-sql=true
```
-   `ddl-auto=update`: Automatically creates/updates tables to match our Java classes.

---

## 4. JPA Entities & Annotations (@Entity, @Table, @Id)
### 📘 Topic Explanation
- **Simple**: An "Entity" is just a Java Class that looks exactly like a Database Table.
- **Technical**:
    -   `@Entity`: Marks class as a DB object.
    -   `@Table`: Customizes table name (optional).
    -   `@Id`: Primary Key.
    -   `@GeneratedValue`: Auto-increment (Identity column).

### 🏗️ Project Usage
-   **User**: `User.java` -> Table `users`.
-   **Movie**: `Movie.java` -> Table `movies`.

### 💻 Code Snippet (User.java)
```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id; // Primary Key (1, 2, 3...)

    @Column(unique = true, nullable = false)
    private String email; // UNIQUE constraint
}
```

---

## 5. Spring Data Repositories & Derived Queries
### 📘 Topic Explanation
- **Simple**: "Derived Queries" mean you just name your method in English, and Spring writes the SQL. "Find By Email" -> `findByEmail`.
- **Technical**: Spring parses method names.
    -   `findByTitleContaining(String t)` -> `SELECT ... WHERE title LIKE %t%`
    -   `existsByEmail(String e)` -> `SELECT COUNT(*) > 0 ... WHERE email = e`

### 🏗️ Project Usage
-   **Search**: In `MovieRepository.java`, we use `findByTitleContainingIgnoreCase`.
-   **Login**: In `UserRepository.java`, we use `findByEmail`.

### 💻 Code Snippet (MovieRepository.java)
```java
public interface MovieRepository extends JpaRepository<Movie, Long> {
    // Search movies by title (Case Insensitive)
    List<Movie> findByTitleContainingIgnoreCase(String title);
    
    // Filter by genre
    List<Movie> findByGenresContainingIgnoreCase(String genre);
}
```

---

## 6. Custom Queries (@Query) & Native Queries
### 📘 Topic Explanation
- **Simple**: Sometimes "Derived" names get too long or complex. We use `@Query` to write the exact SQL (Native) or JPQL (Java-based SQL) we want.
- **Technical**:
    -   **JPQL**: Queries Objects (`SELECT u FROM User u`).
    -   **Native**: Queries Tables (`SELECT * FROM users`). Good for DB-specific syntax.

### 🏗️ Project Usage
-   **Trending Movies**: We need complex logic (joining `watch_history`, counting views, grouping). Derived queries can't do this.
-   **File**: `MovieRepository.java`

### 💻 Code Snippet
```java
// Find Top 5 movies by counting how many times they appear in WatchHistory
@Query(value = "SELECT m.* FROM movies m " +
               "LEFT JOIN watch_history wh ON m.id = wh.movie_id " +
               "GROUP BY m.id " +
               "ORDER BY COUNT(wh.id) DESC " +
               "LIMIT 5", nativeQuery = true)
List<Movie> findTop5ByOrderByViewsDesc();
```

---

## 7. Pagination & Sorting
### 📘 Topic Explanation
- **Simple**: If you have 1,000,000 movies, you don't load them all. You load "Page 1" (10 items).
- **Technical**: We pass a `Pageable` object to the repository. The result is a `Page<T>` object containing the data + metadata (total pages, total elements).

### 🏗️ Project Usage
-   *Can be extended further*: Currently we use `LIMIT 40` in our native queries. Ideally, we would update `MovieController` to accept `page` and `size` parameters.

### 💻 Code Snippet
```java
// In Repository
Page<Movie> findAll(Pageable pageable);

// In Service
Pageable pageRequest = PageRequest.of(0, 10, Sort.by("title").ascending());
Page<Movie> firstPage = movieRepository.findAll(pageRequest);
```

---

## 8. Projections & DTOs
### 📘 Topic Explanation
- **Simple**: Sometimes you don't want the *entire* User object (password, history, etc.). You just want their Name and Email. A "Projection" selects only that subset.
- **Technical**: We can use **Interface-based Projections** (Getters only) or **Class-based Projections** (DTOs used in JPQL).

### 🏗️ Project Usage
-   **Analytics**: We need a summary of movie views, not the full movie object + full watch history object.
-   **File**: `WatchHistoryRepository.java`

### 💻 Code Snippet (Constructor Expression)
```java
// We construct a specific 'AnalyticsResponse' object directly from the DB query
@Query("SELECT new com.mediaconnect.backend.dto.response.AnalyticsResponse(" +
       "m.title, COUNT(wh.id)) " +
       "FROM Movie m JOIN WatchHistory wh ON ... GROUP BY m.id")
List<AnalyticsResponse> findEngagementAnalytics();
```

---

## 9. Frontend → Backend Flow (Data Journey)

### Scenario: Searching for a Movie
1.  **Frontend (Angular)**: User types "Bat" in search bar.
    -   File: `home.component.ts`
    -   Call: `movieService.searchMovies("Bat")` -> HTTP GET `/api/movies/search?query=Bat`
2.  **Controller**: `MovieController` receives `String query`.
3.  **Service**: `MovieServiceImpl` calls `movieRepo.findByTitleContainingIgnoreCase("Bat")`.
4.  **Repository**: Spring generates: `SELECT * FROM movies WHERE UPPER(title) LIKE '%BAT%'`.
5.  **Database**: Returns "Batman", "Battle Royale".
6.  **Return Trip**: DB -> Entity -> Service -> Controller -> JSON -> Frontend.

---

## 10. Interview Answer Templates

### Q: "What is the difference between specific `findBy` methods and `@Query`?"
> **Answer**: "I use `findBy...` (derived queries) for simple lookups like finding a user by email because it's clean and requires no SQL. However, for complex reports like 'Top 5 Trending Movies' which involve Joins and Group By clauses, I use `@Query` to write optimized Native SQL."

### Q: "How do you handle paginated data?"
> **Answer**: "In Spring Data JPA, I pass a `Pageable` object (created via `PageRequest.of(page, size)`) to the repository method. This returns a `Page` object that includes the content and metadata like total pages, which helps the frontend render the 'Next' button correctly."

### Q: "Does your project use Hibernate directly?"
> **Answer**: "We use Spring Data JPA, which uses Hibernate internally as the JPA implementation. We rely on Hibernate for the actual ORM features (mapping Java objects to tables) and DDL generation (`ddl-auto=update`), but our code interacts mainly with the Spring Repository interfaces."


# SPRINT 3: JPA & Data Persistence Guide
This document serves as both a learning module and an interview preparation guide for **Spring Data JPA**. Each topic explains the concept, its usage in our **MediaConnect** project, and provides technical depth for interviews.

---

## 1. Overview of Spring Data JPA
### 📘 Topic Explanation
- **Simple Concept**: Imagine you want to talk to a database (MySQL). Normally, you need to learn SQL (`SELECT * FROM users`). Spring Data JPA is a "Translator" that lets you talk in Java code (`userRepository.findAll()`), and it automatically writes the SQL for you.
- **Technical**: Spring Data JPA is an abstraction layer on top of **JPA (Java Persistence API)**. It reduces boilerplate code by providing standard interfaces (like `JpaRepository`) for CRUD operations. It uses **Hibernate** as the underlying ORM provider to interact with the DB.
- **Problem it Solves**: Eliminates repetitive SQL and JDBC code (opening connections, handling result sets).

### 🏗️ Project Usage
- **Module**: Entire Backend Data Layer.
- **Why**: We need to save Users, Movies, and Watch History. Instead of writing 100 lines of SQL, we just extend `JpaRepository`.

### 🗂️ File Mapping
- **Entity**: `User.java`, `Movie.java`
- **Repository**: `UserRepository.java`, `MovieRepository.java`

### 💻 Code Snippet
```java
// We just create an interface, and Spring implements the logic!
public interface UserRepository extends JpaRepository<User, Long> {
    // Spring automatically knows this means: SELECT * FROM users WHERE email = ?
    User findByEmail(String email); 
}
```

---

## 2. Relationship between JPA and Spring Data JPA
### 📘 Topic Explanation
- **Simple**: JPA is the "Rule Book" (Specification). Spring Data JPA is the "Helper Robot" that reads the book and does the work for you. Hibernate is the "Engine" inside the robot.
- **Technical**:
    - **JPA**: Standard API for ORM in Java (defines annotations like `@Entity`, `@Id`).
    - **Hibernate**: The most popular implementation of JPA.
    - **Spring Data JPA**: A framework that adds extra magic (Repositories) on top of JPA/Hibernate.

### 🏗️ Project Usage
- **Evidence**: We use JPA annotations (`@Entity`) in `User.java` but run them using Spring's Repository interface.

### 🧠 Interview Tip
> "JPA is the specification (Interface), Hibernate is the implementation (Class), and Spring Data JPA is the utility layer that simplifies using them."

---

## 3. Configuring Database Connection (application.properties)
### 📘 Topic Explanation
- **Simple**: How does the app know *which* database to connect to? Use a config file.
- **Technical**: We define the **JDBC URL**, **Username**, **Password**, and **Driver** in `application.properties`. Spring Boot's "Auto-Configuration" reads this and sets up the `DataSource` bean.

### 🗂️ File Mapping
- **File**: `src/main/resources/application.properties`

### 💻 Code Snippet
```properties
# Connect to MySQL on Localhost
spring.datasource.url=jdbc:mysql://localhost:3306/mediaconnect?createDatabaseIfNotExist=true
spring.datasource.username=root
spring.datasource.password=root
# ORM Settings
spring.jpa.hibernate.ddl-auto=update  
spring.jpa.show-sql=true
```
-   `ddl-auto=update`: Automatically creates/updates tables to match our Java classes.

---

## 4. JPA Entities & Annotations (@Entity, @Table, @Id)
### 📘 Topic Explanation
- **Simple**: An "Entity" is just a Java Class that looks exactly like a Database Table.
- **Technical**:
    -   `@Entity`: Marks class as a DB object.
    -   `@Table`: Customizes table name (optional).
    -   `@Id`: Primary Key.
    -   `@GeneratedValue`: Auto-increment (Identity column).

### 🏗️ Project Usage
-   **User**: `User.java` -> Table `users`.
-   **Movie**: `Movie.java` -> Table `movies`.

### 💻 Code Snippet (User.java)
```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id; // Primary Key (1, 2, 3...)

    @Column(unique = true, nullable = false)
    private String email; // UNIQUE constraint
}
```

---

## 5. Spring Data Repositories & Derived Queries
### 📘 Topic Explanation
- **Simple**: "Derived Queries" mean you just name your method in English, and Spring writes the SQL. "Find By Email" -> `findByEmail`.
- **Technical**: Spring parses method names.
    -   `findByTitleContaining(String t)` -> `SELECT ... WHERE title LIKE %t%`
    -   `existsByEmail(String e)` -> `SELECT COUNT(*) > 0 ... WHERE email = e`

### 🏗️ Project Usage
-   **Search**: In `MovieRepository.java`, we use `findByTitleContainingIgnoreCase`.
-   **Login**: In `UserRepository.java`, we use `findByEmail`.

### 💻 Code Snippet (MovieRepository.java)
```java
public interface MovieRepository extends JpaRepository<Movie, Long> {
    // Search movies by title (Case Insensitive)
    List<Movie> findByTitleContainingIgnoreCase(String title);
    
    // Filter by genre
    List<Movie> findByGenresContainingIgnoreCase(String genre);
}
```

---

## 6. Custom Queries (@Query) & Native Queries
### 📘 Topic Explanation
- **Simple**: Sometimes "Derived" names get too long or complex. We use `@Query` to write the exact SQL (Native) or JPQL (Java-based SQL) we want.
- **Technical**:
    -   **JPQL**: Queries Objects (`SELECT u FROM User u`).
    -   **Native**: Queries Tables (`SELECT * FROM users`). Good for DB-specific syntax.

### 🏗️ Project Usage
-   **Trending Movies**: We need complex logic (joining `watch_history`, counting views, grouping). Derived queries can't do this.
-   **File**: `MovieRepository.java`

### 💻 Code Snippet
```java
// Find Top 5 movies by counting how many times they appear in WatchHistory
@Query(value = "SELECT m.* FROM movies m " +
               "LEFT JOIN watch_history wh ON m.id = wh.movie_id " +
               "GROUP BY m.id " +
               "ORDER BY COUNT(wh.id) DESC " +
               "LIMIT 5", nativeQuery = true)
List<Movie> findTop5ByOrderByViewsDesc();
```

---

## 7. Pagination & Sorting
### 📘 Topic Explanation
- **Simple**: If you have 1,000,000 movies, you don't load them all. You load "Page 1" (10 items).
- **Technical**: We pass a `Pageable` object to the repository. The result is a `Page<T>` object containing the data + metadata (total pages, total elements).

### 🏗️ Project Usage
-   *Can be extended further*: Currently we use `LIMIT 40` in our native queries. Ideally, we would update `MovieController` to accept `page` and `size` parameters.

### 💻 Code Snippet
```java
// In Repository
Page<Movie> findAll(Pageable pageable);

// In Service
Pageable pageRequest = PageRequest.of(0, 10, Sort.by("title").ascending());
Page<Movie> firstPage = movieRepository.findAll(pageRequest);
```

---

## 8. Projections & DTOs
### 📘 Topic Explanation
- **Simple**: Sometimes you don't want the *entire* User object (password, history, etc.). You just want their Name and Email. A "Projection" selects only that subset.
- **Technical**: We can use **Interface-based Projections** (Getters only) or **Class-based Projections** (DTOs used in JPQL).

### 🏗️ Project Usage
-   **Analytics**: We need a summary of movie views, not the full movie object + full watch history object.
-   **File**: `WatchHistoryRepository.java`

### 💻 Code Snippet (Constructor Expression)
```java
// We construct a specific 'AnalyticsResponse' object directly from the DB query
@Query("SELECT new com.mediaconnect.backend.dto.response.AnalyticsResponse(" +
       "m.title, COUNT(wh.id)) " +
       "FROM Movie m JOIN WatchHistory wh ON ... GROUP BY m.id")
List<AnalyticsResponse> findEngagementAnalytics();
```

---

## 9. Frontend → Backend Flow (Data Journey)

### Scenario: Searching for a Movie
1.  **Frontend (Angular)**: User types "Bat" in search bar.
    -   File: `home.component.ts`
    -   Call: `movieService.searchMovies("Bat")` -> HTTP GET `/api/movies/search?query=Bat`
2.  **Controller**: `MovieController` receives `String query`.
3.  **Service**: `MovieServiceImpl` calls `movieRepo.findByTitleContainingIgnoreCase("Bat")`.
4.  **Repository**: Spring generates: `SELECT * FROM movies WHERE UPPER(title) LIKE '%BAT%'`.
5.  **Database**: Returns "Batman", "Battle Royale".
6.  **Return Trip**: DB -> Entity -> Service -> Controller -> JSON -> Frontend.

---

## 10. Interview Answer Templates

### Q: "What is the difference between specific `findBy` methods and `@Query`?"
> **Answer**: "I use `findBy...` (derived queries) for simple lookups like finding a user by email because it's clean and requires no SQL. However, for complex reports like 'Top 5 Trending Movies' which involve Joins and Group By clauses, I use `@Query` to write optimized Native SQL."

### Q: "How do you handle paginated data?"
> **Answer**: "In Spring Data JPA, I pass a `Pageable` object (created via `PageRequest.of(page, size)`) to the repository method. This returns a `Page` object that includes the content and metadata like total pages, which helps the frontend render the 'Next' button correctly."

### Q: "Does your project use Hibernate directly?"
> **Answer**: "We use Spring Data JPA, which uses Hibernate internally as the JPA implementation. We rely on Hibernate for the actual ORM features (mapping Java objects to tables) and DDL generation (`ddl-auto=update`), but our code interacts mainly with the Spring Repository interfaces."



# SPRINT 3: Spring REST & API Design (Deep Dive Edition)

This document is an **comprehensive** interview guide for **Spring REST** using **Spring Boot 3**. It expands on core concepts, advanced features, and how they apply (or could apply) to the **MediaConnect** project.

---

## 1. Introduction to Spring REST & Spring Boot 3

### 📘 Topic Explanation
- **RESTful Architecture**: An architectural style where resources (Users, Movies) are accessed via standard HTTP methods (GET, POST). It is "Stateless" (no session memory between requests).
- **Spring REST**: The part of the Spring Framework (Spring MVC) that focuses on building REST APIs. It uses annotations like `@RestController` to simplify handling HTTP requests.
- **Benefits of Spring Boot**:
    -   **Embedded Server**: Comes with Tomcat built-in. No need to install a separate web server.
    -   **Starters**: `spring-boot-starter-web` includes everything needed (Jackson, Tomcat, MVC).
    -   **Auto-Configuration**: Automatically sets up the DispatcherServlet.

### 🚀 What's New in Spring Boot 3?
1.  **Java 17 Baseline**: You *must* use Java 17 or higher (we use Java 17).
2.  **Jakarta EE**: The package namespace changed from `javax.*` to `jakarta.*`.
    -   *Example*: `javax.persistence.Entity` is now `jakarta.persistence.Entity`.
3.  **Observability**: Better built-in support for tracing (Micrometer) to monitor API latency.
4.  **Native Image Support**: Can compile Java code to a native binary (via GraalVM) for instant startup.

---

## 2. Building a Simple REST Controller

### 📘 Topic Explanation
- **@RestController**: The main annotation. It tells Spring "This class handles web requests and returns data (JSON)".
    -   It is a shortcut for `@Controller` + `@ResponseBody`.
-   **@RequestMapping**: Defines the "Base URL" for the controller (e.g., `/api/movies`).

### 🏗️ Project Usage
-   **File**: `MovieController.java`
-   **Methods**:
    -   `GET`: Fetch data.
    -   `POST`: Create data.
    -   `PUT`: Update data.
    -   `DELETE`: Remove data.

### 💻 Code Snippet
```java
@RestController
@RequestMapping("/api/movies")
public class MovieController {
    
    @Autowired private MovieService movieService;

    // HTTP GET /api/movies
    @GetMapping 
    public List<Movie> getAll() {
        return movieService.getAllMovies();
    }
}
```

---

## 3. Request & Response Handling (Deep Dive)

### 📘 Handling Inputs
1.  **Path Variables (`@PathVariable`)**:
    -   Used for unique resource identification.
    -   *Example*: `/api/movies/{id}`. `id` is the variable.
2.  **Query Parameters (`@RequestParam`)**:
    -   Used for filtering or sorting.
    -   *Example*: `/api/movies/search?query=batman`. `query` is the param.
    -   *Optional Params*: `@RequestParam(required = false) String genre`.
3.  **Request Body (`@RequestBody`)**:
    -   Used for sending complex data (like a Forms) in POST/PUT.
    -   Spring uses **Jackson** to convert the incoming JSON string into a Java Object automatically.

### 📘 Customizing Responses (`ResponseEntity`)
Instead of just returning the object, we use `ResponseEntity` to control the **Status Code** and **Headers**.
```java
@PostMapping
public ResponseEntity<UserDTO> createUser(@RequestBody UserDTO user) {
    // Return 201 CREATED instead of 200 OK, and add a custom header
    return ResponseEntity
            .status(HttpStatus.CREATED)
            .header("X-Custom-Header", "UserCreated")
            .body(userService.createUser(user));
}
```

### 📘 Exception Handling
We use `@RestControllerAdvice` to handle errors globally. If a controller throws an exception, this class catches it and returns a clean JSON error.
-   **Project File**: `GlobalExceptionHandler.java`.

---

## 4. RESTful Resource Representation with DTOs

### 📘 Why use DTOs?
**Data Transfer Objects (DTOs)** are plain Java objects used to pass data between the user and the application.
1.  **Security**: Hide fields like `password` or `createdDate` that exist in the Entity.
2.  **Decoupling**: Validates inputs (e.g., "Email cannot be empty") without cluttering the Entity.
3.  **Versioning**: If the DB schema changes, the DTO can stay the same, preventing breaking changes for API clients.

### 🏗️ Versioning Strategies
*How do we update the API without breaking it for old mobile apps?*
1.  **URI Versioning** (Most Common): `/api/v1/movies` vs `/api/v2/movies`.
2.  **Request Param**: `/api/movies?version=1`.
3.  **Header Versioning**: `Accept: application/vnd.company.app-v1+json`.
    -   *Project implementation*: We use a single version logic currently, but URI versioning is the easiest next step.

---

## 5. RESTful CRUD & Advanced Features

### 📘 Validation
We use **Hibernate Validator** (part of `spring-boot-starter-validation`).
-   **Annotations**: `@NotNull`, `@Size(min=5)`, `@Email`.
-   **Usage**: Put `@Valid` before the `@RequestBody` in the controller.
    -   *Example*: `public ResponseEntity register(@Valid @RequestBody AuthRequest req)`.
    -   *If Invalid*: Spring throws `MethodArgumentNotValidException`.

### 📘 Optimistic Locking (Concurrency)
*Problem*: Two admins try to update the same movie at the same time. The last one overrides the first.
*Solution*: **Optimistic Locking**.
-   **How**: Add a `@Version` field to the Entity.
```java
@Entity
public class Movie {
    @Id private Long id;
    
    @Version
    private Integer version; // Spring checks this number 
}
```
-   **Logic**: If Version in DB != Version sent by user, DB throws `OptimisticLockException`.
-   *Status*: Not currently in `Movie.java`, but a key interview topic.

---

## 6. HATEOAS (Hypermedia)

### 📘 Concept
**Hypermedia as the Engine of Application State**.
The API tells the client "What you can do next" by providing links.
-   **Without HATEOAS**: Client must read documentation to know they can subscribe.
-   **With HATEOAS**: Response includes:
```json
{
  "email": "john@test.com",
  "links": [
    { "rel": "self", "href": "/api/users/1" },
    { "rel": "subscribe", "href": "/api/users/subscribe" }
  ]
}
```
*Project Status*: Not implemented. We use standard JSON.

---

## 7. Content Negotiation

### 📘 Concept
The ability of the API to serve different formats (JSON, XML, PDF) based on what the client asks for.
-   **Header**: `Accept: application/xml`.
-   **Spring Config**: By default, Spring Boot only supports JSON. To support XML, we must add the `jackson-dataformat-xml` dependency.
-   *Project Status*: JSON only.

---

## 8. Spring Boot Actuator (Monitoring)

### 📘 Topic Explanation
**Actuator** adds production-ready features to your app to help you monitor and manage it.
-   **Endpoints**:
    -   `/actuator/health`: System health (Up/Down, DB connection status).
    -   `/actuator/metrics`: CPU usage, memory, request counts.
    -   `/actuator/loggers`: Change logging levels at runtime.

### �️ Security Warning
Actuator exposes sensitive info. In a real app, we must secure these endpoints using Spring Security (allow only ADMIN role).
-   *Project Status*: Can be added via `spring-boot-starter-actuator`.

---

## 9. Security & Authentication (Deep Dive)

### 📘 JWT (JSON Web Token)
-   **Structure**: Header + Payload (User ID, Role, Expiry) + Signature.
-   **Flow**:
    1.  User Logins -> Server signs a JWT with a **Secret Key**.
    2.  User sends JWT -> Server uses Secret Key to verify it wasn't tampered with.
-   **Project File**: `JwtUtil.java` (Generates and Validates tokens).

### 📘 CORS (Cross-Origin Resource Sharing)
-   **Problem**: Browsers block requests from one domain (`localhost:4200` - Angular) to another (`localhost:8081` - Java).
-   **Solution**: We must explicitly allow it.
-   **Project File**: `SecurityConfig.java`.
```java
configuration.setAllowedOrigins(List.of("http://localhost:4200")); // Allow Angular
configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
```

---

## 10. Testing REST APIs

### 📘 Unit vs Integration
-   **Unit Test**: Testing *just* the Controller logic. We Mock the Service. (`Mockito`, `MockMvc`).
-   **Integration Test**: Testing the Controller + Service + DB. (`@SpringBootTest`, `TestRestTemplate`).

### 💻 Testing with MockMvc
This allows us to send fake HTTP requests without a network.
```java
// Simulate a POST request with JSON body
mockMvc.perform(post("/api/auth/login")
        .contentType(MediaType.APPLICATION_JSON)
        .content("{\"email\":\"john@test.com\",\"password\":\"123456\"}"))
        .andExpect(status().isOk()); // Assert HTTP 200
```
-   **Project Mapping**: `UserControllerTest.java` uses this extensively.

---

## 11. Documenting with Swagger/OpenAPI

### 📘 Topic Explanation
-   **Swagger UI**: An interactive webpage where you can test API endpoints.
-   **SpringDoc**: The library we use (`springdoc-openapi-starter-webmvc-ui`) to generate this.

### 🏗️ Annotations for Better Docs
We can decorate our controller to make the docs clearer.
-   `@Operation(summary = "Get Movie")`: Describes the endpoint.
-   `@ApiResponse(responseCode = "200", description = "Found")`: Describes successful outcomes.
-   `@ApiResponse(responseCode = "404", description = "Not Found")`: Describes errors.

### 💻 Code Snippet (Hypothetical Enhancement)
```java
@Operation(summary = "Search Movies", description = "Find movies by title")
@GetMapping("/search")
public List<Movie> search(@RequestParam String query) { ... }
```
-   *Project Status*: Configured in `SwaggerConfig.java`. Accessible at `/swagger-ui/index.html`.

---

## 12. Interview Checklist

### Q: "How does Spring convert Java Objects to JSON?"
> **Answer**: "It uses the **Jackson** library, which is included in the web starter. The `MappingJackson2HttpMessageConverter` automatically runs whenever a controller method returns an object annotated with `@ResponseBody` (or acts as a `@RestController`)."

### Q: "What happens if a resource is not found?"
> **Answer**: "Ideally, we throw a custom exception like `ResourceNotFoundException`. Our global `@RestControllerAdvice` catches this and returns a standard JSON error with HTTP Status 404, keeping the API response consistent."

### Q: "Explain how you handled CORS in your project."
> **Answer**: "Since my Angular frontend runs on port 4200 and backend on 8081, I configured a `CorsConfigurationSource` bean in my `SecurityConfig` to explicitly allow requests from the Angular origin."


# MASTER PROJECT CODE WALKTHROUGH

This document provides a line-by-line explanation of the core files in **MediaConnect**, connecting them to the concepts of **REST**, **Spring Boot**, **IoC**, and **JPA**. 

Use this to explain "Your Code" in an interview.

---

## 🏗️ MODULE 1: AUTHENTICATION (Login & Register)

### 1. `AuthController.java` (The Entry Point)
**Role**: Handles incoming HTTP Requests for Login/Register.
**Key Concepts**: `@RestController`, `@RequestBody`, `ResponseEntity`.

```java
@PostMapping("/login") // 1. Maps HTTP POST /api/auth/login
public ResponseEntity<AuthResponse> login(@RequestBody AuthRequest request) { // 2. @RequestBody converts JSON to Java Object
    logger.info("Login request received..."); 
    AuthResponse response = authService.login(request); // 3. Delegates logic to Service (SRP Principle)
    return ResponseEntity.ok(response); // 4. Returns HTTP 200 + JSON
}
```
-   **Interview Point**: "I use `@RequestBody` to deserialize the JSON payload into an `AuthRequest` DTO, ensuring my controller never sees raw string data."

### 2. `AuthServiceImpl.java` (The Business Logic)
**Role**: Validates credentials, talks to DB, generates JWT.
**Key Concepts**: `@Service`, `Dependency Injection`, `JWT`.

```java
@Autowired
private AuthenticationManager authenticationManager; // 1. Injected by Spring (IoC)

@Override
public AuthResponse login(AuthRequest request) {
    // 2. Authenticate: Checks email/password against DB using Spring Security
    authenticationManager.authenticate(
        new UsernamePasswordAuthenticationToken(request.getEmail(), request.getPassword()));

    // 3. Fetch User: Uses JPA Repository
    User user = userRepository.findByEmail(request.getEmail());

    // 4. Generate Token: Creates the "Bearer Token"
    String token = jwtUtil.generateToken(user.getEmail());

    // 5. Return Response: Sends token back to user
    return new AuthResponse(token, user.getRole(), ...); 
}
```
-   **Interview Point**: "My service layer is responsible for orchestrating the authentication flow. It uses the `AuthenticationManager` to verify credentials but relies on `JwtUtil` to creation the actual session token."

### 3. `JwtFilter.java` (The Gatekeeper)
**Role**: Checks *every* request for a valid token.
**Key Concepts**: `OncePerRequestFilter`, `SecurityContext`.

```java
@Override
protected void doFilterInternal(...) {
    // 1. Get Header: "Authorization: Bearer eyJ..."
    String authHeader = request.getHeader("Authorization");

    // 2. Validate & Extract Email
    if (authHeader != null && authHeader.startsWith("Bearer ")) {
        String token = authHeader.substring(7);
        if (jwtUtil.validateToken(token, userDetails)) {
            // 3. Set Context: Tells Spring Security "This user is logged in"
            SecurityContextHolder.getContext().setAuthentication(authToken);
        }
    }
    // 4. Continue: Pass request to next filter (or Controller)
    chain.doFilter(request, response);
}
```

---

## 📽️ MODULE 2: MOVIES & SEARCH (Core Feature)

### 1. `MovieController.java`
**Role**: Browse, Search, and Filter movies.
**Key Concepts**: `@RequestParam`, `@PathVariable`.

```java
// FEATURE: Search Movies by Title
@GetMapping("/search")
public List<Movie> search(@RequestParam String query) {
    // URL: /api/movies/search?query=Batman
    return movieService.searchMovies(query);
}

// FEATURE: Get Single Movie
@GetMapping("/{id}")
public ResponseEntity<Movie> getMovieById(@PathVariable Long id) {
    // URL: /api/movies/101
    return ResponseEntity.ok(movieService.getMovieById(id));
}
```

### 2. `MovieRepository.java` (The Data Layer)
**Role**: Talks to the Database.
**Key Concepts**: `Derived Queries`, `Native Queries`.

```java
// 1. Derived Query: Spring generates SQL "WHERE title LIKE %?%"
List<Movie> findByTitleContainingIgnoreCase(String title);

// 2. Native Query: Complex Logic for Trending
@Query(value = "SELECT * FROM movies ... ORDER BY views DESC", nativeQuery = true)
List<Movie> findTop10ByOrderByViewsDesc();
```
-   **Interview Point**: "For simple searches, I utilized Spring Data's derived query methods because they are readable and require no SQL. For complex reports like 'Trending', I wrote a native SQL query to join the `watch_history` table."

---

## 👤 MODULE 3: USER & SUBSCRIPTION

### 1. `UserController.java`
**Role**: Profile management, Subscription upgrading.

```java
@PostMapping("/subscribe")
public ResponseEntity<UserDTO> subscribe(@RequestBody Map<String, String> payload) {
    // 1. Extract Email from Security Context (Who is logged in?)
    String email = SecurityContextHolder.getContext().getAuthentication().getName();
    
    // 2. Call Service
    return ResponseEntity.ok(userService.subscribe(email, payload.get("plan")));
}
```

### 2. `UserServiceImpl.java` (Subscription Logic)

```java
@Override
public UserDTO subscribe(String email, String plan) {
    User user = userRepository.findByEmail(email);
    
    // 1. Update Plan Details
    user.setCurrentPlan(plan);
    user.setSubscriptionStatus("ACTIVE");
    user.setSubscriptionExpiry(LocalDateTime.now().plusDays(30)); // 30 Day Logic
    
    // 2. Save (Hibernate triggers SQL UPDATE)
    userRepository.save(user);
    
    return mapToDTO(user);
}
```

---

## 📊 MODULE 4: WATCH HISTORY & ANALYTICS

### 1. `WatchHistoryServiceImpl.java` (Logic)
**Role**: Tracks where you stopped watching.

```java
public void saveProgress(Long userId, Long movieId, int seconds) {
    // 1. Check if record exists
    WatchHistory existing = watchHistoryRepository.findByUserIdAndMovieId(userId, movieId);

    if (existing != null) {
        // 2. Update existing
        existing.setProgressSeconds(seconds);
        existing.setLastWatched(LocalDateTime.now());
    } else {
        // 3. Create new
        WatchHistory history = new WatchHistory(user, movie, seconds);
        watchHistoryRepository.save(history); // Save
    }
}
```
-   **Interview Point**: "I implemented an 'Upsert' logic (Update or Insert) for watch history. If a user is re-watching a movie, I update their timestamp; otherwise, I create a new entry."

---

## 📝 GLOBAL FEATURES

### 1. `GlobalExceptionHandler.java` (Error Handling)
**Key Concept**: `AOP` (Aspect Oriented Programming).

```java
@RestControllerAdvice // Targets ALL Controllers
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class) // Catches specific error
    public ResponseEntity<ErrorResponse> handleNotFound(Exception ex) {
        // Returns clean JSON instead of StackTrace
        return new ResponseEntity<>(new ErrorResponse(404, ex.getMessage()), HttpStatus.NOT_FOUND);
    }
}
```

### 2. `CorsConfig` (Frontend Connection)
**Key Concept**: Security Configuration.

```java
configuration.setAllowedOrigins(List.of("http://localhost:4200")); // Trusted Frontend
configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
```

---

## 🧠 HOW TO EXPLAIN "YOUR CODE" IN INTERVIEW

**Q: "Walk me through your Authentication flow."**
> "Sure. When the frontend sends a POST request with credentials to `AuthController`, my `AuthService` validates them using Spring Security's `AuthenticationManager`. If valid, I generate a JWT using `jjwt` library and send it back. For all subsequent requests, my `JwtFilter` intercepts the request, extracts this token, validates it, and sets the user in the Security Context."

**Q: "How did you implement Search?"**
> "I created a `MovieController` that accepts a `query` parameter. This calls `MovieRepository.findByTitleContainingIgnoreCase`, which is a Spring Data derived query, so I didn't have to write any manual SQL for basic searching."

**Q: "How do you handle Database operations?"**
> "I use Spring Data JPA. For example, in my `UserServiceImpl`, I simply call `userRepository.save(user)` to update subscription details, and Hibernate manages the underlying SQL UPDATE statement."


# SPRINT 4: Microservices & Spring Cloud Architecture

This section covers the transition from Monolithic to Microservices, detailing Spring Cloud components, Security, and Resilience patterns.

---

## 1. Introduction to Microservices Architecture (MSA)
### 📘 Monolith vs. Microservices
-   **Monolithic Architecture**: A single large codebase where all modules (User, Movie, Auth) are tightly coupled and deployed as a single WAR/JAR.
    -   *Pros*: Simple to develop/deploy initially.
    -   *Cons*: Single Point of Failure (SPOF), hard to scale individual components, technology lock-in.
-   **Microservices Architecture**: An architectural style where an application is built as a collection of small, independent services.
    -   *Characteristics*: Independently deployable, scalable, loosely coupled, organized around business capabilities.

### 🚀 Advantages & Challenges
-   **Advantages**:
    -   **Scalability**: Scale only the "Movie Service" if traffic spikes, without scaling "User Service".
    -   **Resilience**: If one service fails, others continue to work.
    -   **Tech Agnostic**: One service in Java, another in Python (though we use Spring Boot for all).
-   **Challenges**:
    -   **Complexity**: Distributed systems introduce network latency, consistency issues, and debugging difficulty.
    -   **Data Management**: Data is distributed (Database per Service pattern), making transactions (ACID) hard.

### 💡 Use Cases
-   Large-scale applications (NetFlix, Amazon).
-   Teams requiring independent deployment cycles.
-   Systems with distinct scaling needs per module.

---

## 2. Spring Cloud for Microservices
### 📘 Overview
**Spring Cloud** provides tools for developers to quickly build some of the common patterns in distributed systems (e.g., configuration management, service discovery, circuit breakers).

### 🛠 Key Components
1.  **Service Discovery (Netflix Eureka)**: Acts as a phonebook. Services register themselves ("I am Movie Service on port 8082"). Clients look up services here.
2.  **API Gateway (Spring Cloud Gateway)**: The single entry point for all client requests.
3.  **Config Server (Spring Cloud Config)**: Centralized management for external properties.
4.  **Circuit Breaker (Resilience4j)**: Handling fault tolerance.
5.  **Distributed Tracing (Micrometer/Zipkin)**: Tracking requests across services.

### ⚙️ Configuring Eureka Server
-   **Dependency**: `spring-cloud-starter-netflix-eureka-server`
-   **Annotation**: `@EnableEurekaServer` on main class.
-   **Properties**: `server.port=8761`.

---

## 3. Spring Security for Microservices
### 📘 Overview
In a distributed system, we cannot rely on simple Session IDs because requests jump between services. **Stateless Authentication** is key.

### 🔒 Securing Inter-Service Communication
1.  **mTLS (Mutual TLS)**: Encrypted communication where both client and server authenticate each other.
2.  **Token Relay**: Passing the User's JWT token from the Gateway -> Service A -> Service B.

### 🛡 Configuring Security
-   Each Microservice functions as a **Resource Server**.
-   It validates the JWT token coming in the `Authorization` header.
-   **Dependency**: `spring-boot-starter-oauth2-resource-server`.

---

## 4. Centralized Authentication & Authorization
### 📘 OAuth 2.1 & OIDC
-   **OAuth 2.1**: The industry standard protocol for authorization.
-   **OIDC (OpenID Connect)**: A layer on top of OAuth 2.0 for Authentication (Identity).
-   **Key Roles**:
    -   **Authorization Server**: Issues tokens (e.g., Keycloak, Spring Authorization Server).
    -   **Resource Server**: The microservice protecting data (e.g., Movie Service).
    -   **Client**: The frontend (Angular App).

### 🔑 JWT & SSO
-   **JSON Web Token (JWT)**: A compact, URL-safe means of representing claims to be transferred between two parties. contains `sub` (user), `roles` (admin), `exp` (expiry).
-   **Single Sign-On (SSO)**: User logs in once (via Auth Server) and gains access to all microservices without re-entering credentials.

---

## 5. Microservices Communication
### 📘 Communication Patterns
1.  **Synchronous (Blocking)**: HTTP/REST. Service A waits for Service B.
2.  **Asynchronous (Non-Blocking)**: Messaging (Kafka/RabbitMQ). Service A fires an event and moves on.

### 📞 Spring Cloud OpenFeign
-   **Concept**: A declarative REST client. Instead of `RestTemplate`, you write an interface.
-   **Usage**:
    ```java
    @FeignClient(name = "movie-service")
    public interface MovieClient {
        @GetMapping("/movies/{id}")
        MovieDTO getMovie(@PathVariable Long id);
    }
    ```
-   **Benefit**: Eureka integration (Load Balancing) is built-in.

### 🎭 Orchestration vs. Choreography
-   **Orchestration**: A central conductor (e.g., "Order Service") tells everyone what to do.
-   **Choreography**: decentralized. Service A emits "Order Placed", Service B listens and reacts.

---

## 6. API Gateway & Edge Services
### 📘 Concept
The **API Gateway** sits between the client and the backend microservices. It is the "Edge Service".

### 🚪 Spring Cloud Gateway
-   **Features**: Routing, Filtering, Load Balancing, Auth Limits.
-   **Configuration (`application.yml`)**:
    ```yaml
    spring:
      cloud:
        gateway:
          routes:
            - id: movie-route
              uri: lb://MOVIE-SERVICE  # 'lb' means Load Balanced/Eureka lookup
              predicates:
                - Path=/api/movies/**
    ```
-   **Filters**: Modify requests (add headers) or responses.

---

## 7. Fault Tolerance & Resilience
### 📘 The Problem
In distributed systems, failure is inevitable. If "Service B" is slow, "Service A" shouldn't crash waiting for it.

### 🛡 Resilience Patterns (Resilience4j)
1.  **Circuit Breaker**: Detects failures and "opens" the circuit to fail fast, preventing cascading failures.
    -   *States*: CLOSED (Normal), OPEN (Failing/Blocked), HALF-OPEN (Testing recovery).
2.  **Fallback Mechanism**: Default behavior when a service fails.
    -   *Example*: If "Recommendation Service" is down, return a generic list of top movies.
3.  **Retry**: Automatically retrying a failed request a few times before giving up.

---

## 8. Spring Cloud Config
### 📘 Externalized Configuration
Managing `application.properties` for 50 services is a nightmare. **Spring Cloud Config** solves this.

### ⚙️ How it works
1.  **Config Server**: Reads properties from a Git Repository.
2.  **Config Client** (Microservice): Fetches its config at startup from the Config Server.
3.  **Dynamic Refresh**: Use `@RefreshScope` and `/actuator/refresh` to update config without restarting the service.

---

## 9. Monitoring & Metrics
### 📘 Observability
You cannot fix what you cannot see. We need distinct pillars:
1.  **Logging**: ELK Stack (Elasticsearch, Logstash, Kibana).
2.  **Metrics**: Prometheus & Grafana.
3.  **Tracing**: Zipkin/Jaeger.

### 📈 Spring Boot Actuator
-   Exposes operational endpoints: `/health`, `/info`, `/metrics`.
-   **Prometheus**: Scrapes `/actuator/prometheus` to gather time-series data (e.g., "Requests per second").
-   **Grafana**: Visualizes this data in dashboards.

---

## 10. Security Best Practices
### 🛡 RBAC (Role-Based Access Control)
-   Strictly enforce roles.
-   *Gateway Level*: Ensure `/admin/**` routes are blocked for non-admins.
-   *Method Level*: `@PreAuthorize("hasRole('ADMIN')")`.

### 🔒 Defense in Depth
1.  **Least Privilege**: Services should only talk to services they *need* to.
2.  **Secure Headers**: Use Helmet/Spring Security headers (CSP, XSS protection).
3.  **Sensitive Data**: Never log passwords or PII. Use vaulted storage (Spring Cloud Vault) for secrets.

---

Type **CONTINUE** to proceed with advanced production-level microservices topics.

---

# SPRINT 5: Advanced Microservices & DevOps

This section delves into production-grade patterns, including Event-Driven Architecture, Kubernetes, and Distributed Transactions.

---

## 1. Event-Driven Microservices (Kafka / RabbitMQ)
### 📘 The Concept
Instead of Service A calling Service B (Synchronous/Coupled), Service A publishes an **Event** ("UserRegistered") to a **Message Broker**. Service B, C, and D subscribe to it and react.

### 🚀 Benefits
-   **Decoupling**: The sender doesn't know who is listening.
-   **Buffering**: If the consumer is down, messages wait in the queue (Kafka topic) until it recovers.
-   **Scalability**: Add more consumers to process messages faster.

### 🛠 Tools
-   **Apache Kafka**: High-throughput distributed streaming platform. Good for event sourcing.
-   **RabbitMQ**: Traditional message broker. Good for complex routing.
-   **Spring Cloud Stream**: Framework to build event-driven apps without locking into Kafka/RabbitMQ API directly.

---

## 2. Containerization (Docker)
### 📘 Why Docker?
"It works on my machine" is a classic problem. **Docker** packages the application + JDK + OS dependencies into a lightweight **Image**.

### 🐳 Key Concepts
-   **Dockerfile**: The recipe to build the image.
    ```dockerfile
    FROM openjdk:17-jdk-slim
    COPY target/myapp.jar app.jar
    ENTRYPOINT ["java", "-jar", "/app.jar"]
    ```
-   **Image**: The static snapshot (like a Class).
-   **Container**: The running instance (like an Object).
-   **Multi-Stage Build**: Use distinct stages to build (Maven) and run (JRE) to keep images small.

---

## 3. Orchestration (Kubernetes / K8s)
### 📘 Managing Containers
Running 1 container is easy (Docker). Running 100 containers across 5 servers is hard. **Kubernetes** automates scaling, healing, and deployment.

### ⚓ Key K8s Objects
1.  **Pod**: The smallest unit. Runs one or more containers (e.g., Movie Service).
2.  **Deployment**: Manages Pods. "Ensure 3 replicas of Movie Service are always running."
3.  **Service**: Network abstraction. "Traffic to 'movie-svc' goes to one of the 3 Pods."
4.  **Ingress**: Exposes services to the outside world (like an API Gateway).

---

## 4. Distributed Transactions (Saga Pattern)
### 📘 The Problem
In a Monolith, `One Transaction = Database Commit`. In Microservices, `Order Service` commits, then `Payment Service` fails. How do we rollback the Order?

### 🔄 The Saga Pattern
A sequence of local transactions. If one fails, execute **Compensating Transactions** to undo changes.
1.  **Choreography**: Events trigger steps. (Order Created -> Payment Processed -> Inventory Reserved).
2.  **Orchestration**: A Central Coordinator (State Machine) calls each service.

---

## 5. Service Mesh (Istio / Linkerd)
### 📘 What is it?
A dedicated infrastructure layer for handling service-to-service communication. It uses a **Sidecar Proxy** (Envoy) next to every microservice instance.

### 🛡 Features
-   **Traffic Management**: Canary Deployments (send 10% traffic to v2).
-   **Security**: Automatic mTLS between all services without code changes.
-   **Observability**: Deep stats on latency and errors per hop.

---

## 6. Cloud Native Patterns (12-Factor App)
### 📜 The Gold Standard
A methodology for building apps that run well in the cloud.
1.  **Config**: Store config in the environment (Spring Cloud Config).
2.  **Backing Services**: Treat DB/Queue as attached resources.
3.  **Stateless Processes**: No sticky sessions. State goes to Redis/DB.
4.  **Disposability**: Fast startup and graceful shutdown.

---

## 7. CI/CD for Microservices
### 🚀 Automation
-   **Continuous Integration (CI)**: Developer pushes code -> Jenkins/GitHub Actions runs Tests -> Builds Docker Image -> Pushes to Registry (Docker Hub).
-   **Continuous Deployment (CD)**: Tool (ArgoCD) detects new Image -> Updates Kubernetes Deployment -> New version goes live.
-   **GitOps**: "Infrastructure as Code". The state of the cluster is defined in a Git Repo.

---

## 8. Advanced Observability (Distributed Tracing)
### 🔍 Trace ID & Span ID
-   **Trace ID**: Unique ID generated at the Gateway for a single specific user request. It follows the request everywhere.
-   **Span ID**: Tracks a specific operation within a service.
-   **Tooling**: **Zipkin** or **Jaeger**.
-   **Analogy**: "Trace ID is the Flight Number. Span IDs are the take-off, landing, and layover."

---

## 9. Testing Strategies
### 🧪 Microservices Testing Pyramid
1.  **Unit Tests**: Internal logic (JUnit/Mockito).
2.  **Integration Tests**: Service + DB (TestContainers).
3.  **Contract Tests (Pact)**: Ensures Service A and Service B agree on the JSON format without spinning up both.
4.  **End-to-End (E2E)**: Simulating a user journey across the full stack (Cypress/Selenium).

---

## 10. Database Strategies
### 🗄 Database per Service
-   Each service has its OWN database Schema.
-   **User Service** -> `user_db` (MySQL).
-   **Recommendation Service** -> `neo4j` (Graph DB).
-   **Analytics** -> `mongo` (NoSQL).
-   *Rule*: Service A CANNOT query Service B's DB directly. It must use Service B's API.

---

Type **CONTINUE** to proceed with **System Design Interview Scenarios** (Designing Netflix/Uber).

---

# SPRINT 5 EXTENSION: Applying Advanced Concepts to MediaConnect

This section bridges the gap between **Theory** and **Your Codebase**. Use these examples to demonstrate how you would evolve **MediaConnect** from a Monolith to a sophisticated Cloud-Native Microservices application.

---

## 1. Event-Driven Architecture (Refactoring `WatchHistory`)

### 🚩 Current State (Monolith)
In `WatchHistoryServiceImpl.java`, when a user watches a movie, we write directly to the database:
```java
public void saveProgress(...) {
    watchHistoryRepository.save(new WatchHistory(...)); 
    // This is Synchronous. If DB is slow, user waits.
}
```

### 🚀 Future State (Event-Driven with Kafka)
Instead of writing to the DB, the Service publishes an **Event**.
1.  **Producer (`MovieService`)**:
    ```java
    kafkaTemplate.send("movie-watched-topic", new MovieWatchedEvent(userId, movieId, timestamp));
    // Asynchronous. Returns immediately. Extremely fast.
    ```
2.  **Consumer (`AnalyticsService`)**: Listens to the topic.
    ```java
    @KafkaListener(topics = "movie-watched-topic")
    public void processWatchTime(MovieWatchedEvent event) {
        // 1. Save to WatchHistory DB (Cassandra/Mongo for speed)
        // 2. Update "Trending Movies" counter (Redis)
    }
    ```

### 🗣 Interview Explanation
"Currently, `MediaConnect` handles watch history synchronously. In a production microservices environment, I would decouple this using **Kafka**. The `MovieService` would simply fire a 'UserWatched' event. This allows other services—like a Recommendation Engine or Analytics Service—to consume that same event independently without modifying the core Movie Service code."

---

## 2. Containerization (Dockerizing MediaConnect)

### 🚩 Current State
We run `mvn spring-boot:run` and rely on the host machine having Java 17 installed.

### 🚀 Docker Implementation
We create a `Dockerfile` in the root of the Backend.

**File**: `Backend/Dockerfile`
```dockerfile
# Stage 1: Build
FROM maven:3.8.5-openjdk-17 AS build
COPY . .
RUN mvn clean package -DskipTests

# Stage 2: Run (Multi-stage for smaller image)
FROM openjdk:17-jdk-slim
COPY --from=build /target/mediaconnect-backend.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Interview Answer**:
"I use **Multi-Stage Builds** to keep my images lightweight. The first stage uses the full Maven image to build the JAR, but the final image only contains the JRE and the compiled JAR, reducing the image size from ~800MB to ~150MB."

---

## 3. Orchestration (Kubernetes for MediaConnect)

### 🚩 Current State
We run one instance on `localhost:8081`. If it crashes, the site is down.

### 🚀 K8s Implementation
We define a **Deployment** to ensure high availability.

**File**: `k8s/backend-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mediaconnect-backend
spec:
  replicas: 3                       # Run 3 copies at the same time
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: myrepo/mediaconnect:latest
        ports:
        - containerPort: 8081
```

### 🗣 Interview Explanation
"In Kubernetes, I would deploy `MediaConnect` with **3 Replicas**. K8s monitors them. If one Pod runs out of memory and crashes, the **Kubelet** automatically restarts it (Self-Healing). I would also use a **ClusterIP Service** to balance traffic between these 3 pods."

---

## 4. Distributed Transactions (The SAGA Pattern)

### 🚩 Context: "Subscription Upgrade"
When a user subscribes:
1.  **Payment Service**: Charge Credit Card.
2.  **User Service**: Update Role to 'PREMIUM'.

### 💥 The Problem
If **Payment** succeeds, but **User Service** is down (Network Error), the user is charged but not upgraded.

### 🔄 Choreography Saga Solution
1.  **Payment Service**: Charges card. Publishes Event `PaymentSuccess`.
2.  **User Service**: Listens for `PaymentSuccess`. Attempts to update User.
    -   *Success*: Publish `UserUpgraded`. (Workflow End).
    -   *Failure*: Publish `UserUpgradeFailed`.
3.  **Payment Service**: Listens for `UserUpgradeFailed`. Triggers **Compensating Transaction** (Refund the card).

---

## 5. Database Patterns (Database per Service)

### 🚩 Current State
One MySQL schema (`mediaconnect`) with tables: `users`, `movies`, `watch_history`.

### 🚀 Microservices State
We split the data ownership.
1.  **User Service**: Owns `users` table (MySQL).
2.  **Movie Service**: Owns `movies` table (PostgreSQL).
3.  **History Service**: Owns `watch_history` (MongoDB - better for large, unstructured log data).

### 🗣 Interview Explanation
"Moving to Microservices, I would apply the **Database per Service** pattern. `MovieService` should never query the `users` table directly because that creates tight coupling. Instead, if it needs user name, it calls the `UserService` API. This allows me to migrate `WatchHistory` to MongoDB later without breaking the User Service."

---

## 6. Testing Strategies (Contract Testing)

### 🚩 The Problem
Frontend expects JSON `{ "movieId": 1, "title": "Batman" }`.
Backend changes field to `{ "m_id": 1, "name": "Batman" }`.
Frontend breaks.

### 🛡 Solution: Pact (Contract Testing)
1.  **Consumer (Frontend)** defines a "Contract": "I expect field `movieId`".
2.  **Provider (Backend)** runs a test against this Contract during the build.
3.  If Backend changed `movieId` to `m_id`, the build **FAILS** before deployment.

---

## 7. Simple System Design Scenario: "Design MediaConnect"

**Interviewer**: "Design a system like MediaConnect that scales to 1 million users."

**Whiteboard Answer**:
1.  **Load Balancer (NGINX)**: Distributes traffic.
2.  **API Gateway (Spring Cloud Gateway)**: Security, Rate Limiting.
3.  **Services**:
    -   `Auth Service` (Keycloak/JWT)
    -   `Movie Catalog Service` (Read Heavy -> Add Redis Cache)
    -   `Subscription Service`
4.  **Async Processing**:
    -   Kafka for "Process Video" or "Log History".
5.  **Storage**:
    -   **CDN (Cloudfront)**: Store actual Video Files (MP4). NEVER store videos in DB.
    -   **Relational DB**: Users, Billing.
    -   **NoSQL**: Watch History, Recommendations.

---
Type **CONTINUE** to add advanced Q&A.

---

# SPRINT 6: Architect-Level Interview Q&A

This section covers high-level, "Soft Skill" and "Hard Tech" questions often asked for Senior/Lead roles. These questions test your ability to handle failure, scale, and trade-offs.

---

## 🔥 Scenario 1: Handling Failure
**Q: "The Movie Service is down. How does the rest of MediaConnect behave?"**

**Junior Answer:** "Other services might throw 500 errors."
**Senior Answer:** "We implement **Circuit Breakers** (Resilience4j).
1.  **Immediate**: The Gateway detects the failure and stops sending requests to Movie Service (Circuit **OPEN**).
2.  **Fallback**: The homepage instead serves a cached 'Default List' of movies from Redis, rather than showing a blank screen.
3.  **Recovery**: After 30s, the Circuit enters **HALF-OPEN** state to test if the service is back. The user experience degrades gracefully, but the app stays up."

---

## ⚡ Scenario 2: Caching & Consistency
**Q: "You introduced Redis to cache movie details. What happens when an Admin updates a movie title? The Cache is now stale/old."**

**A:** "We use the **Cache-Aside Pattern** with invalidation.
1.  **Read**: App checks Redis. If missing, read DB -> Write Redis.
2.  **Update**: When Admin saves a change (`updateMovie()`), we publish a `MovieUpdatedEvent`.
3.  **Invalidate**: Ideally, the update logic **deletes** the key from Redis immediately. The next read will re-fetch the fresh data from DB. This is safer than trying to *update* the cache, which can lead to race conditions."

---

## 🚀 Scenario 3: Database Scaling
**Q: "MediaConnect now has 10 Million Users. The 'Users' database is too slow. What do you do?"**

**A:** "I would approach this in phases:
1.  **Read Replicas**: Direct all `GET` requests to 3 Read-Only copies of the DB. Only `POST/PUT` goes to the Master DB.
2.  **Indexing**: Analyze slow queries and ensure proper indexing on columns like `email` or `subscriptionStatus`.
3.  **Sharding (Horizontal Partitioning)**: Split the users into multiple databases based on ID.
    -   *Shard A*: User IDs 1-1,000,000.
    -   *Shard B*: User IDs 1,000,001-2,000,000.
    -   *App Router*: Logic to decide which DB to query."

---

## 🔒 Scenario 4: Distributed Locking
**Q: "We have a limited offer: 'First 100 subscribers get 50% off'. 10,000 users click 'Buy' at the exact same millisecond. How do you prevent overselling?"**

**A:** "Standard Java `synchronized` doesn't work across multiple microservices.
1.  **Optimistic Locking**: Add a `version` column. It works but creates many retries.
2.  **Redis Distributed Lock (Redlock)**:
    -   Before processing the payment, the service attempts to acquire a lock key: `lock:offer:100`.
    -   Only ONE instance gets the lock. It decrements the counter in Redis (Atomic Operation `DECR`).
    -   If the counter > 0, proceed. Else, reject."

---

## 🐢 Scenario 5: Slow 3rd Party APIs
**Q: "MediaConnect uses an external Email Provider to send welcome emails. It suddenly becomes very slow (10s response). Your user registration hangs. What do you do?"**

**A:** "User Registration should **never** be blocked by non-critical side effects like email.
1.  **Async Processing**: The `UserService` should just save the user and publish a `UserRegistered` event.
2.  **Notification Service**: Consumes the event. If the Email Provider is slow, only the Notification Service hangs (or retries later). The User gets a fast 'Success' response immediately."

---

## 📦 Scenario 6: Zero-Downtime Deployment
**Q: "How do you deploy a new version of MediaConnect without kicking logged-in users out?"**

**A:** "We use **Rolling Updates** (Kubernetes).
1.  We have 3 Replicas of V1 running.
2.  K8s spins up 1 new Pod of V2. Waits for the **Health Check** to pass.
3.  K8s kills 1 Pod of V1.
4.  Repeats until all 3 are V2.
5.  **Session Externalization**: Users aren't kicked out because their Session Data (JWT or Session) is stored in **Redis**, not in the application memory."

---

## 🔎 Scenario 7: Thundering Herd
**Q: "The Cache expires. 5,000 users request the 'Home Page' at once. They all miss the cache and hit the DB instantly. The DB crashes. This is a Thundering Herd."**

**A:** "We implement **Request Coalescing** or **Mutex on Cache Miss**.
1.  When Cache Miss occurs, *only one* thread acts to fetch from DB.
2.  It acquires a local lock or promise.
3.  The other 4,999 requests *wait* for that first thread.
4.  Once the first thread updates the Cache, all others read the value and return. The DB only sees 1 query, not 5,000."

---

## 🎤 Behavior/Culture Questions

**Q: "Tell me about a time you disagreed with a senior/manager."**
> **STAR Method Answer**: "In MediaConnect, my manager wanted to store video files in the Database to keep it 'simple'.
> - **Situation**: Storing BLOBs in MySQL would kill performance and make backups huge.
> - **Task**: I needed to convince them to use S3/Cloud storage.
> - **Action**: I didn't just argue. I built a small proof-of-concept. I showed that a 100MB download blocked the DB thread for 5 seconds, whereas S3 was instant. I presented a cost comparison showing Database Storage is 10x more expensive than Object Storage.
> - **Result**: Based on the data, they agreed. We implemented S3 storage, saving cost and improving speed."

**Q: "How do you handle Technical Debt?"**
> **Answer**: "I follow the 'Boy Scout Rule': Always leave the code cleaner than you found it. If I touch a file for a feature, I'll fix variable names or add a missing test. For larger debt, I advocate for setting aside 20% of Sprint time for refactoring, treating debt like financial debt—if you don't pay interest (refactor), you eventually go bankrupt (cannot ship features)."

---

# 🏁 END OF MASTER GUIDE
This document is your **single source of truth** for MediaConnect. 
1.  **Sprint 2**: Foundations & Codebase Walkthrough.
2.  **Sprint 3**: REST, Security, & JPA Deep dives.
3.  **Sprint 4**: Microservices & Distributed Logic.
4.  **Extension**: Advanced Architectures (K8s, Docker, Sagas).
5.  **Sprint 6**: Hard Interview Scenarios.

