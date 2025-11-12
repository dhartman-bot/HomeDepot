# Home Depot Security Code Review

## Description
Perform comprehensive security review of code following Home Depot security standards and OWASP best practices.

Use this skill when:
- Developer asks for security review
- Code handles sensitive data (PII, payments, credentials)
- New endpoints are created
- External integrations are added
- Before production deployment

Activate for phrases like: "security review", "security check", "is this secure?", "check for vulnerabilities"

## Instructions

Perform thorough security analysis across all OWASP Top 10 categories and Home Depot-specific requirements:

### 1. Injection Vulnerabilities

**SQL Injection:**

Check for SQL injection vulnerabilities:
```bash
# Search for string concatenation in SQL
grep -r "\"SELECT.*+\|\"INSERT.*+\|\"UPDATE.*+" --include="*.java" .
```

**Vulnerable Patterns:**
```java
// ❌ VULNERABLE: String concatenation
String query = "SELECT * FROM users WHERE username = '" + username + "'";
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery(query);

// ❌ VULNERABLE: Format string
String query = String.format("SELECT * FROM orders WHERE id = %s", orderId);
```

**Secure Patterns:**
```java
// ✅ SECURE: Parameterized query
String query = "SELECT * FROM users WHERE username = ?";
PreparedStatement pstmt = conn.prepareStatement(query);
pstmt.setString(1, username);
ResultSet rs = pstmt.executeQuery();

// ✅ SECURE: JPA with named parameters
@Query("SELECT u FROM User u WHERE u.username = :username")
User findByUsername(@Param("username") String username);
```

**Check for:**
- [ ] All database queries use parameterized statements
- [ ] No string concatenation in SQL
- [ ] ORM queries use parameter binding
- [ ] Stored procedure calls use parameters

**Command Injection:**

Check for command injection vulnerabilities:
```bash
# Search for dangerous system calls
grep -r "Runtime.getRuntime().exec\|ProcessBuilder\|new ProcessBuilder" --include="*.java" .
```

**Vulnerable Pattern:**
```java
// ❌ VULNERABLE: User input in command
Runtime.getRuntime().exec("convert " + userFilename + " output.pdf");
```

**Secure Pattern:**
```java
// ✅ SECURE: Validated input with whitelist
if (!filename.matches("[a-zA-Z0-9_-]+\\.jpg")) {
    throw new ValidationException("Invalid filename");
}
ProcessBuilder pb = new ProcessBuilder("convert", filename, "output.pdf");
pb.start();
```

### 2. Authentication & Authorization

**Authentication Issues:**

Check for weak authentication:
- [ ] No hardcoded credentials
- [ ] Passwords hashed with bcrypt/Argon2 (not MD5/SHA1)
- [ ] JWT tokens signed with strong keys (HS256 minimum)
- [ ] Session tokens are cryptographically random
- [ ] Multi-factor authentication for sensitive operations

**Search for hardcoded credentials:**
```bash
# Search for hardcoded passwords/keys
grep -r -i "password.*=.*[\"\'][^\"\']*[\"\']" --include="*.java" --include="*.properties" .
grep -r -i "api[_-]?key.*=.*[\"\'][^\"\']*[\"\']" --include="*.java" --include="*.properties" .
```

**Vulnerable Patterns:**
```java
// ❌ VULNERABLE: Hardcoded credentials
String password = "admin123";
String apiKey = "sk-1234567890abcdef";

// ❌ VULNERABLE: Weak hashing
String hash = DigestUtils.md5Hex(password);
```

**Secure Patterns:**
```java
// ✅ SECURE: Environment variables
String password = System.getenv("DB_PASSWORD");
String apiKey = System.getenv("API_KEY");

// ✅ SECURE: Strong password hashing
BCryptPasswordEncoder encoder = new BCryptPasswordEncoder(12);
String hash = encoder.encode(password);
```

**Authorization Issues:**

Check for broken access control:
- [ ] Every endpoint has authorization checks
- [ ] User can only access their own resources
- [ ] Admin functions require admin role
- [ ] No direct object references (use indirect keys)

