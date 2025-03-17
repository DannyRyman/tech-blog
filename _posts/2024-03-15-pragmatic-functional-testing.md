---
title: "Pragmatic Unit Testing: A Practical Approach"
date: 2025-03-15
excerpt: This post explores a real-world approach to unit testing—testing at the API boundary rather than isolating every class. By focusing on system behavior over implementation details, we achieve better coverage, reduced fragility, and greater refactoring confidence. If traditional unit testing has felt limiting, this might offer a fresh perspective.
---

# **Pragmatic Unit Testing: A Practical Approach**

Unit testing is a crucial part of modern software development, ensuring that systems behave as expected from an external consumer’s perspective. This article explores a **pragmatic approach to unit testing**—one that prioritizes **testing observable behavior over implementation details**, avoiding excessive mocking while still maintaining speed and reliability.

## **A Philosophy of Realism in Testing**
Rather than focusing on isolated, white-box unit tests or costly full end-to-end (E2E) tests, this approach emphasizes **unit testing at the API boundary** while minimizing unnecessary stubbing and mocking. The goal is to verify that the system produces the correct **outcomes** rather than testing its internal structure.

### **Key Principles of This Approach**

1. **Unit Testing via Public APIs Using an xUnit Framework**  
   - Tests interact with the system through a **single API endpoint**, mirroring real consumer interactions rather than orchestrating multiple endpoints.  
   - The application runs **in-memory** during tests, leveraging built-in test modes supported by modern web frameworks. For example, Spring Boot, ASP.NET Core, and Django all provide mechanisms to start lightweight instances without requiring full external infrastructure.  
   - Example: Using Spring Boot, a functional test can be set up with `@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)`, which starts the application in-memory and allows API requests via a `WebTestClient`:
   
     ```kotlin
     @SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
     @RunWith(SpringRunner::class)
     class MyApiTest(@Autowired val webTestClient: WebTestClient) {
     
         @Test
         fun `should return expected response from API`() {
             webTestClient.get().uri("/api/example")
                 .exchange()
                 .expectStatus().isOk
                 .expectBody<String>().value { body ->
                     assertTrue(body.contains("expected content"))
                 }
         }
     }
     ```  

2. **Stubbing External Services, Not Internal Components**  
   - **Third-party integrations** are stubbed to ensure the system under test operates in isolation.  
   - However, internal services are **not** mocked—allowing real interactions between components to be tested and avoiding artificial constraints imposed by the test structure.  

3. **Local Infrastructure via Docker**  
   - Dependencies such as **databases, message queues, and caching layers** run in Docker to provide a real execution environment.  
   - This prevents issues that arise from mocking infrastructure that is **an integral part of the domain**.  

4. **Assertions at the Database Level Instead of Another API**  
   - Rather than using another API endpoint than the one under test, assertions are made at the **database level** to verify expected state changes. This approach avoids coupling tests to multiple endpoints, ensures tests remain independent, and allows validation even when there is no secondary API available.
   - Additionally, higher-level integration and system tests will already cover **cross-endpoint** behavior, making such assertions redundant at this level.  
   - Example: To test a `PUT` endpoint has implemented the desired behaviour, we would verify the changes made to the database rather than say using a GET endpoint to get the data for verification.

5. **Data-Driven Setup**  
   - Instead of **mocking repositories**, or using other API endpoints (i.e. `PUT` endpoint), test data is inserted into the database as part of the test to create setup conditions.  
   - Example: Before testing a `GET` request, relevant data is **manually seeded into the database** to ensure realistic test conditions.  

6. **Avoiding White-Box Test Structure Pitfalls**  
   - A common issue with white-box unit testing is that the test structure **mimics the project structure**—leading to a near 1:1 mapping between test classes and implementation classes. This results in unnecessary duplication and rigid test suites.  
   - This pragmatic approach avoids that problem by **organizing tests by behavior** rather than class structure, leaving flexibility to group tests in a way that makes sense for the domain.  
   - **Impact on Test Setup Structure**: Following this approach allows the test suite to be **structured around real-world scenarios and behaviors**, rather than being forced to mirror the internal architecture of the main codebase. This keeps the tests more **maintainable, expressive, and aligned with how the system is actually used**, rather than how it is implemented.

## **Advantages of This Approach**
✅ **High Confidence in System Behavior**  
By testing real components and interactions, this method avoids the **false sense of security** often associated with over-mocking.  

