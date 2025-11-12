# Home Depot Code Review Standards

## Description
Perform automated code review checks following Home Depot's quality standards.

Use this skill when developer:
- Asks for code review
- Creates a pull request
- Wants to check if code is ready for review
- Asks "is this code ready to submit?"

Activate for phrases like: "review this code", "create PR", "ready for review", "check my code"

## Instructions

Perform comprehensive code review following Home Depot's standards across all categories:

### 1. Code Style & Formatting

**Run Linters:**
```bash
# Java projects
mvn checkstyle:check
mvn spotless:check

# JavaScript/TypeScript projects
npm run lint
npm run format:check

# Python projects
black --check .
flake8 .
mypy .
```

**Verify:**
- [ ] No linting errors
- [ ] Code follows team style guide
- [ ] Consistent naming conventions (camelCase for Java, snake_case for Python, etc.)
- [ ] No commented-out code blocks (remove or explain)
- [ ] No TODO/FIXME without JIRA ticket reference

**Auto-fix if possible:**
```bash
# Java
mvn spotless:apply

# JavaScript/TypeScript
npm run lint:fix

# Python
black .
```

### 2. Testing Requirements

**Coverage Check:**
```bash
# Java
mvn jacoco:report
mvn jacoco:check -Dcoverage.minimum=80

# JavaScript
npm run test:coverage -- --coverageThreshold=80

# Python
pytest --cov=. --cov-report=term --cov-fail-under=80
```

**Verify:**
- [ ] Unit tests exist for new functions/methods
- [ ] Integration tests for new endpoints/features
- [ ] Test coverage ≥ 80% (project-wide)
- [ ] New code coverage ≥ 90% (for new files)
- [ ] All tests pass locally
- [ ] No flaky tests (tests pass consistently)
- [ ] Test names clearly describe what they test

