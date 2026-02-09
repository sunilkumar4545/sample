# JUnit & Mockito: The Ultimate Guide for MediaConnect

This document provides a detailed, interview-focused explanation of the testing framework implemented in your **MediaConnect** application. It covers concepts, specific code breakdowns, and a comprehensive list of interview questions.

---

## 1. High-Level Overview

### **JUnit 5 (The Engine)**
*   **What is it?** It is the framework that **runs** your tests. It provides the `@Test` annotation and methods like `assertEquals()` to check usage.
*   **Analogy:** The **Exam Proctor**. It hands out the test paper, watches the clock, and marks your answers as Pass/Fail.

### **Mockito (The tool)**
*   **What is it?** A **mocking framework**. It allows you to create "fake" versions of external dependencies (like your `UserRepository` or `AuthService`) so you can test one class in isolation.
*   **Analogy:** A **Stunt Double**. When your Controller needs to call a Service, Mockito steps in and says, "I'm the Service, and here is the fake data you asked for," without actually doing the heavy work.

---

## 2. Code Walkthrough: How Your Tests Work

### **Scenario A: Controller Testing (`UserControllerTest.java`)**
**Goal:** Test if `UserController` endpoints work correctly **without** loading the entire application or hitting the database.

```java
@WebMvcTest(UserController.class) // 1. Load ONLY the Controller layer. Fast.
@AutoConfigureMockMvc(addFilters = false) // 2. Disable Security (bypass login/CSRF) for simplicity.
class UserControllerTest {

    @Autowired private MockMvc mockMvc; // 3. The "Fake Browser" to send requests.
    @MockBean private UserService userService; // 4. The "Fake Service" (Stunt Double).
    @MockBean private SubscriptionPlanRepository planRepository; // 5. The "Fake Repo".

    @Test
    void subscribe_shouldReturnOk_whenPlanIsValid() throws Exception {
        // --- ARRANGE (Prepare) ---
        // Teach the mock what to do when called
        when(planRepository.findByName("Premium")).thenReturn(new SubscriptionPlan());
        when(userService.subscribe("john@test.com", "Premium")).thenReturn(new UserDTO());

        // --- ACT (Execute) ---
        // Send a fake POST request
        mockMvc.perform(post("/api/users/subscribe")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"plan\":\"Premium\"}"))
                
        // --- ASSERT (Verify) ---
                .andExpect(status().isOk()); // Check if response is 200 OK
    }
}
```
**Key Takeaway:** The test passes because the **Mock** returned valid data. The real database was never touched.

---

### **Scenario B: Service Testing (`UserServiceImplTest.java`)**
**Goal:** Verify complex business logic (e.g., checking subscription expiry) in isolation.

```java
@ExtendWith(MockitoExtension.class) // 1. Use pure Mockito (No Spring Context). Blazing fast.
class UserServiceImplTest {

    @Mock private UserRepository userRepository; // 2. The Mock Repository.
    @InjectMocks private UserServiceImpl userService; // 3. Real Service, with Mocks inside.

    @Test
    void getUserByEmail_shouldCheckSubscriptionExpiry() {
        // --- ARRANGE ---
        User expiredUser = new User();
        expiredUser.setSubscriptionExpiry(LocalDateTime.now().minusDays(1)); // Expired
        
        // Teach the mock to return this specific user
        when(userRepository.findByEmail("test@example.com")).thenReturn(expiredUser);

        // --- ACT ---
        userService.getUserByEmail("test@example.com");

        // --- ASSERT ---
        // Did the service try to save the status change to the DB?
        verify(userRepository).save(expiredUser); 
    }
}
```
**Key Takeaway:** We use `verify()` to ensure the service logic *attempted* to save the user, proving the business logic works.

---

## 3. Interview Question Bank (CRITICAL SECTION)

This section contains questions an interviewer is likely to ask based specifically on your code, followed by "General" Java testing questions.

