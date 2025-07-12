# Java Application Development Guidelines

These guidelines serve as a standard for developing Java 21 applications. They cover general coding standards, 
architectural principles, logging, and testing practices to ensure consistency, maintainability, performance, and 
robustness. The instructions are also optimized to guide AI code assistants like GitHub Copilot.

---

## 1. General Coding Standards

Foundational principles for writing clean, modern, and efficient Java code.

* **Leverage Java 21 Features:**
    * **Records:** Use `record` for immutable data carriers (DTOs, configuration classes, simple value objects) to 
    reduce boilerplate.
        ```java
        public record Product(String id, String name, double price) {}
        ```
    * **Sealed Types:** Employ `sealed` classes and interfaces with `permits` for controlled inheritance hierarchies, ensuring all possible subtypes are known and handled. Use `non-sealed` where appropriate to allow further extension.
        ```java
        public sealed interface Shape permits Circle, Rectangle {}
        public final class Circle implements Shape {}
        public non-sealed class Rectangle implements Shape {} // Allows further extension
        ```
    * **Switch Expressions with Pattern Matching:** Use `switch` expressions for concise, exhaustive handling of multiple cases, especially with pattern matching.
        ```java
        double area = switch (shape) {
            case Circle c    -> Math.PI * c.radius() * c.radius();
            case Rectangle r -> r.length() * r.width();
            default          -> throw new IllegalArgumentException("Unknown shape");
        };
        ```
    * **Pattern Matching for `instanceof`:** Utilize `if (obj instanceof MyType t)` for type checking and casting in a single, safe step.
        ```java
        if (event instanceof UserCreatedEvent userEvent) {
            // 'userEvent' is automatically cast and available here
            logger.info("User created: {}", userEvent.userId());
        }
        ```
* **Default to Immutability:**
    * Use the `final` keyword wherever possible for variables, parameters, and fields to promote immutability and reduce side effects.
    * Use immutable collections: `List.of()`, `Set.of()`, `Map.of()` for small, fixed collections. For mutable collections that are returned or passed, consider `Collections.unmodifiableList()`, `Collections.unmodifiableSet()`, etc., or Guava's immutable collections.
* **Handling Optionality:**
    * Use `Optional<T>` instead of `null` to explicitly indicate the absence of a value, improving readability and preventing `NullPointerExceptions`.
    * Avoid using `Optional.get()` without checking for presence (`Optional.isPresent()` or `Optional.orElseThrow()`).
    * Prefer methods like `orElse()`, `orElseGet()`, `map()`, `flatMap()`, `filter()`, and `ifPresent()` for processing `Optional` values.
* **Service Design (DI-enabled):**
    * **Avoid static utility classes** with many static methods. Prefer creating DI-enabled services that can be injected, allowing for easier testing, mockability, and adherence to SOLID principles.
* **SOLID Principles & Clean Architecture:**
    * Continuously apply **SOLID principles** (Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion) to design robust and maintainable code.
    * Follow **Clean Architecture** (or Hexagonal Architecture) guidelines (see Section 2).
* **Stream Operations:**
    * Prefer **Stream API operations** over traditional manual loops for collection processing. This leads to more concise, readable, and often more efficient code.
    * **Use `Collectors`:** Leverage `Collectors.groupingBy`, `mapping`, `joining`, `reducing`, `partitioningBy` for complex aggregation and transformation.
    * **`flatMap`:** Use `flatMap` to flatten nested collections or streams into a single stream.
    * **Avoid in-place mutation:** Do not mutate collections within stream pipelines; instead, use `map()`, `filter()`, and `collect()` to produce new collections.
* **Control Flow Simplification:**
    * Replace complex `if-else` chains with **early returns and guard clauses** to improve readability and reduce nesting.
    ```java
    // Bad
    if (isValid) {
        if (isAuthenticated) {
            // ... logic ...
        } else {
            // handle not authenticated
        }
    } else {
        // handle not valid
    }

    // Good
    if (!isValid) {
        // handle not valid, return
    }
    if (!isAuthenticated) {
        // handle not authenticated, return
    }
    // ... main logic ...
    ```

---

## 2. Architecture & Multi-Module Structure

Applying Clean Architecture principles with a multi-module Gradle setup for modular and scalable applications.