✅ **Catches Integration Issues Early**  
Internal services are tested **in context**, reducing the risk of integration failures only appearing in production.  

✅ **Minimal Refactoring Overhead**  
Tests are **less fragile** because they validate **outcomes rather than implementation details**.  

✅ **More Realistic Than Pure White-Box Unit Testing**  
Ensures that the system as a whole works as expected, rather than verifying only individual methods.  

✅ **Test Structure is Flexible**  
The test suite is **not bound** to the structure of the application, allowing for better organization and maintainability.  

## **When to Break the Convention**
While this unit testing approach is robust, there are cases where alternative strategies make more sense:

1. **Testing Edge Cases and Large Permutation Sets**  
   - If testing requires **dozens of permutations** (e.g., validation rules, calculation logic), isolated unit tests may be more efficient than API-driven tests.  
  
2. **Faster Feedback for Core Business Logic**  
   - Computationally heavy business logic (e.g., pricing calculations) can be **tested independently** in unit tests for rapid iteration.  
  
3. **Testing Behavior That Is Hard to Simulate in a Full System**  
   - Error handling, **time-sensitive workflows**, and **network failures** can often be tested more reliably with **targeted component tests**.  

4. **Performance and Load Testing**  
   - Unit tests are not designed for load testing. Tools like **Gatling or k6** are better suited for performance validation.  

## **Further Reading**
For a deeper discussion on why this approach works, we highly recommend:

- Ian Cooper’s talk ["TDD: Where Did It All Go Wrong?"](https://www.youtube.com/watch?v=EZ05e7EMOLM). It explores how TDD was originally intended to test behaviors rather than just isolated units of code, aligning closely with the pragmatic unit testing philosophy.
- Dan North’s article ["Introducing BDD"](https://dannorth.net/introducing-bdd/), which originally defined Behavior-Driven Development (BDD) as **outside-in testing**. Over time, however, BDD has become more associated with **Given-When-Then (GWT) syntax and tooling like Cucumber**, shifting the focus toward structured test descriptions rather than the underlying testing philosophy.

## **Conclusion**
This **pragmatic unit testing approach** provides a middle ground between **isolated unit testing and full end-to-end tests**, balancing **realism, maintainability, and execution speed**. By testing against a **single API endpoint** and asserting at system boundaries, this method delivers **high confidence in production readiness** without excessive fragility.

For teams struggling with **white-box unit tests** that mirror the code structure or overly coupled test suites, this approach offers a structured yet flexible alternative—one that maximizes **test effectiveness and system reliability** while keeping test suites maintainable.

\n
---
\n

> ### 📝 An Evolution of Thought: How I Came to This Approach  
>  
> I’ve refined my perspective on testing over a long time—**with plenty of missteps along the way**. Having been an early(ish) adopter of **Extreme Programming (XP)**, I’ve spent years experimenting with different testing styles, often influenced by shifting industry trends.  
>  
> I remember early successes where we managed to **refactor large areas of legacy code** that no one wanted to touch. By **first wrapping the existing behavior in tests**, we were able to replace fragile code with confidence, ensuring that the system still worked—even if we didn’t fully understand why certain things were originally built the way they were.  
>  
> Then came a shift in the industry, where this **style of testing was frowned upon**. A new wave of opinionated developers advocated that **everything be tested in isolation**, promoting strict **mock-heavy, white-box unit tests**. This led to tests that felt **detached from real system behavior**, didn’t provide confidence when refactoring, and ultimately, **strayed from some of the original intents of TDD**—including documenting the system and preventing regressions. 
>  
> This approach put off many developers, including **David Heinemeier Hansson (DHH), the creator of Ruby on Rails**, who famously wrote ["TDD is Dead, Long Live Testing"](https://dhh.dk/2014/tdd-is-dead-long-live-testing.html). His frustration sparked a broader debate, leading to conversations with **Kent Beck and Martin Fowler**—who clarified that **TDD was never about isolating every unit of code**, but rather about driving design through **meaningful** tests ([see discussion](https://martinfowler.com/articles/is-tdd-dead/)).  
>  
> Reflecting on those early successes, I realized that **TDD itself wasn’t the problem**. The problem was the **shift in focus from testing behaviors to testing isolated units**. The best results didn’t come from testing every internal class in isolation, but from **testing behaviors through the system’s public API**—treating a “unit” as a **unit of behavior** rather than a **unit of code**.  
