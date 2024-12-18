---
title: "Why You Don’t Need a DI Framework: A Case for Manual Dependency Injection"
date: 2024-12-18
---

Dependency Injection (DI) frameworks are so widely adopted that many developers assume they are the only option for managing dependencies in an application. However, this prevailing wisdom often goes unchallenged, leading to unnecessary complexity, opaque behavior, and an over-reliance on tools that add abstraction with limited benefits.

This article will argue that **manual dependency injection** is not only viable but often the better choice, even for large-scale applications. By focusing on **explicitness** and simplicity, you can avoid the pitfalls of DI frameworks while maintaining clean, testable, and scalable code.

While the examples here are written in **Kotlin**, the concepts are **language agnostic** and can be applied to any modern programming language.

---

## Why People Think They Need a DI Framework

Before addressing why DI frameworks aren’t always necessary, let’s outline the typical reasons developers reach for them:

1. **Automating Dependency Resolution**: Frameworks promise to simplify the wiring of complex dependency graphs.
2. **Managing Lifecycle Scopes**: Frameworks handle singleton, transient, and request-scoped objects seamlessly.
3. **Simplifying Testing**: Dependency injection enables easy mocking and swapping of implementations.
4. **Avoiding Boilerplate**: Manually managing dependencies can seem verbose and repetitive.
5. **Consistency Across Teams**: Frameworks enforce a standardized approach to managing dependencies.

These points may sound convincing, but upon closer inspection, each can be addressed with manual dependency injection (manual DI) while retaining the benefits of explicit, understandable code.

---

## Addressing the Arguments for DI Frameworks

### 1. Automating Dependency Resolution

Dependency resolution is often seen as the primary benefit of DI frameworks. By registering components, frameworks resolve dependencies automatically.

**Counterpoint**:
Manual DI is explicit. By bootstrapping dependencies at the **edge** of your application, you retain full control and clarity over how objects are created and passed.

**Example in Kotlin**:

```kotlin
// Bootstrapping dependencies
class Logger
class Database(val logger: Logger)
class UserService(val logger: Logger, val database: Database)

fun bootstrap(): UserService {
    val logger = Logger()
    val database = Database(logger)
    return UserService(logger, database)
}

fun main() {
    val userService = bootstrap()
    // Application starts here
}
```

Here, dependencies are manually created and passed explicitly. While this might seem verbose for small examples, in practice it keeps the wiring **clear and traceable**.

> **Aside**: Interestingly, a common pattern has emerged among developers using frameworks like Spring to centralize their dependency bootstrapping in dedicated configuration files instead of relying on annotations scattered throughout the code. This practice simplifies reasoning about dependencies but highlights the irony of frameworks' promises of "automation." If centralizing configuration improves clarity, it suggests that fully automated dependency resolution may not always be beneficial or necessary. This approach brings Spring closer to manual DI principles, emphasizing that explicitness and intentional design often lead to better outcomes.

### 2. Managing Lifecycle Scopes

DI frameworks provide lifecycle scopes like **singleton**, **transient**, and **request-scoped** objects. For instance, a singleton service is only instantiated once.

**Counterpoint**:
Manual DI can handle lifecycles easily:

- **Singleton**: Create a shared instance during application bootstrapping.
- **Transient**: Use factory functions to create new instances when needed.
- **Request Scope**: Pass request-specific data as function arguments.

**Example**:

```kotlin
class RequestHandler(val logger: Logger, val database: Database)

fun createRequestHandler(logger: Logger, database: Database): RequestHandler {
    return RequestHandler(logger, database)
}

fun main() {
    val logger = Logger() // Singleton
    val database = Database(logger) // Singleton

    // Transient instance for each request
    val handler = createRequestHandler(logger, database)
    handler.process()
}
```

This approach removes the need for lifecycle annotations or runtime resolution.

### 3. Simplifying Testing

DI frameworks claim to simplify testing by enabling mocks to be injected automatically.

**Counterpoint**:
Manual DI makes testing easier by keeping dependencies **explicit**. You don’t need a framework to replace real implementations with mocks:

**Example**:

```kotlin
val mockLogger = mock<Logger>()
val mockDatabase = mock<Database>()
val service = UserService(mockLogger, mockDatabase)
```

With manual DI, there is no hidden wiring—dependencies are clear and under your control.

### 4. Avoiding Boilerplate

DI frameworks reduce boilerplate by automating dependency resolution.

**Counterpoint**:
While manual DI may seem more verbose, this verbosity is **explicitness** in disguise. Explicit dependency wiring prevents surprises and reduces debugging time. Boilerplate can also be minimized with patterns like **factories** and dependency grouping:

**Example**:

