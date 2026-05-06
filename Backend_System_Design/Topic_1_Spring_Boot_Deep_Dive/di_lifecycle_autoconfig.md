# Deep Dive: Dependency Injection & Bean Lifecycle

## 1. IOC (Inversion of Control) vs DI (Dependency Injection)
This is a classic question, but let's understand it architecturally.

- **IOC (The Concept):** Instead of you creating the object (using `new`), the control is given to a container (Spring). The container creates the object, manages its lifecycle, and gives it to you when needed.
- **DI (The Implementation):** DI is the specific pattern used to implement IOC. It's the act of "injecting" the required dependency into a class.

**Why do we need it?** 
1. **Decoupling:** Your class doesn't need to know *how* to instantiate its dependencies.
2. **Testability:** You can easily swap real implementations with "Mocks" during unit testing.

## 2. The Spring Bean Lifecycle (The "How It Works")
When Spring starts, it goes through a very specific sequence of steps to manage a bean. If you are building high-performance systems or custom frameworks, you MUST know this.

### The Phases:
1. **Instantiation:** Spring creates the bean instance (like calling the constructor).
2. **Populate Properties:** Spring injects the dependencies (using `@Autowired` on fields, setters, or constructors).
3. **Aware Interfaces:** If your bean implements `BeanNameAware`, `BeanFactoryAware`, etc., Spring calls these methods to give the bean information about its environment.
4. **BeanPostProcessor (Before Initialization):** Spring calls the `postProcessBeforeInitialization` method of all registered `BeanPostProcessor`s.
5. **@PostConstruct / InitializingBean:** Spring calls your custom initialization logic.
6. **BeanPostProcessor (After Initialization):** Spring calls `postProcessAfterInitialization`. **This is where AOP/Proxies are created!** If your bean is annotated with `@Transactional`, this is where the real bean is wrapped in a proxy.
7. **Bean is Ready:** The bean is now ready for use in the application.
8. **PreDestroy / DisposableBean:** When the container shuts down, Spring calls your cleanup logic.

---

## 3. The 3-Level Cache & Circular Dependencies
**Production Scenario:** 
`Class A` depends on `Class B`, and `Class B` depends on `Class A`. 
How does Spring solve this without getting stuck in an infinite loop?

Spring uses a **3-level cache** (Three Maps) in the `DefaultSingletonBeanRegistry`:
1. **singletonObjects (1st Level):** Stores fully initialized beans.
2. **earlySingletonObjects (2nd Level):** Stores "partially created" beans (instantiated but properties not yet populated).
3. **singletonFactories (3rd Level):** Stores ObjectFactories that can create a bean (useful for AOP proxies).

**How it works:**
- Spring starts creating `Bean A`. It puts an ObjectFactory for `A` in the **3rd cache**.
- It sees `A` needs `B`. It tries to get `B`.
- `B` isn't there, so it starts creating `B`. It sees `B` needs `A`.
- It looks for `A` in the 1st, then 2nd, then **3rd cache**.
- It finds the factory for `A` in the 3rd cache, creates an "early reference" of `A`, moves it to the **2nd cache**, and gives it to `B`.
- `B` finishes initialization and goes to the **1st cache**.
- Now `A` can finish because `B` is ready!

**Note:** Circular dependencies only work with **Setter/Field Injection**. Constructor injection will still fail with a `BeanCurrentlyInCreationException` because Spring can't even instantiate the object without its dependencies.

---

## 4. Bean Scopes
| Scope | Description |
| :--- | :--- |
| **Singleton (Default)** | Only one instance per Spring Container. Used for 99% of beans. |
| **Prototype** | A new instance is created every time it is requested from the container. |
| **Request** | One instance per HTTP request (Web applications only). |
| **Session** | One instance per HTTP session. |
| **Application** | One instance per ServletContext. |

**Trap:** If you inject a `Prototype` bean into a `Singleton` bean, the prototype bean will be created **only once** (when the singleton is initialized). To fix this, you must use **Method Injection** or `@Lookup`.

---

## 5. Production Level Best Practices
1. **Prefer Constructor Injection:** It makes dependencies explicit, ensures the object is never in an invalid state, and allows for `final` fields.
2. **Avoid Circular Dependencies:** Even though Spring can handle them, they are a sign of bad architecture. Refactor the common logic into a third class.
3. **Be careful with Scopes:** Using `Prototype` beans incorrectly can lead to memory leaks if you don't manage their destruction (Spring does not manage the full lifecycle of prototype beans after initialization).
