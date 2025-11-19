# Inventory Sync Patterns Skill

## Description
Enforce Home Depot's inventory synchronization standards for Oracle Retail and SAP integration. Ensures real-time inventory accuracy across 2,300+ stores, distribution centers, and e-commerce platforms.

**Use when:** Creating or modifying inventory-related services, store inventory APIs, fulfillment logic, or integrating with Oracle Retail/SAP systems.

**Activate for phrases like:** "inventory", "stock level", "Oracle Retail", "SAP", "store inventory", "availability", "inventory sync", "ATP" (available to promise), "fulfillment"

## Instructions

### 1. Inventory Data Model Standards

Always use the canonical inventory model:

```java
public class InventoryPosition {
    private String storeId;           // Format: HD####  (e.g., HD0123)
    private String skuId;             // 9-digit SKU
    private String locationId;        // Aisle-Bay-Shelf (e.g., "A12-B03-S02")

    // Quantity breakdown (all required)
    private Integer onHandQty;        // Physical count
    private Integer allocatedQty;     // Reserved for orders
    private Integer inTransitQty;     // En route to location
    private Integer availableQty;     // onHand - allocated (computed)

    // Timestamps (always UTC)
    private Instant lastCountTime;    // Physical count timestamp
    private Instant lastSyncTime;     // Oracle Retail sync
    private Instant lastModified;     // Any update

    // Source system tracking
    private String sourceSystem;      // ORACLE_RETAIL, SAP_EWM, POS, MANUAL
    private String syncId;            // Correlation ID for tracing
}
```

### 2. Oracle Retail Integration

When integrating with Oracle Retail (RMS/RIB):

```java
@Service
public class OracleRetailInventoryAdapter {

    private static final Duration SYNC_TIMEOUT = Duration.ofSeconds(30);
    private static final int MAX_BATCH_SIZE = 1000;

    @Retryable(maxAttempts = 3, backoff = @Backoff(delay = 1000))
    public void publishInventoryUpdate(InventoryPosition position) {
        // Always use RIB (Retail Integration Bus) message format
        RIBInventoryMessage message = RIBInventoryMessage.builder()
            .messageType("InvAdjustDesc")
            .businessDate(LocalDate.now(ZoneId.of("America/New_York")))
            .location(convertToOracleLocation(position.getStoreId()))
            .item(position.getSkuId())
            .adjustmentQty(position.getOnHandQty())
            .reasonCode(determineReasonCode(position))
            .userId(SecurityContext.getCurrentUser())
            .build();

        // Log to Splunk BEFORE sending
        auditLogger.log("INVENTORY_SYNC_ATTEMPT", Map.of(
            "store", position.getStoreId(),
            "sku", position.getSkuId(),
            "qty", position.getOnHandQty(),
            "syncId", position.getSyncId()
        ));

        try {
            ribPublisher.publish(message, SYNC_TIMEOUT);

            auditLogger.log("INVENTORY_SYNC_SUCCESS", Map.of(
                "syncId", position.getSyncId(),
                "latencyMs", stopwatch.elapsed()
            ));
        } catch (RIBException e) {
            // Dead letter queue for failed syncs
            dlqPublisher.send(new FailedInventorySync(position, e));
            throw new InventorySyncException("Oracle Retail sync failed", e);
        }
    }
}
```

### 3. Real-Time vs Batch Sync Rules

| Change Type | Sync Method | Max Latency | Example |
|-------------|-------------|-------------|---------|
| Sale transaction | Real-time Kafka | < 5 seconds | POS sale decrements |
| Inventory receipt | Real-time | < 30 seconds | Truck unload |
| Cycle count | Batch | 15 minutes | Physical inventory |
| Store transfer | Real-time | < 1 minute | Store-to-store |
| E-commerce reserve | Real-time | < 2 seconds | Cart checkout |

```java
// Determine sync strategy based on change type
public SyncStrategy determineSyncStrategy(InventoryChange change) {
    return switch (change.getType()) {
        case SALE, ECOM_RESERVE, TRANSFER_OUT -> SyncStrategy.REALTIME_KAFKA;
        case RECEIPT, TRANSFER_IN -> SyncStrategy.REALTIME_RIB;
        case CYCLE_COUNT, ADJUSTMENT -> SyncStrategy.BATCH_15MIN;
        case SHRINKAGE, DAMAGE -> SyncStrategy.BATCH_HOURLY;
    };
}
```

### 4. Inventory Availability Calculation

Always compute availability consistently:

```java
public class InventoryAvailabilityService {

    /**
     * Calculate Available-To-Promise (ATP) quantity
     * This is the ONLY approved formula for availability
     */
    public int calculateATP(String storeId, String skuId) {
        InventoryPosition pos = inventoryRepo.findByStoreAndSku(storeId, skuId);

        int onHand = pos.getOnHandQty();
        int allocated = pos.getAllocatedQty();
        int safetyStock = getSafetyStock(storeId, skuId);  // Min threshold
        int pendingReceipts = getPendingReceipts(storeId, skuId, Duration.ofHours(24));

        // ATP = OnHand - Allocated - SafetyStock + PendingReceipts(24h)
        int atp = onHand - allocated - safetyStock + pendingReceipts;

        // Never return negative - that means oversold
        return Math.max(0, atp);
    }

    /**
     * Check if store can fulfill quantity
     */
    public FulfillmentEligibility checkFulfillment(String storeId, String skuId, int requestedQty) {
        int atp = calculateATP(storeId, skuId);

        if (atp >= requestedQty) {
            return FulfillmentEligibility.available(storeId, atp);
        }

        // Check nearby stores within 25 miles
        List<StoreInventory> nearbyStores = findNearbyStoresWithStock(storeId, skuId, requestedQty, 25);
        if (!nearbyStores.isEmpty()) {
            return FulfillmentEligibility.availableNearby(nearbyStores);
        }

        // Check DC for ship-to-store
        int dcStock = checkDistributionCenters(skuId);
        if (dcStock >= requestedQty) {
            return FulfillmentEligibility.shipToStore(estimateDelivery(storeId));
        }

        return FulfillmentEligibility.unavailable();
    }
}
```

