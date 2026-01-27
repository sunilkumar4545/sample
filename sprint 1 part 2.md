# Spring Framework Curriculum - Part 2 (Data Persistence)

This document maps **Spring Data JPA, Hibernate, and Entity Mapping** concepts directly to your **MediaConnect** project.

---

## 1️⃣ Introduction to Spring Data JPA

### 💻 Code from Your Project
**File:** `Backend/src/main/java/com/mediaconnect/backend/repository/UserRepository.java`

```java
public interface UserRepository extends JpaRepository<User, Long> {
    // 🪄 Magic: We didn't write the SQL definition for this method!
    User findByEmail(String email);
}
```

### 🔵 Technical Explanation
*   **JPA (Java Persistence API):** A specification (interface) for object-relational mapping in Java.
*   **Hibernate:** The most popular *implementation* of JPA. It does the actual work of mapping Java objects to DB tables.
*   **Spring Data JPA:** An abstraction layer *on top of* JPA/Hibernate. It reduces boilerplate code by providing standard repositories (`JpaRepository`) with built-in methods like `save()`, `findAll()`, `delete()`.

### 🧐 Why is it used?
In `MediaConnect`, we need to store Users, Movies, and WatchHistory. Writing raw JDBC code for CRUD operations is tedious and error-prone. Spring Data JPA allows us to interact with the database using simple Java interfaces, speeding up development by 70-80%.

### ⚙️ Internal Working
1.  **Proxy Creation:** When the app starts, Spring creates a dynamic proxy implementation of your `UserRepository` interface.
2.  **Method Parsing:** For `findByEmail(String email)`, it analyzes the name:
    *   `find` -> `SELECT`
    *   `By` -> `WHERE`
    *   `Email` -> `email = ?`
3.  **Execution:** It uses the underlying **EntityManager** (provided by Hibernate) to execute the generated JPQL/SQL query.

### 🙋‍♂️ Interview Q&A

**Q1: What is the difference between JPA, Hibernate, and Spring Data JPA?**
> **Answer:**
> *   **JPA**: The *Specification* (Interface/Rules).
> *   **Hibernate**: The *Implementation* (The code that follows the rules).
> *   **Spring Data JPA**: A framework that simplifies using JPA by removing boilerplate code (Repository pattern).

**Q2: What is the `JpaRepository` interface?**
> **Answer:** It is a Spring Data interface that extends `PagingAndSortingRepository` and `CrudRepository`. It provides JPA specific methods like flushing the persistence context and deleting records in batch.

**Q3: Can we use Spring Data JPA without Hibernate?**
> **Answer:** Yes, theoretically, if you use another JPA implementation like EclipseLink, but Hibernate is the default and most common choice in Spring Boot.

---

## 2️⃣ Setting Up a Spring Boot Project with Spring Data JPA

### 💻 Code from Your Project
**File:** `Backend/pom.xml`
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId> <!-- Adds Hibernate + Spring Data -->
</dependency>
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId> <!-- Driver to talk to MySQL -->
    <scope>runtime</scope>
</dependency>
```

**File:** `Backend/src/main/resources/application.properties`
```properties
spring.datasource.url=jdbc:mysql://localhost:3306/mediaconnect?createDatabaseIfNotExist=true
spring.datasource.username=root
spring.datasource.password=root
# Automatically updates the database schema when Java code changes
spring.jpa.hibernate.ddl-auto=update
# Shows the exact SQL Hibernate generates in the console
spring.jpa.show-sql=true
```

### 🔵 Technical Explanation
Setting up involves:
1.  **Dependencies**: Adding `spring-boot-starter-data-jpa` pulls in Hibernate, Jakarta Persistence API, and Spring Data libraries. The JDBC Driver (MySQL) is required for physical connection.
2.  **DataSource**: Spring Boot's **Auto-Configuration** detects these dependencies and creates a `DataSource` bean connecting to the URL provided in properties.
3.  **EntityManagerFactory**: It also auto-configures the `EntityManagerFactory` and `TransactionManager`.

### 🧐 Why is it used?
Manual configuration of `SessionFactory`, `DataSource`, and transaction management in standard Spring (non-Boot) takes massive XML or Java config files. Spring Boot does this in 4 lines of `application.properties`.

### ⚙️ Internal Working
*   `ddl-auto=update`: When the app starts, Hibernate looks at your Java Entities (`User`, `Movie`). It compares them to the actual Database Tables. If a column is missing in the DB, it adds it (`ALTER TABLE`).
*   `show-sql=true`: Hibernate prints the SQL it generates to `System.out`, helping you debug.

### 🙋‍♂️ Interview Q&A

**Q1: What does `spring.jpa.hibernate.ddl-auto` do?**
> **Answer:** It configures the database schema generation behavior. Common values:
> *   `none`: Do nothing (Production).
> *   `update`: Update the schema if changed (Development).
> *   `create`: Drop and create tables at startup.
> *   `create-drop`: Create at startup, drop at shutdown.

**Q2: How does Spring Boot know which database to connect to?**
> **Answer:** It looks for `spring.datasource.*` properties in `application.properties`. If `HikariCP` (default pool) is on the classpath, it configures the DataSource using those credentials.

**Q3: Why isn't `mysql-connector` scope provided as `compile`?**
> **Answer:** It uses `runtime` scope because our code relies on JDBC interfaces (standard Java), not MySQL-specific classes directly. The driver is only needed effectively when the application runs.

---

## 3️⃣ Entity Mapping

### 💻 Code from Your Project
**File:** `Backend/src/main/java/com/mediaconnect/backend/entity/User.java`
```java
@Entity // 1. Tells Hibernate: "This class maps to a table"
@Table(name = "users") // 2. Specific table name
public class User {

