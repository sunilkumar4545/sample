# Spring Framework Curriculum - Part 1 (Backend Core)

This document provides a topic-by-topic explanation of the Spring Framework, specifically mapped to the **MediaConnect** project.

---

## 1️⃣ Introduction to Spring Framework

### 💻 Code from Your Project
**File:** `Backend/src/main/java/com/mediaconnect/backend/MediaconnectApplication.java`

```java
@SpringBootApplication
public class MediaconnectApplication {
    public static void main(String[] args) {
        // This line starts the Spring Framework (IoC Container)
        SpringApplication.run(MediaconnectApplication.class, args);
    }
}
```

### 🟢 Simple Explanation
Think of **Spring** as a "manager" for your Java application. Instead of you manually creating objects (like `new UserServiceImpl()`) and connecting them together, Spring does it for you. It provides a huge toolbox (modules) to build web apps, extract data from databases, and secure your app without writing everything from scratch.

### 🔵 Technical Explanation
The **Spring Framework** is an open-source, enterprise-level framework for Java. Its core feature is the **Inversion of Control (IoC)** container, which manages the lifecycle and configuration of application objects (Beans). It promotes **Dependency Injection (DI)** to achieve loose coupling.

### 🧐 Why is it used?
In `MediaConnect`, we use Spring (specifically Spring Boot) to avoid writing boilerplate code for setting up a server, connecting to MySQL, and handling HTTP requests. The `@SpringBootApplication` annotation tells Spring: "This is the starting point, please scan my project and wire everything up."

### ⚙️ Internal Working
1.  **Scanning**: When `SpringApplication.run()` executes, Spring scans your package `com.mediaconnect.backend` and its sub-packages.
2.  **Registering**: It looks for classes marked with annotations like `@Component`, `@Service`, `@Controller`.
3.  **Wiring**: It creates objects (beans) from these classes and stores them in the **ApplicationContext** (the memory container).

### 🙋‍♂️ Interview Q&A

**Q1: What is the core feature of the Spring Framework?**
> **Answer:** The core feature is the **IoC Container** (Inversion of Control), which manages object creation and dependencies using Dependency Injection (DI).

**Q2: What is the difference between Spring and Spring Boot?**
> **Answer:** Spring is the framework itself. Spring Boot is an extension that provides "opinionated defaults" and auto-configuration to set up a Spring application quickly without minimal XML configuration.

**Q3: What are the main modules in Spring?**
> **Answer:** Core (IoC, DI), AOP (Aspect Oriented Programming), Data Access (JDBC, ORM), Web (MVC), and Test.

**Q4: What is tight coupling vs loose coupling?**
> **Answer:** Tight coupling is when classes directly create their dependencies (`new Service()`), making testing hard. Loose coupling (via Spring DI) means dependencies are provided externally, making code flexible and testable.

---

## 2️⃣ Spring with Maven

### 💻 Code from Your Project
**File:** `Backend/pom.xml`

```xml
<dependencies>
    <!-- We didn't download this jar manually; Maven did it -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Maven manages the version for us (mostly) via the parent -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
</dependencies>
```

### 🟢 Simple Explanation
**Maven** is a tool that goes to the internet (Maven Central), downloads the libraries (JAR files) your project needs, and puts them in your project. It also helps compile your code and build the final executable file.

### 🔵 Technical Explanation
Maven is a **build automation** and **dependency management** tool. It uses a `pom.xml` (Project Object Model) file to define the project structure, dependencies, and build plugins. It follows a standard lifecycle: `validate`, `compile`, `test`, `package`, `verify`, `install`, `deploy`.

### 🧐 Why is it used?
Without Maven, you would have to manually download `spring-web.jar`, `spring-core.jar`, `hibernate.jar`, etc., and add them to your classpath. Maven automates this. In `MediaConnect`, `spring-boot-starter-web` pulls in dozens of libraries (like Tomcat, Jackson) automatically.

### ⚙️ Internal Working
1.  **Reading POM**: Maven reads `pom.xml`.
2.  **Resolution**: It checks your local repository (`.m2` folder). If the library isn't there, it downloads it from the central repository.
3.  **Transitive Dependencies**: If Library A needs Library B, Maven downloads both automatically.

