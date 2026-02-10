# Backend Spring Boot Concepts: Comprehensive Analysis

This document provides a detailed breakdown of all Spring Boot and Java concepts used in the `Backend` project (MediaConnect), mapped directly to the files where they appear.

## 1. Project Architecture & Configuration

The application is built using **Spring Boot 3.3.0 + Java 17**, utilizing **Maven** for dependency management.

### Core Configuration Files
- **`pom.xml`**: Project Object Model.
  - **Concept**: Dependency Management.
  - **Dependencies**: `spring-boot-starter-web`, `spring-boot-starter-data-jpa`, `spring-boot-starter-security`, `jjwt` (JWT), `mysql-connector` (Database), `springdoc-openapi` (Swagger).
- **`MediaconnectApplication.java`**: Main Entry Point.
  - **Concept**: `@SpringBootApplication`. It performs component scanning, auto-configuration, and bootstraps the application context.
- **`src/main/resources/application.properties`**: (Inferred) Configuration.
  - **Concept**: Externalized Configuration (Database URL, Server Port, JWT Secrets).

---

## 2. REST API & Controllers (Web Layer)

The application uses **Spring Web MVC** to expose RESTful endpoints.

- **`AuthController.java`** (`src/.../controller/AuthController.java`):
  - **Annotation**: `@RestController` (Combines `@Controller` and `@ResponseBody`).
  - **Routing**: `@RequestMapping("/api/auth")` defines the base URL.
  - **Endpoints**: `@PostMapping("/login")` maps HTTP POST requests to methods.
  - **Data Binding**: `@RequestBody` deserializes JSON from the request body into Java Objects (`AuthRequest`).
  - **Response**: `ResponseEntity<T>` is used to return HTTP status codes and body content securely.
  - **Logging**: Uses `SLF4J` (`Logger`, `LoggerFactory`) for info/debug logs.

---

## 3. Data Access Layer (JPA & Hibernate)

The application uses **Spring Data JPA** to interact with the MySQL database without writing boilerplate SQL.

- **`UserRepository.java`** (`src/.../repository/UserRepository.java`):
  - **Concept**: Repository Pattern.
  - **Inheritance**: Extends `JpaRepository<User, Long>`. This provides built-in methods like `save()`, `findAll()`, `findById()`, `delete()`.
  - **Derived Query Methods**: `User findByEmail(String email);` - Spring automatically generates the SQL query based on the method name.
  - **Annotation**: `@Repository` (Optional when extending JpaRepository, but good for clarity).

- **`User.java`** (`src/.../entity/User.java`):
  - **ORM Mapping**:
    - `@Entity`: Marks class as a Database Table.
    - `@Table(name = "users")`: Customizes table name.
    - `@Id`, `@GeneratedValue(strategy = GenerationType.IDENTITY)`: Primary Key with auto-increment.
    - `@Column(nullable = false, unique = true)`: Database constraints.

---

## 4. Security & Authentication (Spring Security 6)

The project implements **Stateless JWT Authentication**.

- **`SecurityConfig.java`** (`src/.../security/SecurityConfig.java`):
  - **Configuration**: `@Configuration` and `@EnableWebSecurity`.
  - **Security Filter Chain**:
    - Disables CSRF (`.csrf(AbstractHttpConfigurer::disable)`) because API is stateless.
    - Enables CORS (`.cors(...)`) to allow the Angular frontend (`localhost:4200`) to communicate.
    - **Authorization**: `.requestMatchers("/admin/**").hasAuthority("ADMIN")` protects routes.
    - **Session**: `SessionCreationPolicy.STATELESS` ensures no server-side sessions are created.
  - **Password Encoding**: Defines `BCryptPasswordEncoder` bean.
  - **Filter Registration**: `.addFilterBefore(jwtFilter, ...)` inserts the custom JWT filter before standard authentication.