### **Part A: Questions Based on YOUR Code**

**Q1: I see `@WebMvcTest` in your Controller tests and `@ExtendWith(MockitoExtension.class)` in your Service tests. Why the difference?**
> **Answer:** 
> "I use `@WebMvcTest` for Controllers because it's a **slice test**—it loads the Spring Web context allows me to use `MockMvc` to simulate HTTP requests.
> For Services, I don't need Spring at all. I use `@ExtendWith(MockitoExtension.class)` to run **pure unit tests**. This is much faster because it doesn't start the Spring container, it just uses Mockito to mock the repositories."

**Q2: In `UserControllerTest`, you used `@AutoConfigureMockMvc(addFilters = false)`. Why?**
> **Answer:** 
> "Since I am using Spring Security, every request normally requires a valid JWT token or CSRF token. For unit testing the *logic* of the controller, I didn't want to deal with generating complex auth tokens for every test case. Passing `addFilters = false` disables the security filter chain for that test, allowing me to test the endpoint logic directly."

**Q3: How exactly does `@InjectMocks` work in your `UserServiceImplTest`?**
> **Answer:** 
> "It’s a Mockito annotation. When I mark `UserServiceImpl` with `@InjectMocks`, Mockito creates a real instance of that class. Then, it looks for any fields marked with `@Mock` (like the `UserRepository`) and injects them into the Service instance. It essentially does Dependency Injection manually for the test."

**Q4: You are mocking `UserService` in the Controller test. Why not use the real Service?**
> **Answer:** 
> "If I used the real Service, my Controller test would depend on the Service logic, which depends on the Repository, which depends on the Database. If the Database fails, my Controller test fails.
> By mocking the Service, I isolate the Controller. I am testing *only* how the Controller handles HTTP requests and responses, assuming the Service behaves correctly."

**Q5: What does `verify(userRepository).save(user)` do, and why is it important?**
> **Answer:** 
> "Since I am mocking the repository, nothing actually gets written to a database. `verify` checks that the `save` method *was called* on the mock object. It proves that my business logic reached the point where it *intended* to save data. If the logic had failed earlier, `verify` would fail the test."

---

### **Part B: General / conceptual Interview Questions**

**Q6: What is the difference between `@Mock` and `@MockBean`? (Common Question)**
> **Answer:** 
> *   `@Mock` is a **Mockito** annotation. It is used in simple unit tests (like Service tests) where there is no Spring Context.
> *   `@MockBean` is a **Spring Boot** annotation. It is used in integration tests (like Controller tests). It adds the mock to the Spring Application Context, replacing any existing bean of that type.

**Q7: Explain the "Arrange-Act-Assert" pattern.**
> **Answer:**
> "It's the standard structure of a unit test:
> 1.  **Arrange:** Prepare the data and interacting mocks (e.g., `when(...).thenReturn(...)`).
> 2.  **Act:** Call the actual method being tested.
> 3.  **Assert:** Verify the result matches expectations (e.g., `assertEquals` or `verify`)."

**Q8: "Stubbing" vs "Mocking" - what's the difference?**
> **Answer:** 
> "In Mockito, we technically do both.
> *   **Stubbing** is when we define the behavior: `when(repo.find()).thenReturn(user)`. We are telling it *what to return*.
> *   **Mocking** is the object itself and verifying behavior: `verify(repo).delete()`. We are checking *what happened*."

**Q9: How do you handle exceptions in testing?**
> **Answer:**
> "I can use `assertThrows` in JUnit 5. For example:
> `assertThrows(UserNotFoundException.class, () -> userService.getUser(99));`
> This asserts that the code correctly throws usage logic exceptions when invalid data is provided."

**Q10: Why is 'Code Coverage' important, and how much is enough?**
> **Answer:**
> "Code Coverage measures what percentage of code lines were executed during tests. While 100% is ideal, it is often impractical. I aim for **80%+ coverage**, focusing specifically on business logic (Services) and input handling (Controllers). Setters/Getters usually don't need explicit testing."