    @Id // 3. Primary Key
    @GeneratedValue(strategy = GenerationType.IDENTITY) // 4. Auto Increment
    private Long id;

    @Column(nullable = false, unique = true) // 5. Column Constraints
    private String email;
    
    // ...
}
```

**File:** `Backend/src/main/java/com/mediaconnect/backend/entity/WatchHistory.java`
```java
@Entity
@Table(name = "watch_history")
public class WatchHistory {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne // 6. Relationship: Many History records -> One User
    @JoinColumn(name = "user_id", nullable = false) // Foreign Key
    private User user;

    @ManyToOne // Relationship: Many History records -> One Movie
    @JoinColumn(name = "movie_id", nullable = false)
    private Movie movie;
}
```

### 🔵 Technical Explanation
JPA allows mapping Plain Old Java Objects (POJOs) to Relational Database Tables using **Annotations**.
*   **@Entity**: Marks a class as a JPA entity.
*   **@Table**: Specify details like exact table name, indexes, or schema.
*   **@Id** & **@GeneratedValue**: Defines the Primary Key (PK) and its generation strategy (Identity, Sequence, Auto).
*   **@ManyToOne**: Defines a Master-Detail relationship. It typically owns the foreign key (`@JoinColumn`).

### 🧐 Why is it used?
In `MediaConnect`, users watch movies.
*   We created `WatchHistory` to track this.
*   Ideally, we want to know *which* user watched *which* movie.
*   The `@ManyToOne` annotations in `WatchHistory` allow us to do `watchHistory.getUser().getEmail()` easily in Java.

### ⚙️ Internal Working
*   **@GeneratedValue(IDENTITY)**: Hibernate relies on the Database's auto-increment feature (MySQL `AUTO_INCREMENT`). It inserts a null ID, lets DB generate it, and then retrieves it back.
*   **Relationships**: When you load a `WatchHistory` object, Hibernate can also load the associated `User` object (either instantly via Eager Loading or when asked via Lazy Loading).

### 🙋‍♂️ Interview Q&A

**Q1: What is the difference between `@Entity` and `@Table`?**
> **Answer:** `@Entity` is mandatory (maps class to DB entity). `@Table` is optional; it's used to specify details like the exact table name (if different from class name), indexes, or schema.

**Q2: What is the difference between `GenerationType.IDENTITY` and `SEQUENCE`?**
> **Answer:**
> *   `IDENTITY`: Relies on the database's auto-increment column (efficient for MySQL).
> *   `SEQUENCE`: Uses a database sequence object (Oracle/PostgreSQL). Hibernate gets the next ID *before* inserting.

**Q3: Explain `@ManyToOne` vs `@OneToMany`.**
> **Answer:**
> *   `@ManyToOne`: (Child side) Multiple records here point to one record there (e.g., Many Employees belong to One Department). Holds the Foreign Key.
> *   `@OneToMany`: (Parent side) One record here has a list of records there.

**Q4: What is Lazy vs Eager Loading?**
> **Answer:**
> *   **Eager**: Fetches related data immediately with the main query (JOIN).
> *   **Lazy**: Fetches related data only when `getRelated()` is called (Separate SELECT). Default for `@OneToMany` is Lazy; `@ManyToOne` is Eager.

---

## 4️⃣ Spring Data Repositories

### 💻 Code from Your Project
**File:** `Backend/src/main/java/com/mediaconnect/backend/repository/MovieRepository.java`

```java
@Repository
public interface MovieRepository extends JpaRepository<Movie, Long> {
    
    // We didn't write logic for this! Spring created this query automatically!
    List<Movie> findByTitleContainingIgnoreCase(String title);
    