* **Multi-Module Gradle Structure:**
    * Use **Gradle multi-module projects** to enforce clear architectural boundaries and enable independent development of features.
    * **Application Module (`app`):** The main entry point that assembles all feature modules, contains the main Spring Boot application class, and defines the overall application configuration.
    * **Feature Modules:** Each major feature or bounded context should be its own Gradle module (e.g., `user-management`, `product-catalog`, `order-processing`).
    * **Libraries Module:** Contains shared utilities, common exceptions, base classes, and cross-cutting concerns used across multiple feature modules.
    * Example project structure:
        ```
        my-app/
        ├── settings.gradle
        ├── build.gradle
        ├── app/                           // Main application module
        │   ├── build.gradle
        │   └── src/main/java/
        │       └── com/yourorg/app/
        │           └── Application.java   // @SpringBootApplication
        ├── user-management/               // Feature module
        │   ├── build.gradle
        │   └── src/main/java/
        │       └── com/yourorg/user/
        ├── product-catalog/               // Feature module
        │   ├── build.gradle
        │   └── src/main/java/
        │       └── com/yourorg/product/
        └── libraries/                     // Common utilities
            ├── build.gradle
            └── src/main/java/
                └── com/yourorg/libraries/
        ```

* **Clean Architecture / Hexagonal Architecture:**
    * Strictly apply the principles of **Clean Architecture** (also known as Hexagonal Architecture or Ports and Adapters) within each feature module.
    * **Domain Layer:** Contains pure business logic (entities, value objects, domain services, interfaces defining ports). It has no dependencies on outer layers.
    * **Application Layer:** Defines application-specific use cases, orchestrating domain logic. It contains service interfaces (ports) and their implementations (use case interactors). Depends on the Domain Layer.
    * **Infrastructure Layer:** Contains concrete implementations of interfaces defined in the Application or Domain layers. This includes persistence (JPA repositories, custom DAOs), external REST clients, message queue clients, etc. Depends on Application and Domain Layers.
    * **Interface Layer:** Contains entry points into the application, such as REST controllers, WebSockets handlers, or CLI commands. Adapts external requests into use case calls. Depends on Application Layer.

* **Feature Module Organization:**
    * Organize code **by feature (or bounded context), not by technical type**. Each feature module should contain its complete vertical slice of functionality.
    * Within each feature module, organize by Clean Architecture layers:
        ```
        user-management/src/main/java/com/yourorg/user/
        ├── domain/           // User, UserService (interface), UserRepository (interface)
        ├── application/      // CreateUserUseCase, GetUserByIdQuery, UserApplicationService
        ├── infrastructure/   // JpaUserRepository, UserRestClient, UserEventPublisher
        └── api/              // UserController, UserRestDto, UserMapper
        ```

* **Module Dependencies:**
    * **Application module** depends on all feature modules and is responsible for assembling the complete application.
    * **Feature modules** can depend on the libraries module but should NOT depend on each other directly.
    * **Cross-feature communication** should happen through well-defined interfaces (events, shared DTOs, or application services) rather than direct module dependencies.

* **Service Definitions:**
    * Always define **interfaces for service definitions** (ports) in the Domain or Application layers within each feature module. Implementations (adapters) belong in the Infrastructure layer.
    * For cross-feature communication, define shared interfaces in the libraries module or use event-driven patterns.

* **Class Cohesion:**
    * **Avoid "God Classes"** (large, monolithic classes that do too many things).
    * Each class should adhere to the **Single Responsibility Principle**, doing one thing well.

---

## 3. Logging & Observability

Guidelines for effective logging and system observability.

* **SLF4J:**
    * Use **SLF4J** as the logging facade with an appropriate logging backend (e.g., Logback, Log4j2).
    * Use **appropriate log levels**:
        * `INFO`: For significant business events, major state changes, or application startup/shutdown.
        * `WARN`: For recoverable issues, potential problems, or situations that don't immediately cause failure but warrant attention.
        * `ERROR`: For unexpected failures, exceptions, or critical issues that prevent normal operation.
        * `DEBUG`/`TRACE`: For detailed technical information during development or troubleshooting.
