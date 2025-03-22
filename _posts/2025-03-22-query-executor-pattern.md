---
layout: post
title: "Rethinking Repositories: The QueryExecutor Pattern"
date: 2025-03-22
tags: [architecture, kotlin, di, repositories]
---

The repository pattern has long been the go-to for managing database access in layered architectures. It gives us a clean separation between our domain logic and persistence concerns — but it also comes with some trade-offs that can get in the way as a project scales.

In this post, I want to share a pattern I’ve found more scalable, testable, and in line with modern architecture principles: the **QueryExecutor** pattern.

---

## The Problems With Repositories

Repositories do a good job of abstracting DB logic, but they have a few common drawbacks:

1. **Violates SRP (Single Responsibility Principle)**  
   Putting every query and update into a single class causes bloat. Repository classes grow unwieldy, mixing unrelated logic together — just so it can all live behind a common interface.

2. **No Seam for Cross-Cutting Concerns**  
   Since queries go directly through repository methods, there's no natural place to plug in retries, logging, metrics, or tracing. This often leads to the use of AOP or proxy-based magic, which introduces unnecessary complexity and hides the actual control flow.

3. **Unnecessary Registration of Dependencies**  
   Whether you’re using manual DI or declarative configuration (as in some Spring setups), you end up registering every repository or individual query. This adds boilerplate and makes your wiring harder to reason about — especially as the number of queries grows.

> Manual DI, on the other hand, encourages you to:
>
> - Limit dependencies to external interfaces (e.g., ports like databases, HTTP clients, or external systems).
> - Intentionally introduce seams where extensibility is required (e.g., adhering to the Open-Closed Principle).  
>
> — From [You Don’t Need a DI Framework](https://dannyryman.github.io/tech-blog/2024/12/18/you-do-not-need-a-di-framework.html)

---

## A Better Alternative: The QueryExecutor Pattern

The **QueryExecutor** flips the control. Instead of injecting dependencies into each query, you inject the *query* into the executor, and the executor provides the shared context.

This becomes a single registration point for all queries and commands, and creates a clean seam where you can apply cross-cutting concerns.

```kotlin
interface Query<T> {
    fun execute(queryContext: QueryContext): T
}

class QueryExecutor(
    val dbContext: DSLContext,
    val requestContext: RequestContext,
    val timeSource: TimeSource
) {
    fun <T> execute(query: Query<T>): T {
        // Add any cross-cutting concerns such as retries, logging, metrics here
        return query.execute(QueryContext(dbContext, requestContext, timeSource))
    }
}

data class QueryContext(
    val dbContext: DSLContext,
    val requestContext: RequestContext,
    val timeSource: TimeSource
)

data class RequestContext(val loggedOnUser: User)
```

## Why QueryExecutor Works Well

This pattern offers several benefits:

- **Cross-cutting concerns in one place** — logging, retries, tracing, metrics... no magic needed.
- **Minimal dependency wiring** — only the executor needs to be registered.
- **Testable and focused** — each query is a standalone unit with no external dependencies.
- **Scales gracefully** — you can have as many queries/commands as you like without bloated classes.
- **Simple mental model** — one input (the query), one output (the result), clean flow.

> 💡 **Aside**: The real magic here is the inversion of typical injection. You pass business inputs (like `userId`) in the constructor of the query, and inject *infrastructure* (DB, time, request context) into the function. This is the opposite of what most developers are used to — but it gives you total control and clarity.

---

## Example Usage

A simple read query:

```kotlin
class GetUserByIdQuery(val userId: UUID) : Query<User?> {
    override fun execute(queryContext: QueryContext): User? {
        return queryContext.dbContext
            .selectFrom(USER)
            .where(USER.ID.eq(userId))
            .fetchOneInto(User::class.java)
    }
}
```

A write (command) example — just return `Unit`:

```kotlin
class CreateAuditLogCommand(val message: String) : Query<Unit> {
    override fun execute(queryContext: QueryContext) {
        queryContext.dbContext.insertInto(AUDIT_LOG)
            .set(AUDIT_LOG.MESSAGE, message)
            .set(AUDIT_LOG.TIMESTAMP, queryContext.timeSource.now())
            .execute()
    }
}
```

Application logic becomes simple and declarative:

```kotlin
val user = queryExecutor.execute(GetUserByIdQuery(userId))
queryExecutor.execute(CreateAuditLogCommand("Fetched user $userId"))
```

---

## What About Grouping?

A common objection is that this doesn’t group related queries together like a repository does. But this is fuzzy thinking — you can still organize your code by domain:

```
/user/queries/GetUserByIdQuery.kt
/audit/commands/CreateAuditLogCommand.kt
```

You get logical grouping *without* introducing large, coupled classes.

---

## Final Thoughts

The QueryExecutor pattern offers a simple, scalable alternative to traditional repositories. It gives you:

- Clear separation of concerns.  
- Natural seams for cross-cutting logic.  
- Minimal dependency setup.  
- Easy testing.  
- Clean, composable business logic.

In practice, I’ve found it to be a cleaner and more flexible approach — especially on projects where complexity grows over time.

If you’re tired of bloated repositories and DI tangles, give this pattern a try. You might not go back.
