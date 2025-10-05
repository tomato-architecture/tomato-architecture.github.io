# Concepts
Tomato Architecture is built around the following concepts:

## 1. Separation of Concerns
A typical web application consists of multiple components such as web controllers, service layers implementing business logic, and database access layers. Separating these components into distinct layers improves maintainability, testability, and extensibility.

For example, the web controller should focus on handling incoming HTTP requests and delegating work to the service layer. The service layer implements the business logic and interacts with the persistence layer for data operations.

### Keypoints:
* Web controllers should only extract necessary data from incoming requests and pass it to the service layer.
* Scheduled jobs should prepare input data from their execution context and delegate actual processing to the service layer.
* Service layer should expose APIs that execute business use cases as atomic operations.
* Service layer should remain independent of the context in which it is invoked (e.g., web requests, scheduled jobs, or message listeners).
* Persistence layer should be solely responsible for database interactions.

For example, the service layer should not look up the current login user details from the HTTP Session or request context.
Instead, the web layer should extract user information and pass it to the service layer. This approach makes the service layer reusable across different contexts, such as web requests or background jobs.

## 2. Modularity
Enforcing strict modularity prevents the codebase from devolving into a tightly coupled and unmanageable structure.
Organize code by following the [package-by-feature](https://phauer.com/2020/package-by-feature/) approach, where the related feature(s) is encapsulated within its own module.

Each module should expose only its public APIs and hide its internal implementation details. This ensures that internal changes within a module do not impact other parts of the system.

You can leverage tools such as [Spring Modulith](https://docs.spring.io/spring-modulith/reference/) and [ArchUnit](https://www.archunit.org/) to enforce module boundaries programmatically.

## 3. Testability
Automated testing is crucial to maintaining code quality and preventing regressions. A well-structured test suite provides confidence in the stability of the application as it evolves.

### Unit tests
Unit tests validate individual components—whether a single class or a small cluster of related classes—to ensure they behave as expected.
Although mocking dependencies is a common practice, I prefer using real objects whenever possible to keep tests meaningful and closer to real scenarios.

When dependencies involve external systems, tools such as [Mockito](https://site.mockito.org/) and [WireMock](http://wiremock.org/) can be used to mock interactions.

While some developers prefer creating in-memory implementations over using Mockito, IMO, Mockito generally offers greater flexibility and simplicity.

### Integration tests
While unit tests are valuable for validating isolated logic quickly, integration tests verify that entire features work correctly as a whole.

Use [Testcontainers](https://testcontainers.com/) to start dependent services and perform realistic end-to-end validation.
Though integration tests take longer to run, they provide stronger guarantees that the system behaves correctly in real environments.

A healthy combination of both unit and integration tests ensures overall robustness and reliability.
While aiming for good test coverage is important, focus on writing meaningful tests that add value rather than chasing arbitrary coverage metrics.

## 4. Simplicity over Excessive Abstractions
Modern frameworks and libraries already address many recurring architectural challenges. In real-world scenarios, organizations rarely switch frameworks, databases, or message brokers overnight. Creating multiple abstraction layers to guard against such unlikely events often leads to unnecessary complexity and reduced productivity.

Some interpretations of Hexagonal Architecture discourage direct use of frameworks or libraries, leading developers to reinvent features such as annotations and AOP-based mechanisms (e.g., custom @UseCase annotations for handling transactions or caching). This approach often adds overhead without real benefit.

Favor simplicity: minimize coupling with frameworks and libraries but leverage their built-in capabilities when they provide clear value. Avoid reinventing the wheel—use abstractions only when they serve a tangible purpose.