### 🙋‍♂️ Interview Q&A

**Q1: What is the purpose of `pom.xml`?**
> **Answer:** It stands for Project Object Model. It defines project configuration, dependencies, plugins, and build settings.

**Q2: What is a "Starter" dependency in Spring Boot?**
> **Answer:** A Starter (like `spring-boot-starter-web`) is a bundle of dependencies convenient for a specific purpose. It aggregates common libraries so you don't have to add them individually.

**Q3: Difference between `dependency` and `plugin` in Maven?**
> **Answer:** A **dependency** is a library your code uses (like Spring MVC). A **plugin** is a tool Maven uses to perform tasks (like compiling code or creating a JAR).

---

## 3️⃣ Setting up a Spring Project with Maven

### 💻 Code from Your Project
**Location:** Root Directory Structure
```text
Backend/
├── mvnw & mvnw.cmd      <-- The Maven Wrapper script
├── pom.xml              <-- The instructions
└── src/
    ├── main/
    │   ├── java/        <-- Your Java Code (com.mediaconnect...)
    │   └── resources/   <-- Config (application.properties)
```

### 🟢 Simple Explanation
Setting up means creating the folder structure that Maven understands. You put your Java code in `src/main/java` and your settings in `src/main/resources`.

### 🔵 Technical Explanation
A standard Maven project follows a specific directory layout convention (`Convention over Configuration`). Spring Boot initializes this via the Spring Initializr or CLI. The `mvnw` wrapper allows you to run Maven commands without having Maven installed globally.

### 🧐 Why is it used?
Using `mvnw` ensures that everyone working on `MediaConnect` uses the exact same version of Maven, preventing "it works on my machine" issues.

### ⚙️ Internal Working
When you run `./mvnw spring-boot:run`:
1.  The script checks if the compatible Maven version exists locally.
2.  It downloads Maven if missing.
3.  It executes the `spring-boot:run` goal which compiles the code and starts the embedded Tomcat server.

### 🙋‍♂️ Interview Q&A

**Q1: What is the Maven Lifecycle?**
> **Answer:** The sequence of phases: `clean` (delete target), `compile` (source to class), `test` (run JUnit), `package` (create JAR/WAR), `install` (save to local repo), `deploy` (push to remote server).

**Q2: Why do we use `mvnw` instead of just `mvn`?**
> **Answer:** The wrapper (`mvnw`) ensures the project runs with the specific Maven version defined in the project, guaranteeing consistency across different developer machines and CI/CD environments.

---

## 4️⃣ Spring IoC Container

### 💻 Code from Your Project
Because IoC happens "behind the scenes," we see its effects, not the container code itself.

**File:** `Backend/src/main/java/com/mediaconnect/backend/MediaconnectApplication.java`
```java
// When this runs, the ApplicationContext (IoC Container) is created.
SpringApplication.run(MediaconnectApplication.class, args);
```
**File:** `UserController.java`
```java
@Autowired // <-- We ask the IoC Container: "Give me the UserService object you made."
private UserService userService;
```

### 🟢 Simple Explanation
The **IoC (Inversion of Control) Container** is the "brain" of Spring. Usually, your code controls the flow (creating objects). In Spring, you give control to the container. It creates the objects (Beans) and keeps them alive in memory for you to use.

### 🔵 Technical Explanation
The IoC Container is responsible for instantiating, configuring, and assembling beans. The implementation is the **ApplicationContext** interface (e.g., `AnnotationConfigApplicationContext` in Boot). It manages the complete lifecycle of beans.

### 🧐 Why is it used?
It decouples components. `UserController` doesn't need to know *how* to create `UserService`. It just asks the container for it. This makes swapping implementations (e.g., a mock service for testing) easy.

### ⚙️ Internal Working
1.  **Instantiation**: The container finds `@Component`, `@Service`, etc.
2.  **Population**: It creates the object instances (using `new` internally via Reflection).
3.  **Property Filling**: It injects dependencies (`@Autowired`).
4.  **Initialization**: It runs any `@PostConstruct` methods.
5.  **Ready**: The bean is ready for use.