### 5. Kafka Event Schema for Inventory Changes

Use the standard Avro schema for inventory events:

```java
// Kafka topic: inventory.position.updates
// Partition key: storeId + skuId

@KafkaListener(topics = "inventory.position.updates")
public void handleInventoryUpdate(InventoryUpdateEvent event) {
    // Validate event
    validateEvent(event);

    // Idempotency check - use eventId to prevent duplicate processing
    if (processedEvents.contains(event.getEventId())) {
        log.info("Duplicate event ignored: {}", event.getEventId());
        return;
    }

    // Process update
    inventoryService.applyUpdate(event);

    // Mark processed
    processedEvents.add(event.getEventId(), Duration.ofHours(24));
}

// Standard event structure
public record InventoryUpdateEvent(
    String eventId,           // UUID
    String eventType,         // SALE, RECEIPT, ADJUSTMENT, TRANSFER, RESERVE
    String storeId,
    String skuId,
    int quantityChange,       // Positive or negative
    int newOnHandQty,
    String sourceSystem,
    String userId,
    Instant eventTime,
    Map<String, String> metadata
) {}
```

### 6. Error Handling and Recovery

```java
@Service
public class InventorySyncRecoveryService {

    // Reconciliation job runs every 15 minutes
    @Scheduled(fixedRate = 900000)
    public void reconcileInventoryDiscrepancies() {
        List<InventoryDiscrepancy> discrepancies =
            findDiscrepanciesBetweenSystems();

        for (InventoryDiscrepancy d : discrepancies) {
            // Oracle Retail is source of truth for on-hand
            if (d.getOracleQty() != d.getLocalQty()) {
                auditLogger.logDiscrepancy(d);

                // Auto-correct if within threshold
                if (Math.abs(d.getDelta()) <= 5) {
                    autoCorrect(d, "ORACLE_RETAIL");
                } else {
                    // Large discrepancies need manual review
                    alertService.sendToInventoryTeam(d);
                }
            }
        }
    }

    // Dead letter queue processor
    @KafkaListener(topics = "inventory.sync.dlq")
    public void processFailedSyncs(FailedInventorySync failed) {
        int retryCount = failed.getRetryCount();

        if (retryCount < 5) {
            // Exponential backoff retry
            scheduleRetry(failed, Duration.ofMinutes((long) Math.pow(2, retryCount)));
        } else {
            // Escalate to ops team
            alertService.criticalAlert(
                "Inventory sync failed after 5 retries",
                failed
            );
        }
    }
}
```

### 7. Monitoring and Alerting

Always include these metrics:

```java
// Required metrics for inventory services
@Component
public class InventoryMetrics {

    private final MeterRegistry registry;

    // Sync latency by source system
    public void recordSyncLatency(String sourceSystem, Duration latency) {
        registry.timer("inventory.sync.latency", "source", sourceSystem)
            .record(latency);
    }

    // Discrepancy tracking
    public void recordDiscrepancy(String storeId, int delta) {
        registry.counter("inventory.discrepancy.count", "store", storeId)
            .increment();
        registry.gauge("inventory.discrepancy.delta", delta);
    }

    // ATP accuracy (compare prediction vs actual)
    public void recordATPAccuracy(String skuId, boolean wasAccurate) {
        registry.counter("inventory.atp.accuracy",
            "sku_category", getCategoryFromSku(skuId),
            "accurate", String.valueOf(wasAccurate)
        ).increment();
    }
}
```

### 8. Testing Requirements

All inventory services must include:

```java
@SpringBootTest
class InventorySyncServiceTest {

    @Test
    void shouldSyncToOracleRetailWithinTimeout() {
        // Real-time syncs must complete in < 30 seconds
    }

    @Test
    void shouldHandleOracleRetailOutage() {
        // Dead letter queue must capture failed syncs
    }

    @Test
    void shouldCalculateATPCorrectly() {
        // ATP formula must match exactly
    }

    @Test
    void shouldPreventOverselling() {
        // Concurrent reservations must not exceed ATP
    }

    @Test
    void shouldReconcileDiscrepancies() {
        // Reconciliation must detect and log differences
    }

    @Test
    void shouldMaintainIdempotency() {
        // Duplicate events must not double-count
    }
}
```

## Output Format

When creating inventory-related code, always include:
1. Proper data model with all required fields
2. Oracle Retail/SAP integration following adapter pattern
3. Kafka events with Avro schema
4. ATP calculation using approved formula
5. Error handling with DLQ and reconciliation
6. Metrics for monitoring
7. Comprehensive tests

## Tools Available
All tools (Read, Edit, Write, Bash, Grep, Glob)