- **`JwtFilter.java`** (`src/.../security/JwtFilter.java`):
  - **Concept**: `OncePerRequestFilter`.
  - **Logic**: Intercepts every request, extracts the "Bearer" token from the header, validates it, and sets the `SecurityContext` if valid.

- **`CustomUserDetailsService.java`**:
  - **Concept**: `UserDetailsService` interface.
  - **Logic**: Loads user data from the database (`userRepository.findByEmail()`) and converts it to a purely security-focused `UserDetails` object.

---

## 5. Exception Handling

- **`GlobalExceptionHandler.java`** (`src/.../exception/GlobalExceptionHandler.java`):
  - **Aspect-Oriented Programming (AOP)**: `@RestControllerAdvice` allows global handling of exceptions across all controllers.
  - **Handlers**: `@ExceptionHandler(ResourceNotFoundException.class)` catches specific errors and returns a clean JSON `ErrorResponse` with standard HTTP status codes (404, 500).

---

## 6. Understanding Imports

This section explains the common packages used in your Java files.

### Spring Ecosystem
- **`org.springframework.web.bind.annotation.*`**: REST annotations (`@GetMapping`, `@RestController`).
- **`org.springframework.beans.factory.annotation.Autowired`**: Dependency Injection (Field or Constructor).
- **`org.springframework.stereotype.*`**: Component markers (`@Service`, `@Repository`, `@Component`).
- **`org.springframework.http.*`**: HTTP utilities (`ResponseEntity`, `HttpStatus`).
- **`org.springframework.security.*`**: Security configurations.

### Java Persistence API (JPA) / Jakarta
- **`jakarta.persistence.*`** (or `javax.persistence.*` depending on Boot version): ORM annotations (`@Entity`, `@Id`, `@OneToMany`).

### Utilities
- **`java.util.*`**: Collections (`List`, `Optional`, `Map`).
- **`org.slf4j.*`**: Simple Logging Facade for Java.

---

## 7. Complete List of Backend Concepts Used

Below is a quick-reference list of every Spring/Java concept found in the source code.

### Annotations
- **`@SpringBootApplication`**: Bootstraps the app.
- **`@RestController / @Controller`**: Defines Web components.
- **`@Service`**: Defines Business Logic components.
- **`@Repository`**: Defines Data Access components.
- **`@Component`**: Generic Spring Bean.
- **`@Autowired`**: Injects dependencies.
- **`@Bean`**: Manually creates a Spring Bean (used in `SecurityConfig`).
- **`@RequestMapping / @GetMapping / @PostMapping`**: URL mapping.
- **`@RequestBody`**: Maps JSON to Java Object.
- **`@PathVariable / @RequestParam`**: Extracts URL parameters.
- **`@Entity / @Table / @Id / @Column`**: Database mapping.
- **`@ExceptionHandler / @RestControllerAdvice`**: Global Error Handling.
- **`@Configuration`**: Defines a configuration class.

### Security Concepts
- **`SecurityFilterChain`**: The chain of filters protecting the app.
- **`AuthenticationManager`**: The core interface that validates credentials.
- **`BCrypt`**: Hashing algorithm for passwords.
- **`UserDetails`**: Interface representing a security user.
- **`CORS`**: Cross-Origin Resource Sharing (allowing Frontend access).
- **`CSRF`**: Cross-Site Request Forgery (disabled for APIs).
- **`JWT`**: JSON Web Token (Stateless auth).

### Database Concepts
- **`JpaRepository`**: Interface for CRUD operations.
- **`Derived Queries`**: Methods like `findByEmail` that generate SQL.
- **`DTO (Data Transfer Object)`**: Plain Java objects used to decouple the Database Entity from the API Response.

### Architecture
- **Layered Architecture**: Controller -> Service -> Repository -> Database.
- **Dependency Injection (IoC)**: Spring manages object life cycles.
- **Restlessness**: API does not store state; every request is independent.
