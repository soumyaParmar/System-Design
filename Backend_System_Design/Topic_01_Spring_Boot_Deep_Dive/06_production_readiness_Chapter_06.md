# Deep Dive: Production Readiness (Actuator, Metrics, Profiling)

## 1. What is "Production Readiness"?
In a production environment, your code working is only 50% of the job. The other 50% is **Observability**: knowing *how* it is running, *why* it is slow, and *if* it is healthy. Spring Boot provides this via **Actuator**.

## 2. Spring Boot Actuator: The Window into your App
Actuator adds several HTTP endpoints to your application that let you monitor it.

### Core Endpoints:
- `/health`: Shows if the app is UP or DOWN. Crucial for Kubernetes Liveness/Readiness probes.
- `/info`: Displays arbitrary application information.
- `/metrics`: Shows JVM memory, CPU usage, HTTP request counts, etc.
- `/env`: Shows all the environment variables and configuration properties (careful with secrets!).
- `/thread-dump`: Generates a thread dump for debugging deadlocks.
- `/heap-dump`: Generates a heap dump for debugging memory leaks.

---

## 3. Metrics & Monitoring (Micrometer)
Spring Boot uses **Micrometer**, which is like the "SLF4J for metrics." It allows you to publish metrics to various monitoring systems like **Prometheus, Datadog, or New Relic** without changing your code.

### Standard Dashboard Setup (Prometheus + Grafana):
1. **App:** Exposes `/actuator/prometheus`.
2. **Prometheus:** Scrapes (pulls) metrics from this endpoint every 15-30 seconds.
3. **Grafana:** Connects to Prometheus and displays beautiful graphs of memory, response times, and error rates.

---

## 4. Logging & Tracing (MDC)
In production, you have thousands of users. If one user reports an error, how do you find *their* specific logs?

### MDC (Mapped Diagnostic Context):
We use `MDC` to attach a unique `correlationId` or `traceId` to every log line for a specific request.
1. A **Filter** generates a `UUID` for each incoming request.
2. The ID is put into `MDC.put("traceId", uuid)`.
3. Every log line (Service, DAO, Controller) will automatically include this `traceId`.
4. When you see an error, you search your log aggregator (ELK/Splunk) for that `traceId` and see the entire journey of that request.

---

## 5. Profiling & Performance Tuning
### Spring Profiles
Use `@Profile("prod")` to enable specific beans or configurations (like a real database instead of H2) only in production.

### JVM Profiling Tools:
If your app is slow or consuming too much memory, use:
- **JVisualVM / JConsole:** Standard tools included with the JDK for real-time monitoring.
- **JProfiler / YourKit:** Premium tools for deep memory and CPU analysis.
- **Arthas:** A powerful open-source diagnostic tool from Alibaba that lets you profile a running Java app *without* restarting it.

### Common Tuning Parameters:
- **JVM Memory:** `-Xms2g -Xmx2g` (Setting min and max memory same prevents heap resizing overhead).
- **HikariCP:** Adjust `maximum-pool-size` based on your DB capacity.
- **Tomcat Threads:** Adjust `server.tomcat.threads.max` (default is 200).

---

## 6. Summary Checklist
- [x] Enable Actuator and secure sensitive endpoints (don't expose `/env` to the public!).
- [x] Use Prometheus/Grafana for long-term monitoring.
- [x] Implement Request Tracing using MDC.
- [x] Use Spring Profiles to separate Dev/Stage/Prod configs.
- [x] Perform load testing and tune your thread pools and connection pools before going live.
