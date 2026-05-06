# Deep Dive: Spring Security Architecture

## 1. The Big Picture: How Security Hooks into Spring
Spring Security is primarily a **Chain of Servlet Filters**. It sits in front of your application and intercepts every request before it even reaches your `DispatcherServlet`.

### The Key Components:
- **DelegatingFilterProxy:** A standard Servlet Filter that delegates the actual work to a Spring Bean.
- **FilterChainProxy:** A Spring Bean that manages a list of `SecurityFilterChain`s.
- **SecurityFilterChain:** A collection of security filters (e.g., `UsernamePasswordAuthenticationFilter`, `JwtAuthenticationFilter`, `ExceptionTranslationFilter`).

## 2. Authentication vs. Authorization
- **Authentication (Who are you?):** The process of verifying the identity of a user.
- **Authorization (What can you do?):** The process of verifying if the authenticated user has permission to access a specific resource or perform an action.

---

## 3. The Authentication Flow (Production Level)
When a user tries to log in, Spring Security goes through this flow:

1. **Filter:** A filter (like `UsernamePasswordAuthenticationFilter`) extracts the credentials (username/password) from the request and creates an `Authentication` object (unauthenticated).
2. **AuthenticationManager:** The filter hands this object to the `AuthenticationManager` (usually `ProviderManager`).
3. **AuthenticationProvider:** The manager iterates through several `AuthenticationProvider`s to find one that can handle this request (e.g., `DaoAuthenticationProvider` for DB, `JwtAuthenticationProvider` for tokens).
4. **UserDetailsService:** The provider calls `UserDetailsService` to load the user's data from the database.
5. **Password Encoding:** The provider compares the provided password with the encoded password in the database using `PasswordEncoder`.
6. **Success:** If they match, a fully populated `Authentication` object (including roles/authorities) is created.
7. **SecurityContextHolder:** The authenticated object is stored in the `SecurityContextHolder`, which uses a `ThreadLocal` to keep the user's data available throughout the entire request.

---

## 4. JWT (Stateless) vs. Session (Stateful) Security
In modern microservices, we almost always use **JWT (JSON Web Tokens)** because they are stateless.

| Feature | Session-Based (Stateful) | JWT-Based (Stateless) |
| :--- | :--- | :--- |
| **Storage** | Stored on the server (In-memory/Redis). | Stored on the client (Local Storage/Cookie). |
| **Scalability** | Harder to scale (requires sticky sessions or shared Redis). | Easy to scale (server doesn't store anything). |
| **Security** | Risk of CSRF (Cross-Site Request Forgery). | Risk of XSS (Cross-Site Scripting) if stored in LocalStorage. |

### JWT Production Workflow:
1. User logs in with credentials.
2. Server validates and signs a JWT.
3. Client sends the JWT in the `Authorization: Bearer <token>` header for every subsequent request.
4. Server validates the signature of the token. No database hit is required to "know" who the user is!

---

## 5. Method Level Security
You can protect specific methods (not just URLs) using annotations:

```java
@Service
public class SalaryService {

    @PreAuthorize("hasRole('ADMIN')") // SpEL (Spring Expression Language)
    public void updateSalary(Long employeeId) {
        // ...
    }
}
```
**How it works:** Spring creates a **Proxy** around your service. When the method is called, the proxy checks the `SecurityContextHolder` to see if the current user has the 'ADMIN' role.

---

## 6. Common Misconceptions & Traps
- **Trap: Storing sensitive data in JWT.** JWT is Base64 encoded, NOT encrypted. Anyone can read the contents. Only store public IDs and roles.
- **Trap: Forgeting to permit the login/public endpoints.** If you don't explicitly `permitAll()` your `/login` or `/public/**` endpoints, you will get a 403 even for guests.
- **Trap: SecurityContext and Async.** Since `SecurityContextHolder` uses `ThreadLocal`, if you start a new thread (e.g., using `@Async`), the security context is lost. You must use `DelegatingSecurityContextExecutorService` to pass the context to the new thread.

---

## 7. Summary Checklist
- [x] Spring Security is a chain of filters.
- [x] Use `SecurityContextHolder` to get the current user.
- [x] Prefer stateless JWT for Microservices.
- [x] Always use a `BCryptPasswordEncoder` for passwords.
- [x] Use `@PreAuthorize` for fine-grained authorization logic.