* **Structured Logging:**
    * Use **structured logging** (e.g., Logback's `StructuredArguments`, `logstash-logback-encoder`) instead of simple string concatenation. This makes logs easier to parse and analyze by logging aggregation systems.
    * Avoid direct string concatenation in log messages:
        ```java
        // Bad
        logger.info("User " + userId + " logged in.");
        // Good (using parameterized logging)
        logger.info("User {} logged in.", userId);
        ```
* **Sensitive Data:**
    * **Never log sensitive data** such as passwords, authentication tokens, personally identifiable information (PII), or financial details. Ensure sensitive fields are properly masked or redacted before logging.

---

## 4. Testing Guidelines (Spock Framework)

Comprehensive guidelines for writing effective and reliable tests with Spock Framework.

### 4.1. General Best Practices

* **Spock Specifications:**
    * Use **Spock specifications** extending `Specification` class for all test classes.
    * Name test classes with `Spec` suffix (e.g., `UserServiceSpec`, `CreateUserUseCaseSpec`).
    * Use **descriptive feature method names** that read like natural language sentences describing the behavior being tested.
        ```groovy
        def "should create user when valid input is provided"() {
            // test implementation
        }
        
        def "should throw validation exception when email is invalid"() {
            // test implementation
        }
        ```

* **Spock Blocks:**
    * Use **Spock's structured blocks** for clear test organization:
        * **`given:`** (or `setup:`) - Set up test fixtures and preconditions
        * **`when:`** - Execute the action being tested
        * **`then:`** - Verify the expected outcomes and side effects
        * **`where:`** - Provide parameterized test data
        * **`and:`** - Continue any block when additional clarity is needed
        * **`expect:`** - For simple single-expression tests
        * **`cleanup:`** - Clean up resources after test execution
    * Example:
        ```groovy
        def "should calculate total price correctly"() {
            given: "a product with price and quantity"
            def product = new Product(id: "123", name: "Test Product", price: 10.0)
            def quantity = 3
            
            when: "calculating total price"
            def totalPrice = orderService.calculateTotalPrice(product, quantity)
            
            then: "total should be price multiplied by quantity"
            totalPrice == 30.0
        }
        ```

* **Spock Annotations:**
    * Use **`@Subject`** to mark the class under test for better readability.
    * Use **`@Shared`** for expensive setup that can be reused across feature methods.
    * Use **`@Unroll`** for parameterized tests to see individual test cases in reports.
    * Use **`@Timeout`** for tests that should complete within a specific time limit.
    * Use **`@IgnoreIf`** or **`@Requires`** for conditional test execution.

### 4.2. Data-Driven Testing

* **Where Blocks:**
    * Use **`where:` blocks** extensively for parameterized testing to test multiple scenarios concisely.
    * Use **data pipes (`|`)** for tabular data representation.
    * Use **`@Unroll`** annotation to execute each parameter set as a separate test.
        ```groovy
        @Unroll
        def "should validate email format for '#email' and return #expectedResult"() {
            expect: "email validation returns expected result"
            emailValidator.isValid(email) == expectedResult
            
            where: "different email formats are tested"
            email                    | expectedResult
            "user@example.com"       | true
            "invalid-email"          | false
            "user@"                  | false
            ""                       | false
            null                     | false
        }
        ```

* **Data Providers:**
    * Use **data providers** for complex test data generation.
    * Leverage **Groovy's collection methods** for generating test data.
        ```groovy
        def "should handle various user inputs"() {
            expect:
            userService.processUser(user) != null
            
            where:
            user << generateTestUsers()
        }
        
        def generateTestUsers() {
            return (1..10).collect { 
                new User(id: it, name: "User$it", email: "user$it@example.com") 
            }
        }
        ```

### 4.3. Mocking and Stubbing

* **Spock Mocks:**
    * Use **Spock's built-in mocking** instead of external frameworks like Mockito.
    * Create mocks using **`Mock()`**, **`Stub()`**, or **`Spy()`** methods.
    * Use **interaction verification** in `then:` blocks to verify method calls.
        ```groovy
        def "should save user and send notification"() {
            given: "mocked dependencies"
            def userRepository = Mock(UserRepository)
            def notificationService = Mock(NotificationService)
            def userService = new UserService(userRepository, notificationService)
            def user = new User(id: "123", name: "John Doe")
            
            when: "creating a user"
            userService.createUser(user)
            
            then: "user is saved and notification is sent"
            1 * userRepository.save(user) >> user
            1 * notificationService.sendWelcomeEmail(user.email)
        }
        ```

* **Stubbing:**
    * Use **method stubbing** with `>>` operator for return values.
    * Use **`>>>>`** for returning different values on subsequent calls.
    * Use **closures** for complex stubbing logic.
        ```groovy
        given: "stubbed external service"
        def externalService = Stub(ExternalService)
        externalService.getData(_) >> { String id -> 
            if (id == "valid") return new Data("result")
            else throw new NotFoundException("Data not found")
        }
        ```

* **Argument Matching:**
    * Use **argument matchers** for flexible interaction verification.
    * Use **`_`** for any argument, **`!null`** for non-null arguments.
    * Use **custom matchers** with closures for complex argument validation.
        ```groovy
        then: "notification is sent with correct parameters"
        1 * notificationService.sendEmail(_ as String, { it.contains("welcome") })
        ```

### 4.4. Test Coverage Guidelines

* **Coverage Targets:**
    * Aim for **90%+ code coverage** on the **Domain** and **Application (Service/Use Case)** layers. These layers contain the core business logic and should be thoroughly tested.
    * Infrastructure and Interface layers may have slightly lower targets but should still have adequate coverage for critical paths.

* **Test Scenarios:**
    * Write tests for both the **happy path** (expected successful execution) and **edge cases**.
    * **Edge cases** include: invalid inputs, empty collections, boundary conditions, error conditions, null values, concurrency issues (where applicable).
    * Use **`where:` blocks** to test multiple scenarios efficiently.

* **Exception Testing:**
    * Use **`thrown()`** method to test expected exceptions.
    * Use **`notThrown()`** to verify that exceptions are not thrown.
        ```groovy
        def "should throw validation exception for invalid input"() {
            given: "invalid user data"
            def invalidUser = new User(id: null, name: "")
            
            when: "attempting to create user"
            userService.createUser(invalidUser)
            
            then: "validation exception is thrown"
            def exception = thrown(ValidationException)
            exception.message.contains("User ID cannot be null")
        }
        ```

* **Exclusions:**
    * **Do not test simple POJOs** (Plain Old Java Objects) like DTOs, records, or entities that only contain getters/setters and have no custom logic.
    * **Do not test generated code** (e.g., Lombok-generated methods, framework proxies).
    * **Do not test framework code** or third-party libraries.

### 4.5. Spring Boot Integration Testing

* **Spring Boot Test Annotations:**
    * Use **`@SpringBootTest`** for full integration tests.
    * Use **`@WebMvcTest`** for testing web layer components.
    * Use **`@DataJpaTest`** for testing JPA repositories.
    * Use **`@TestConfiguration`** for test-specific configuration.

* **Test Slices:**
    * Use **Spring Boot test slices** to test specific layers in isolation.
    * Combine with Spock's mocking capabilities for comprehensive testing.
        ```groovy
        @WebMvcTest(UserController)
        class UserControllerSpec extends Specification {
            @Autowired
            MockMvc mockMvc
            
            @MockBean
            UserService userService
            
            def "should return user when valid ID is provided"() {
                given: "a valid user ID"
                def userId = "123"
                def user = new User(id: userId, name: "John Doe")
                userService.findById(userId) >> Optional.of(user)
                
                when: "requesting user by ID"
                def response = mockMvc.perform(get("/users/{id}", userId))
                
                then: "user is returned with OK status"
                response.andExpect(status().isOk())
                       .andExpect(jsonPath('$.id').value(userId))
                       .andExpect(jsonPath('$.name').value("John Doe"))
            }
        }
        ```

### 4.6. Best Practices Summary

* **Readable Tests:** Write tests that serve as living documentation of your system's behavior.
* **Fast Execution:** Keep unit tests fast by mocking external dependencies.
* **Isolated Tests:** Each test should be independent and not rely on other tests.
* **Maintainable Tests:** Use Spock's expressive syntax to write tests that are easy to understand and maintain.
* **Comprehensive Coverage:** Test both positive and negative scenarios, edge cases, and error conditions.
* **Leverage Spock Features:** Use data-driven testing, built-in mocking, and expressive assertions to write better tests.

---