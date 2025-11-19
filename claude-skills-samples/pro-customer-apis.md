# Pro Customer APIs Skill

## Description
Enforce Home Depot's Pro Xtra loyalty program and B2B customer API standards. Ensures proper handling of Pro accounts, volume pricing, purchase tracking, and business customer features that differ from consumer patterns.

**Use when:** Creating or modifying APIs for Pro customers, business accounts, Pro Xtra loyalty features, volume pricing, job/project tracking, or B2B integrations.

**Activate for phrases like:** "Pro customer", "Pro Xtra", "business account", "volume pricing", "commercial", "B2B", "contractor", "trade professional", "job tracking", "purchase account"

## Instructions

### 1. Pro Customer Data Model

Pro accounts have additional fields beyond consumer accounts:

```java
@Entity
@Table(name = "pro_customers")
public class ProCustomer {
    @Id
    private String proXtraId;          // Format: PRO-XXXXXXXXXX

    // Business information
    private String businessName;
    private String businessType;        // CONTRACTOR, PROPERTY_MANAGER, TRADESPERSON, etc.
    private String taxId;               // Encrypted - for tax exemptions
    private String resaleCertificate;   // For resale tax exemption

    // Pro Xtra tier
    @Enumerated(EnumType.STRING)
    private ProTier tier;               // MEMBER, SILVER, GOLD, PLATINUM

    // Spending tracking (for tier qualification)
    private BigDecimal ytdSpend;        // Year-to-date spending
    private BigDecimal lifetimeSpend;

    // Perks earned
    private BigDecimal perksBalance;    // 2% back on Pro purchases
    private LocalDate perksExpirationDate;

    // Account relationships
    @OneToMany(mappedBy = "proCustomer")
    private List<AuthorizedBuyer> authorizedBuyers;  // Employees who can purchase

    @OneToMany(mappedBy = "proCustomer")
    private List<PurchaseAccount> purchaseAccounts;  // Credit/billing accounts

    @OneToMany(mappedBy = "proCustomer")
    private List<Job> activeJobs;                    // Project/job tracking

    // Preferences
    private String preferredStoreId;
    private boolean paperlessInvoice;
    private String deliveryInstructions;
}

@Entity
public class AuthorizedBuyer {
    @Id
    private String id;
    private String firstName;
    private String lastName;
    private String phone;
    private BigDecimal purchaseLimit;    // Max per transaction
    private BigDecimal dailyLimit;       // Max per day
    private boolean canUsePurchaseAccount;
}

@Entity
public class Job {
    @Id
    private String jobId;
    private String jobName;
    private String jobAddress;
    private LocalDate startDate;
    private LocalDate expectedEndDate;
    private BigDecimal budgetAmount;
    private BigDecimal spentAmount;
    private JobStatus status;

    // Purchases linked to this job
    @OneToMany(mappedBy = "job")
    private List<Order> orders;
}
```

### 2. Pro Authentication & Authorization

Pro accounts require additional auth checks:

```java
@RestController
@RequestMapping("/api/v1/pro")
public class ProCustomerController {

    @GetMapping("/account/{proXtraId}")
    @PreAuthorize("hasRole('PRO_CUSTOMER') and @proAuthService.canAccessAccount(#proXtraId, authentication)")
    public ProAccountResponse getAccount(@PathVariable String proXtraId) {
        return proCustomerService.getAccount(proXtraId);
    }

    @PostMapping("/account/{proXtraId}/buyers")
    @PreAuthorize("hasRole('PRO_ADMIN') and @proAuthService.isAccountAdmin(#proXtraId, authentication)")
    public AuthorizedBuyerResponse addAuthorizedBuyer(
            @PathVariable String proXtraId,
            @RequestBody @Valid AddBuyerRequest request) {
        return proCustomerService.addAuthorizedBuyer(proXtraId, request);
    }
}

@Service
public class ProAuthService {

    /**
     * Check if user can access Pro account
     * - Account owner can access
     * - Authorized buyers can access (limited views)
     * - Account admins can access all
     */
    public boolean canAccessAccount(String proXtraId, Authentication auth) {
        ProUserDetails user = (ProUserDetails) auth.getPrincipal();

        // Check if owner
        if (user.getProXtraId().equals(proXtraId)) {
            return true;
        }

        // Check if authorized buyer
        ProCustomer account = proCustomerRepository.findById(proXtraId).orElse(null);
        if (account != null) {
            return account.getAuthorizedBuyers().stream()
                .anyMatch(b -> b.getId().equals(user.getBuyerId()));
        }

        return false;
    }

    /**
     * Verify buyer has sufficient purchase authority
     */
    public void validatePurchaseAuthority(String proXtraId, String buyerId, BigDecimal amount) {
        AuthorizedBuyer buyer = authorizedBuyerRepository
            .findByProXtraIdAndBuyerId(proXtraId, buyerId)
            .orElseThrow(() -> new UnauthorizedBuyerException(buyerId));

        // Check transaction limit
        if (amount.compareTo(buyer.getPurchaseLimit()) > 0) {
            throw new PurchaseLimitExceededException(
                "Transaction amount $" + amount + " exceeds limit $" + buyer.getPurchaseLimit()
            );
        }

        // Check daily limit
        BigDecimal todaySpend = orderRepository.sumTodaySpend(proXtraId, buyerId);
        if (todaySpend.add(amount).compareTo(buyer.getDailyLimit()) > 0) {
            throw new DailyLimitExceededException(
                "Daily spend would exceed limit $" + buyer.getDailyLimit()
            );
        }
    }
}
```

### 3. Volume Pricing API

Pro customers get special pricing based on quantity and tier:

```java
@RestController
@RequestMapping("/api/v1/pro/pricing")
public class ProPricingController {

    @GetMapping("/quote")
    public PricingQuoteResponse getQuote(
            @RequestParam String proXtraId,
            @RequestBody List<QuoteLineItem> items) {

        ProCustomer customer = proCustomerService.getCustomer(proXtraId);

        List<PricedLineItem> pricedItems = items.stream()
            .map(item -> priceForPro(item, customer))
            .toList();

        BigDecimal subtotal = calculateSubtotal(pricedItems);
        BigDecimal perksEarned = calculatePerksEarned(subtotal, customer.getTier());

        return PricingQuoteResponse.builder()
            .items(pricedItems)
            .subtotal(subtotal)
            .tierDiscount(calculateTierDiscount(customer.getTier(), subtotal))
            .volumeDiscount(calculateVolumeDiscount(pricedItems))
            .estimatedPerksEarned(perksEarned)
            .quoteValidUntil(Instant.now().plus(7, ChronoUnit.DAYS))
            .build();
    }

    private PricedLineItem priceForPro(QuoteLineItem item, ProCustomer customer) {
        Product product = productService.getProduct(item.getSkuId());

        // Get best price from hierarchy
        BigDecimal bestPrice = determineBestPrice(
            product,
            customer.getTier(),
            item.getQuantity()
        );

        return PricedLineItem.builder()
            .skuId(item.getSkuId())
            .quantity(item.getQuantity())
            .retailPrice(product.getRetailPrice())
            .proPrice(bestPrice)
            .savings(product.getRetailPrice().subtract(bestPrice).multiply(BigDecimal.valueOf(item.getQuantity())))
            .priceType(determinePriceType(product, customer.getTier(), item.getQuantity()))
            .build();
    }

    /**
     * Price hierarchy for Pro customers:
     * 1. Contract pricing (if customer has negotiated contract)
     * 2. Volume pricing (quantity breaks)
     * 3. Tier pricing (based on Pro Xtra tier)
     * 4. Standard Pro price
     * Return the LOWEST of applicable prices
     */
    private BigDecimal determineBestPrice(Product product, ProTier tier, int quantity) {
        List<BigDecimal> applicablePrices = new ArrayList<>();

        // Standard Pro price
        applicablePrices.add(product.getProPrice());

        // Tier pricing
        BigDecimal tierPrice = getTierPrice(product, tier);
        if (tierPrice != null) {
            applicablePrices.add(tierPrice);
        }

        // Volume pricing
        BigDecimal volumePrice = getVolumePriceBreak(product, quantity);
        if (volumePrice != null) {
            applicablePrices.add(volumePrice);
        }

        // Return lowest
        return applicablePrices.stream()
            .min(BigDecimal::compareTo)
            .orElse(product.getRetailPrice());
    }

    private BigDecimal getVolumePriceBreak(Product product, int quantity) {
        // Volume pricing tiers
        return product.getVolumePricing().stream()
            .filter(vp -> quantity >= vp.getMinQuantity())
            .max(Comparator.comparing(VolumePricing::getMinQuantity))
            .map(VolumePricing::getPrice)
            .orElse(null);
    }
}
```