---

## 4. Key Annotations Cheat Sheet

| Annotation | Library | Purpose |
| :--- | :--- | :--- |
| `@Test` | JUnit | Marks a method as a test case. |
| `@SpringBootTest` | Spring | Loads the *entire* application context (Integration Test). |
| `@WebMvcTest` | Spring | Loads only the Web layer (Controller Test). |
| `@MockBean` | Spring | Adds a mock to the Spring Context (replaces real bean). |
| `@Mock` | Mockito | Creates a mock object (Unit Test). |
| `@InjectMocks` | Mockito | Injects `@Mock`s into the class under test. |
| `@BeforeEach` | JUnit | Runs before *every* test method (setup). |
| `@DisplayName` | JUnit | Custom name for the test in reports ("Should return user..."). |


# JUnit & Mockito Deep Dive: How It Works Without a Database

This document explains **how your specific Controller and Service tests run without a database**. It breaks down the "Test Magic" (Mocking) used in `UserControllerTest` and `UserServiceImplTest`.

---

## 1. The Core Concept: "The Stunt Double"
In a real application, your data flows like this:
`Controller` -> `Service` -> `Repository` -> **Real Database**

In **Testing (JUnit + Mockito)**, we cut the cord to the database. We replace the *next* layer with a **Mock** (a Stunt Double).
*   **Controller Test:** We test `Controller` -> `Mock Service`. (No Repository, No DB).
*   **Service Test:** We test `Service` -> `Mock Repository`. (No DB).

---

## 2. Example 1: Controller Testing (`UserControllerTest.java`)

**Goal:** Test if `UserController` correctly handles a request to subscribe to a plan, **without** actually updating the database or even running the Service logic.

### **The "Magic" Setup**
```java
@WebMvcTest(UserController.class) // 1. Load ONLY UserController (No Service/Repo beans)
class UserControllerTest {

    @Autowired private MockMvc mockMvc; // 2. Fake HTTP Client
    
    // 3. THE REPLACEMENT: instead of real UserService, use a Mock
    @MockBean private UserService userService; 
    
    // 4. THE REPLACEMENT: instead of real Repository, use a Mock
    @MockBean private SubscriptionPlanRepository planRepository; 

    @Test
    void subscribe_shouldReturnOk_whenPlanIsValid() throws Exception {
        // --- ARRANGE (Prepare the Stage) ---
        String planName = "Premium";
        
        // 5. THE SCRIPT: "If the Controller asks the Repo for 'Premium', 
        //    DON'T look in DB. JUST return this new SubscriptionPlan object."
        when(planRepository.findByName("Premium"))
            .thenReturn(new SubscriptionPlan()); 

        // 6. THE SCRIPT: "If the Controller calls userService.subscribe(), 
        //    DON'T run the real code. JUST return this updated UserDTO."
        UserDTO mockResponse = new UserDTO();
        mockResponse.setCurrentPlan("Premium");
        when(userService.subscribe("john@test.com", "Premium"))
            .thenReturn(mockResponse);

        // --- ACT (Run the Scene) ---
        // 7. Send a Fake HTTP POST request
        mockMvc.perform(post("/api/users/subscribe")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"plan\":\"Premium\"}"))
                
        // --- ASSERT (Did it follow the script?) ---
                .andExpect(status().isOk()) // Did it return 200 OK?
                .andExpect(jsonPath("$.currentPlan").value("Premium")); // Did JSON match?
    }
}
```

### **What is happening here?**
*   **Line 3 & 4 (@MockBean):** Spring removes the *real* `UserService` and `Repository`. It inserts empty shell objects (Mocks) in their place.
*   **Line 5 & 6 (when...thenReturn):** This is crucial. Since the Mock is an empty shell, it knows nothing. We must **teach** it. We say, "When method X is called, return Y."
    *   **Effect:** The Controller *thinks* it successfully talked to the database, but it actually just talked to your script.