    // Custom SQL? No problem.
    @Query(value = "SELECT * FROM movies ORDER BY views DESC LIMIT 10", nativeQuery = true)
    List<Movie> findTop10ByOrderByViewsDesc();
}
```

### 🔵 Technical Explanation
A Spring Data Repository is an interface that provides data access functionality. By extending `JpaRepository<Entity, ID>`, you inherent standard CRUD operations (`save`, `delete`, `findAll`) plus paging methods.
*   **Derived Queries**: Generates SQL based on method naming convention (`findByTitle...`).
*   **@Query**: Allows defining custom HQL (Hibernate Query Language) or Native SQL queries.

### 🧐 Why is it used?
To keep our code clean. We don't want SQL inside our Service logic. The Repository layer handles *data access* exclusively.

### ⚙️ Internal Working
At runtime, Spring uses JDK Dynamic Proxies to create an instance of your interface.
*   For derived queries (`findByTitle`), it parses the method name tokens (`find`, `By`, `Title`) and constructs a Criteria Query equivalent to `SELECT * FROM movies WHERE title = ?`.
*   For `@Query`, it executes the SQL/JPQL string directly when the method is called.

### 🙋‍♂️ Interview Q&A

**Q1: What are Derived Query Methods?**
> **Answer:** Methods in the repository interface that Spring Data parses to generate SQL automatically based on the name (e.g., `findByNameAndAge`).

**Q2: What is nativeQuery = true in @Query?**
> **Answer:** It tells Hibernate to execute the query exactly as written in standard SQL (targeting tables) instead of JPQL (targeting Entities).

**Q3: Can we add custom logic to a Repository?**
> **Answer:** Interfaces can't have logic, but you can create a implementation class (e.g., `MovieRepositoryImpl`) or use Default Methods (Java 8+) to add custom behavior.

---

## 5️⃣ CRUD Operations with Spring Data JPA

### 💻 Code from Your Project
**Concept Usage:** `Backend/src/main/java/com/mediaconnect/backend/service/implementations/MovieServiceImpl.java`

```java
@Autowired
private MovieRepository movieRepository;

public MovieDTO createMovie(MovieDTO dto) {
    Movie movie = mapToEntity(dto);
    
    // 1. Create (C)
    movieRepository.save(movie); // <-- Inherited method!
    
    return mapToDTO(movie);
}

public List<MovieDTO> getAllMovies() {
    // 2. Read (R)
    List<Movie> movies = movieRepository.findAll(); // <-- Inherited method!
    return ...;
}
```

### 🔵 Technical Explanation
`JpaRepository` provides generalized CRUD methods:
*   `save(S entity)`: If the Entity ID is null, it performs `persist()` (INSERT). If ID exists, it performs `merge()` (UPDATE).
*   `findById(ID id)`: Returns an `Optional<T>`, encouraging null-safety checking.
*   `findAll()`: Returns `List<T>`.

### 🧐 Why is it used?
It is the heart of `MediaConnect`. Every time a user registers (Create), logs in (Read), watches a movie (Update/Create history), or cancels a sub (Update), we rely on these CRUD methods.

### ⚙️ Internal Working
Hibernate maintains a **Persistence Context** (First Level Cache).
*   When you call `save()`, the entity is attached to the context.
*   Hibernate waits until the transaction ends (or `flush()` is called) to send the SQL to the DB. This is "Transactional Write-Behind" for performance.

### 🙋‍♂️ Interview Q&A

**Q1: How does `save()` decide between Insert and Update?**
> **Answer:** It checks if the Primary Key (ID) is null. If null (or 0), it assumes a new record (INSERT). If the ID exists, it assumes an update (MERGE).

**Q2: What is the return type of `findById()`?**
> **Answer:** It returns `Optional<T>`. This forces the developer to handle the "Not Found" case explicitly using `.orElse()` or `.orElseThrow()`.

**Q3: Does `deleteById()` issue the delete query immediately?**
> **Answer:** It typically fetches the entity first to ensure it exists (triggering lifecycle events), then issues the delete.

---

## 6️⃣ Query Methods and Named Queries

### 💻 Code from Your Project
**File:** `Backend/src/main/java/com/mediaconnect/backend/repository/WatchHistoryRepository.java`

```java
// 1. Derived Query Method (Keywords: And, OrderBy, Desc)
List<WatchHistory> findByUserIdOrderByLastWatchedDesc(Long userId);

// 2. Dynamic JPQL Query (using @Query)
@Query("SELECT new com.mediaconnect.backend.dto.response.AnalyticsResponse(" +
        "m.id, m.title, COUNT(DISTINCT wh.user.id)) " +
        "FROM Movie m LEFT JOIN WatchHistory wh ON ...")