```kotlin
data class AppDependencies(val logger: Logger, val database: Database)

class UserService(val deps: AppDependencies) {
    fun process() {
        deps.logger.log("Processing")
    }
}
```

---

## The Downsides of Using a DI Framework

### 1. Hidden Complexity

DI frameworks introduce abstraction that obscures how dependencies are created and resolved. This hidden behavior can make debugging difficult, especially when errors occur at runtime due to misconfiguration.

### 2. Over-Injection

The ease of adding dependencies in DI frameworks often leads to over-injection, where classes depend on too many services or components. This results in bloated constructors, reduced clarity, and violations of the **Single Responsibility Principle**.

Moreover, DI frameworks **require every class that needs a dependency to be registered** in the framework's container. This means even minor utility classes or internal components must be treated as seams, adding unnecessary complexity to the dependency graph.

**Manual DI**, on the other hand, encourages you to:

- **Limit dependencies to external interfaces** (e.g., ports like databases, HTTP clients, or external systems).
- **Intentionally introduce seams** where extensibility is required (e.g., adhering to the **Open-Closed Principle**).

Not everything needs to be abstracted or dynamically replaceable. Core application logic and stable internal components can be passed directly without unnecessary indirection. This reduces cognitive load and keeps the codebase simple.

For example, you do not need to inject a utility class like `Logger` via a framework when it can simply be instantiated and passed explicitly:

```kotlin
class Service(val logger: Logger, val database: Database)

fun main() {
    val logger = Logger()
    val database = Database(logger)
    val service = Service(logger, database)
    service.process()
}
```

By avoiding the automatic registration of every class, manual DI keeps the system **focused and intentional**, reducing over-abstraction.

### 3. Framework Lock-In

Frameworks impose specific patterns and behaviors, making it difficult to refactor or migrate your application in the future.

### 4. Learning Curve

Developers need to learn and understand the DI framework in addition to managing dependencies, adding overhead for teams.

---

## How to Do Manual DI Effectively

### 1. Constructor Injection

Pass dependencies explicitly via class constructors.

```kotlin
class UserService(val logger: Logger, val database: Database) {
    fun processUser(userId: String) {
        val user = database.queryUser(userId)
        logger.log("Processing user: $user")
    }
}

fun main() {
    val logger = Logger()
    val database = Database(logger)
    val service = UserService(logger, database)
    service.processUser("123")
}
```

This is simple, clear, and makes each class’s dependencies explicit.

### 2. Factory Functions

For dependencies that need to be created on demand, use factory functions:

```kotlin
fun createRequestHandler(logger: Logger, database: Database): RequestHandler {
    return RequestHandler(logger, database)
}
```

### 3. Group Related Dependencies

To reduce argument lists, group related dependencies into a single container:

```kotlin
data class AppDependencies(val logger: Logger, val database: Database)
```

---

## Alternatives to Constructor Injection

For the purpose of this article I have used constructor based injection for both the DI framework and manual bootstrapping examples. However, there are alternatives that can be considered, which leads to a more functional style.

### 1. Passing Dependencies as Function Arguments

For small, focused functions, pass dependencies directly as arguments:

```kotlin
fun fetchUser(logger: Logger, database: Database, userId: String) {
    val user = database.queryUser(userId)
    logger.log("Fetched user: $user")
}
```

This is simple and avoids retaining unnecessary state.

### 2. The Reader Pattern

For more functional-style dependency injection, the **Reader Monad** allows you to thread dependencies implicitly through computations.

**Example** (using Arrow in Kotlin):

```kotlin
typealias Reader<Env, A> = (Env) -> A

fun fetchUser(userId: String): Reader<Config, String> = { config ->
    "User($userId) fetched from ${config.databaseUrl}"
}

fun main() {
    val config = Config(databaseUrl = "http://localhost")
    val user = fetchUser("123")(config)
    println(user)
}
```

This approach avoids passing dependencies manually while keeping the process explicit and testable.

---

## Conclusion

Dependency Injection frameworks are often adopted out of habit, not necessity. While they promise automation, lifecycle management, and reduced boilerplate, they come with hidden complexity, over-injection, and framework lock-in.

Manual dependency injection offers a **simpler, clearer, and more explicit alternative** that works for projects of all sizes, even large ones. By focusing on intentional design, you:

- Retain control over **how dependencies are wired**.
- Avoid over-abstraction by limiting seams to **external interfaces** and extensibility points.
- Keep your codebase simple, testable, and easy to refactor.

Start with manual DI, use patterns like constructor injection and factories, and only introduce tools when you have a demonstrated need. By prioritizing explicitness, you avoid unnecessary complexity and write code that is easier to understand and maintain.