**Vulnerable Pattern:**
```java
// ❌ VULNERABLE: No authorization check
@GetMapping("/api/orders/{orderId}")
public Order getOrder(@PathVariable Long orderId) {
    return orderService.findById(orderId); // Anyone can access any order!
}
```

**Secure Pattern:**
```java
// ✅ SECURE: Authorization check
@GetMapping("/api/orders/{orderId}")
public Order getOrder(@PathVariable Long orderId, @AuthenticationPrincipal User user) {
    Order order = orderService.findById(orderId);

    // Verify user owns this order
    if (!order.getCustomerId().equals(user.getId()) && !user.isAdmin()) {
        throw new AccessDeniedException("Not authorized to access this order");
    }

    return order;
}
```

### 3. Sensitive Data Exposure

**Check for exposed sensitive data:**

```bash
# Search for logging sensitive data
grep -r "log.*password\|log.*ssn\|log.*creditCard\|log.*cardNumber" --include="*.java" .
```

**Data That Must Be Protected:**
- Credit card numbers (PCI DSS)
- Social Security Numbers
- Passwords/API keys
- Customer PII (name, email, phone, address)
- Health information (HIPAA if applicable)

**Vulnerable Patterns:**
```java
// ❌ VULNERABLE: Logging sensitive data
logger.info("Processing payment for card: {}", creditCardNumber);
logger.debug("User login - username: {}, password: {}", username, password);

// ❌ VULNERABLE: Storing unencrypted PII
customer.setSsn(ssn); // Stored in plaintext
```

**Secure Patterns:**
```java
// ✅ SECURE: Mask sensitive data in logs
logger.info("Processing payment for card ending in: {}", creditCardNumber.substring(creditCardNumber.length() - 4));

// ✅ SECURE: Never log passwords
logger.info("User login - username: {}", username);

// ✅ SECURE: Encrypt sensitive data
String encryptedSsn = cryptoUtil.encrypt(ssn);
customer.setEncryptedSsn(encryptedSsn);
```

**Encryption Requirements:**
- [ ] All sensitive data encrypted at rest (AES-256)
- [ ] TLS 1.2+ for data in transit
- [ ] Encryption keys stored in key management service (not code)
- [ ] Key rotation policy implemented

### 4. XML/JSON External Entities (XXE)

**Check XML parsing:**
```bash
# Find XML parsers
grep -r "DocumentBuilderFactory\|SAXParserFactory\|XMLInputFactory" --include="*.java" .
```

**Vulnerable Pattern:**
```java
// ❌ VULNERABLE: XXE attack possible
DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
DocumentBuilder builder = factory.newDocumentBuilder();
Document doc = builder.parse(userInput);
```

**Secure Pattern:**
```java
// ✅ SECURE: XXE protection
DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
factory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
factory.setFeature("http://xml.org/sax/features/external-general-entities", false);
factory.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
DocumentBuilder builder = factory.newDocumentBuilder();
```

### 5. Broken Access Control

**Check for insecure direct object references:**

```bash
# Find endpoints with path variables
grep -r "@PathVariable\|@RequestParam" --include="*.java" .
```

**Common Issues:**
- [ ] Check all endpoints validate user owns resource
- [ ] Admin endpoints require admin role
- [ ] No functions that bypass access control
- [ ] File paths validated (no path traversal)

**Path Traversal Check:**
```java
// ❌ VULNERABLE: Path traversal
@GetMapping("/files/{filename}")
public File getFile(@PathVariable String filename) {
    return new File("/uploads/" + filename); // Can access ../../etc/passwd
}

// ✅ SECURE: Validate path
@GetMapping("/files/{filename}")
public File getFile(@PathVariable String filename) {
    // Whitelist allowed characters
    if (!filename.matches("[a-zA-Z0-9_-]+\\.[a-zA-Z]{3,4}")) {
        throw new ValidationException("Invalid filename");
    }

    File file = new File("/uploads/" + filename);

    // Ensure file is within uploads directory
    if (!file.getCanonicalPath().startsWith("/uploads/")) {
        throw new SecurityException("Invalid file path");
    }

    return file;
}
```