List<AnalyticsResponse> findEngagementAnalytics();
```

### 🔵 Technical Explanation
*   **Query Keywords**: Keywords are parsed into a predicate. Example: `findTop5ByUserIdOrderByLastWatchedDesc`.
    *   `Top5`: Limit result set.
    *   `ByUserId`: `WHERE user_id = ?`.
    *   `OrderByLastWatchedDesc`: `ORDER BY last_watched DESC`.
*   **Named Queries**: Pre-defined queries on the Entity class using `@NamedQuery`. (Note: **Not used** in your project as it clutters Entity classes; we prefer `@Query` in Repositories).
*   **Dynamic Queries**: Queries constructed at runtime. You used `@Query` which is static but processes dynamic parameters. For truly dynamic filtering, we would use `Specifications` (Criteria API).

### 🧐 Why is it used?
The `findEngagementAnalytics()` method in your project is too complex for a derived name (it joins tables, groups data, and counts). So we use `@Query`.

### ⚙️ Internal Working
JPQL queries are compiled by the EntityManager on startup. They are cached to avoid re-parsing overhead. This makes `@Query` slightly faster than constructing queries dynamically every time.

### 🙋‍♂️ Interview Q&A

**Q1: What are Named Queries?**
> **Answer:** Queries defined via `@NamedQuery` annotation on the Entity class itself. They are validated at startup. However, they couple the Entity with the Query logic, so `@Query` in repositories is often preferred.

**Q2: What is JPQL?**
> **Answer:** Java Persistence Query Language. It queries the *Entity Model* rather than the Data Model. It is portable across different databases (MySQL, Oracle, etc.).

**Q3: How do you handle pagination in custom queries?**
> **Answer:** You can pass a `Pageable` parameter to the repository method (e.g., `PageRequest.of(0, 10)`), and return a `Page<T>` object.

---

## 7️⃣ Pagination and Sorting

### 💻 Code from Your Project (Implementation)
While your current `MovieController` uses simple lists, here is how Pagination **WOULD** be implemented in your project to handle thousands of movies efficiently.

**File:** `Backend/src/main/java/com/mediaconnect/backend/repository/MovieRepository.java`
```java
// Logic provided by JpaRepository extension
// public interface MovieRepository extends JpaRepository<Movie, Long> { ... }
```

**Service Layer Implementation (Hypothetical Upgrade):**
```java
public Page<MovieDTO> getMovies(int page, int size, String sortBy) {
    // 1. Create Pageable object
    Pageable pageable = PageRequest.of(page, size, Sort.by(sortBy).descending());
    
    // 2. Fetch Page<Entity>
    Page<Movie> moviePage = movieRepository.findAll(pageable);
    
    // 3. Convert to Page<DTO>
    return moviePage.map(this::mapToDTO);
}
```

### 🔵 Technical Explanation
**Pagination** refers to dividing a large dataset into smaller chunks (pages) to improve performance and user experience. **Sorting** orders these records.
In Spring Data JPA, `PagingAndSortingRepository` (extended by `JpaRepository`) provides:
*   `findAll(Pageable pageable)`: Returns a `Page<T>` containing metadata (total pages, total elements) and the data itself.
*   **Pageable Interface**: Represents query parameters like page number (0-indexed), size, and sort order.
*   **Slice vs Page**: `Page` executes an extra COUNT query to know the total elements. `Slice` only knows if there is a "next" slice (keeps the query cheaper).

### ⚙️ Internal Working
When `findAll(pageable)` is called, Hibernate generates **two SQL queries**:
1.  **Data Query**: `SELECT * FROM movies ORDER BY views DESC LIMIT ? OFFSET ?` (MySQL specific using LIMIT/OFFSET).
2.  **Count Query**: `SELECT COUNT(*) FROM movies` (Only if returning `Page`, not `Slice`).

This mechanism prevents loading the entire table into the Java Heap, preventing `OutOfMemoryError` on large datasets.

### 🙋‍♂️ Interview Q&A
**Q1: What is the difference between `Page` and `Slice`?**
> **Answer:** `Page` includes the total count of records and total pages (expensive COUNT query). `Slice` only creates a boolean flag `hasNext()` indicating if more data exists (cheaper, no COUNT query).

**Q2: How do you handle sorting dynamically?**
> **Answer:** By passing a `Sort` object to the `PageRequest`. E.g., `Sort.by(Sort.Direction.DESC, "releaseDate")`.

---

## 8️⃣ Auditing with Spring Data JPA

### 💻 Code Mapping
**❌ NOT YET USED IN YOUR PROJECT.**

**Where it WOULD be used:**
In `User.java` or `SubscriptionPlan.java` to track *when* a record was created and *who* created it.

**Example Implementation:**
```java
@EntityListeners(AuditingEntityListener.class) // 1. Listener
@MappedSuperclass
public abstract class BaseEntity {
    
    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;
    