### 🙋‍♂️ Interview Q&A

**Q1: What is Inversion of Control (IoC)?**
> **Answer:** It is a design principle where the control of object creation and flow is transferred from the programmer to a framework (container).

**Q2: ApplicationContext vs BeanFactory?**
> **Answer:** `BeanFactory` is the basic container with lazy loading. `ApplicationContext` extends it, adding enterprise features like Event publication, AOP integration, and eager loading (creating beans at startup). Spring Boot uses `ApplicationContext`.

**Q3: What is the default scope of a Spring Bean?**
> **Answer:** **Singleton**. One instance per ApplicationContext.

---

## 5️⃣ Spring Bean Configuration

### 💻 Code from Your Project

**Method 1: Component Scanning (Most of your code)**
**File:** `Backend/src/main/java/com/mediaconnect/backend/service/implementations/UserServiceImpl.java`
```java
@Service // <-- Tells Spring: "Make this a Bean! It holds business logic."
public class UserServiceImpl implements UserService { ... }
```

**Method 2: Java Configuration (Explicit)**
**File:** `Backend/src/main/java/com/mediaconnect/backend/config/SwaggerConfig.java`
```java
@Configuration // <-- Tells Spring: "This class contains bean definitions."
public class SwaggerConfig {
    
    @Bean // <-- Tells Spring: "Run this method and manage the returned object as a Bean."
    public OpenAPI customOpenAPI() { 
        return new OpenAPI()...; 
    }
}
```

### 🟢 Simple Explanation
A **Bean** is just a Java object that Spring manages. We need to tell "The Manager" (Spring) which objects to manage. We do this by putting stickers (annotations) on our classes like "This is a Service" or "This is a Controller."

### 🔵 Technical Explanation
Bean configuration defines how beans are created and wired.
Two main ways:
1.  **Component Scanning**: Using `@Component`, `@Service`, `@Repository`, `@Controller`.
2.  **Java Configuration**: Using `@Configuration` classes and `@Bean` methods (for third-party libs).

### 🧐 Why is it used?
*   `@Service` etc. are used for **your** classes because it's faster.
*   `@Bean` is used for **external classes** (like `OpenAPI`) because you can't edit their source code to add `@Component`.

### ⚙️ Internal Working
Spring Boot's `@SpringBootApplication` includes `@ComponentScan`. It scans the current package and sub-packages. When it finds a class with a stereotype annotation (`@Service`), it creates an instance and registers it in the context map.

### 🙋‍♂️ Interview Q&A

**Q1: Difference between `@Component`, `@Service`, `@Repository`, and `@Controller`?**
> **Answer:**
> *   `@Component`: Generic stereotype for any Spring-managed component.
> *   `@Service`: Specialization of Component for business logic layer.
> *   `@Repository`: Specialization for DAO layer (catches DB exceptions).
> *   `@Controller`: Specialization for Web MVC layer.
> Technically, they are all the same, but they show intent.

**Q2: Can we declare a bean without an annotation?**
> **Answer:** Yes, by defining it in XML (legacy) or using a `@Bean` method inside a `@Configuration` class.

---

## 6️⃣ Dependency Injection (DI) in Spring

### 💻 Code from Your Project
**File:** `Backend/src/main/java/com/mediaconnect/backend/service/implementations/UserServiceImpl.java`

```java
@Service
public class UserServiceImpl implements UserService {

    @Autowired  // <-- FIELD INJECTION
    private UserRepository userRepository;

    // ... code using userRepository
}
```

### 🟢 Simple Explanation
**DI** is how Spring gives an object what it needs. If `UserController` needs `UserService`, simplified "The Manager" injects it. You don't say `userService = new UserServiceImpl()`. You just say `@Autowired`, and it magically appears.

### 🔵 Technical Explanation
Dependency Injection is the implementation of IoC. There are three types:
1.  **Constructor Injection** (Recommended).
2.  **Field Injection** (Used in your project).
3.  **Setter Injection**.

### 🧐 Why is it used?
It removes hard dependencies. `UserServiceImpl` relies on an *interface* (`UserRepository`), not a specific database implementation. Spring finds the implementation and plugs it in.