*   **Usefulness:** 
    *   **Speed:** No database connection needed (0ms latency).
    *   **Isolation:** If the *real* `UserService` is broken, this Controller test still passes because it uses a *perfect* Mock Service. This helps you pinpoint bugs.

---

## 3. Example 2: Service Testing (`UserServiceImplTest.java`)

**Goal:** Test if `UserService` correctly deactivates an expired subscription. This is pure Java logic (Business Logic).

### **The "Magic" Setup**
```java
@ExtendWith(MockitoExtension.class) // 1. Use Mockito (No Spring Context needed)
class UserServiceImplTest {

    @Mock private UserRepository userRepository; // 2. The Fake Database Interface
    @InjectMocks private UserServiceImpl userService; // 3. The Real Service (uses the Mock above)

    @Test
    void getUserByEmail_shouldCheckSubscriptionExpiry() {
        // --- ARRANGE ---
        // 4. Create a User object that IS expired
        User expiredUser = new User();
        expiredUser.setEmail("test@test.com");
        expiredUser.setSubscriptionStatus("ACTIVE");
        expiredUser.setSubscriptionExpiry(LocalDateTime.now().minusDays(1)); // Yesterday

        // 5. TEACH THE MOCK: "If asked for 'test@test.com', return this implementation of expiredUser"
        when(userRepository.findByEmail("test@test.com")).thenReturn(expiredUser);

        // --- ACT ---
        // 6. Call the REAL service method
        UserDTO result = userService.getUserByEmail("test@test.com");

        // --- ASSERT ---
        // 7. Verify the Logic: Did the Service change the status to INACTIVE?
        assertEquals("INACTIVE", result.getSubscriptionStatus()); 
        
        // 8. VERIFY DB INTERACTION: Did the Service TRY to save the user?
        verify(userRepository).save(expiredUser);
    }
}
```

### **What is happening here?**
*   **Line 2 (@Mock):** Creates a dummy `UserRepository` class.
*   **Line 3 (@InjectMocks):** Creates a *real* `UserServiceImpl` instance but **injects** the dummy repository into it.
*   **Line 5 (when...thenReturn):** We simulate a specific database state (finding a user).
*   **Line 8 (verify):** This is the killer feature. Since there is no real database to check, we ask Mockito: *"Hey, did `userService` call the `save()` method on your mock?"*
    *   If yes -> Test Passes.
    *   If no -> Test Fails (Meaning the Service didn't try to save the update).

---

## 4. Summary: Why is this useful for your project?

| Real World Problem | JUnit/Mockito Solution | Benefit |
| :--- | :--- | :--- |
| **"I need to test if checkout works, but I don't want to charge a real credit card."** | Mock the `PaymentService` to always return "Success". | Test checkout logic safely without external side effects. |
| **"I need to test user registration, but my database isn't running."** | Mock the `UserRepository` to simulate saving a user. | Run tests anywhere (CI/CD pipeline, friend's laptop) without DB setup. |
| **"I broke the User Service, and now my Controller test fails. I don't know where the bug is."** | Controller tests use a **Mock** Service. They will still pass! | You know immediately the bug is in the *Service*, not the *Controller*. |

### **Interview Explanation (The perfect answer)**
*"In my project, I use **JUnit 5** and **Mockito** to perform unit testing. I strictly follow the **3-Tier Architecture** even in testing. For Controllers, I use `@WebMvcTest` and **mock** the Service layer using `@MockBean`. This allows me to test endpoint status codes and JSON paths without loading the whole application. For Services, I use `@ExtendWith(MockitoExtension.class)` and **mock** the Repository layer. This lets me test business logic—like checking subscription expiry—instantaneously without hitting the database. I use `when().thenReturn()` to stub behaviors and `verify()` to ensure the correct dependencies were called."*
