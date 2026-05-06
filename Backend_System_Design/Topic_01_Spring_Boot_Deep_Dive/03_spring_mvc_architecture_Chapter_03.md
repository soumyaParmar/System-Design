# Deep Dive: Spring MVC & Request Processing Architecture

## 1. What is Spring MVC?
Spring MVC is a web framework built on the **Servlet API**. It follows the **Model-View-Controller** design pattern, which decouples the business logic, the data (Model), and the presentation layer (View). In modern microservices, the "View" is often just a JSON response.

## 2. The Heart of the System: DispatcherServlet
The `DispatcherServlet` is the "Front Controller" of Spring MVC. Every single HTTP request coming into your application first hits this Servlet.

### The Request Flow (The Journey of a Request):
1. **Request Received:** The HTTP request arrives at the Servlet container (e.g., Tomcat).
2. **DispatcherServlet:** Tomcat hands the request to `DispatcherServlet`.
3. **HandlerMapping:** The `DispatcherServlet` asks `HandlerMapping` (e.g., `RequestMappingHandlerMapping`) which Controller should handle this URL.
4. **HandlerAdapter:** Once the Controller is found, the `DispatcherServlet` uses a `HandlerAdapter` to actually call the Controller's method. (This handles parameter binding, converting JSON to Java objects using Jackson, etc.).
5. **Controller Execution:** Your `@RestController` method executes and returns an object or a String.
6. **Return Value Handling:**
   - If it's a `@ResponseBody` (or `@RestController`), the `HandlerAdapter` uses `HttpMessageConverters` to convert the Java object into JSON/XML and writes it directly to the HTTP response.
   - If it's a traditional MVC app, it returns a `ModelAndView`.
7. **ViewResolver:** (For traditional apps) The `DispatcherServlet` asks the `ViewResolver` to find the actual HTML/Thymeleaf template.
8. **Response:** The final response is sent back to the client.

---

## 3. Filters vs. Interceptors (Crucial for System Design)
This is a common production confusion. Both allow you to execute code before/after a request, but they live in different layers.

| Feature | Filter (Servlet API) | Interceptor (Spring MVC) |
| :--- | :--- | :--- |
| **Layer** | Lives outside Spring, in the Servlet Container (Tomcat). | Lives inside the Spring Context. |
| **Access** | Access to `HttpServletRequest` and `Response`. No access to Spring's `Handler` or `ModelAndView`. | Access to the actual Controller `Handler` object. |
| **Usage** | Global tasks: Logging, Authentication (Spring Security), GZIP compression, CORS. | Business-specific tasks: Performance monitoring, adding common model attributes, fine-grained Auth. |

**The Hierarchy:**
`Request` -> `Filter 1` -> `Filter 2` -> `DispatcherServlet` -> `Interceptor` -> `Controller`

---

## 4. Production Grade Concepts

### Exception Handling (`@ControllerAdvice`)
In a production system, you never want your raw stack traces to leak to the client. We use a global `ExceptionMapper` using `@ControllerAdvice` and `@ExceptionHandler`.
Spring MVC uses the `HandlerExceptionResolver` to find these global handlers when a controller throws an error.

### Content Negotiation
Spring MVC can serve different formats (JSON, XML, HTML) for the *same* URL based on the `Accept` header from the client. This is handled by `ContentNegotiationManager`.

### Threading Model
In a standard Spring MVC app, each request is handled by a **separate thread** from Tomcat's thread pool (usually 200 threads by default). 
**Problem:** If your controller calls a slow external API, that thread sits idle and "blocked," waiting for the response. If 200 people call that slow API, your entire server freezes.
**Solution:** This is why "Reactive Programming" (Spring WebFlux) was created, or more recently, "Virtual Threads" (Java 21/Project Loom) to handle blocking calls without wasting real OS threads.

---

## 5. Summary Checklist
- [x] `DispatcherServlet` is the entry point.
- [x] `HandlerMapping` finds the controller; `HandlerAdapter` executes it.
- [x] Filters are for low-level Servlet tasks; Interceptors are for Spring-level logic.
- [x] Use `@ControllerAdvice` for global error handling.
- [x] Understand that Spring MVC is "One Thread Per Request" (Blocking IO).
