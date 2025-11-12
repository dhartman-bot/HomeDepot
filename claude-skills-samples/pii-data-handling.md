# Home Depot PII Data Handling Standards

## Description
Ensure proper handling of Personally Identifiable Information (PII) to comply with CCPA, GDPR, and Home Depot privacy policies.

Use this skill when working with customer data including:
- Names, email addresses, phone numbers
- Physical addresses
- Social Security Numbers
- Date of birth
- Payment information
- Driver's license numbers
- Any data that can identify an individual

Activate for phrases like: "customer data", "user information", "personal data", "PII", "privacy", or when working with User, Customer, Profile, or Account models.

## Instructions

Apply Home Depot's privacy and data protection standards to all code handling PII:

### 1. PII Identification

**Automatically identify PII fields:**

Common PII fields in Home Depot systems:
- `firstName`, `lastName`, `fullName`
- `email`, `emailAddress`
- `phone`, `phoneNumber`, `mobileNumber`
- `address`, `streetAddress`, `homeAddress`
- `ssn`, `socialSecurityNumber`
- `dateOfBirth`, `birthDate`
- `driversLicense`, `governmentId`
- `creditCardNumber`, `bankAccount`

**Mark PII in Data Models:**

```java
@Entity
@Table(name = "customers")
public class Customer {

    @Id
    private Long id;

    @PII(type = PII_Type.NAME)
    @Column(nullable = false)
    private String firstName;

    @PII(type = PII_Type.NAME)
    @Column(nullable = false)
    private String lastName;

    @PII(type = PII_Type.EMAIL)
    @Column(unique = true, nullable = false)
    private String email;

    @PII(type = PII_Type.PHONE)
    private String phoneNumber;

    @PII(type = PII_Type.ADDRESS)
    @Embedded
    private Address address;

    @PII(type = PII_Type.SSN, encrypted = true)
    @Column(name = "encrypted_ssn")
    private String encryptedSsn;

    // Non-PII fields don't need annotation
    private Instant createdAt;
    private String preferredStore;
}
```

### 2. Encryption Requirements

**Encrypt Sensitive PII at Rest:**

**High Sensitivity (Must Encrypt):**
- Social Security Numbers
- Driver's License Numbers
- Credit Card Numbers
- Bank Account Numbers
- Health Information

**Medium Sensitivity (Should Encrypt):**
- Full Addresses
- Phone Numbers
- Email Addresses (in some contexts)

**Encryption Implementation:**

```java
@Service
public class CustomerService {

    @Autowired
    private CryptoUtil cryptoUtil;

    public void saveCustomer(CustomerDTO dto) {
        Customer customer = new Customer();

        // Regular PII - store as-is (encrypted at DB level)
        customer.setFirstName(dto.getFirstName());
        customer.setEmail(dto.getEmail());

        // High-sensitivity PII - encrypt before storage
        if (dto.getSsn() != null) {
            String encryptedSsn = cryptoUtil.encrypt(dto.getSsn());
            customer.setEncryptedSsn(encryptedSsn);
        }

        customerRepository.save(customer);
    }

    public CustomerDTO getCustomer(Long customerId, User requestingUser) {
        Customer customer = customerRepository.findById(customerId)
            .orElseThrow(() -> new NotFoundException("Customer not found"));

        // Authorization check
        if (!canAccessCustomerData(requestingUser, customer)) {
            throw new AccessDeniedException("Not authorized");
        }

        // Decrypt sensitive data only if authorized
        CustomerDTO dto = new CustomerDTO();
        dto.setFirstName(customer.getFirstName());
        dto.setEmail(customer.getEmail());

        if (requestingUser.hasPermission("VIEW_SSN")) {
            String decryptedSsn = cryptoUtil.decrypt(customer.getEncryptedSsn());
            dto.setSsn(decryptedSsn);
        }

        // Log access for audit
        auditLogger.logPiiAccess(requestingUser.getId(), customerId, "VIEW");

        return dto;
    }
}
```

### 3. Logging and Masking

**Never Log Full PII:**

```java
// ❌ BAD: Logging full PII
logger.info("Customer login - email: {}, phone: {}", email, phone);
logger.debug("Processing order for: {}", customer.toString()); // May expose PII

// ✅ GOOD: Masked PII in logs
logger.info("Customer login - email: {}, phone: {}",
            maskEmail(email),
            maskPhone(phone));
```