### ❌ Why Field Injection Explained (and strict advice)
Your project uses **Field Injection** (`@Autowired` on variables).
*   **Pros**: Very concise/short code.
*   **Cons**: Hard to unit test (can't pass mocks without reflection), hides dependencies.
*   **Professional Note**: In interviews, say you know **Constructor Injection** is better (because it allows final fields and easier testing), but Field Injection is acceptable for simple projects.

### ⚙️ Internal Working
1.  Spring sees `@Autowired`.
2.  It looks in its bag of Beans for something that matches the type `UserRepository`.
3.  Are there multiple matches? No -> Inject. Yes -> Check `@Qualifier`.

### 🙋‍♂️ Interview Q&A

**Q1: What are the types of DI?**
> **Answer:** Constructor, Setter, and Field injection.

**Q2: Which DI is recommended and why?**
> **Answer:** **Constructor Injection** is recommended. It ensures the bean is immutable (fields can be `final`), prevents partial initialization (object works immediately), and makes Unit Testing easier (no reflection needed).

**Q3: What happens if Spring finds two beans of the same type?**
> **Answer:** It throws a `NoUniqueBeanDefinitionException`. You must resolve it using `@Qualifier("beanName")` or `@Primary`.

---

## 7️⃣ Spring AOP (Aspect-Oriented Programming)

### 💻 Code from Your Project
The best example in `MediaConnect` is the **Global Exception Handler**. It "intercepts" errors from ALL controllers without modifying the controllers themselves.

**File:** `Backend/src/main/java/com/mediaconnect/backend/exception/GlobalExceptionHandler.java`

```java
@RestControllerAdvice // <-- This is an Aspect! It watches all RestControllers.
public class GlobalExceptionHandler {

    // Advice: What to do
    // Pointcut: When ResourceNotFoundException happens
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleResourceNotFound(ResourceNotFoundException ex) {
        // ... logic to return nice JSON error
        return new ResponseEntity<>(error, HttpStatus.NOT_FOUND);
    }
}
```

### 🟢 Simple Explanation
**AOP** allows you to separate "cross-cutting" logic (stuff that applies everywhere, like logging or error handling) from your main business logic. Instead of writing `try-catch` in every single function, AOP lets you write it once and apply it globally.

### 🔵 Technical Explanation
AOP breaks program logic into distinct parts called **concerns**.
*   **Aspect**: The module handling the concern.
*   **Advice**: *What* action to take (code).
*   **Pointcut**: *Where* to apply the action.
*   **JoinPoint**: The specific point of execution.

### 🧐 Why is it used?
If we didn't use this, every method in `UserController` would need a `try-catch` block, making the code messy and repetitive.

### ⚙️ Internal Working
Spring creates a **Proxy** (a wrapper object) around your Controller. When a request comes in:
1.  The Proxy receives it.
2.  It runs the real method.
3.  If an exception is thrown, the Proxy catches it and delegates it to the `@RestControllerAdvice` class.

### 🙋‍♂️ Interview Q&A

**Q1: What is a Cross-Cutting Concern?**
> **Answer:** It is functionality that affects multiple layers of an application, such as Logging, Security, Transaction Management, and Exception Handling.

**Q2: What is the difference between specific AOP and Filters?**
> **Answer:** Filters (Servlet Filters) run outside the Spring Context (before the request hits the DispatcherServlet). AOP runs inside Spring (wrapping Beans).

**Q3: Does Spring AOP modify the byte code?**
> **Answer:** No, it uses **Dynamic Proxies** (JDK Proxy or CGLIB) at runtime to wrap the targeted beans.

---

## 8️⃣ Spring MVC and ORM

### 💻 Code from Your Project (MVC Layer)
**File:** `Backend/src/main/java/com/mediaconnect/backend/controller/UserController.java`
```java
@RestController // <-- MVC Controller
@RequestMapping("/api/users")
public class UserController {
    @GetMapping("/me") // <-- Maps HTTP GET request
    public ResponseEntity<UserDTO> getCurrentUser() { ... }
}
```

### 💻 Code from Your Project (ORM Layer)
**File:** `Backend/src/main/java/com/mediaconnect/backend/entity/User.java`
```java
@Entity // <-- ORM: Maps this class to a DB Table
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    // ...
}
```

**File:** `Backend/src/main/java/com/mediaconnect/backend/repository/UserRepository.java`
```java
// Spring Data JPA creates the implementation automatically!
public interface UserRepository extends JpaRepository<User, Long> {
    User findByEmail(String email); // <-- Generates SQL: SELECT * FROM users WHERE email = ?
}
```

### 🟢 Simple Explanation
**MVC** (Model-View-Controller) is how we handle web requests. user asks -> Controller -> Service -> Database.
**ORM** (Object Relational Mapping) is how we treat database tables (SQL) as Java Classes. You don't write `SELECT * FROM users`. You write `userRepository.findAll()`.

### 🔵 Technical Explanation
*   **MVC**: Spring Web MVC is built on the Servlet API. The `DispatcherServlet` acts as the Front Controller handling all incoming requests.
*   **ORM**: Uses Hibernate (via Spring Data JPA) to map POJOs (Entities) to RDBMS tables.

### 🧐 Why is it used?
*   **MVC**: Gives us clear structure. API endpoints are distinct from Logic.
*   **ORM**: Saves huge amounts of time not writing raw SQL queries.

### ⚙️ Internal Working
*   **MVC Flow**: Remote Request -> `DispatcherServlet` -> HandlerMapping -> `UserController` -> Service.
*   **JPA Flow**: `findAll()` call -> Spring Data Proxy -> Hibernate Session -> JDBC Driver -> SQL Query.

### 🙋‍♂️ Interview Q&A

**Q1: What is the DispatcherServlet?**
> **Answer:** It is the **Front Controller** in Spring MVC. It receives all incoming HTTP requests and delegates them to the appropriate controllers.

**Q2: JPA vs Hibernate?**
> **Answer:** JPA is the **Interface** (Standard specification). Hibernate is the **Implementation** (Library code). Spring Data JPA is a layer on top that simplifies using JPA.

**Q3: What is `@RestController`?**
> **Answer:** It is a convenience annotation combining `@Controller` and `@ResponseBody`. It implies that methods return domain objects (JSON/XML) directly, not HTML Views.

---

## 9️⃣ Spring Boot (Introduction)

### 💻 Code from Your Project
**File:** `Backend/src/main/resources/application.properties`

```properties
# We just add properties, Spring Boot does the heavy lifting
spring.datasource.url=jdbc:mysql://localhost:3306/mediaconnect
spring.datasource.username=root
spring.jpa.hibernate.ddl-auto=update
```

Because of `spring-boot-starter-data-jpa` and these properties, Spring Boot automatically configures the **DataSource**, **EntityManager**, and **TransactionManager** beans.

### 🟢 Simple Explanation
Spring Boot is the "Easy Button" for Spring. Standard Spring requires a lot of setup (XML, server config). Spring Boot guesses what you need (`AutoConfiguration`). If it sees "MySQL" in your specific files, it sets up MySQL automatically.

### 🔵 Technical Explanation
Spring Boot makes it easy to create stand-alone, production-grade Spring-based Applications.
Key features:
1.  **Auto-Configuration**.
2.  **Starter Dependencies**.
3.  **Embedded Server** (Tomcat/Jetty).
4.  **Production-ready metrics** (Actuator, though not used in your project yet).

### 🧐 Why is it used?
To reduce development time. We focus on business logic (`UserService`), not infrastructure config.

### ⚙️ Internal Working
At startup (`@SpringBootApplication`), the `@EnableAutoConfiguration` annotation tells Boot to look at the classpath.
*   "Oh, I see `mysql-connector.jar` on the classpath?" -> "I will configure a MySQL DataSource."
*   "Oh, I see `spring-web.jar`?" -> "I will configure an embedded Tomcat server on port 8080 (or 8081 as defined)."

### 🙋‍♂️ Interview Q&A

**Q1: How does Spring Boot Auto-Configuration work?**
> **Answer:** It scans the classpath dependencies. If specific JARs are found (like H2 or MySQL), it configures relevant beans automatically unless you define your own.

**Q2: Can we override default properties?**
> **Answer:** Yes, via `application.properties`, environment variables, or YAML files.

**Q3: What is the embedded server?**
> **Answer:** Spring Boot includes a web server (Tomcat by default) inside the JAR file. You don't need to install a separate external Tomcat; you just run the Java application.

---

## 🔟 Reactive Programming with Spring WebFlux

### 💻 Code Mapping
**❌ NOT USED IN YOUR PROJECT.**

**Map to Project:**
Your project uses **Spring MVC** (Blocking I/O).
`Backend/pom.xml`: `spring-boot-starter-web` (Standard MVC), NOT `spring-boot-starter-webflux`.

**Why not used?**
WebFlux is complex and useful for high-concurrency streaming services (like actual Netflix video streaming servers). For the metadata API (Login, get plans, get movie descriptions), standard MVC is easier to write and debug, and fast enough.

### 🟢 Simple Explanation
Standard Spring MVC (which we use) is like a waiter who waits at the table until the customer orders, then waits for the kitchen, then brings food. One waiter per table.
**WebFlux** is like a waiter who takes an order, runs to the kitchen, then immediately goes to another table. One waiter handles many tables at once. It's used for super-high performance.

### 🔵 Technical Explanation
Reactive programming is a paradigm about non-blocking, asynchronous data streams. **Spring WebFlux** uses Project Reactor (`Mono` and `Flux`).
*   **Mono**: Returns 0 or 1 result (`Mono<User>`).
*   **Flux**: Returns 0 to N results (`Flux<Movie>`).

### ⚙️ Internal Working (If used)
It runs on **Netty** (an event loop server) instead of Tomcat (thread-per-request). It never "blocks" a thread waiting for a database response.

### 🙋‍♂️ Interview Q&A

**Q1: What is the difference between Spring MVC and Spring WebFlux?**
> **Answer:**
> *   **MVC**: Blocking (Synchronous). Uses Thread-per-request model. Good for traditional CRUD.
> *   **WebFlux**: Non-blocking (Asynchronous). Uses Event-loop model. Good for high concurrency or streaming.

**Q2: What are Mono and Flux?**
> **Answer:** Types from Project Reactor. `Mono` acts as a publisher for 0 or 1 item. `Flux` acts as a publisher for 0 to N items.

**Q3: Can we use JDBC with WebFlux?**
> **Answer:** Traditionally no, because JDBC is blocking. You need R2DBC (Reactive Relational Database Connectivity) drivers for true reactive SQL access.

---

## 🧠 Quick Recap (Sharpen Your Memory)

| Topic | 🟢 Simple (The Hook) | 🔵 Technical (The Keyword) |
| :--- | :--- | :--- |
| **Intro to Spring** | The "Manager" who connects your objects. | **IoC (Inversion of Control)** and **DI (Dependency Injection)**. |
| **Spring Modules** | The Toolbox (Core, Web, Data). | **Core, AOP, Data Access, Web (MVC)**. |
| **Maven** | The "Amazon Delivery" for JARs. | **Build Automation** & **Dependency Management**. |
| **IoC Container** | The Brain/Engine that runs everything. | `ApplicationContext` manages the **Life Cycle** of Beans. |
| **Bean Config** | Putting "Stickies" on classes (`@Service`). | **Component Scanning** (Annotations) vs **Java Config** (`@Configuration`). |
| **Dependency Injection** | Getting what you need without asking `new`. | **Constructor**, **Setter**, **Field Injection**. |
| **AOP** | The "Security Guard" (checks every request). | **Aspects**, **Advice**, **Pointcuts**, **Proxies** (Cross-Cutting Concerns). |
| **Spring MVC** | The Waiter (Request -> Response). | **DispatcherServlet**, **Front Controller** Pattern. |
| **Spring Boot** | The "Easy Button" (Auto-start). | **Auto-Configuration**, **Starters**, **Embedded Server**. |
| **WebFlux** | The Super-Fast Multi-tasking Waiter. | **Reactive Programming**, **Non-blocking I/O**, **Event Loop**. |

---
