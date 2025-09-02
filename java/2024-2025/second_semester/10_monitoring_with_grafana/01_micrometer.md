## **1. Введение в Micrometer**
Micrometer — это библиотека для сбора метрик из Java-приложений, предоставляющая фасад над популярными системами мониторинга, такими как Prometheus, Graphite, InfluxDB и другие.

**Ключевые термины:**
- **Метрика** — числовая характеристика состояния системы (например, latency, error rate).
- **Тэги (Tags)** — ключ-значение для дополнительной классификации метрик.
- **Реестр (MeterRegistry)** — компонент, отвечающий за хранение и экспорт метрик.

---

## **2. Основные типы метрик**
Micrometer поддерживает несколько видов метрик:

### **2.1. Counter (Счётчик)**
Измеряет количество событий (монотонно возрастает).

```java
Counter requestCounter = Counter
    .builder("http.requests")
    .description("Total HTTP requests")
    .tag("method", "GET")
    .register(Metrics.globalRegistry);

requestCounter.increment(); // Увеличиваем счётчик
```

### **2.2. Gauge (Гейдж)**
Измеряет мгновенное значение (например, размер очереди).

```java
List<String> queue = new ArrayList<>();
Gauge.builder("queue.size", queue, List::size)
     .description("Current queue size")
     .register(Metrics.globalRegistry);
```

### **2.3. Timer (Таймер)**
Измеряет продолжительность операций и их частоту.

```java
Timer timer = Timer
    .builder("http.request.duration")
    .description("HTTP request duration")
    .tag("method", "POST")
    .register(Metrics.globalRegistry);

timer.record(() -> {
    // Логика запроса
});
```

### **2.4. DistributionSummary (Сводка распределения)**
Фиксирует распределение значений (например, размеры ответов).

```java
DistributionSummary summary = DistributionSummary
    .builder("response.size")
    .description("Response size in bytes")
    .register(Metrics.globalRegistry);

summary.record(response.getBytes().length);
```

---

## **3. Интеграция с Spring Boot Actuator**
Spring Boot автоматически конфигурирует Micrometer через `spring-boot-starter-actuator`.

**Настройка в `application.yml`:**
```yaml
management:
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      service: "user-service"
```

**Пример метрики в контроллере:**
```java
@RestController
public class UserController {
    private final Counter userRegistrations;

    public UserController(MeterRegistry registry) {
        this.userRegistrations = Counter.builder("user.registrations")
            .register(registry);
    }

    @PostMapping("/users")
    public void createUser() {
        userRegistrations.increment();
        // Логика регистрации
    }
}
```

---

## **4. Конфигурация Docker Compose**

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus
    ports:
      - 9090:9090
    volumes:
      - ./config/prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    ports:
      - 3000:3000
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
```

## **5. Экспорт метрик в Prometheus**
Micrometer поддерживает экспорт в Prometheus через `micrometer-registry-prometheus`.

**Зависимость Maven:**
```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

**Иллюстрация метрики в Prometheus-формате:**
```
# HELP http_requests_total Total HTTP requests  
# TYPE http_requests_total counter  
http_requests_total{method="GET",} 42.0  
```

---