**Masking Utilities:**

```java
@Component
public class PiiMaskingUtil {

    /**
     * Masks email: john.doe@example.com -> j***@example.com
     */
    public String maskEmail(String email) {
        if (email == null || !email.contains("@")) {
            return "***";
        }
        String[] parts = email.split("@");
        return parts[0].charAt(0) + "***@" + parts[1];
    }

    /**
     * Masks phone: 555-123-4567 -> ***-***-4567
     */
    public String maskPhone(String phone) {
        if (phone == null || phone.length() < 4) {
            return "***";
        }
        return "***-***-" + phone.substring(phone.length() - 4);
    }

    /**
     * Masks SSN: 123-45-6789 -> ***-**-6789
     */
    public String maskSsn(String ssn) {
        if (ssn == null || ssn.length() < 4) {
            return "***";
        }
        return "***-**-" + ssn.substring(ssn.length() - 4);
    }

    /**
     * Masks name: John Smith -> J*** S***
     */
    public String maskName(String name) {
        if (name == null || name.isEmpty()) {
            return "***";
        }
        String[] parts = name.split(" ");
        StringBuilder masked = new StringBuilder();
        for (String part : parts) {
            if (part.length() > 0) {
                masked.append(part.charAt(0)).append("*** ");
            }
        }
        return masked.toString().trim();
    }

    /**
     * Masks address: 123 Main St -> ***
     */
    public String maskAddress(String address) {
        return "***";
    }
}
```

**Structured Logging with Masked PII:**

```java
@Component
@Slf4j
public class AuditLogger {

    @Autowired
    private PiiMaskingUtil maskingUtil;

    public void logCustomerAccess(Long userId, Customer customer, String action) {
        Map<String, Object> logData = new HashMap<>();
        logData.put("userId", userId);
        logData.put("customerId", customer.getId());
        logData.put("action", action);
        logData.put("email", maskingUtil.maskEmail(customer.getEmail()));
        logData.put("timestamp", Instant.now());

        // Send to Splunk with PII audit tag
        logger.info("Customer access - {}", new ObjectMapper().writeValueAsString(logData));
    }
}
```

### 4. Error Handling

**Never Expose PII in Error Messages:**

```java
// ❌ BAD: PII in error message
throw new ValidationException("Invalid email: " + email);
return ResponseEntity.badRequest().body("User " + email + " not found");

// ✅ GOOD: Generic error without PII
throw new ValidationException("Invalid email format");
return ResponseEntity.badRequest().body("User not found");

// Log details internally (with masking)
logger.warn("Email validation failed - email: {}", maskEmail(email));
```

**Error Response Format:**

```java
@ExceptionHandler(ValidationException.class)
public ResponseEntity<ErrorResponse> handleValidation(ValidationException ex, HttpServletRequest request) {
    // Log with context (masked PII)
    logger.warn("Validation error - RequestID: {}, Error: {}",
                request.getAttribute("requestId"),
                ex.getMessage());

    // Generic error to client (no PII)
    ErrorResponse response = new ErrorResponse();
    response.setError("Validation failed");
    response.setErrorCode("VALIDATION_ERROR");
    response.setRequestId(request.getAttribute("requestId").toString());

    return ResponseEntity.badRequest().body(response);
}
```

### 5. Access Control and Authorization

**Implement Role-Based Access for PII:**

```java
@RestController
@RequestMapping("/api/customers")
public class CustomerController {

    @GetMapping("/{customerId}")
    @PreAuthorize("hasPermission(#customerId, 'Customer', 'READ')")
    public CustomerDTO getCustomer(@PathVariable Long customerId,
                                   @AuthenticationPrincipal User user) {

        Customer customer = customerService.findById(customerId);

        // Users can only view their own data
        if (!customer.getUserId().equals(user.getId()) && !user.isAdmin()) {
            throw new AccessDeniedException("Not authorized to view this customer");
        }

        // Return appropriate level of detail based on role
        if (user.hasRole("CUSTOMER_SERVICE")) {
            return toDetailedDTO(customer); // Full PII for CS reps
        } else if (user.hasRole("CUSTOMER")) {
            return toBasicDTO(customer); // Own data only
        } else {
            return toMinimalDTO(customer); // Masked data
        }
    }
}
```