**Test Quality Checks:**
- Tests follow AAA pattern (Arrange, Act, Assert)
- No sleeps/waits in tests (use proper async handling)
- Test data is self-contained (no external dependencies)
- Mocks are used appropriately (don't mock everything)

### 3. Security Scan

**Run Security Checks:**
```bash
# Dependency vulnerabilities
mvn dependency-check:check
npm audit
pip-audit

# Static security analysis
# Java
mvn spotbugs:check -Deffort=Max

# JavaScript
npm run security:check

# Python
bandit -r . -ll
```

**Manual Security Review:**
- [ ] No hardcoded secrets, API keys, passwords
- [ ] No SQL queries vulnerable to injection
- [ ] Input validation on all user-facing endpoints
- [ ] Authentication/authorization checks present
- [ ] Sensitive data encrypted (PII, payment data)
- [ ] No exposure of internal error details to clients
- [ ] CORS configured properly (not `*` in production)
- [ ] Rate limiting on public endpoints

**Check for Common Vulnerabilities:**
- SQL Injection: Check for string concatenation in queries
- XSS: Check for unescaped user input in HTML
- CSRF: Verify CSRF tokens for state-changing operations
- Path Traversal: Check file operations validate paths
- Insecure Deserialization: Avoid deserializing untrusted data

### 4. Performance Considerations

**Check for Performance Issues:**
- [ ] No N+1 query problems (check database access patterns)
- [ ] Proper indexing on database queries
- [ ] Caching used where appropriate (Redis/in-memory)
- [ ] Large collections paginated
- [ ] Async operations for long-running tasks
- [ ] Connection pooling configured correctly
- [ ] File uploads have size limits
- [ ] API responses include pagination for lists

**Database Query Review:**
```java
// ❌ BAD: N+1 query problem
for (Order order : orders) {
    order.getItems(); // Triggers separate query each time
}

// ✅ GOOD: Fetch join
@Query("SELECT o FROM Order o LEFT JOIN FETCH o.items WHERE o.id IN :ids")
List<Order> findOrdersWithItems(@Param("ids") List<Long> ids);
```

### 5. Documentation

**Code Documentation:**
- [ ] Public APIs have JSDoc/Javadoc comments
- [ ] Complex logic has explanatory comments
- [ ] README updated if public API changed
- [ ] API documentation (Swagger/OpenAPI) updated
- [ ] CHANGELOG.md entry added with PR number

**Required for Public Methods:**
```java
/**
 * Processes a customer payment transaction.
 *
 * @param paymentRequest the payment details including card info and amount
 * @return PaymentResponse containing transaction ID and status
 * @throws PaymentException if payment processing fails
 * @throws ValidationException if payment request is invalid
 */
public PaymentResponse processPayment(PaymentRequest paymentRequest) {
    // implementation
}
```

### 6. Error Handling

**Verify Proper Error Handling:**
- [ ] All exceptions caught at appropriate level
- [ ] No empty catch blocks
- [ ] Errors logged with sufficient context
- [ ] User-facing errors are friendly (no stack traces)
- [ ] Internal errors include request ID for debugging
- [ ] Database transactions rolled back on error
- [ ] Resources cleaned up in finally blocks or try-with-resources

**Good Error Handling Example:**
```java
try {
    paymentService.processPayment(request);
} catch (PaymentProcessorException ex) {
    logger.error("Payment failed - RequestID: {}, UserID: {}, Error: {}",
                 requestId, userId, ex.getMessage(), ex);

    // User-friendly error
    return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                         .body(new ErrorResponse("Payment processing failed. Please try again."));
} catch (Exception ex) {
    logger.error("Unexpected error - RequestID: {}", requestId, ex);

    // Generic error for unexpected issues
    return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                         .body(new ErrorResponse("An error occurred. Please contact support."));
}
```

### 7. Code Complexity

**Check Complexity Metrics:**
```bash
# Java
mvn pmd:check

# JavaScript
npm run complexity-check

# General: Check cyclomatic complexity
# Methods should have complexity < 10
# Classes should have complexity < 50
```

**Refactor if Needed:**
- [ ] No methods longer than 50 lines
- [ ] No classes longer than 300 lines
- [ ] Cyclomatic complexity < 10 per method
- [ ] Deeply nested code refactored (max 3 levels)
- [ ] Duplicated code extracted to helper methods

### 8. Git Hygiene

**Commit Message Quality:**
- [ ] Commit messages follow conventional commits format
- [ ] Each commit is atomic (one logical change)
- [ ] No merge commits in feature branch (rebase instead)

**Good Commit Message Format:**
```
feat(payments): add support for Apple Pay

- Integrate Apple Pay SDK
- Add Apple Pay button to checkout
- Update payment processing to handle Apple Pay tokens

Closes HD-1234
```

**Branch Hygiene:**
- [ ] Branch name follows pattern: `feature/HD-1234-short-description`
- [ ] Branch is up to date with main/develop
- [ ] No unnecessary files committed (.DS_Store, .idea/, etc.)

### 9. Dependencies

**Check Dependencies:**
```bash
# Check for outdated dependencies
mvn versions:display-dependency-updates
npm outdated
pip list --outdated
```

**Verify:**
- [ ] No unnecessary dependencies added
- [ ] New dependencies approved by architecture team
- [ ] Dependencies have no known vulnerabilities
- [ ] License compatibility verified (no GPL in production)
- [ ] Version pinned (not using `LATEST` or `*`)

### 10. Backwards Compatibility

**Breaking Change Check:**
- [ ] No breaking API changes without migration plan
- [ ] Database migrations are backwards compatible
- [ ] Feature flags used for risky changes
- [ ] Deprecation warnings added for removed features

### 11. Observability

**Logging & Monitoring:**
- [ ] Appropriate log levels used (INFO for business events, ERROR for failures)
- [ ] No sensitive data in logs (passwords, card numbers, SSNs)
- [ ] Structured logging format (JSON)
- [ ] Key operations include metrics/timing
- [ ] New endpoints have monitoring/alerting configured

**Required Logging for Business Operations:**
```java
// Log key business events
logger.info("Order created - OrderID: {}, CustomerID: {}, Amount: {}, StoreID: {}",
            orderId, customerId, amount, storeId);

// Log performance metrics
long duration = System.currentTimeMillis() - startTime;
logger.info("Payment processed - Duration: {}ms, Status: {}", duration, status);
```

### 12. Home Depot Specific Standards

**Retail Operations:**
- [ ] Store ID validation for store-related operations
- [ ] SKU validation for product operations
- [ ] Inventory checks before order confirmation

**Data Handling:**
- [ ] Customer PII encrypted and logged per compliance
- [ ] Payment data follows PCI DSS requirements
- [ ] Data retention policies implemented

**Integration Points:**
- [ ] Oracle Retail integration patterns followed
- [ ] SAP integration uses approved connectors
- [ ] Legacy system calls include circuit breakers

## Review Output Format

Generate a review summary in this format:

```markdown
# Code Review Summary

## ✅ Passing Checks (X/12)
- Code style and formatting
- Test coverage (85%)
- Security scan clean
[... list all passing checks]

## ⚠️ Issues Found

### High Priority (Must Fix Before Merge)
1. **Security**: Hardcoded API key found in `config/application.properties`
   - File: config/application.properties:42
   - Fix: Move to environment variable

2. **Testing**: Coverage below threshold (65%, need 80%)
   - Missing tests for: PaymentService.processRefund()
   - Fix: Add unit tests

### Medium Priority (Should Fix)
1. **Performance**: N+1 query detected in OrderController.getOrders()
   - File: OrderController.java:156
   - Fix: Use fetch join

### Low Priority (Nice to Have)
1. **Documentation**: Missing Javadoc on public method
   - File: InventoryService.java:89
   - Fix: Add method documentation

## 📊 Metrics
- Test Coverage: 85%
- Code Complexity: 6.2 (Good)
- Security Issues: 1 (Fix before merge)
- Documentation: 90% complete

## ✓ Ready for Review
- [ ] All high priority issues fixed
- [ ] Tests passing
- [ ] Security scan clean
```

## Automated Actions

After review, automatically:

1. **Run all checks** (linting, tests, security scans)
2. **Generate review summary** with specific line numbers
3. **Categorize issues** by priority (high/medium/low)
4. **Provide specific fixes** with code examples
5. **Update PR description** with checklist
6. **Add labels** to PR based on findings (needs-tests, security-review, etc.)

## PR Description Template

Generate PR description:

```markdown
## Description
[Brief description of changes]

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Manual testing completed

## Checklist
- [ ] Code follows style guidelines
- [ ] Tests pass locally
- [ ] Coverage ≥ 80%
- [ ] Security scan clean
- [ ] Documentation updated
- [ ] No breaking changes (or migration plan included)

## Related Issues
Closes HD-1234

## Screenshots (if applicable)
[Add screenshots for UI changes]
```

## Tools Available
All tools (Read, Edit, Write, Bash, Grep, Glob)
