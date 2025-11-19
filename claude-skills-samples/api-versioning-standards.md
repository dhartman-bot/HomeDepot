# API Versioning Standards Skill

## Description
Enforce Home Depot's API versioning, deprecation, and backward compatibility standards. Prevents breaking changes and ensures smooth API evolution across all services.

**Use when:** Creating new APIs, adding endpoints, modifying existing APIs, deprecating features, or planning API changes.

**Activate for phrases like:** "API version", "versioning", "deprecate", "breaking change", "backward compatible", "API evolution", "v1", "v2", "endpoint", "REST API"

## Instructions

### 1. Version Format Standards

Use semantic versioning in URL path:

```
Pattern: /api/v{major}/resource
Examples:
  - /api/v1/products
  - /api/v2/orders
  - /api/v1/stores/{storeId}/inventory
```

**Version Rules:**
- **Major (v1 → v2):** Breaking changes only
- **Minor/Patch:** NO URL changes - use headers or response fields
- **Never:** /api/v1.1/ or /api/v1-beta/ in URLs

### 2. What Constitutes a Breaking Change

| Change Type | Breaking? | Action Required |
|-------------|-----------|-----------------|
| Remove endpoint | YES | New major version |
| Remove required field | YES | New major version |
| Change field type | YES | New major version |
| Rename field | YES | New major version |
| Add required field to request | YES | New major version |
| Add optional field to request | NO | Minor version |
| Add field to response | NO | Minor version |
| Add new endpoint | NO | Minor version |
| Performance improvements | NO | Patch version |

### 3. API Controller Structure

```java
// Always use explicit version in path
@RestController
@RequestMapping("/api/v1/products")
@Tag(name = "Products API v1", description = "Product catalog operations")
public class ProductControllerV1 {

    // Document version lifecycle in OpenAPI
    @Operation(
        summary = "Get product by SKU",
        description = "Retrieves product details. **Stable since v1.0**"
    )
    @GetMapping("/{skuId}")
    public ProductResponse getProduct(@PathVariable String skuId) {
        return productService.getProduct(skuId);
    }

    // Mark deprecated endpoints clearly
    @Deprecated(since = "2024-01", forRemoval = true)
    @Operation(
        summary = "Get product by UPC (DEPRECATED)",
        description = "**DEPRECATED:** Use GET /products/{skuId} instead. " +
                      "Will be removed 2024-07-01.",
        deprecated = true
    )
    @GetMapping("/by-upc/{upc}")
    public ProductResponse getProductByUpc(@PathVariable String upc) {
        // Log deprecation usage for monitoring
        deprecationMetrics.recordUsage("products.by-upc", getClientId());
        return productService.getProductByUpc(upc);
    }
}

// V2 controller for breaking changes
@RestController
@RequestMapping("/api/v2/products")
@Tag(name = "Products API v2", description = "Product catalog - enhanced schema")
public class ProductControllerV2 {

    @Operation(summary = "Get product by SKU with enhanced details")
    @GetMapping("/{skuId}")
    public ProductResponseV2 getProduct(@PathVariable String skuId) {
        // V2 returns enhanced schema
        return productService.getProductV2(skuId);
    }
}
```

### 4. Response Schema Evolution

**Adding fields (non-breaking):**
```java
// V1 Response - original
public record ProductResponseV1(
    String skuId,
    String name,
    BigDecimal price
) {}

// V1 Response - evolved (still v1, added optional fields)
public record ProductResponseV1(
    String skuId,
    String name,
    BigDecimal price,
    // New fields added - clients should ignore unknown fields
    @Schema(description = "Added in v1.2")
    String brand,
    @Schema(description = "Added in v1.3")
    List<String> categories
) {}
```

**Breaking changes require v2:**
```java
// V2 Response - breaking schema change
public record ProductResponseV2(
    String skuId,
    String name,
    // BREAKING: price changed from BigDecimal to Money object
    Money price,
    // BREAKING: categories changed from List<String> to structured objects
    List<Category> categories,
    // BREAKING: dimensions now required (was absent in v1)
    Dimensions dimensions
) {}
```

### 5. Deprecation Process

Follow the 6-month deprecation timeline:

```java
@Component
public class DeprecationManager {

    /**
     * Deprecation Timeline:
     * 1. Announce: Mark deprecated in OpenAPI + Changelog
     * 2. Monitor: Track usage for 30 days
     * 3. Notify: Contact high-usage clients directly
     * 4. Sunset: Remove after 6 months minimum
     */

    // Step 1: Add deprecation header to responses
    @Aspect
    @Component
    public class DeprecationHeaderAspect {

        @Around("@annotation(Deprecated)")
        public Object addDeprecationHeader(ProceedingJoinPoint joinPoint) throws Throwable {
            Object result = joinPoint.proceed();

            HttpServletResponse response = getCurrentResponse();
            Deprecated annotation = getDeprecatedAnnotation(joinPoint);

            // Standard deprecation headers
            response.setHeader("Deprecation", annotation.since());
            response.setHeader("Sunset", annotation.forRemoval() ?
                calculateSunsetDate(annotation.since()) : "");
            response.setHeader("Link", "</api/v2/products>; rel=\"successor-version\"");

            return result;
        }
    }

    // Step 2: Monitor deprecated endpoint usage
    public void recordDeprecatedUsage(String endpoint, String clientId) {
        // Track in Splunk for reporting
        auditLogger.log("DEPRECATED_API_USAGE", Map.of(
            "endpoint", endpoint,
            "clientId", clientId,
            "timestamp", Instant.now()
        ));

        // Increment metric
        meterRegistry.counter("api.deprecated.calls",
            "endpoint", endpoint,
            "client", clientId
        ).increment();
    }

    // Step 3: Generate deprecation report
    @Scheduled(cron = "0 0 9 * * MON")  // Every Monday 9am
    public void generateWeeklyDeprecationReport() {
        List<DeprecatedEndpointUsage> usage = getDeprecatedUsage(Duration.ofDays(7));

        // Alert clients still using deprecated endpoints
        for (DeprecatedEndpointUsage u : usage) {
            if (u.getCallCount() > 100) {
                notifyClientTeam(u.getClientId(), u.getEndpoint(), u.getSunsetDate());
            }
        }
    }
}
```

