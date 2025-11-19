# Recommendations

## What is your recommended tech stack for Java backend development?
- **Language:** Java / Kotlin
- **Frameworks:** Spring Boot, Quarkus
- **Design & Architecture:** Modular Monolith, Package-by-Feature
- **Database:** PostgreSQL, Flyway Migrations
- **Testing:** Integration Tests with Testcontainers, Unit Tests with Mockito, API Stubbing with WireMock
- **Architecture Tests:** Spring Modulith Tests, ArchUnit Tests with Taikai
- **Library Upgrades:** Renovate, Dependabot, OpenRewrite
- **IDE:** IntelliJ IDEA
- **Build Tool:** Maven
- **Code Formatting:** Spotless Plugin with Palantir Java Format

## Unit vs Integration Tests
I used to strictly follow the [Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html) approach, but over time, I've realized that it's not always the most effective strategy. Many of the applications I've worked on have relatively simple business logic but rely heavily on integrations with external systems such as SQL databases, messaging systems, and third-party REST APIs. For such applications, unit tests often provide limited value. Instead, I recommend focusing on integration tests that validate how the application behaves when interacting with real external systems.

That said, it's not about choosing only unit tests or only integration tests. The most effective testing strategy involves a balanced mix of different types of tests to ensure comprehensive coverage and reliable behavior.