### 4. Job/Project Tracking API

Pro customers can track purchases by job:

```java
@RestController
@RequestMapping("/api/v1/pro/{proXtraId}/jobs")
public class JobTrackingController {

    @PostMapping
    @PreAuthorize("@proAuthService.canAccessAccount(#proXtraId, authentication)")
    public JobResponse createJob(
            @PathVariable String proXtraId,
            @RequestBody @Valid CreateJobRequest request) {

        Job job = Job.builder()
            .jobId(generateJobId())
            .proXtraId(proXtraId)
            .jobName(request.getJobName())
            .jobAddress(request.getJobAddress())
            .startDate(request.getStartDate())
            .expectedEndDate(request.getExpectedEndDate())
            .budgetAmount(request.getBudget())
            .status(JobStatus.ACTIVE)
            .build();

        jobRepository.save(job);

        auditLogger.log("JOB_CREATED", Map.of(
            "proXtraId", proXtraId,
            "jobId", job.getJobId(),
            "budget", request.getBudget()
        ));

        return toJobResponse(job);
    }

    @GetMapping("/{jobId}/purchases")
    public JobPurchasesResponse getJobPurchases(
            @PathVariable String proXtraId,
            @PathVariable String jobId,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "50") int size) {

        Job job = jobRepository.findByProXtraIdAndJobId(proXtraId, jobId)
            .orElseThrow(() -> new JobNotFoundException(jobId));

        Page<Order> orders = orderRepository.findByJobId(jobId, PageRequest.of(page, size));

        // Calculate spend by category for reporting
        Map<String, BigDecimal> spendByCategory = orders.stream()
            .flatMap(o -> o.getItems().stream())
            .collect(Collectors.groupingBy(
                item -> item.getProduct().getCategory(),
                Collectors.reducing(BigDecimal.ZERO, OrderItem::getTotal, BigDecimal::add)
            ));

        return JobPurchasesResponse.builder()
            .jobId(jobId)
            .jobName(job.getJobName())
            .budgetAmount(job.getBudgetAmount())
            .spentAmount(job.getSpentAmount())
            .remainingBudget(job.getBudgetAmount().subtract(job.getSpentAmount()))
            .percentUsed(job.getSpentAmount().divide(job.getBudgetAmount(), 2, RoundingMode.HALF_UP).multiply(BigDecimal.valueOf(100)))
            .spendByCategory(spendByCategory)
            .purchases(orders.map(this::toOrderSummary))
            .build();
    }

    @PostMapping("/{jobId}/link-purchase")
    public void linkPurchaseToJob(
            @PathVariable String proXtraId,
            @PathVariable String jobId,
            @RequestBody LinkPurchaseRequest request) {

        Order order = orderRepository.findById(request.getOrderId())
            .orElseThrow();

        // Verify order belongs to this Pro account
        if (!order.getProXtraId().equals(proXtraId)) {
            throw new UnauthorizedAccessException("Order does not belong to this account");
        }

        Job job = jobRepository.findByProXtraIdAndJobId(proXtraId, jobId)
            .orElseThrow();

        // Link order to job
        order.setJobId(jobId);
        orderRepository.save(order);

        // Update job spending
        job.setSpentAmount(job.getSpentAmount().add(order.getTotal()));
        jobRepository.save(job);

        // Check budget alert
        if (job.getSpentAmount().compareTo(job.getBudgetAmount().multiply(BigDecimal.valueOf(0.9))) >= 0) {
            notificationService.sendBudgetAlert(proXtraId, job);
        }
    }
}
```

### 5. Purchase Account API

Manage Pro credit/billing accounts:

```java
@RestController
@RequestMapping("/api/v1/pro/{proXtraId}/purchase-accounts")
public class PurchaseAccountController {

    @GetMapping
    public List<PurchaseAccountResponse> getAccounts(@PathVariable String proXtraId) {
        List<PurchaseAccount> accounts = purchaseAccountRepository.findByProXtraId(proXtraId);

        return accounts.stream()
            .map(this::toPurchaseAccountResponse)
            .toList();
    }

    @GetMapping("/{accountId}/statement")
    public StatementResponse getStatement(
            @PathVariable String proXtraId,
            @PathVariable String accountId,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate startDate,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate endDate) {

        PurchaseAccount account = purchaseAccountRepository
            .findByProXtraIdAndAccountId(proXtraId, accountId)
            .orElseThrow();

        List<Transaction> transactions = transactionRepository
            .findByAccountIdAndDateBetween(accountId, startDate, endDate);

        return StatementResponse.builder()
            .accountId(accountId)
            .accountName(account.getAccountName())
            .statementPeriod(new DateRange(startDate, endDate))
            .openingBalance(calculateOpeningBalance(account, startDate))
            .closingBalance(calculateClosingBalance(account, endDate))
            .creditLimit(account.getCreditLimit())
            .availableCredit(account.getAvailableCredit())
            .minimumPaymentDue(account.getMinimumPaymentDue())
            .paymentDueDate(account.getPaymentDueDate())
            .transactions(transactions.stream().map(this::toTransactionResponse).toList())
            .build();
    }

    @PostMapping("/{accountId}/authorize")
    @PreAuthorize("@proAuthService.isAccountAdmin(#proXtraId, authentication)")
    public AuthorizationResponse authorizeForPurchaseAccount(
            @PathVariable String proXtraId,
            @PathVariable String accountId,
            @RequestBody AuthorizeBuyerRequest request) {

        PurchaseAccount account = purchaseAccountRepository
            .findByProXtraIdAndAccountId(proXtraId, accountId)
            .orElseThrow();

        AuthorizedBuyer buyer = authorizedBuyerRepository.findById(request.getBuyerId())
            .orElseThrow();

        // Add purchase account access to buyer
        buyer.setCanUsePurchaseAccount(true);
        buyer.setPurchaseAccountId(accountId);
        authorizedBuyerRepository.save(buyer);

        auditLogger.log("PURCHASE_ACCOUNT_ACCESS_GRANTED", Map.of(
            "proXtraId", proXtraId,
            "accountId", accountId,
            "buyerId", request.getBuyerId(),
            "grantedBy", SecurityContext.getCurrentUserId()
        ));

        return AuthorizationResponse.success(buyer.getId(), accountId);
    }
}
```

### 6. Pro Xtra Perks API

Manage loyalty perks and rewards:

```java
@RestController
@RequestMapping("/api/v1/pro/{proXtraId}/perks")
public class ProPerksController {

    // Perks earn rates by tier
    private static final Map<ProTier, BigDecimal> PERK_RATES = Map.of(
        ProTier.MEMBER, BigDecimal.valueOf(0.01),     // 1%
        ProTier.SILVER, BigDecimal.valueOf(0.015),   // 1.5%
        ProTier.GOLD, BigDecimal.valueOf(0.02),      // 2%
        ProTier.PLATINUM, BigDecimal.valueOf(0.025)  // 2.5%
    );

    @GetMapping("/balance")
    public PerksBalanceResponse getPerksBalance(@PathVariable String proXtraId) {
        ProCustomer customer = proCustomerService.getCustomer(proXtraId);

        BigDecimal pendingPerks = calculatePendingPerks(proXtraId);

        return PerksBalanceResponse.builder()
            .proXtraId(proXtraId)
            .currentBalance(customer.getPerksBalance())
            .pendingPerks(pendingPerks)  // Perks from recent purchases not yet posted
            .expirationDate(customer.getPerksExpirationDate())
            .currentTier(customer.getTier())
            .earnRate(PERK_RATES.get(customer.getTier()).multiply(BigDecimal.valueOf(100)) + "%")
            .ytdEarned(calculateYtdPerksEarned(proXtraId))
            .ytdRedeemed(calculateYtdPerksRedeemed(proXtraId))
            .build();
    }

    @PostMapping("/redeem")
    public RedemptionResponse redeemPerks(
            @PathVariable String proXtraId,
            @RequestBody @Valid RedeemPerksRequest request) {

        ProCustomer customer = proCustomerService.getCustomer(proXtraId);

        if (request.getAmount().compareTo(customer.getPerksBalance()) > 0) {
            throw new InsufficientPerksException(
                "Requested " + request.getAmount() + " but balance is " + customer.getPerksBalance()
            );
        }

        // Create redemption voucher
        PerksVoucher voucher = PerksVoucher.builder()
            .voucherId(generateVoucherId())
            .proXtraId(proXtraId)
            .amount(request.getAmount())
            .validUntil(LocalDate.now().plusDays(90))
            .status(VoucherStatus.ACTIVE)
            .build();

        perksVoucherRepository.save(voucher);

        // Deduct from balance
        customer.setPerksBalance(customer.getPerksBalance().subtract(request.getAmount()));
        proCustomerRepository.save(customer);

        auditLogger.log("PERKS_REDEEMED", Map.of(
            "proXtraId", proXtraId,
            "amount", request.getAmount(),
            "voucherId", voucher.getVoucherId(),
            "newBalance", customer.getPerksBalance()
        ));

        return RedemptionResponse.builder()
            .voucherId(voucher.getVoucherId())
            .amount(voucher.getAmount())
            .barcode(generateBarcode(voucher))
            .validUntil(voucher.getValidUntil())
            .remainingBalance(customer.getPerksBalance())
            .build();
    }

    @GetMapping("/tier-progress")
    public TierProgressResponse getTierProgress(@PathVariable String proXtraId) {
        ProCustomer customer = proCustomerService.getCustomer(proXtraId);

        // Tier thresholds
        Map<ProTier, BigDecimal> thresholds = Map.of(
            ProTier.SILVER, BigDecimal.valueOf(2500),
            ProTier.GOLD, BigDecimal.valueOf(5000),
            ProTier.PLATINUM, BigDecimal.valueOf(10000)
        );

        ProTier nextTier = getNextTier(customer.getTier());
        BigDecimal nextThreshold = thresholds.get(nextTier);
        BigDecimal remaining = nextThreshold.subtract(customer.getYtdSpend());

        return TierProgressResponse.builder()
            .currentTier(customer.getTier())
            .nextTier(nextTier)
            .ytdSpend(customer.getYtdSpend())
            .spendRequiredForNextTier(nextThreshold)
            .remainingToNextTier(remaining.max(BigDecimal.ZERO))
            .percentComplete(customer.getYtdSpend().divide(nextThreshold, 2, RoundingMode.HALF_UP).multiply(BigDecimal.valueOf(100)))
            .tierBenefits(getTierBenefits(customer.getTier()))
            .nextTierBenefits(getTierBenefits(nextTier))
            .build();
    }
}
```

### 7. Reporting API

Pro-specific reports for business customers:

```java
@RestController
@RequestMapping("/api/v1/pro/{proXtraId}/reports")
public class ProReportingController {

    @GetMapping("/spending-summary")
    public SpendingReportResponse getSpendingSummary(
            @PathVariable String proXtraId,
            @RequestParam(defaultValue = "YTD") ReportPeriod period) {

        LocalDate startDate = calculateStartDate(period);
        List<Order> orders = orderRepository.findByProXtraIdAndDateAfter(proXtraId, startDate);

        // Aggregate by various dimensions
        Map<String, BigDecimal> byCategory = aggregateByCategory(orders);
        Map<String, BigDecimal> byJob = aggregateByJob(orders);
        Map<YearMonth, BigDecimal> byMonth = aggregateByMonth(orders);
        Map<String, BigDecimal> byStore = aggregateByStore(orders);
        Map<String, BigDecimal> byBuyer = aggregateByBuyer(orders);

        return SpendingReportResponse.builder()
            .proXtraId(proXtraId)
            .period(period)
            .totalSpend(orders.stream().map(Order::getTotal).reduce(BigDecimal.ZERO, BigDecimal::add))
            .orderCount(orders.size())
            .averageOrderValue(calculateAverage(orders))
            .spendByCategory(byCategory)
            .spendByJob(byJob)
            .spendByMonth(byMonth)
            .spendByStore(byStore)
            .spendByBuyer(byBuyer)
            .topProducts(getTopProducts(orders, 10))
            .build();
    }

    @GetMapping("/export")
    public ResponseEntity<Resource> exportPurchaseHistory(
            @PathVariable String proXtraId,
            @RequestParam ExportFormat format,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate startDate,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate endDate) {

        List<Order> orders = orderRepository.findByProXtraIdAndDateBetween(proXtraId, startDate, endDate);

        byte[] content;
        String filename;
        MediaType mediaType;

        switch (format) {
            case CSV -> {
                content = exportService.toCsv(orders);
                filename = "purchases_" + proXtraId + ".csv";
                mediaType = MediaType.parseMediaType("text/csv");
            }
            case EXCEL -> {
                content = exportService.toExcel(orders);
                filename = "purchases_" + proXtraId + ".xlsx";
                mediaType = MediaType.parseMediaType("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
            }
            case PDF -> {
                content = exportService.toPdf(orders);
                filename = "purchases_" + proXtraId + ".pdf";
                mediaType = MediaType.APPLICATION_PDF;
            }
            default -> throw new UnsupportedFormatException(format);
        }

        auditLogger.log("REPORT_EXPORTED", Map.of(
            "proXtraId", proXtraId,
            "format", format,
            "recordCount", orders.size()
        ));

        return ResponseEntity.ok()
            .contentType(mediaType)
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + filename + "\"")
            .body(new ByteArrayResource(content));
    }
}
```

### 8. Testing Pro APIs

```java
@SpringBootTest
class ProCustomerApiTest {

    @Test
    void shouldApplyVolumeDiscountForProCustomer() {
        // Given
        ProCustomer customer = createProCustomer(ProTier.GOLD);
        QuoteLineItem item = new QuoteLineItem("123456789", 100);  // 100 units

        // When
        PricingQuoteResponse quote = pricingController.getQuote(customer.getProXtraId(), List.of(item));

        // Then
        assertThat(quote.getItems().get(0).getProPrice())
            .isLessThan(quote.getItems().get(0).getRetailPrice());
    }

    @Test
    void shouldEnforceBuyerPurchaseLimit() {
        // Given
        ProCustomer customer = createProCustomer(ProTier.MEMBER);
        AuthorizedBuyer buyer = addAuthorizedBuyer(customer, BigDecimal.valueOf(500));

        // When/Then
        assertThrows(PurchaseLimitExceededException.class, () -> {
            proAuthService.validatePurchaseAuthority(
                customer.getProXtraId(),
                buyer.getId(),
                BigDecimal.valueOf(600)
            );
        });
    }

    @Test
    void shouldTrackJobSpending() {
        // Given
        ProCustomer customer = createProCustomer(ProTier.SILVER);
        Job job = createJob(customer, BigDecimal.valueOf(10000));
        Order order = createOrder(customer, BigDecimal.valueOf(500));

        // When
        jobController.linkPurchaseToJob(customer.getProXtraId(), job.getJobId(),
            new LinkPurchaseRequest(order.getId()));

        // Then
        Job updated = jobRepository.findById(job.getJobId()).orElseThrow();
        assertThat(updated.getSpentAmount()).isEqualTo(BigDecimal.valueOf(500));
    }

    @Test
    void shouldCalculateCorrectPerkRate() {
        // Given
        ProCustomer platinumCustomer = createProCustomer(ProTier.PLATINUM);
        BigDecimal purchaseAmount = BigDecimal.valueOf(1000);

        // When
        BigDecimal perksEarned = perksService.calculatePerksEarned(platinumCustomer, purchaseAmount);

        // Then - Platinum gets 2.5%
        assertThat(perksEarned).isEqualTo(BigDecimal.valueOf(25.00));
    }
}
```

## Output Format

When creating Pro customer APIs, always include:
1. ProXtraId validation and authorization checks
2. Multi-tier pricing logic (volume, tier, contract)
3. Authorized buyer permission checks
4. Job/project tracking integration
5. Perks earning and redemption
6. Comprehensive audit logging
7. B2B-specific reporting capabilities

## Tools Available
All tools (Read, Edit, Write, Bash, Grep, Glob)