    @LastModifiedDate
    private LocalDateTime updatedAt;
    
    @CreatedBy
    private String createdBy; // User who did the action
}
```

### 🔵 Technical Explanation
**JPA Auditing** automatically populates fields responsible for tracking the modification history of an entity.
Key Annotations:
*   `@CreatedDate`: Sets timestamp on persist.
*   `@LastModifiedDate`: Updates timestamp on every update.
*   `@CreatedBy` / `@LastModifiedBy`: Sets the current user (requires `AuditorAware` implementation).
*   `@EntityListeners(AuditingEntityListener.class)`: Must be added to the entity to trigger the capture.

To enable, you must add `@EnableJpaAuditing` to the main configuration class.

### ⚙️ Internal Working
JPA uses **Entity Lifecycle Events** (`@PrePersist`, `@PreUpdate`).
1.  When `save()` is called, the `AuditingEntityListener` intercepts the call *before* it hits the database.
2.  it uses reflection to inject `LocalDateTime.now()` into fields marked with `@CreatedDate`.
3.  If `@CreatedBy` is used, it calls the `AuditorAware.getCurrentAuditor()` bean to fetch the logged-in user from `SecurityContextHolder`.

### 🙋‍♂️ Interview Q&A
**Q1: How do you enable JPA Auditing?**
> **Answer:** Apply `@EnableJpaAuditing` on a configuration class and attach `@EntityListeners(AuditingEntityListener.class)` to the entities.

**Q2: How does Spring know "Who" modified the record?**
> **Answer:** You must implement the `AuditorAware<T>` interface, typically fetching the principal from Spring Security's `SecurityContext`.

---

## 9️⃣ Spring Data JPA Projections

### 💻 Code from Your Project
**File:** `Backend/src/main/java/com/mediaconnect/backend/repository/WatchHistoryRepository.java`

```java
// CLASS-BASED PROJECTION (DTO)
@Query("SELECT new com.mediaconnect.backend.dto.response.AnalyticsResponse(" +
        "m.id, m.title, m.genres, m.language, COUNT(DISTINCT wh.user.id), COALESCE(SUM(wh.progressSeconds), 0L)) " +
        "FROM Movie m ...")
List<AnalyticsResponse> findEngagementAnalytics();
```

### 🔵 Technical Explanation
**Projections** allow fetching only a *subset* of attributes rather than the entire Entity graph. This optimizes performance by reducing network bandwidth and memory usage.
Types:
1.  **Interface-based (Closed)**: Spring proxies an interface with getter methods matching entity properties.
2.  **Class-based (DTO)**: Uses a constructor expression (`SELECT new com.example.Dto(...)`) generally used for complex aggregations.
3.  **Dynamic Projections**: Passing the class type as a parameter to the repository method.

### ⚙️ Internal Working
*   **Constructor Expression (Your Project)**: Hibernate parses the JPQL. Instead of mapping the ResultSet to the `Movie` entity, it directly invokes the `AnalyticsResponse` constructor for each row. This bypasses the persistence context management for these objects (they are effectively "read-only" snapshots).
*   **Interface Projections**: Spring creates a lightweight proxy wrapper around the result set.

### 🙋‍♂️ Interview Q&A
**Q1: Why use Projections instead of fetching the Entity?**
> **Answer:** To avoid "Over-fetching". If an Entity has 50 fields and relation joins, but the UI only needs the "Name" and "Count", fetching the whole entity wastes memory. Projections fetch strictly what is needed.

**Q2: Can Projections be updated?**
> **Answer:** Generally No. DTO projections are not managed entities. Interface projections backed by an entity *can* technically expose setters, but it defeats the purpose.

---

## 🔟 Spring Boot Integration & Hibernate Config

### 💻 Code from Your Project
**File:** `Backend/src/main/resources/application.properties`

```properties
# Spring Boot Auto-Configuration hook
spring.datasource.url=jdbc:mysql://...

# Hibernate Specific Configuration being passed through Spring Boot
spring.jpa.hibernate.ddl-auto=update
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQLDialect
```

### 🔵 Technical Explanation
**Integration**:
Spring Boot's `HibernateJpaAutoConfiguration` detects the `EntityManager` and `DataSource`. It configures the `LocalContainerEntityManagerFactoryBean`.
It maps `spring.jpa.properties.*` directly to the underlying Hibernate configuration map.

**Hibernate Specifics**:
*   **Batch Processing**: Not enabled by default. To enable: `spring.jpa.properties.hibernate.jdbc.batch_size=30`. This forces Hibernate to group insertions (`INSERT INTO ... VALUES (...), (...)`) reduces round-trips to the DB.
*   **Dialect**: The `MySQLDialect` tells Hibernate how to translate generic JPQL into MySQL-specific SQL (e.g., using backticks `` ` `` or generic pagination `LIMIT`).

