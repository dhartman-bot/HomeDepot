# Payment API Security Standards

## Description
Apply PCI DSS compliance requirements to all payment-related APIs at Home Depot.
Use this skill when creating or modifying endpoints that handle credit cards, customer payment data, financial transactions, or any endpoint in the payments service.

Activate when developer mentions: payment, credit card, transaction, checkout, billing, PCI, or working in `/services/payment-api/` directory.

## Instructions

Every payment endpoint must follow these Home Depot PCI DSS requirements:

### 1. Data Encryption

**At Rest:**
- Encrypt all payment data using AES-256
- Use Home Depot's `CryptoUtil` library (available in all services)
- Never store full card numbers - tokenize immediately
- Store only last 4 digits + token reference

**In Transit:**
- Require TLS 1.2 or higher for all connections
- Configure endpoint with `@RequiresTLS` annotation
- Reject non-HTTPS requests with 403

**Code Example:**
```java
@RestController
@RequiresTLS
public class PaymentController {

    @Autowired
    private CryptoUtil cryptoUtil;

    @PostMapping("/api/payments")
    public PaymentResponse processPayment(@RequestBody PaymentRequest request) {
        // Tokenize card immediately
        String token = cryptoUtil.tokenizeCard(request.getCardNumber());

        // Store only token + last 4 digits
        Payment payment = new Payment();
        payment.setToken(token);
        payment.setLast4Digits(request.getCardNumber().substring(request.getCardNumber().length() - 4));

        // Never log full card number
        logger.info("Processing payment - Last 4: {}", payment.getLast4Digits());

        return paymentService.process(payment);
    }
}
```

### 2. Input Validation

**Card Number Validation:**
- Validate using Luhn algorithm (use `CardValidator.isValid()`)
- Check card length (13-19 digits)
- Remove all spaces, dashes before validation
- Reject invalid cards immediately with 400 status

**Amount Validation:**
- Maximum transaction: $50,000 (configurable via `payment.max.amount`)
- Minimum transaction: $0.01
- Validate currency code (USD only for domestic)

**Request Sanitization:**
- Sanitize all string inputs to prevent injection
- Use `InputSanitizer.sanitize()` from security library
- Validate against allowed character sets

**Code Example:**
```java
public void validatePaymentRequest(PaymentRequest request) {
    // Validate card number
    if (!CardValidator.isValid(request.getCardNumber())) {
        throw new InvalidCardException("Invalid card number");
    }

    // Validate amount
    if (request.getAmount() > MAX_TRANSACTION_AMOUNT) {
        throw new AmountExceededException("Transaction exceeds maximum allowed");
    }

    if (request.getAmount() <= 0) {
        throw new InvalidAmountException("Amount must be positive");
    }

    // Sanitize inputs
    request.setBillingName(InputSanitizer.sanitize(request.getBillingName()));
    request.setBillingAddress(InputSanitizer.sanitize(request.getBillingAddress()));
}
```

### 3. Audit Logging

**Required Log Fields:**
- Timestamp (ISO 8601 format)
- User ID (customer ID, not email)
- Transaction amount
- Result (success/failure)
- Masked card number (last 4 digits only)
- Request ID (for tracing)
- Store ID or online indicator

**Log to Splunk:**
- Use `AuditLogger.logPaymentEvent()` method
- Tag with `source=payment_audit` for compliance queries
- Include custom index: `index=pci_compliance`

**What NOT to Log:**
- Full card numbers (NEVER)
- CVV codes (NEVER)
- Full expiration dates (log month/year only)
- Internal error messages that expose system info

**Code Example:**
```java
public void logPaymentEvent(PaymentEvent event) {
    Map<String, Object> auditData = new HashMap<>();
    auditData.put("timestamp", Instant.now().toString());
    auditData.put("userId", event.getUserId());
    auditData.put("amount", event.getAmount());
    auditData.put("result", event.getResult());
    auditData.put("last4", event.getCardLast4());
    auditData.put("requestId", event.getRequestId());
    auditData.put("storeId", event.getStoreId());

    // Send to Splunk with PCI compliance tags
    AuditLogger.logPaymentEvent(auditData, "pci_compliance", "payment_audit");
}
```

### 4. Error Handling

**Never Expose Internal Details:**
- Return generic error messages to clients
- Log detailed errors internally only
- Don't reveal payment processor names, gateway endpoints, etc.

**Standard Error Responses:**
```json
{
  "error": "Payment processing failed",
  "errorCode": "PAYMENT_DECLINED",
  "requestId": "req-123-abc",
  "supportContact": "1-800-HOME-DEPOT"
}
```

**Alert on Suspicious Patterns:**
- Multiple failed attempts from same user (>3 in 10 minutes)
- Invalid card format attempts (potential testing)
- Large transactions outside normal patterns
- Geographic anomalies

**Code Example:**
```java
@ExceptionHandler(PaymentException.class)
public ResponseEntity<ErrorResponse> handlePaymentException(PaymentException ex, HttpServletRequest request) {
    // Log detailed error internally
    logger.error("Payment failed - RequestID: {}, User: {}, Error: {}",
                 request.getAttribute("requestId"),
                 SecurityContextHolder.getContext().getAuthentication().getName(),
                 ex.getMessage(), ex);

    // Return generic error to client
    ErrorResponse response = new ErrorResponse();
    response.setError("Payment processing failed");
    response.setErrorCode(ex.getErrorCode());
    response.setRequestId(request.getAttribute("requestId").toString());
    response.setSupportContact("1-800-HOME-DEPOT");

    // Check for suspicious patterns
    if (fraudDetectionService.isSuspicious(ex, request)) {
        securityAlertService.notifySecurityTeam(ex, request);
    }

    return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(response);
}
```

### 5. Security Headers

Add these headers to all payment endpoints:

```java
response.setHeader("X-Content-Type-Options", "nosniff");
response.setHeader("X-Frame-Options", "DENY");
response.setHeader("X-XSS-Protection", "1; mode=block");
response.setHeader("Strict-Transport-Security", "max-age=31536000; includeSubDomains");
response.setHeader("Content-Security-Policy", "default-src 'self'");
```

### 6. Rate Limiting

- Maximum 10 payment attempts per user per minute
- Use `RateLimiter` annotation: `@RateLimit(maxRequests=10, window="1m")`
- Return 429 status when exceeded
- Include `Retry-After` header

### 7. Testing Requirements

Before marking payment endpoint complete, ensure:

- [ ] Unit tests for validation logic
- [ ] Integration tests with payment gateway (use sandbox)
- [ ] Security scan passes (no SQL injection, XSS, etc.)
- [ ] Load test handles 1000 requests/second
- [ ] PCI compliance checklist completed (use internal tool)

### 8. Code Review Checklist

When creating payment endpoint PR, verify:

- [ ] No full card numbers in code, logs, or comments
- [ ] All sensitive data encrypted
- [ ] Input validation comprehensive
- [ ] Audit logging complete with Splunk tags
- [ ] Error handling doesn't expose internals
- [ ] Security headers present
- [ ] Rate limiting configured
- [ ] Tests pass and coverage >90%
- [ ] PCI security review approved (tag @security-team)

## Apply These Standards Automatically

When developer asks to create or modify payment-related endpoints:

1. Generate code with all encryption, validation, logging in place
2. Include complete error handling with generic messages
3. Add audit logging with proper Splunk configuration
4. Set up rate limiting and security headers
5. Create unit and integration tests
6. Generate code review checklist for PR description

**The code should be PCI compliant from the first commit, not added later.**

## Tools Available
All tools (Read, Edit, Write, Bash, Grep, Glob)