### 6. Security Misconfiguration

**Check configurations:**

**CORS Configuration:**
```java
// ❌ VULNERABLE: Allowing all origins in production
@CrossOrigin(origins = "*")

// ✅ SECURE: Specific origins
@CrossOrigin(origins = "https://www.homedepot.com")
```

**Security Headers:**

Verify these headers are set:
```java
// Required security headers
response.setHeader("X-Content-Type-Options", "nosniff");
response.setHeader("X-Frame-Options", "DENY");
response.setHeader("X-XSS-Protection", "1; mode=block");
response.setHeader("Strict-Transport-Security", "max-age=31536000; includeSubDomains");
response.setHeader("Content-Security-Policy", "default-src 'self'");
```

**Error Handling:**
```java
// ❌ VULNERABLE: Exposing stack traces
@ExceptionHandler(Exception.class)
public String handleError(Exception ex) {
    return ex.toString(); // Exposes internal details
}

// ✅ SECURE: Generic error message
@ExceptionHandler(Exception.class)
public ResponseEntity<ErrorResponse> handleError(Exception ex, HttpServletRequest request) {
    logger.error("Error occurred - RequestID: {}", request.getAttribute("requestId"), ex);
    return ResponseEntity.status(500).body(new ErrorResponse("An error occurred"));
}
```

### 7. Cross-Site Scripting (XSS)

**Check for XSS vulnerabilities:**

```bash
# Find potential XSS issues
grep -r "innerHTML\|document.write\|eval(" --include="*.js" --include="*.jsx" .
```

**Vulnerable Patterns:**
```javascript
// ❌ VULNERABLE: Unescaped user input
element.innerHTML = userInput;
document.write(userComment);

// ❌ VULNERABLE: Eval with user data
eval("var x = " + userInput);
```

**Secure Patterns:**
```javascript
// ✅ SECURE: Use textContent (auto-escapes)
element.textContent = userInput;

// ✅ SECURE: Use React (auto-escapes)
<div>{userInput}</div>

// ✅ SECURE: Explicit sanitization if HTML needed
import DOMPurify from 'dompurify';
element.innerHTML = DOMPurify.sanitize(userInput);
```

**Backend XSS Prevention:**
```java
// Escape output in templates
<div th:text="${userInput}"></div> // ✅ Auto-escaped
<div th:utext="${userInput}"></div> // ❌ NOT escaped
```

### 8. Insecure Deserialization

**Check for unsafe deserialization:**

```bash
# Find deserialization
grep -r "readObject\|XMLDecoder\|ObjectInputStream" --include="*.java" .
```

**Vulnerable Pattern:**
```java
// ❌ VULNERABLE: Deserializing untrusted data
ObjectInputStream ois = new ObjectInputStream(userInputStream);
Object obj = ois.readObject();
```

**Secure Pattern:**
```java
// ✅ SECURE: Use JSON with schema validation
ObjectMapper mapper = new ObjectMapper();
User user = mapper.readValue(jsonString, User.class);
```

### 9. Components with Known Vulnerabilities

**Run dependency checks:**
```bash
# Java
mvn dependency-check:check
mvn versions:display-dependency-updates

# JavaScript
npm audit
npm outdated

# Python
pip-audit
safety check
```

**Verify:**
- [ ] No critical/high severity vulnerabilities
- [ ] All dependencies reasonably up-to-date
- [ ] No deprecated packages
- [ ] Licenses compatible with commercial use

### 10. Insufficient Logging & Monitoring

**Required Security Logging:**

Log these security events:
- [ ] Authentication attempts (success and failure)
- [ ] Authorization failures
- [ ] Input validation failures
- [ ] Sensitive operations (payment, PII access)
- [ ] Configuration changes
- [ ] Security exceptions