### ⚙️ Internal Working
1.  **Bootstrapping**: Spring Boot reads `spring.datasource.*`.
2.  **Pooling**: Creates a `HikariDataSource` (High-performance connection pool).
3.  **ORM Setup**: Passes the DataSource to Hibernate.
4.  **Transaction Management**: Wraps the EntityManager in a `JpaTransactionManager`, allowing `@Transactional` to work.

### 🙋‍♂️ Interview Q&A
**Q1: How do you configure multiple Data Sources in Spring Boot?**
> **Answer:** You disable the default `DataSourceAutoConfiguration`. You define two `DataSource` beans (e.g., `@Primary` and `@Qualifier("second")`). You then create two `LocalContainerEntityManagerFactoryBean` instances, each pointing to a package of entities and a specific DataSource.

**Q2: What is the N+1 Select Problem and how to fix it?**
> **Answer:** It happens when fetching 1 Parent causes N separate queries for N Children.
> *   **Fix 1**: Use `@EntityGraph` in the repository.
> *   **Fix 2**: Use `JOIN FETCH` in JPQL (`SELECT m FROM Movie m JOIN FETCH m.watchHistory`).

---

## 🧠 Updated Recap (Sharpen Your Memory)

| Topic | 🔵 Technical Concept (The Keyword) |
| :--- | :--- |
| **JPA vs Hibernate** | **Specification** vs **Implementation**. |
| **Spring Data JPA** | **Repository Pattern**, **Abstraction** over Hibernate. |
| **DDL-Auto** | **Schema Generation** (`update`, `create`, `none`). |
| **@Entity / @Id** | **ORM Mapping**, **Persistence Context**, **Identifier**. |
| **@ManyToOne** | **Cardinality**, **Foreign Key**, **Eager Loading** (Default). |
| **Repositories** | **Proxy Pattern**, **Derived Queries**, **@Query**. |
| **CRUD** | **Transactional Write-Behind**, **Persist vs Merge**. |
| **Pagination** | `Pageable`, `PageRequest`, `Page` (with count) vs `Slice`. |
| **Auditing** | `@EnableJpaAuditing`, **Entity Lifecycle Events**. |
| **Projections** | partial fetching. **DTO Projection** vs **Interface Projection**. |
| **Integration** | `HibernateJpaAutoConfiguration`, **Dialect**, **Batching**. |

---

# 📚 Ultimate Annotation & Dependency Cheat Sheet

This section breaks down every annotation referenced in this module, separated by their **Source Dependency** and **Use Case**. This is strictly designed for **Technical Interviews**—knowing *where* an annotation comes from shows seniority.

## 📦 Dependency: `spring-boot-starter-data-jpa`
This is a **Starter Dependency**. It does not contain code itself; it pulls in three major libraries:
1.  **Hibernate Core** (`org.hibernate`): The engine doing the work.
2.  **Jakarta Persistence API** (`jakarta.persistence`): The standard interface rules.
3.  **Spring Data JPA** (`org.springframework.data`): The simplified Repository layer.

---

### 🟢 1. Entity Mapping (The "Table" Level)
**Source Library:** `jakarta.persistence-api` (Standard Java)

| Annotation | Technical Definition | Interview "Senior" Explanation |
| :--- | :--- | :--- |
| `@Entity` | Marks a Plain Old Java Object (POJO) as a JPA Managed Entity. | "It tells the `EntityManager` to manage the lifecycle of this class and map it to a DB table." |
| `@Table` | Specifies table details (name, schema, indexes). | "Optional. I use it when the class name (`User`) is a reserved SQL keyword or doesn't match the table name (`users`)." |
| `@Id` | Specifies the Primary Key of the entity. | "The unique identifier. Every Entity **MUST** have one." |
| `@GeneratedValue`| Configures the identifier generation strategy. | "I prefer `IDENTITY` for MySQL (auto-increment) and `SEQUENCE` for high-performance batch inserts in PostgreSQL." |
| `@Column` | specifies mapped column details. | "Used to enforce constraints at the schema level, like `nullable=false` or `unique=true`." |
| `@Transient` | Specifies that the property or field is not persistent. | "Spring will ignore this field completely. Use it for calculated fields (like `age` calculated from `dob`)." |
| `@Enumerated` | Specifies how an Enum should be persisted. | "Always use `EnumType.STRING`. `ORDINAL` saves numbers (0, 1), which breaks data if you reorder the Enums later." |

---