**Permission Checking:**

```java
@Component
public class PiiPermissionEvaluator implements PermissionEvaluator {

    @Override
    public boolean hasPermission(Authentication auth, Object targetDomainObject, Object permission) {
        User user = (User) auth.getPrincipal();

        if (targetDomainObject instanceof Customer) {
            Customer customer = (Customer) targetDomainObject;

            // User can access their own data
            if (customer.getUserId().equals(user.getId())) {
                return true;
            }

            // Customer service can access with valid reason
            if (user.hasRole("CUSTOMER_SERVICE") && hasValidAccessReason()) {
                auditLogger.logPiiAccess(user.getId(), customer.getId(), "CS_ACCESS");
                return true;
            }

            // Admins need special permission
            if (user.hasRole("ADMIN") && user.hasPermission("VIEW_ALL_PII")) {
                auditLogger.logPiiAccess(user.getId(), customer.getId(), "ADMIN_ACCESS");
                return true;
            }
        }

        return false;
    }
}
```

### 6. Data Retention and Deletion

**Implement Data Retention Policies:**

```java
@Service
public class DataRetentionService {

    /**
     * CCPA: Delete customer data upon request
     */
    @Transactional
    public void deleteCustomerData(Long customerId, String reason) {
        Customer customer = customerRepository.findById(customerId)
            .orElseThrow(() -> new NotFoundException("Customer not found"));

        // Log deletion for audit
        auditLogger.logDataDeletion(customerId, reason);

        // Anonymize instead of hard delete (preserve referential integrity)
        customer.setFirstName("DELETED");
        customer.setLastName("USER");
        customer.setEmail("deleted_" + customerId + "@anonymized.com");
        customer.setPhoneNumber(null);
        customer.setAddress(null);
        customer.setEncryptedSsn(null);
        customer.setDeletionDate(Instant.now());
        customer.setDeletedReason(reason);

        customerRepository.save(customer);

        // Also delete from related tables
        orderService.anonymizeCustomerOrders(customerId);
        reviewService.anonymizeCustomerReviews(customerId);
    }

    /**
     * Auto-delete inactive accounts per policy
     */
    @Scheduled(cron = "0 0 2 * * ?") // Daily at 2 AM
    public void cleanupInactiveAccounts() {
        Instant cutoffDate = Instant.now().minus(7, ChronoUnit.YEARS);

        List<Customer> inactive = customerRepository
            .findByLastLoginBeforeAndNotDeleted(cutoffDate);

        logger.info("Found {} inactive accounts for cleanup", inactive.size());

        for (Customer customer : inactive) {
            deleteCustomerData(customer.getId(), "INACTIVE_7_YEARS");
        }
    }
}
```

### 7. API Response Filtering

**Filter PII Based on Consumer:**

```java
@RestController
public class CustomerApiController {

    /**
     * Public API - minimal PII
     */
    @GetMapping("/public/customers/{id}")
    public PublicCustomerDTO getPublicCustomer(@PathVariable Long id) {
        Customer customer = customerService.findById(id);

        // Only non-sensitive data
        PublicCustomerDTO dto = new PublicCustomerDTO();
        dto.setId(customer.getId());
        dto.setDisplayName(customer.getFirstName()); // First name only
        dto.setMemberSince(customer.getCreatedAt());
        return dto;
    }

    /**
     * Internal API - full PII for authorized services
     */
    @GetMapping("/internal/customers/{id}")
    @PreAuthorize("hasRole('SERVICE')")
    public FullCustomerDTO getFullCustomer(@PathVariable Long id,
                                           @AuthenticationPrincipal ServiceAccount service) {
        // Verify service is authorized for PII access
        if (!service.hasPermission("READ_PII")) {
            throw new AccessDeniedException("Service not authorized for PII access");
        }

        // Log access for audit
        auditLogger.logServicePiiAccess(service.getName(), id);

        Customer customer = customerService.findById(id);
        return toFullDTO(customer);
    }
}
```

### 8. Database Security

**Column-Level Encryption:**

```sql
-- Enable transparent data encryption for PII columns
ALTER TABLE customers
  ENCRYPT COLUMN encrypted_ssn USING 'AES256';

ALTER TABLE customers
  ENCRYPT COLUMN phone_number USING 'AES256';
```