**Good Security Logging:**
```java
// Authentication
logger.info("Login successful - UserID: {}, IP: {}", userId, ipAddress);
logger.warn("Login failed - Username: {}, IP: {}, Reason: {}", username, ipAddress, reason);

// Authorization
logger.warn("Access denied - UserID: {}, Resource: {}, Action: {}", userId, resource, action);

// Sensitive operations
logger.info("Payment processed - UserID: {}, Amount: {}, Last4: {}", userId, amount, cardLast4);

// Security exceptions
logger.error("Security violation - UserID: {}, Type: {}, Details: {}", userId, violationType, details);
```

### 11. Home Depot Specific Security

**PCI DSS Compliance (Payment Data):**
- [ ] No storage of full card numbers (tokenize immediately)
- [ ] No storage of CVV/CVC codes (ever)
- [ ] Audit logging for all payment operations
- [ ] Encryption at rest and in transit
- [ ] Access restricted to authorized services

**PII Protection (Customer Data):**
- [ ] Customer SSN encrypted
- [ ] Email/phone masked in logs
- [ ] Access logged for audit
- [ ] Data retention policies enforced
- [ ] CCPA/GDPR compliance (right to deletion)

**Store Operations Security:**
- [ ] Store credentials not shared across stores
- [ ] POS integration uses mutual TLS
- [ ] Offline mode has proper security
- [ ] Store data synced securely

### 12. Rate Limiting & DDoS Protection

**Check for rate limiting:**
- [ ] Public endpoints have rate limits
- [ ] Login endpoints rate limited (prevent brute force)
- [ ] API endpoints return 429 when limit exceeded
- [ ] Rate limits per user/IP/API key

**Implementation:**
```java
@RateLimit(maxRequests = 10, window = "1m")
@PostMapping("/api/login")
public ResponseEntity<LoginResponse> login(@RequestBody LoginRequest request) {
    // Login logic
}
```

## Security Review Output

Generate security review in this format:

```markdown
# Security Review Summary

## 🔴 Critical Issues (Must Fix Immediately)
1. **SQL Injection Risk** - OrderController.java:145
   - Issue: String concatenation in SQL query
   - Risk: Database compromise
   - Fix: Use parameterized query
   - Code: [show vulnerable code and fix]

## 🟠 High Priority Issues
1. **Hardcoded API Key** - config/application.properties:42
   - Issue: API key committed to repository
   - Risk: Unauthorized access
   - Fix: Move to environment variable

2. **Missing Authorization Check** - UserController.java:89
   - Issue: No check if user owns resource
   - Risk: Data exposure
   - Fix: Add ownership validation

## 🟡 Medium Priority Issues
1. **Weak Password Hashing** - UserService.java:123
   - Issue: Using SHA-256 instead of bcrypt
   - Risk: Passwords easier to crack
   - Fix: Use BCryptPasswordEncoder

## 🟢 Low Priority / Recommendations
1. **Security Headers Missing** - WebSecurityConfig.java
   - Recommendation: Add CSP and other security headers
   - Impact: Defense in depth

## ✅ Security Strengths
- Strong input validation on all endpoints
- Proper error handling without info disclosure
- Up-to-date dependencies with no known vulnerabilities
- Good logging of security events

## 📊 Security Score: 7/10

### Breakdown:
- Injection Protection: 6/10 (SQL injection found)
- Authentication: 8/10 (Strong but missing MFA)
- Authorization: 7/10 (One missing check)
- Data Protection: 9/10 (Good encryption)
- Configuration: 8/10 (Headers missing)

## Action Required:
1. Fix critical SQL injection immediately
2. Remove hardcoded API key
3. Add missing authorization check
4. Schedule: Upgrade password hashing
```

## Automated Security Checks

Run these automatically:

```bash
# Static analysis
mvn spotbugs:check
npm run security:check

# Dependency vulnerabilities
mvn dependency-check:check
npm audit

# Secrets scanning
git secrets --scan

# OWASP ZAP dynamic scan (if staging environment available)
zap-cli quick-scan https://staging.homedepot.com
```

## Tools Available
All tools (Read, Edit, Write, Bash, Grep, Glob)