### 🔗 2. Relationships (Connecting Tables)
**Source Library:** `jakarta.persistence-api`

| Annotation | Technical Definition | Interview "Senior" Explanation |
| :--- | :--- | :--- |
| `@ManyToOne` | Defines a single-valued association to another entity class. | "The 'Child' side. It typically owns the relationship because it holds the Foreign Key." |
| `@OneToMany` | Defines a many-valued association map or collection. | "The 'Parent' side. Always use `mappedBy` here to tell Hibernate 'The other guy owns the key', otherwise you get duplicate tables." |
| `@JoinColumn` | Specifies the Foreign Key column. | "Defines the physical name of the Foreign Key column (e.g., `user_id`)." |
| `@OneToOne` | Defines a 1:1 relationship. | "Tricky. Lazy loading often doesn't work on the inverse side (the side without the FK). Use sparingly." |

---

### 🗄️ 3. Repository & Data Access (The "Access" Level)
**Source Library:** `spring-data-jpa` & `spring-context`

| Annotation | Technical Definition | Interview "Senior" Explanation |
| :--- | :--- | :--- |
| `@Repository` | Stereotype annotation for Persistence Layer components. | "It's a specialization of `@Component`. Its superpower is **Exception Translation**—converting scary SQL exceptions into clean Spring DataAccessExceptions." |
| `@Query` | Declares finder queries directly on repository methods. | "I use it when Method Naming (`findBy...`) gets too long or when I need a complex JOIN that JPA defaults can't handle." |
| `@Param` | Binds method parameters to query parameters. | "Required when using named parameters in JPQL (e.g., `:email`)." |
| `@Modifying` | Indicates a query method should be considered a modifying query. | "You MUST put this on custom `UPDATE` or `DELETE` queries, otherwise Hibernate expects a ResultSet and throws an error." |

---

### 📝 4. Auditing (Tracking Changes)
**Source Library:** `spring-data-jpa`

| Annotation | Technical Definition | Interview "Senior" Explanation |
| :--- | :--- | :--- |
| `@EnableJpaAuditing` | Enables JPA Auditing via annotation configuration. | "The 'On Switch'. Goes on the main Application class." |
| `@EntityListeners` | Specifies the listener classes to be used for an entity. | "Connects the Entity to the `AuditingEntityListener`. Without this, `@CreatedDate` does nothing." |
| `@CreatedDate` | Declares a field as the creation date. | "Auto-filled once on INSERT." |
| `@LastModifiedDate` | Declares a field as the last modified date. | "Auto-updated on every UPDATE." |

---

## 📦 Dependency: `spring-boot-starter-web`
**Purpose:** Used for building RESTful Web Services.
**Includes:** `spring-web`, `spring-webmvc`, `jackson-databind`, `tomcat-embed-core`.

### 🌐 5. Web Logic & REST Controllers
**Source Library:** `spring-web` & `spring-webmvc`

| Annotation | Package | Technical Definition | Interview "Senior" Explanation |
| :--- | :--- | :--- | :--- |
| `@RestController` | `org.springframework.web.bind.annotation` | A convenience annotation combining `@Controller` and `@ResponseBody`. | "It tells Spring that every method in this class returns Domain Objects (JSON/XML) directly in the HTTP Response, not an HTML view." |
| `@RequestMapping` | `org.springframework.web.bind.annotation` | Maps HTTP requests to handler methods of MVC and REST controllers. | "I use this at the Class level to define the base URL (e.g., `/api/users`), keeping method-level mappings clean." |
| `@GetMapping` / `@PostMapping` | `org.springframework.web.bind.annotation` | Shortcut composed annotations for HTTP GET/POST methods. | "Syntactic sugar for `@RequestMapping(method = RequestMethod.GET)`. It's cleaner and explicit about intent." |
| `@RequestBody` | `org.springframework.web.bind.annotation` | Maps the HTTPS request body to a Java object using a `HttpMessageConverter`. | "Triggers Jackson deserialization. If the JSON is invalid, it throws `MethodArgumentNotValidException` (400 Bad Request)." |
| `@PathVariable` | `org.springframework.web.bind.annotation` | Extracts a template variable from the URI. | "Used for fetching Resources by ID (e.g., `/users/{id}`). Mandatory by default unless `required=false` is set." |
| `@RequestParam` | `org.springframework.web.bind.annotation` | Extracts query parameters from the URL. | "Used for filtering or pagination (e.g., `/users?page=1`). Can define default values easily." |
| `@CrossOrigin` | `org.springframework.web.bind.annotation` | Enables Cross-Origin Resource Sharing (CORS) on the specific handler. | "Essential for letting my Angular frontend (localhost:4200) talk to my Backend (localhost:8080). Otherwise browser blocks it." |