**Access Auditing:**

```sql
-- Enable audit logging for PII table access
CREATE AUDIT POLICY customer_pii_audit
  ACTIONS SELECT, UPDATE, DELETE ON customers
  WHEN 'SYS_CONTEXT(''USERENV'', ''SESSION_USER'') != ''SYSTEM'''
  EVALUATE PER STATEMENT;
```

### 9. CCPA/GDPR Compliance Features

**Right to Access:**

```java
@GetMapping("/api/privacy/my-data")
public DataExportDTO exportMyData(@AuthenticationPrincipal User user) {
    // User can download all their data
    DataExportDTO export = new DataExportDTO();
    export.setCustomer(customerService.getFullData(user.getCustomerId()));
    export.setOrders(orderService.getCustomerOrders(user.getCustomerId()));
    export.setReviews(reviewService.getCustomerReviews(user.getCustomerId()));
    export.setExportDate(Instant.now());

    auditLogger.logDataExport(user.getId());

    return export;
}
```

**Right to Deletion:**

```java
@PostMapping("/api/privacy/delete-my-account")
public ResponseEntity<String> deleteMyAccount(@AuthenticationPrincipal User user,
                                              @RequestBody DeletionRequest request) {
    // Verify user identity
    if (!passwordEncoder.matches(request.getPassword(), user.getPassword())) {
        throw new UnauthorizedException("Invalid password");
    }

    // Queue for deletion (30-day grace period)
    dataRetentionService.queueForDeletion(user.getCustomerId(),
                                          request.getReason(),
                                          30); // days

    auditLogger.logDeletionRequest(user.getId());

    return ResponseEntity.ok("Account scheduled for deletion in 30 days");
}
```

**Right to Correction:**

```java
@PutMapping("/api/privacy/update-my-data")
public CustomerDTO updateMyData(@AuthenticationPrincipal User user,
                                @RequestBody @Valid CustomerUpdateDTO updates) {
    // User can update their own PII
    Customer customer = customerService.findById(user.getCustomerId());

    // Log change for audit
    auditLogger.logPiiUpdate(user.getId(), "SELF_UPDATE", updates);

    customerService.update(customer, updates);

    return toDTO(customer);
}
```

### 10. Testing Considerations

**Never Use Real PII in Tests:**

```java
@Test
public void testCustomerCreation() {
    // ✅ GOOD: Fake data
    CustomerDTO dto = new CustomerDTO();
    dto.setFirstName("Test");
    dto.setLastName("User");
    dto.setEmail("test.user@test.homedepot.com");
    dto.setPhone("555-0100"); // Clearly fake

    // ❌ BAD: Real-looking PII
    dto.setEmail("john.smith@gmail.com"); // Could be real
    dto.setPhone("404-555-1234"); // Could be real
}
```

**Use Test Data Generators:**

```java
@Component
public class TestDataGenerator {

    public Customer generateTestCustomer() {
        Customer customer = new Customer();
        customer.setFirstName("TestUser" + UUID.randomUUID().toString().substring(0, 8));
        customer.setEmail("test-" + System.currentTimeMillis() + "@test.homedepot.com");
        customer.setPhone("555-0" + ThreadLocalRandom.current().nextInt(100, 199));
        return customer;
    }
}
```

## PII Handling Checklist

When working with PII, verify:

- [ ] PII fields identified and annotated
- [ ] Sensitive PII encrypted at rest
- [ ] PII masked in all logs
- [ ] PII not exposed in error messages
- [ ] Access control implemented (users access own data only)
- [ ] Audit logging for all PII access
- [ ] Data retention policy implemented
- [ ] CCPA/GDPR features implemented (access, deletion, correction)
- [ ] TLS 1.2+ for data in transit
- [ ] No real PII in test data

## Apply These Standards Automatically

When developer works with customer/user data:

1. **Identify PII fields** in data models
2. **Add encryption** for sensitive fields (SSN, government IDs)
3. **Implement masking** in all logging statements
4. **Add access control** - verify user owns data
5. **Add audit logging** for PII access
6. **Implement error handling** that doesn't expose PII
7. **Add data retention/deletion** methods
8. **Verify CCPA/GDPR compliance** features

The code should be privacy-compliant from the first commit.

## Tools Available
All tools (Read, Edit, Write, Bash, Grep, Glob)
