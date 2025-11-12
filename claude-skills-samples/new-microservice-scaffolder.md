# Home Depot Microservice Scaffolder

## Description
Create new microservices following Home Depot's standard architecture patterns and conventions.

Use this skill when developer asks to:
- Create a new microservice
- Scaffold a new service
- Set up a new backend service
- Start a new API project

Activate for phrases like: "create new service", "new microservice", "scaffold service", "set up API service"

## Instructions

Generate a complete microservice with Home Depot's standard structure, configurations, and integrations.

### 1. Project Structure

Create this folder structure:

```
<service-name>/
├── src/
│   ├── main/
│   │   ├── java/com/homedepot/<service>/
│   │   │   ├── Application.java
│   │   │   ├── config/
│   │   │   │   ├── SecurityConfig.java
│   │   │   │   ├── DatabaseConfig.java
│   │   │   │   └── ObservabilityConfig.java
│   │   │   ├── controller/
│   │   │   │   └── HealthController.java
│   │   │   ├── service/
│   │   │   ├── repository/
│   │   │   ├── model/
│   │   │   └── exception/
│   │   │       └── GlobalExceptionHandler.java
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-dev.yml
│   │       ├── application-staging.yml
│   │       └── application-prod.yml
│   └── test/
│       ├── java/com/homedepot/<service>/
│       │   ├── integration/
│       │   └── unit/
│       └── resources/
│           └── application-test.yml
├── k8s/
│   ├── deployment.yml
│   ├── service.yml
│   ├── configmap.yml
│   └── ingress.yml
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── cd.yml
├── Dockerfile
├── docker-compose.yml
├── pom.xml (or build.gradle)
├── README.md
└── .gitignore
```

### 2. Technology Stack Defaults

**Backend Framework:**
- Java 17 with Spring Boot 3.x
- Maven for dependency management
- Spring Cloud for microservices patterns

**Database:**
- PostgreSQL 14+ (default)
- Flyway for migrations
- Connection pooling with HikariCP

**Observability:**
- New Relic APM for monitoring
- Splunk for centralized logging
- Prometheus metrics endpoint (/actuator/prometheus)
- Spring Boot Actuator for health checks

**Security:**
- Spring Security
- OAuth2/JWT for authentication
- API key validation for internal services
- Rate limiting with Bucket4j

### 3. Application.java Template

```java
package com.homedepot.<service>;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

@SpringBootApplication
@EnableDiscoveryClient
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### 4. Health Check Endpoint (Required)

Every service must expose health checks for Kubernetes:

```java
@RestController
@RequestMapping("/actuator/health")
public class HealthController {

    @Autowired
    private DatabaseHealthCheck databaseHealthCheck;

    @GetMapping
    public ResponseEntity<HealthStatus> health() {
        HealthStatus status = new HealthStatus();
        status.setStatus("UP");
        status.setTimestamp(Instant.now());

        // Check database connectivity
        if (!databaseHealthCheck.isHealthy()) {
            status.setStatus("DOWN");
            status.addDetail("database", "Connection failed");
            return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE).body(status);
        }

        return ResponseEntity.ok(status);
    }

    @GetMapping("/ready")
    public ResponseEntity<String> readiness() {
        // Readiness check for Kubernetes
        return ResponseEntity.ok("READY");
    }

    @GetMapping("/live")
    public ResponseEntity<String> liveness() {
        // Liveness check for Kubernetes
        return ResponseEntity.ok("ALIVE");
    }
}
```

### 5. Application Configuration (application.yml)

```yaml
spring:
  application:
    name: ${SERVICE_NAME}

  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME}
    username: ${DB_USER}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000

  flyway:
    enabled: true
    locations: classpath:db/migration

server:
  port: 8080
  compression:
    enabled: true
  error:
    include-message: never
    include-stacktrace: never

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  metrics:
    export:
      prometheus:
        enabled: true

logging:
  level:
    com.homedepot: INFO
    org.springframework: WARN
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} - %msg%n"

# Home Depot specific configs
homedepot:
  service:
    name: ${SERVICE_NAME}
    version: ${VERSION:1.0.0}
  newrelic:
    enabled: true
    app-name: ${SERVICE_NAME}
  splunk:
    enabled: true
    index: services
```

### 6. Dockerfile (Multi-stage Build)

```dockerfile
# Build stage
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests

# Runtime stage
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app

# Add non-root user for security
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