---

## 📦 Dependency: `spring-boot-starter-security`
**Purpose:** Authentication and Authorization.
**Includes:** `spring-security-web`, `spring-security-config`, `spring-security-core`.

### 🔒 6. Security & Authorization
**Source Library:** `spring-security-core` & `config`

| Annotation | Package | Technical Definition | Interview "Senior" Explanation |
| :--- | :--- | :--- | :--- |
| `@EnableWebSecurity` | `org.springframework.security.config.annotation.web.configuration` | Enables Spring Security’s web security support. | "The 'On Switch' for the security filter chain. Required to provide a custom `SecurityFilterChain` bean." |
| `@Configuration` | `org.springframework.context.annotation` | Indicates that a class declares one or more `@Bean` methods. | "Tells the IoC container: 'Process this class first to generate Bean definitions'." |
| `@Bean` | `org.springframework.context.annotation` | Indicates that a method produces a bean to be managed by the Spring container. | "Unlike `@Component` (Class-level), `@Bean` allows me to instantiate 3rd party classes (like `BCryptPasswordEncoder`) that I can't annotate directly." |
| `@PreAuthorize` | `org.springframework.security.access.prepost` | Checks authorization before entering the method using SpEL. | "Powerful AOP annotation. I use it for role-based access control, like `@PreAuthorize(\"hasRole('ADMIN')\")`." |
| `@AuthenticationPrincipal`| `org.springframework.security.core.annotation` | Resolves the `Authentication.getPrincipal()` property. | "Instead of parsing the Context manually, it injects the currently logged-in User details directly into my controller method." |

---

## 📦 Dependency: `spring-boot-starter` (Core)
**Purpose:** Core Spring functionalities (IoC, DI, Lifecycle).
**Includes:** `spring-core`, `spring-context`, `spring-beans`, `spring-aop`.

### ⚙️ 7. Core Containers & Dependency Injection
**Source Library:** `spring-context` & `spring-beans`

| Annotation | Package | Technical Definition | Interview "Senior" Explanation |
| :--- | :--- | :--- | :--- |
| `@SpringBootApplication`| `org.springframework.boot.autoconfigure` | Composes `@Configuration`, `@EnableAutoConfiguration`, and `@ComponentScan`. | "The entry point. It triggers the classpath scanning to create beans and the auto-configuration logic to set up DB/Web defaults." |
| `@Component` | `org.springframework.stereotype` | Generic stereotype for any Spring-managed component. | "The base annotation. `@Service`, `@Repository`, and `@Controller` are just specialized aliases of this." |
| `@Service` | `org.springframework.stereotype` | Indicates that an annotated class is a "Service". | "Semantically clearer than `@Component`. It tells the dev 'Business Logic lives here'. Ideally, Transactions start here." |
| `@Autowired` | `org.springframework.beans.factory.annotation` | Marks a constructor, field, or setter method to be autowired by Spring's DI. | "I prefer **Constructor Injection** over `@Autowired` on fields. It makes testing easier (no Reflection needed) and ensures immutability." |
| `@Value` | `org.springframework.beans.factory.annotation` | Injects values into fields from property files. | "Used to read from `application.properties`. Supports SpEL for complex logic." |
| `@Primary` | `org.springframework.context.annotation` | Indicates a preference when multiple beans candidate for autowiring. | "Resolves usage conflicts. If I have two `EmailService` impls, `@Primary` tells Spring which one to inject by default." |

---

## 📦 Dependency: `spring-boot-starter-test`
**Purpose:** Unit and Integration Testing.
**Includes:** `junit`, `mockito`, `spring-test`, `assertj`.

### 🧪 8. Testing Annotations
**Source Library:** `spring-test` & `junit`

| Annotation | Package | Technical Definition | Interview "Senior" Explanation |
| :--- | :--- | :--- | :--- |
| `@SpringBootTest` | `org.springframework.boot.test.context` | Bootstraps the complete Spring Application Context. | "Used for Integration Tests. It starts the *actual* container, DB connections (or H2), and full context. Slow but accurate." |
| `@WebMvcTest` | `org.springframework.boot.test.autoconfigure.web.servlet` | Focuses only on the MVC components (`@Controller`). | "Used for Unit Testing Controllers. It *mocks* the Service layer and does NOT load the full context, making it extremely fast." |
| `@MockBean` | `org.springframework.boot.test.mock.mockito` | Adds a mock to the Spring ApplicationContext. | "Replaces a real Bean (like `UserService`) with a Mockito Mock. Essential for isolating layers during testing." |
| `@Test` | `org.junit.jupiter.api` | Signals that a method is a test method. | "From JUnit 5. Unlike JUnit 4, it doesn't need to be public." |