1. **Integration Tests:** Validate the behavior of a feature end-to-end by interacting with real dependencies using tools like [Testcontainers](https://testcontainers.com/). An exception to this is third-party REST APIs, where you can use mocking tools such as WireMock since you may not have control over the external service.
2. **Unit Tests:** Validate the behavior of a single unit (which could be one class or a group of closely related classes) by mocking external dependencies.
3. **Slice Tests:** While integration tests are valuable, it's not always necessary to spin up all dependencies for certain scenarios. For example, when testing invalid REST API payloads, you don't need a database or message broker. In such cases, slice tests are ideal — they load only a subset of components while mocking the rest. In Spring Boot, you can use annotations like `@WebMvcTest`, `@DataJpaTest`, and others to achieve this.


## Testcontainers tests are taking too much time. What should I do?
1. Use [Testcontainers Singleton approach](https://www.sivalabs.in/blog/run-spring-boot-testcontainers-tests-at-jet-speed/) to use the same containers for all tests.
2. Use [Testcontainers reuse](https://java.testcontainers.org/features/reuse/) feature to keep the containers running in the background instead of stopping and restarting.
3. If you are using Testcontainers with PostgreSQL, consider using [dbsandboxer](https://github.com/misirio/dbsandboxer) to speed up test execution by using PostgreSQL's template database feature.

## Mockito vs In-Memory Implementation for Mocking
When writing unit tests for a service layer class(`CustomerService`), a common challenge is managing its dependencies, such as the `CustomerRepository`. We need a way to isolate the service logic from the actual data access implementation to ensure the test only validates the service's behavior. Two primary techniques for achieving this isolation are using a mocking framework like Mockito or creating an In-Memory implementation of the dependency.

Let's assume the following simplified interfaces and class structures:

```java
public class Customer {
    private Long id;
    private String name;
    // ... constructors, getters, and setters
}

public interface CustomerRepository {
    List<Customer> findAll();
    List<Customer> search(String query);
    void update(Customer customer);
    void delete(Long customerId);
    // ... other methods
}

@Service
class CustomerService {
    private final CustomerRepository repository;

    public CustomerService(CustomerRepository repository) {
        this.repository = repository;
    }

    public List<Customer> findAll() {
        return repository.findAll();
    }

    public List<Customer> search(String query) {
        if (query == null || query.trim().isEmpty()) {
            return repository.findAll();
        }
        return repository.search(query);
    }

    public void update(Customer customer) {
        // Assume some business logic here before calling repository
        repository.update(customer);
    }

    public void delete(Long customerId) {
        repository.delete(customerId);
    }
}
```

Let's explore how we can test using the Mockito-based approach and the In-Memory implementation approach.

### Testing with Mockito Mocks
This approach uses Mockito to create a mock instance of `CustomerRepository`.

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Arrays;
import java.util.List;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class CustomerServiceMockitoTest {

    // Creates a mock instance of CustomerRepository
    @Mock
    private CustomerRepository mockRepository;

    // Injects the mockRepository into a CustomerService instance
    @InjectMocks
    private CustomerService customerService;

    @Test
    void search_ShouldReturnFilteredCustomers_WhenQueryIsProvided() {
        // Arrange: Setup *specific* data for this test
        String searchName = "Alice";
        List<Customer> expectedCustomers = Arrays.asList(
            new Customer(1L, "Alice Smith"),
            new Customer(2L, "Alice Johnson")
        );
        
        // Stubbing: Program the mock to return the *expected* data 
        // when its 'search' method is called with the specific argument.
        when(mockRepository.search(searchName)).thenReturn(expectedCustomers);

        // Act
        List<Customer> actualCustomers = customerService.search(searchName);

        // Assert
        assertEquals(2, actualCustomers.size());
        assertEquals("Alice Smith", actualCustomers.getFirst().getName());
        
        // Verification: Ensure the dependency was called as expected
        verify(mockRepository, times(1)).search(searchName);
        verify(mockRepository, never()).findAll(); // Ensure findAll wasn't called
    }
    
    @Test
    void delete_ShouldCallRepositoryDelete() {
        // Arrange
        Long customerId = 5L;
        
        // Act
        customerService.delete(customerId);
        
        // Assert/Verify
        // Ensure the delete method on the repository was called exactly once with the correct ID
        verify(mockRepository, times(1)).delete(customerId);
    }
}
```

**Benefits of the Mockito Approach:**

* **Independent Test Data Setup for Each Test:** As shown in the example, you define exactly what the mock returns (stubbing) right before the test execution. This means each test operates with a fresh, isolated set of data. A successful data setup in one test cannot affect the data or assertions of any other test.

* **No Extra Code to Maintain:** You do not write a second, simplified implementation of `CustomerRepository`. All the required "test behavior" is defined concisely within the test method itself using the mocking API. This significantly reduces code maintenance overhead.


### Testing with In-Memory Implementation
This approach involves creating a concrete implementation of `CustomerRepository` that stores data in simple Java collections (like `List` or `Map`) instead of connecting to a real database.

```java
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;
import java.util.stream.Collectors;

class InMemoryCustomerRepository implements CustomerRepository {
    // This collection holds the data for testing
    private final Map<Long, Customer> store = new ConcurrentHashMap<>();
    private final AtomicLong nextId = new AtomicLong(1);

    public void saveInitialData(List<Customer> customers) {
        store.clear(); // Important: clear data before each setup
        customers.forEach(c -> {
            c.setId(nextId.getAndIncrement());
            store.put(c.getId(), c);
        });
    }

    @Override
    public List<Customer> findAll() {
        return new ArrayList<>(store.values());
    }

    @Override
    public List<Customer> search(String query) {
        return store.values().stream()
                .filter(c -> c.getName().contains(query))
                .collect(Collectors.toList());
    }
    
    // Simplified implementation for the test
    @Override
    public void update(Customer customer) {
        store.put(customer.getId(), customer);
    }

    @Override
    public void delete(Long customerId) {
        store.remove(customerId);
    }
}
```

The test relies on setting up the shared data structure in the `InMemoryCustomerRepository`.

```java
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.Arrays;
import java.util.List;
import static org.junit.jupiter.api.Assertions.assertEquals;

class CustomerServiceInMemoryTest {

    private InMemoryCustomerRepository inMemoryRepository;
    private CustomerService customerService;

    @BeforeEach
    void setup() {
        inMemoryRepository = new InMemoryCustomerRepository();
        customerService = new CustomerService(inMemoryRepository);
        
        // Initial setup for ALL tests using this repository
        List<Customer> initialData = Arrays.asList(
            new Customer(null, "Charlie Brown"),
            new Customer(null, "Alice Smith"),
            new Customer(null, "David Jones")
        );
        inMemoryRepository.saveInitialData(initialData);
    }

    @Test
    void search_ShouldReturnFilteredCustomers_WhenQueryIsProvided() {
        // Arrange is done in @BeforeEach, we rely on the initial data.
        
        // Act
        List<Customer> actualCustomers = customerService.search("li");

        // Assert
        assertEquals(1, actualCustomers.size());
        assertEquals("Charlie Brown", actualCustomers.get(0).getName());
    }
}
```

**Problems with the In-Memory Repository Approach:**

While an in-memory repository can feel more "real" than a mock, it introduces significant issues for unit testing:

**1. Need to Maintain Two Versions of the Repository Implementations:** You must keep the production code (e.g., Spring Data JPA implementation) and the `InMemoryCustomerRepository` perfectly compatible. Every time a method is added or the behavior of an existing method is changed (e.g., subtle ordering or filtering logic), both implementations must be updated. This creates a maintenance burden and a risk of divergence (where the in-memory version doesn't accurately reflect the production version).

**2. Test Execution Order Matters (Shared State Problem):** The in-memory repository keeps data in a shared collection (`List<Customer>` or `Map<Long, Customer>`). If one test modifies this shared state (e.g., a delete test), that modification persists and can affect subsequent tests. For example, a search test that relies on 5 records being present will fail if a preceding delete test removed one of them. While a `@BeforeEach` method can help reset the state, managing complex, multi-state resets quickly becomes cumbersome and brittle.

**3. Brittle Test Data Setup:** As the application grows and more functionality is added, maintaining a global data set for the in-memory repository becomes extremely difficult.

For example, a test verifying a feature expects a search query to return 23 records based on the current global data setup. If new data is added for a different test or feature, and that new data also matches the search criteria, the count might become 24. The original test's assertion (`assertEquals(23, actualCount)`) will fail, even though the service logic itself is correct. The test setup becomes hard to maintain as the application expands.

So, I recommend using Mockito so that you control the exact conditions of the test without the maintenance overhead and brittleness associated with managing a secondary, in-memory dependency implementation.

## Should I aim for 100% Test Coverage?
Striving for 100% test coverage often leads to a blind ritual rather than a genuine pursuit of quality. While the metric was intended to encourage thorough testing, obsessively chasing the final few percentage points frequently results in writing trivial, brittle tests that simply confirm getter/setter calls or other low-value code, without actually verifying meaningful system behavior. 

The reality is that high test coverage doesn't necessarily mean high-quality code; a project can have 100% coverage yet still fail to test critical business logic or edge cases. 

Instead of a rigid target, a more pragmatic approach is to aim for a solid foundation, such as 80% code coverage, prioritizing a good test suite focused on verifying system behavior, complex logic, and integration points, which provides the most value for maintaining a robust application.

## How to enforce a common coding style in a team?
Code formatting and linting are frequently discussed — and sometimes hotly debated — topics within development teams. In reality, these discussions often consume more time than they should. Once a team agrees on a standard formatting approach, it typically becomes second nature within a week, and few developers think about it again.

To maintain consistency and avoid unnecessary debates, it’s best to adopt a well-established tool and automate formatting as part of your build or CI process. Some popular options include:

* [Spotless](https://github.com/diffplug/spotless)
* [Google Java Format](https://github.com/google/google-java-format)
* [Spring Java Format](https://github.com/spring-io/spring-javaformat)
* [Prettier Java](https://github.com/jhipster/prettier-java)