COPY --from=build /app/target/*.jar app.jar

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s \
  CMD wget --quiet --tries=1 --spider http://localhost:8080/actuator/health || exit 1

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 7. Kubernetes Deployment

**deployment.yml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${SERVICE_NAME}
  namespace: homedepot-services
  labels:
    app: ${SERVICE_NAME}
    team: ${TEAM_NAME}
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ${SERVICE_NAME}
  template:
    metadata:
      labels:
        app: ${SERVICE_NAME}
    spec:
      containers:
      - name: ${SERVICE_NAME}
        image: homedepot/${SERVICE_NAME}:${VERSION}
        ports:
        - containerPort: 8080
          protocol: TCP
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "prod"
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: ${SERVICE_NAME}-secrets
              key: db-host
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /actuator/health/live
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/ready
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 5
```

### 8. CI/CD Pipeline (GitHub Actions)

**.github/workflows/ci.yml:**
```yaml
name: CI

on:
  pull_request:
    branches: [ main, develop ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3

    - name: Set up JDK 17
      uses: actions/setup-java@v3
      with:
        java-version: '17'
        distribution: 'temurin'

    - name: Build with Maven
      run: mvn clean verify

    - name: Run tests
      run: mvn test

    - name: Check coverage
      run: mvn jacoco:report jacoco:check

    - name: Security scan
      run: mvn dependency-check:check

    - name: SonarQube analysis
      run: mvn sonar:sonar
      env:
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

### 9. Logging Configuration

Integrate with Splunk for centralized logging:

```java
@Configuration
public class LoggingConfig {

    @Bean
    public LoggingInterceptor loggingInterceptor() {
        return new LoggingInterceptor();
    }
}

@Component
public class LoggingInterceptor implements HandlerInterceptor {

    private static final Logger logger = LoggerFactory.getLogger(LoggingInterceptor.class);

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        String requestId = UUID.randomUUID().toString();
        request.setAttribute("requestId", requestId);

        logger.info("Request - Method: {}, URI: {}, RequestID: {}",
                    request.getMethod(),
                    request.getRequestURI(),
                    requestId);

        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
        logger.info("Response - Status: {}, RequestID: {}, Duration: {}ms",
                    response.getStatus(),
                    request.getAttribute("requestId"),
                    System.currentTimeMillis() - (Long) request.getAttribute("startTime"));
    }
}
```

### 10. Global Exception Handler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger logger = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleException(Exception ex, HttpServletRequest request) {
        logger.error("Unhandled exception - RequestID: {}", request.getAttribute("requestId"), ex);

        ErrorResponse error = new ErrorResponse();
        error.setError("Internal server error");
        error.setRequestId(request.getAttribute("requestId").toString());
        error.setTimestamp(Instant.now());

        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex, HttpServletRequest request) {
        ErrorResponse error = new ErrorResponse();
        error.setError("Resource not found");
        error.setRequestId(request.getAttribute("requestId").toString());
        error.setTimestamp(Instant.now());

        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
}
```

### 11. README.md Template

Generate comprehensive README:

```markdown
# ${SERVICE_NAME}

## Overview
[Brief description of what this service does]

## Tech Stack
- Java 17
- Spring Boot 3.x
- PostgreSQL
- Docker/Kubernetes

## Getting Started

### Prerequisites
- JDK 17+
- Maven 3.8+
- Docker & Docker Compose
- kubectl (for K8s deployment)

### Local Development
\`\`\`bash
# Start dependencies
docker-compose up -d

# Run service
mvn spring-boot:run

# Run tests
mvn test
\`\`\`

## API Documentation
Swagger UI: http://localhost:8080/swagger-ui.html

## Monitoring
- Health: http://localhost:8080/actuator/health
- Metrics: http://localhost:8080/actuator/prometheus
- New Relic Dashboard: [link]

## Deployment
[Instructions for deploying to dev/staging/prod]

## Team
- Owner: [Team Name]
- Slack: #team-channel
- On-call: [PagerDuty rotation]
```

### 12. Testing Setup

Create sample integration test:

```java
@SpringBootTest
@AutoConfigureMockMvc
class IntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void healthCheckReturnsOk() throws Exception {
        mockMvc.perform(get("/actuator/health"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.status").value("UP"));
    }
}
```

## Checklist for New Service

When scaffolding complete, verify:

- [ ] All folders and files created
- [ ] Dependencies configured in pom.xml/build.gradle
- [ ] Health check endpoints working
- [ ] Database migrations set up (if needed)
- [ ] Logging configured for Splunk
- [ ] Dockerfile builds successfully
- [ ] Kubernetes manifests valid
- [ ] CI/CD pipeline configured
- [ ] README complete with all sections
- [ ] Tests pass locally
- [ ] Service runs in Docker Compose

## Apply These Patterns Automatically

When developer asks to create a new microservice:

1. Ask for service name and brief description
2. Generate complete folder structure
3. Create all configuration files with Home Depot standards
4. Set up health checks, logging, monitoring
5. Generate Dockerfile and K8s manifests
6. Create CI/CD pipeline
7. Add README with all documentation
8. Create sample controller and test

The service should be deployment-ready from the first commit.

## Tools Available
All tools (Read, Edit, Write, Bash, Grep, Glob)