### 6. Version Negotiation

Support header-based minor version selection:

```java
@RestController
@RequestMapping("/api/v1/products")
public class ProductController {

    // Client can request specific minor version via header
    // X-API-Version: 1.3
    @GetMapping("/{skuId}")
    public ResponseEntity<ProductResponse> getProduct(
            @PathVariable String skuId,
            @RequestHeader(value = "X-API-Version", defaultValue = "1.0") String apiVersion) {

        ProductResponse response = productService.getProduct(skuId);

        // Add version metadata to response
        return ResponseEntity.ok()
            .header("X-API-Version", apiVersion)
            .header("X-API-Latest", "1.5")
            .body(response);
    }
}
```

### 7. Changelog Requirements

Every API change must be documented:

```markdown
# Product API Changelog

## [v2.0.0] - 2024-03-01
### Breaking Changes
- `price` field changed from `number` to `Money` object with `amount` and `currency`
- `categories` changed from `string[]` to `Category[]` with `id` and `name`
- Added required `dimensions` field

### Migration Guide
```json
// v1 response
{ "price": 29.99 }

// v2 response
{ "price": { "amount": 29.99, "currency": "USD" } }
```

## [v1.5.0] - 2024-02-15
### Added
- `brand` field to product response
- `relatedSkus` array for product recommendations

### Deprecated
- `GET /products/by-upc/{upc}` - Use `GET /products/{skuId}` instead
  Sunset date: 2024-08-15

## [v1.4.0] - 2024-01-10
### Added
- Pagination support via `page` and `size` query parameters
- `totalCount` field in list responses
```

### 8. Client SDK Versioning

When generating client SDKs:

```java
// SDK should match API major version
// Maven: com.homedepot:product-api-client:1.x.x for v1
// Maven: com.homedepot:product-api-client:2.x.x for v2

public class ProductApiClient {

    private final String baseUrl;
    private final String apiVersion;

    public ProductApiClient(String baseUrl) {
        this.baseUrl = baseUrl;
        this.apiVersion = "1";  // Hardcoded to major version this SDK supports
    }

    public ProductResponse getProduct(String skuId) {
        return webClient.get()
            .uri(baseUrl + "/api/v" + apiVersion + "/products/" + skuId)
            .header("X-Client-SDK-Version", getClass().getPackage().getImplementationVersion())
            .retrieve()
            .bodyToMono(ProductResponse.class)
            .block();
    }
}
```

### 9. Testing Version Compatibility

```java
@SpringBootTest
class ApiVersioningTest {

    @Test
    void v1ClientShouldWorkWithV1Api() {
        // Basic compatibility
    }

    @Test
    void v1ClientShouldIgnoreNewFieldsInV1Response() {
        // Clients must ignore unknown fields
        String json = """
            {
                "skuId": "123456789",
                "name": "Hammer",
                "price": 19.99,
                "newFieldAddedInV1_5": "should be ignored"
            }
            """;

        ProductResponseV1 response = objectMapper.readValue(json, ProductResponseV1.class);
        assertThat(response.skuId()).isEqualTo("123456789");
        // No exception thrown for unknown field
    }

    @Test
    void deprecatedEndpointShouldReturnDeprecationHeaders() {
        ResponseEntity<ProductResponse> response = restTemplate.getForEntity(
            "/api/v1/products/by-upc/123456789012",
            ProductResponse.class
        );

        assertThat(response.getHeaders().get("Deprecation")).isNotNull();
        assertThat(response.getHeaders().get("Sunset")).isNotNull();
    }

    @Test
    void v2EndpointShouldRejectOldClients() {
        // Optional: If you want to enforce minimum client version
    }
}
```

### 10. OpenAPI Documentation

Always include version metadata:

```yaml
openapi: 3.0.3
info:
  title: Product API
  version: 1.5.0
  description: |
    Product catalog API for Home Depot systems.

    ## Versioning Policy
    - Major versions (v1, v2) in URL path
    - Minor versions in X-API-Version header
    - 6-month deprecation notice for breaking changes

    ## Current Versions
    - **v1** (Current): Stable, recommended for production
    - **v2** (Preview): Enhanced schema, available for testing

    ## Deprecated Endpoints
    - `GET /products/by-upc/{upc}` - Sunset: 2024-08-15

servers:
  - url: https://api.homedepot.com/api/v1
    description: Production v1
  - url: https://api.homedepot.com/api/v2
    description: Production v2 (Preview)

paths:
  /products/{skuId}:
    get:
      operationId: getProduct
      x-api-version-added: "1.0.0"
      x-api-version-current: "1.5.0"
```

## Output Format

When creating or modifying APIs, always include:
1. Explicit version in URL path (/api/v1/)
2. OpenAPI documentation with version metadata
3. Deprecation headers for sunset endpoints
4. Changelog entry for all changes
5. Version compatibility tests
6. Migration guide for breaking changes

## Tools Available
All tools (Read, Edit, Write, Bash, Grep, Glob)
