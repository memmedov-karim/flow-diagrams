# Withdraw Flow - Code-Level Technical Documentation

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Component Breakdown](#component-breakdown)
3. [Step-by-Step Flow Explanation](#step-by-step-flow-explanation)
4. [Data Structures & DTOs](#data-structures--dtos)
5. [Configuration Details](#configuration-details)
6. [Error Handling & Retry Mechanisms](#error-handling--retry-mechanisms)
7. [State Management](#state-management)
8. [Database Operations](#database-operations)
9. [External Service Integrations](#external-service-integrations)
10. [Temporal Implementation Details](#temporal-implementation-details)

---

## Architecture Overview

### Technology Stack
- **Java 17+** with Spring Boot
- **Temporal.io** for workflow orchestration
- **PostgreSQL** for persistent storage
- **Redis** for distributed locking and workflow mapping
- **Feign Clients** for external service communication
- **Lombok** for reducing boilerplate code
- **MapStruct** for entity-DTO mapping

### Design Patterns Used
1. **Workflow Pattern**: Temporal workflows orchestrate long-running business processes
2. **Activity Pattern**: Business logic encapsulated in retriable activities
3. **Signal Pattern**: External events trigger workflow continuation
4. **Query Pattern**: Read workflow state without side effects
5. **Compensation/Saga Pattern**: Rollback on failures
6. **Repository Pattern**: Data access abstraction
7. **Service Layer Pattern**: Business logic separation

---

## Component Breakdown

### 1. Controller Layer

#### `WithdrawTemporalController.java`

**Purpose**: REST API endpoints for withdraw operations.

**Location**: `az.abb.investment.usm.funding.ms.controller`

**Key Methods**:

```java
@PostMapping("/initiate")
public ResponseDTO<AuthorisationDTO> initiateWithdraw(
        @RequestBody @Valid WithdrawInitiationRequestDTO request)
```
- **Input**: `WithdrawInitiationRequestDTO` containing:
  - `amount`: Withdrawal amount
  - `currency`: Currency code (e.g., USD)
  - `productNumber`: Card/account number
  - `productType`: CARD or ACCOUNT
  - `calculatedRate`: Exchange rate
  - `frontendSimaReady`: Whether SIMA is ready
  
- **Process**:
  1. Validates request using `@Valid` annotation
  2. Delegates to `WithdrawWorkflowService.initiateWithdraw(request)`
  3. Logs initiation attempt
  4. Handles exceptions and returns appropriate HTTP response
  
- **Output**: `ResponseDTO<AuthorisationDTO>` containing:
  - `authorisationId`: UUID for authorization
  - `correlationId`: Unique identifier for tracking
  - `otpRequired`: Boolean flag
  - `simaRequired`: Boolean flag
  - `commission`: Fee details

**Code Flow**:
```java
// Line 54-71
try {
    ResponseDTO<AuthorisationDTO> response = withdrawWorkflowService.initiateWithdraw(request);
    
    if (response.getData() != null) {
        log.info("Withdraw initiated successfully. CorrelationId: {}", 
                response.getData().getCorrelationId());
    } else {
        log.error("Failed to initiate withdraw");
    }
    
    return response;
    
} catch (Exception e) {
    log.error("Error initiating withdraw", e);
    throw e; // Let Spring's exception handler deal with it
}
```

```java
@PutMapping(value = "/finalize/{authorisationId}/{correlationId}")
public ResponseDTO<InvestmentReceiptDTO> finalizeWithdraw(
        @PathVariable("authorisationId") UUID authorisationId,
        @PathVariable("correlationId") String correlationId,
        @RequestBody(required = false) TransferAuthoriseDTO transferAuthoriseDTO)
```
- **Input**: 
  - Path variables: `authorisationId`, `correlationId`
  - Optional body: `TransferAuthoriseDTO` (OTP, SIMA details)
  
- **Process**:
  1. Logs finalization attempt
  2. Calls `withdrawWorkflowService.finalizeWithdraw()`
  3. This sends a signal to the running Temporal workflow
  4. Waits for workflow to complete
  
- **Output**: `ResponseDTO<InvestmentReceiptDTO>` containing receipt details

---

### 2. Service Layer

#### `WithdrawWorkflowService.java`

**Purpose**: Bridge between controller and Temporal workflows. Manages workflow lifecycle.

**Location**: `az.abb.investment.usm.funding.ms.temporal.service`

**Dependencies**:
```java
private final WorkflowClient workflowClient;        // Temporal client
private final SessionService sessionService;        // User session management
private final WorkflowMappingService workflowMappingService; // Redis-based mapping
```

**Configuration Properties**:
```java
@Value("${temporal.task-queue.withdraw:withdraw-task-queue}")
private String withdrawTaskQueue;

@Value("${temporal.workflow.withdraw-timeout-hours:24}")
private int withdrawTimeoutHours;
```

##### Method: `initiateWithdraw(WithdrawInitiationRequestDTO request)`

**Step-by-Step Code Execution**:

1. **Get User Context** (Line 71-73):
```java
var user = sessionService.getCurrentContext().getUser();
String userId = user.getId().toString();
String dwAccountId = user.getDwAccountId();
```
- Retrieves authenticated user from session
- Extracts user ID and DriveWealth account ID

2. **Generate Workflow ID** (Line 77):
```java
String workflowId = generateWorkflowId(userId);
// Format: "withdraw-{userId}-{timestamp}"
// Example: "withdraw-123e4567-e89b-12d3-a456-426614174000-1698765432100"
```
- Creates unique, deterministic workflow identifier
- Timestamp ensures uniqueness for multiple withdraws
- Prevents duplicate workflow execution (idempotency)

3. **Configure Workflow Options** (Line 82-91):
```java
WorkflowOptions options = WorkflowOptions.newBuilder()
    .setTaskQueue(withdrawTaskQueue)           // Queue name for worker
    .setWorkflowId(workflowId)                 // Unique identifier
    .setWorkflowExecutionTimeout(Duration.ofHours(withdrawTimeoutHours)) // Max lifetime
    .setWorkflowRunTimeout(Duration.ofHours(withdrawTimeoutHours))       // Single run max
    .setWorkflowTaskTimeout(Duration.ofSeconds(30))  // Decision task timeout
    .build();
```
- **TaskQueue**: Routes workflow to specific workers
- **WorkflowId**: Idempotency key
- **ExecutionTimeout**: Total workflow lifetime (24 hours)
- **RunTimeout**: Single run before continue-as-new (24 hours)
- **TaskTimeout**: Decision task processing limit (30 seconds)

4. **Create Workflow Stub** (Line 94):
```java
WithdrawWorkflow workflow = workflowClient.newWorkflowStub(
    WithdrawWorkflow.class, 
    options
);
```
- Creates typed proxy to workflow interface
- Temporal generates implementation that communicates with server

5. **Start Workflow Asynchronously** (Line 98):
```java
WorkflowClient.start(workflow::executeWithdraw, request, userId, dwAccountId);
```
- **Non-blocking**: Returns immediately after starting
- Workflow executes in background on worker
- Parameters serialized and passed to workflow

6. **Poll for Initiation Completion** (Line 106-127):
```java
String currentState;
int maxWaitSeconds = 30;
long startTime = System.currentTimeMillis();

do {
    currentState = workflow.getCurrentState(); // Query method
    if ("WAITING_FINALIZATION".equals(currentState) || 
        "INITIATED".equals(currentState) || 
        "INITIATION_FAILED".equals(currentState)) {
        break;
    }
    
    if (System.currentTimeMillis() - startTime > maxWaitSeconds * 1000) {
        throw new RuntimeException("Timeout waiting for workflow initiation");
    }
    
    Thread.sleep(500); // Poll every 500ms
} while (true);
```
- **Query Pattern**: Read-only state inspection
- Polls until workflow reaches expected state
- 30-second timeout prevents indefinite waiting
- **Trade-off**: Blocks HTTP thread but ensures consistency

7. **Retrieve Initiation Response** (Line 129):
```java
initiateResponse = workflow.getInitiateResponse();
```
- Another query method
- Returns authorization details from workflow state
- Contains: `AuthorisationDTO`, `correlationId`, `success`, `errorMessage`

8. **Store Workflow Mapping in Redis** (Line 148-157):
```java
String correlationId = initiateResponse.getCorrelationId();
log.info("Storing mapping: correlationId={} -> workflowId={}", correlationId, workflowId);

workflowMappingService.storeMapping(correlationId, workflowId);

// Verify storage
String storedWorkflowId = workflowMappingService.getWorkflowId(correlationId);
if (storedWorkflowId == null) {
    log.error("Failed to store workflow mapping!");
} else {
    log.info("Verified mapping stored successfully");
}
```
- **Critical**: Enables finding workflow for finalization
- Redis key: `temporal:workflow:mapping:{correlationId}`
- TTL: 48 hours
- Verification ensures reliability

9. **Build and Return Response** (Line 160-165):
```java
AuthorisationDTO authDTO = initiateResponse.getAuthorisationDTO();
authDTO.setCorrelationId(correlationId);

return ResponseDTO.<AuthorisationDTO>builder()
    .data(authDTO)
    .build();
```

##### Method: `finalizeWithdraw(UUID authorisationId, String correlationId, TransferAuthoriseDTO transferAuthoriseDTO)`

**Step-by-Step Code Execution**:

1. **Get User Context** (Line 185-186):
```java
var user = sessionService.getCurrentContext().getUser();
String userId = user.getId().toString();
```

2. **Find Workflow by Correlation ID** (Line 192):
```java
String workflowId = findWorkflowIdByCorrelationId(correlationId, userId);

if (workflowId == null) {
    throw new RuntimeException("Workflow not found for correlation ID: " + correlationId);
}
```
- Retrieves workflow ID from Redis using `WorkflowMappingService`
- Throws exception if mapping not found (404 to client)

3. **Get Workflow Stub** (Line 202-205):
```java
WithdrawWorkflow workflow = workflowClient.newWorkflowStub(
    WithdrawWorkflow.class,
    workflowId  // Connect to existing workflow
);
```
- Creates stub connected to running workflow
- No new workflow created - connects to existing one

4. **Verify Workflow State** (Line 208-213):
```java
String currentState = workflow.getCurrentState();
log.info("Current workflow state: {}", currentState);

if (!"WAITING_FINALIZATION".equals(currentState)) {
    log.warn("Workflow is not waiting for finalization. Current state: {}", currentState);
}
```
- Safety check before sending signal
- Warns if workflow in unexpected state

5. **Build Finalize Request** (Line 216-220):
```java
WithdrawFinalizeRequest finalizeRequest = WithdrawFinalizeRequest.builder()
    .authorisationId(authorisationId)
    .correlationId(correlationId)
    .transferAuthoriseDTO(transferAuthoriseDTO)
    .build();
```

6. **Send Signal to Workflow** (Line 222):
```java
workflow.finalizeWithdraw(finalizeRequest);
```
- **Signal Method**: External event to running workflow
- Non-blocking: Returns immediately
- Workflow's `Workflow.await()` condition satisfied
- Workflow continues execution

7. **Wait for Workflow Completion** (Line 227-230):
```java
WorkflowStub untypedStub = WorkflowStub.fromTyped(workflow);
WithdrawFinalizeResponse result = untypedStub.getResult(
    WithdrawFinalizeResponse.class
);
```
- **Blocking**: Waits until workflow completes
- Temporal handles timeout (24 hours max)
- Returns final workflow result

8. **Handle Result** (Line 232-236):
```java
if (result == null || !result.isSuccess()) {
    throw new RuntimeException("Workflow finalization failed: " + 
            (result != null ? result.getErrorMessage() : "Unknown error"));
}

return result.getReceipt();
```

---

#### `WorkflowMappingService.java`

**Purpose**: Manages correlation ID to workflow ID mappings in Redis.

**Location**: `az.abb.investment.usm.funding.ms.temporal.service`

**Key Constants**:
```java
private static final String WORKFLOW_MAPPING_PREFIX = "temporal:workflow:mapping:";
private static final int MAPPING_TTL_HOURS = 48;
```

**Key Methods**:

1. **Store Mapping**:
```java
public void storeMapping(String correlationId, String workflowId) {
    String key = WORKFLOW_MAPPING_PREFIX + correlationId;
    redisService.set(key, workflowId, MAPPING_TTL_HOURS * 3600);
    log.info("Stored workflow mapping: correlationId={}, workflowId={}", 
            correlationId, workflowId);
}
```
- Redis key: `temporal:workflow:mapping:abc-123-def`
- Value: `withdraw-userid-timestamp`
- TTL: 172800 seconds (48 hours)

2. **Get Workflow ID**:
```java
public String getWorkflowId(String correlationId) {
    String key = WORKFLOW_MAPPING_PREFIX + correlationId;
    String workflowId = redisService.get(key);
    return workflowId;
}
```
- Returns null if not found or expired

3. **Check Mapping Exists**:
```java
public boolean hasMapping(String correlationId) {
    String key = WORKFLOW_MAPPING_PREFIX + correlationId;
    return redisService.exists(key);
}
```

---

### 3. Workflow Layer

#### `WithdrawWorkflow.java` (Interface)

**Purpose**: Defines workflow contract with Temporal annotations.

**Location**: `az.abb.investment.usm.funding.ms.temporal.workflow`

**Key Annotations**:
- `@WorkflowInterface`: Marks as Temporal workflow
- `@WorkflowMethod`: Main workflow entry point (only one allowed)
- `@SignalMethod`: External event handler (can have multiple)
- `@QueryMethod`: State inspection (can have multiple)

**Interface Definition**:
```java
@WorkflowInterface
public interface WithdrawWorkflow {
    
    @WorkflowMethod
    WithdrawFinalizeResponse executeWithdraw(
        WithdrawInitiationRequestDTO request, 
        String userId, 
        String dwAccountId
    );
    
    @SignalMethod
    void finalizeWithdraw(WithdrawFinalizeRequest request);
    
    @QueryMethod
    boolean isWaitingForFinalization();
    
    @QueryMethod
    String getCurrentState();
    
    @QueryMethod
    String getCorrelationId();
    
    @QueryMethod
    WithdrawInitiateResponse getInitiateResponse();
}
```

**Important Temporal Rules**:
1. Only ONE `@WorkflowMethod` allowed per interface
2. Multiple `@SignalMethod` and `@QueryMethod` allowed
3. Workflow methods must be deterministic
4. No direct I/O in workflow code (use activities)

---

#### `WithdrawWorkflowImpl.java` (Implementation)

**Purpose**: Orchestrates withdraw process with state management.

**Location**: `az.abb.investment.usm.funding.ms.temporal.workflow.impl`

**State Variables** (Persisted by Temporal):
```java
private String currentState = "INITIALIZED";
private boolean waitingForFinalization = false;
private WithdrawInitiateResponse initiateResponse;
private WithdrawFinalizeRequest finalizeRequest;
private WithdrawFinalizeResponse finalizeResponse;
private String correlationId;
private String userId;
```

**Activity Stubs** (Different retry policies):
```java
private final WithdrawActivities initiateActivities = 
    TemporalWorkflowConfig.createInitiateActivitiesStub();

private final WithdrawActivities finalizeActivities = 
    TemporalWorkflowConfig.createFinalizeActivitiesStub();

private final WithdrawActivities statusCheckActivities = 
    TemporalWorkflowConfig.createStatusCheckActivitiesStub();

private final WithdrawActivities rollbackActivities = 
    TemporalWorkflowConfig.createRollbackActivitiesStub();
```

##### Main Workflow Method: `executeWithdraw()`

**Complete Code Flow**:

```java
@Override
public WithdrawFinalizeResponse executeWithdraw(
        WithdrawInitiationRequestDTO request, 
        String userId, 
        String dwAccountId) {
    
    this.userId = userId;
    
    try {
        // ========== PHASE 1: INITIATION ==========
        currentState = "INITIATING";
        log.info("Workflow: Starting withdraw initiation for userId: {}", userId);
        
        initiateResponse = initiateWithdraw(request);
        
        if (!initiateResponse.isSuccess()) {
            currentState = "INITIATION_FAILED";
            return WithdrawFinalizeResponse.builder()
                .success(false)
                .errorMessage(initiateResponse.getErrorMessage())
                .workflowState(currentState)
                .build();
        }
        
        correlationId = initiateResponse.getCorrelationId();
        currentState = "INITIATED";
        
        // ========== PHASE 2: WAIT FOR SIGNAL ==========
        currentState = "WAITING_FINALIZATION";
        waitingForFinalization = true;
        
        // CRITICAL: Temporal's deterministic await
        Workflow.await(Duration.ofSeconds(20), () -> finalizeRequest != null);
        
        waitingForFinalization = false;
        
        // Timeout check
        if (finalizeRequest == null) {
            currentState = "FINALIZATION_TIMEOUT";
            initiateActivities.saveWithdrawLog(correlationId, "TRANSFER_FINALIZATION_TIMEOUT");
            rollbackWithdraw();
            return WithdrawFinalizeResponse.builder()
                .success(false)
                .errorMessage("Timeout - user did not authorize within 20 seconds")
                .workflowState(currentState)
                .build();
        }
        
        // ========== PHASE 3: FINALIZATION ==========
        currentState = "FINALIZING";
        finalizeResponse = finalizeWithdraw();
        
        if (finalizeResponse.isSuccess()) {
            currentState = "FINALIZED";
            notifyWithdrawStatus("SUCCESS");
        } else {
            currentState = "FINALIZATION_FAILED";
            rollbackWithdraw();
            notifyWithdrawStatus("FAILED");
        }
        
        return finalizeResponse;
        
    } catch (ActivityFailure e) {
        currentState = "ACTIVITY_FAILED";
        rollbackWithdraw();
        return buildFailureResponse(e);
    } catch (Exception e) {
        currentState = "WORKFLOW_FAILED";
        rollbackWithdraw();
        return buildFailureResponse(e);
    }
}
```

**Critical Temporal Concept - `Workflow.await()`**:
```java
Workflow.await(Duration.ofSeconds(20), () -> finalizeRequest != null);
```
- **Deterministic**: Not a real thread sleep
- **Persistent**: Saved in workflow history
- **Resumable**: Works across app restarts
- **Condition**: Lambda checked after each workflow task
- **Timeout**: Returns after 20 seconds if condition not met
- **Signal Effect**: When `finalizeWithdraw()` signal sets `finalizeRequest`, condition becomes true

##### Helper Method: `initiateWithdraw()`

**Code Breakdown**:
```java
private WithdrawInitiateResponse initiateWithdraw(WithdrawInitiationRequestDTO request) {
    try {
        // Generate correlation ID using Temporal's deterministic UUID
        String correlationId = Workflow.randomUUID().toString();
        
        // Activity 1: Create DB log
        initiateActivities.createWithdrawDbLog(request, userId, correlationId);
        
        // Activity 2: Check restrictions
        boolean isRestricted = initiateActivities.checkWithdrawRestrictions(userId);
        if (isRestricted) {
            return WithdrawInitiateResponse.builder()
                .success(false)
                .errorMessage("Withdraw is not allowed for this user")
                .build();
        }
        
        // Activity 3: Call transfer service
        AuthorisationDTO authDTO = initiateActivities
            .callTransferServiceInitiate(request, correlationId);
        
        // Activity 4: Save log with success status
        initiateActivities.saveWithdrawLog(correlationId, "INITIAL");
        
        return WithdrawInitiateResponse.builder()
            .authorisationDTO(authDTO)
            .correlationId(correlationId)
            .success(true)
            .build();
            
    } catch (Exception e) {
        throw ApplicationFailure.newFailure(
            "Failed to initiate withdraw: " + e.getMessage(),
            "INITIATION_ERROR"
        );
    }
}
```

**Key Points**:
- `Workflow.randomUUID()` is deterministic (replays generate same UUID)
- Activities called sequentially
- Each activity can be retried independently
- Exception propagates to main workflow method

##### Helper Method: `finalizeWithdraw()`

**Code Breakdown**:
```java
private WithdrawFinalizeResponse finalizeWithdraw() {
    try {
        String correlationId = finalizeRequest.getCorrelationId();
        
        // Activity 1: Lock withdraw
        finalizeActivities.lockWithdraw(correlationId);
        
        try {
            // Activity 2: Finalize transfer
            boolean transferSuccess = finalizeActivities.finalizeTransfer(finalizeRequest);
            if (!transferSuccess) {
                return buildFailureResponse("Transfer finalization failed");
            }
            
            // Activity 3: Get DW token
            String dwToken = finalizeActivities.getDriveWealthToken(correlationId);
            
            // Activity 4: Create DW withdraw
            finalizeActivities.createDriveWealthWithdraw(correlationId, dwToken);
            
            // Activity 5: Save final status
            finalizeActivities.saveWithdrawLog(correlationId, "WITHDRAW_SUCCESS");
            
            return WithdrawFinalizeResponse.builder()
                .success(true)
                .workflowState("FINALIZED")
                .build();
                
        } finally {
            // Activity 6: Always unlock (even on failure)
            try {
                finalizeActivities.unlockWithdraw(correlationId);
            } catch (Exception e) {
                log.error("Failed to unlock withdraw", e);
                // Don't fail workflow if unlock fails
            }
        }
        
    } catch (Exception e) {
        finalizeActivities.saveWithdrawLog(correlationId, "WITHDRAW_FAILED");
        throw ApplicationFailure.newFailure(
            "Failed to finalize withdraw: " + e.getMessage(),
            "FINALIZATION_ERROR"
        );
    }
}
```

**Important Patterns**:
- Try-finally ensures unlock always executes
- Boolean return from transfer check
- Token passed to subsequent activity
- Failure logged before throwing exception

##### Signal Method: `finalizeWithdraw()`

```java
@Override
public void finalizeWithdraw(WithdrawFinalizeRequest request) {
    log.info("Workflow: Received finalization signal for correlationId: {}", 
            request.getCorrelationId());
    this.finalizeRequest = request;
}
```
- **Extremely simple**: Just sets instance variable
- **Unblocks await**: `Workflow.await()` condition becomes true
- **Persisted**: Temporal saves this state change
- **Immediate**: Returns right away

##### Query Methods:

```java
@Override
public String getCurrentState() {
    return currentState;
}

@Override
public WithdrawInitiateResponse getInitiateResponse() {
    return initiateResponse;
}

@Override
public boolean isWaitingForFinalization() {
    return waitingForFinalization;
}

@Override
public String getCorrelationId() {
    return correlationId;
}
```
- **Read-only**: Cannot modify workflow state
- **No side effects**: Safe to call repeatedly
- **Instant**: No I/O or processing

---

### 4. Activity Layer

#### `WithdrawActivities.java` (Interface)

**Purpose**: Defines activities (business logic operations).

**Location**: `az.abb.investment.usm.funding.ms.temporal.activities`

**All Activities**:
```java
@ActivityInterface
public interface WithdrawActivities {
    
    @ActivityMethod
    String createWithdrawDbLog(WithdrawInitiationRequestDTO request, String userId, String correlationId);
    
    @ActivityMethod
    boolean checkWithdrawRestrictions(String dwAccountId);
    
    @ActivityMethod
    AuthorisationDTO callTransferServiceInitiate(WithdrawInitiationRequestDTO request, String correlationId);
    
    @ActivityMethod
    void saveWithdrawLog(String correlationId, String status);
    
    @ActivityMethod
    void lockWithdraw(String correlationId);
    
    @ActivityMethod
    boolean finalizeTransfer(WithdrawFinalizeRequest request);
    
    @ActivityMethod
    String getDriveWealthToken(String correlationId);
    
    @ActivityMethod
    void createDriveWealthWithdraw(String correlationId, String token);
    
    @ActivityMethod
    void unlockWithdraw(String correlationId);
    
    @ActivityMethod
    String checkTransferStatus(String correlationId);
    
    @ActivityMethod
    void rollbackWithdraw(String correlationId);
    
    @ActivityMethod
    void notifyWithdrawStatus(String correlationId, String status, String userId);
}
```

---

#### `WithdrawActivitiesImpl.java` (Implementation)

**Purpose**: Implements actual business logic with external service calls.

**Location**: `az.abb.investment.usm.funding.ms.temporal.activities.impl`

**Dependencies** (Injected via Constructor):
```java
private final TransferCheckerMsClient transferCheckerMsClient;
private final TransferServiceClient transferServiceClient;
private final DriveWealthClient driveWealthClient;
private final AuthorizationService authorizationService;
private final AuthorizationEvictionService authorizationEvictionService;
private final WithdrawRepository withdrawRepository;
private final WithdrawHistoryRepository withdrawHistoryRepository;
private final WithdrawHistoryMapper withdrawHistoryMapper;
private final FlexInfoService flexInfoService;
private final RedisService redisService;
```

##### Activity: `createWithdrawDbLog()`

**Purpose**: Creates initial database records for withdraw operation.

**Code**:
```java
@Override
public String createWithdrawDbLog(
        WithdrawInitiationRequestDTO request, 
        String userId, 
        String correlationId) {
    
    log.info("Activity: Creating withdraw DB log for userId: {}, correlationId: {}", 
            userId, correlationId);
    
    // Build entity from request
    InvestmentWithdraw withdrawLog = initiateDbLog(request, userId, correlationId);
    
    // Save to main table
    withdrawRepository.save(withdrawLog);
    
    // Save to history table (audit trail)
    withdrawHistoryRepository.save(withdrawHistoryMapper.map(withdrawLog));
    
    log.info("Activity: DB log created successfully for correlationId: {}", correlationId);
    return correlationId;
}

private InvestmentWithdraw initiateDbLog(
        WithdrawInitiationRequestDTO request, 
        String userId, 
        String correlationId) {
    
    return InvestmentWithdraw.builder()
        .amount(request.getAmount())
        .currency(CurrencyEnum.USD)
        .productCurrency(request.getCurrency())
        .productType(request.getProductType())
        .productNumber(request.getProductNumber())
        .calculatedRate(request.getCalculatedRate())
        .correlationId(correlationId)
        .userId(userId)
        .internalWithdrawStatus(InternalWithdrawStatusEnum.INITIAL)
        // User details populated from session
        .cif("MOCK_CIF")  // TODO: Fetch from user service
        .fin("MOCK_FIN")
        .phone("MOCK_PHONE")
        .dwAccountNo("MOCK_DW_ACCOUNT_NO")
        .dwCustomerId("MOCK_DW_CUSTOMER_ID")
        .dwAccountId("MOCK_DW_ACCOUNT_ID")
        .address("Test Address")
        .customerName("Test Customer")
        .iban("test_iban")
        .build();
}
```

**Database Tables**:
- `investment_withdraw`: Main transaction table
- `investment_withdraw_history`: Immutable audit log

**Why Two Tables?**
- Main table: Mutable, updated throughout process
- History table: Immutable snapshots for compliance/audit

##### Activity: `checkWithdrawRestrictions()`

**Purpose**: Validates if user can perform withdraw.

**Code**:
```java
@Override
public boolean checkWithdrawRestrictions(String dwAccountId) {
    log.info("Activity: Checking withdraw restrictions for dwAccountId: {}", dwAccountId);
    
    boolean isRestricted = checkIfWithdrawIsRestricted(dwAccountId);
    
    if (isRestricted) {
        log.warn("Activity: Withdraw is restricted for user: {}", dwAccountId);
    }
    
    return isRestricted;
}

private boolean checkIfWithdrawIsRestricted(String dwAccId) {
    // Check 1: Redis lock exists
    if (redisService.exists(redisService.generateKey(WITHDRAW_LOCK_PREFIX, dwAccId))) {
        return true;
    }
    
    // Check 2: Pending/failed withdraws in database
    LocalDate fromDate = LocalDate.parse(withdrawRetryStartDate);
    LocalDateTime fromDateTime = fromDate.atStartOfDay();
    
    return withdrawRepository.existsByUserIdAndDwStatusInOrInternalWithdrawStatusIn(
        dwAccId,
        List.of(PENDING.getMessage(), FAILED.getMessage()),
        List.of(InternalWithdrawStatusEnum.TRANSFER_FINALIZATION_TIMEOUT),
        fromDateTime
    );
}
```

**Restriction Checks**:
1. **Redis Lock**: Another withdraw in progress
2. **Pending Status**: Previous withdraw not completed
3. **Failed Status**: Previous withdraw failed and not resolved
4. **Timeout Status**: Previous withdraw timed out

**Database Query**:
```sql
SELECT EXISTS(
    SELECT 1 FROM investment_withdraw
    WHERE dw_account_id = ?
    AND (
        dw_status IN ('PENDING', 'FAILED')
        OR internal_withdraw_status = 'TRANSFER_FINALIZATION_TIMEOUT'
    )
    AND created_at >= ?
)
```

##### Activity: `callTransferServiceInitiate()`

**Purpose**: Initiates transfer through ABB's transfer service.

**Code**:
```java
@Override
public AuthorisationDTO callTransferServiceInitiate(
        WithdrawInitiationRequestDTO request, 
        String correlationId) {
    
    log.info("Activity: Calling transfer service for initiation, correlationId: {}", 
            correlationId);
    
    // Build transfer request
    WithdrawInitiationTransferRequestDTO transferRequestDTO = 
        buildTransferRequest(request, correlationId);
    
    try {
        // Feign client call (commented for now - mocked)
        // var response = transferServiceClient.initiateWithdraw(
        //     headersSupply.get(), 
        //     transferRequestDTO
        // );
        // return response.getData();
        
        // Mock response
        return AuthorisationDTO.builder()
            .authorisationId(UUID.randomUUID())
            .otpRequired(false)
            .simaRequired(true)
            .simaAuthorisationId(UUID.randomUUID())
            .commission(CommissionDTO.builder()
                .amount(BigDecimal.TEN)
                .currency(CurrencyEnum.USD)
                .type(CommissionTypeEnum.OUR)
                .build())
            .build();
            
    } catch (Exception e) {
        log.error("Activity: Error calling transfer service", e);
        
        // Update database with error status
        InvestmentWithdraw withdrawLog = 
            withdrawRepository.findFirstByCorrelationId(correlationId);
        
        if (withdrawLog != null) {
            if (e.getCause() instanceof SocketTimeoutException) {
                withdrawLog.setInternalWithdrawStatus(
                    InternalWithdrawStatusEnum.TRANSFER_INITIATION_TIMEOUT
                );
            } else {
                withdrawLog.setInternalWithdrawStatus(
                    InternalWithdrawStatusEnum.TRANSFER_INITIATION_FAILED
                );
            }
            withdrawRepository.save(withdrawLog);
            withdrawHistoryRepository.save(withdrawHistoryMapper.map(withdrawLog));
        }
        
        throw e; // Temporal will retry based on policy
    }
}

private WithdrawInitiationTransferRequestDTO buildTransferRequest(
        WithdrawInitiationRequestDTO request,
        String correlationId) {
    
    return WithdrawInitiationTransferRequestDTO.builder()
        .amount(request.getAmount())
        .currency(CurrencyEnum.USD)
        .correlationId(correlationId)
        .preOtpSend(Boolean.FALSE)
        .frontendSimaReady(request.isFrontendSimaReady())
        .receiver(InvestProductDTO.builder()
            .productNumber(request.getProductNumber())
            .productType(request.getProductType())
            .build())
        .build();
}
```

**Transfer Service Request**:
```json
{
  "amount": 100.00,
  "currency": "USD",
  "correlationId": "abc-123-def",
  "preOtpSend": false,
  "frontendSimaReady": true,
  "receiver": {
    "productNumber": "4169738800001234",
    "productType": "CARD"
  }
}
```

**Transfer Service Response**:
```json
{
  "authorisationId": "550e8400-e29b-41d4-a716-446655440000",
  "otpRequired": false,
  "simaRequired": true,
  "simaAuthorisationId": "660e8400-e29b-41d4-a716-446655440001",
  "commission": {
    "amount": 10.00,
    "currency": "USD",
    "type": "OUR"
  }
}
```

##### Activity: `saveWithdrawLog()`

**Purpose**: Updates withdraw status in database.

**Code**:
```java
@Override
public void saveWithdrawLog(String correlationId, String status) {
    log.info("Activity: Saving withdraw log with status: {} for correlationId: {}", 
            status, correlationId);
    
    InvestmentWithdraw withdrawLog = 
        withdrawRepository.findFirstByCorrelationId(correlationId);
    
    if (withdrawLog != null) {
        withdrawLog.setInternalWithdrawStatus(InternalWithdrawStatusEnum.valueOf(status));
        withdrawRepository.save(withdrawLog);
        withdrawHistoryRepository.save(withdrawHistoryMapper.map(withdrawLog));
        log.info("Activity: Withdraw log saved successfully");
    } else {
        log.error("Activity: Withdraw log not found for correlationId: {}", correlationId);
    }
}
```

**Status Values**:
- `INITIAL`
- `TRANSFER_FINALIZATION_TIMEOUT`
- `WITHDRAW_SUCCESS`
- `WITHDRAW_FAILED`
- etc. (defined in `InternalWithdrawStatusEnum`)

##### Activity: `lockWithdraw()`

**Purpose**: Prevents concurrent withdraw operations.

**Code**:
```java
@Override
public void lockWithdraw(String correlationId) {
    log.info("Activity: Locking withdraw for correlationId: {}", correlationId);
    
    InvestmentWithdraw withdrawLog = 
        withdrawRepository.findFirstByCorrelationId(correlationId);
    
    if (withdrawLog != null) {
        // Redis key: withdraw:lock:{dwAccountId}
        var lockedKey = redisService.generateKey(
            WITHDRAW_LOCK_PREFIX, 
            withdrawLog.getDwAccountId()
        );
        
        // Set with TTL
        redisService.set(lockedKey, WITHDRAW_LOCKED_VALUE, withdrawLockTimeSeconds);
        log.info("Activity: Withdraw locked successfully");
    } else {
        throw new RuntimeException("Withdraw log not found");
    }
}
```

**Redis Lock**:
- Key: `withdraw:lock:{dwAccountId}`
- Value: `"LOCKED"`
- TTL: Configurable (e.g., 300 seconds)

**Purpose**:
- Prevents duplicate finalization
- Ensures one withdraw at a time per account

##### Activity: `finalizeTransfer()`

**Purpose**: Completes transfer through transfer service.

**Code**:
```java
@Override
public boolean finalizeTransfer(WithdrawFinalizeRequest request) {
    log.info("Activity: Finalizing transfer for correlationId: {}", 
            request.getCorrelationId());
    
    InvestmentWithdraw withdrawLog = 
        withdrawRepository.findFirstByCorrelationId(request.getCorrelationId());
    
    if (withdrawLog == null) {
        throw new RuntimeException("Withdraw log not found");
    }
    
    ResponseDTO<TransferReceiptDTO> result = 
        finalizeTransferInternal(request, withdrawLog);
    
    if (!result.getData().isSuccess()) {
        log.warn("Activity: Transfer finalization failed");
        withdrawLog.setExternalWithdrawStatus(result.getData().getStatusText());
        withdrawLog.setInternalWithdrawStatus(
            InternalWithdrawStatusEnum.TRANSFER_FINALIZATION_FAILED
        );
        withdrawRepository.save(withdrawLog);
        withdrawHistoryRepository.save(withdrawHistoryMapper.map(withdrawLog));
        return false;
    }
    
    withdrawRepository.save(withdrawLog);
    withdrawHistoryRepository.save(withdrawHistoryMapper.map(withdrawLog));
    return true;
}

private ResponseDTO<TransferReceiptDTO> finalizeTransferInternal(
        WithdrawFinalizeRequest request,
        InvestmentWithdraw withdrawLog) {
    
    try {
        // Feign client call (currently mocked)
        // ResponseDTO<TransferReceiptDTO> result = transferServiceClient
        //     .finalizeWithdraw(
        //         request.getAuthorisationId(), 
        //         request.getTransferAuthoriseDTO(), 
        //         headersSupply.get()
        //     );
        
        // Mock response
        ResponseDTO<TransferReceiptDTO> result = new ResponseDTO<>();
        result.setData(TransferReceiptDTO.builder()
            .fee(BigDecimal.ZERO)
            .cashCode("test_cash_code")
            .statusText("success")
            .success(true)
            .amount(BigDecimal.valueOf(100))
            .currency(CurrencyEnum.USD)
            .sender("test_sender")
            .receiver("test_receiver")
            .build());
        
        // Update withdraw log with receipt details
        withdrawLog.setTransferDate(new Date());
        withdrawLog.setTransferReferenceNumber("test_ref_num");
        withdrawLog.setFlexContractNumber("test_contract_ref_num");
        withdrawLog.setRrn("test_rrn");
        
        return result;
        
    } catch (Exception e) {
        if (UtilFunctions.transferTimedOut.test(e)) {
            withdrawLog.setInternalWithdrawStatus(
                InternalWithdrawStatusEnum.TRANSFER_FINALIZATION_TIMEOUT
            );
            throw new IbamServerErrorException(ErrorEnum.TIMEOUT_HAPPENED);
        } else {
            withdrawLog.setInternalWithdrawStatus(
                InternalWithdrawStatusEnum.TRANSFER_FINALIZATION_FAILED
            );
        }
        throw e;
    }
}
```

**Transfer Finalize Request**:
```json
{
  "authorisationId": "550e8400-e29b-41d4-a716-446655440000",
  "otp": "123456",
  "simaCode": "789012"
}
```

**Transfer Receipt Response**:
```json
{
  "success": true,
  "statusText": "success",
  "amount": 100.00,
  "currency": "USD",
  "fee": 0.00,
  "cashCode": "CC123",
  "sender": "Investment Account",
  "receiver": "Card ****1234",
  "referenceNumber": "REF123456"
}
```

##### Activity: `getDriveWealthToken()`

**Purpose**: Obtains authorization token for DriveWealth API.

**Code**:
```java
@Override
public String getDriveWealthToken(String correlationId) {
    log.info("Activity: Getting DriveWealth token for correlationId: {}", correlationId);
    
    InvestmentWithdraw withdrawLog = 
        withdrawRepository.findFirstByCorrelationId(correlationId);
    
    if (withdrawLog == null) {
        throw new RuntimeException("Withdraw log not found");
    }
    
    // Actual implementation:
    // String token = getDwToken(withdrawLog);
    
    // Mock for now
    log.info("Activity: DriveWealth token obtained successfully");
    return "test-token";
}

private String getDwToken(InvestmentWithdraw withdrawLog) {
    try {
        return authorizationService.getAuthorizationToken();
    } catch (Exception e) {
        if (e.getCause() instanceof SocketTimeoutException) {
            withdrawLog.setInternalWithdrawStatus(
                InternalWithdrawStatusEnum.DW_AUTHORIZATION_TIMEOUT
            );
        } else {
            withdrawLog.setInternalWithdrawStatus(
                InternalWithdrawStatusEnum.DW_AUTHORIZATION_FAILED
            );
        }
        throw e;
    }
}
```

**Authorization Process**:
1. Check cache for existing token
2. If expired/missing, request new token from DriveWealth
3. Cache token for reuse (typical TTL: 1 hour)

##### Activity: `createDriveWealthWithdraw()`

**Purpose**: Creates withdrawal in DriveWealth system.

**Code**:
```java
@Override
public void createDriveWealthWithdraw(String correlationId, String token) {
    log.info("Activity: Creating DriveWealth withdraw for correlationId: {}", correlationId);
    
    InvestmentWithdraw withdrawLog = 
        withdrawRepository.findFirstByCorrelationId(correlationId);
    
    if (withdrawLog == null) {
        throw new RuntimeException("Withdraw log not found");
    }
    
    // Actual implementation with retry on 401:
    // makeDwWithdrawByHandlingAuthorization(withdrawLog, withdrawLog.getDwAccountNo(), token);
    
    withdrawRepository.save(withdrawLog);
    withdrawHistoryRepository.save(withdrawHistoryMapper.map(withdrawLog));
    
    log.info("Activity: DriveWealth withdraw created successfully");
}

private void makeDwWithdraw(
        InvestmentWithdraw withdrawLog, 
        String accountNo, 
        String token) {
    
    // Build request
    DriveWealthWithdrawRequestDto request = DriveWealthWithdrawRequestDto.builder()
        .accountNo(accountNo)
        .type(DW_WITHDRAW_TYPE)  // "ACH"
        .amount(withdrawLog.getAmount())
        .currency(withdrawLog.getCurrency())
        .build();
    
    // Call DriveWealth API
    DriveWealthWithdrawResponseDto response = 
        driveWealthClient.createWithdrawal(key, request, token);
    
    // Update entity with response
    withdrawLog.setDwWithdrawId(response.getId());
    withdrawLog.setDwAccountId(response.getAccountId());
    withdrawLog.setDwAccountNo(response.getAccountNo());
    withdrawLog.setDwCategory(response.getCategory());
    withdrawLog.setDwTransactionCode(response.getTransactionCode());
    withdrawLog.setDwStatus(response.getStatus().getMessage());
    withdrawLog.setDwWlpFinTranTypeId(response.getWlpFinTranTypeId());
    withdrawLog.setDwCreated(response.getCreated());
    withdrawLog.setDwPaymentRef(response.getPaymentRef());
    
    // Set internal status based on DW status
    withdrawLog.setInternalWithdrawStatus(
        DwStatusEnum.findByCode(response.getStatus().getId()).isSuccess()
            ? InternalWithdrawStatusEnum.WITHDRAW_SUCCESS
            : InternalWithdrawStatusEnum.DW_WITHDRAW_FAILED
    );
}

private void makeDwWithdrawByHandlingAuthorization(
        InvestmentWithdraw withdrawLog,
        String accountNo, 
        String token) {
    
    try {
        makeDwWithdraw(withdrawLog, accountNo, token);
    } catch (FeignException.Unauthorized e) {
        // Token expired - evict cache and retry
        authorizationEvictionService.evictAuthorizationToken();
        var newToken = getDwToken(withdrawLog);
        
        try {
            makeDwWithdraw(withdrawLog, accountNo, newToken);
        } catch (Exception exception) {
            catchDwWithdrawException(exception, withdrawLog);
        }
    } catch (Exception e) {
        catchDwWithdrawException(e, withdrawLog);
    }
}
```

**DriveWealth Request**:
```json
{
  "accountNo": "DWAC000001",
  "type": "ACH",
  "amount": 100.00,
  "currency": "USD"
}
```

**DriveWealth Response**:
```json
{
  "id": "WD123456",
  "accountId": "acc-id-123",
  "accountNo": "DWAC000001",
  "category": "CASH",
  "transactionCode": "WDL",
  "status": {
    "id": 1,
    "message": "PENDING"
  },
  "wlpFinTranTypeId": "type-123",
  "created": "2023-10-25T14:30:00Z",
  "paymentRef": "REF789012"
}
```

##### Activity: `unlockWithdraw()`

**Purpose**: Releases Redis lock after completion.

**Code**:
```java
@Override
public void unlockWithdraw(String correlationId) {
    log.info("Activity: Unlocking withdraw for correlationId: {}", correlationId);
    
    InvestmentWithdraw withdrawLog = 
        withdrawRepository.findFirstByCorrelationId(correlationId);
    
    if (withdrawLog != null) {
        var lockedKey = redisService.generateKey(
            WITHDRAW_LOCK_PREFIX, 
            withdrawLog.getDwAccountId()
        );
        redisService.delete(lockedKey);
        log.info("Activity: Withdraw unlocked successfully");
    } else {
        log.warn("Activity: Withdraw log not found for unlock");
    }
}
```

##### Activity: `rollbackWithdraw()`

**Purpose**: Compensation logic for failed operations.

**Code**:
```java
@Override
public void rollbackWithdraw(String correlationId) {
    try {
        log.info("Activity: Rolling back withdraw for correlationId: {}", correlationId);
        
        InvestmentWithdraw withdraw = 
            withdrawRepository.findFirstByCorrelationId(correlationId);
        
        if (withdraw != null) {
            withdraw.setInternalWithdrawStatus(
                InternalWithdrawStatusEnum.TRANSFER_FINALIZATION_FAILED
            );
            withdrawRepository.save(withdraw);
            withdrawHistoryRepository.save(withdrawHistoryMapper.map(withdraw));
            log.info("Withdraw rolled back successfully");
        }
    } catch (Exception e) {
        log.error("Error rolling back withdraw", e);
        // Rollback errors should not fail the workflow
    }
}
```

**Rollback Strategy**:
- Best effort (doesn't fail workflow)
- Marks transaction as failed
- Could be extended to:
  - Reverse transfer
  - Cancel DriveWealth withdrawal
  - Send notifications

##### Activity: `notifyWithdrawStatus()`

**Purpose**: Sends notifications about withdraw status.

**Code**:
```java
@Override
public void notifyWithdrawStatus(String correlationId, String status, String userId) {
    try {
        log.info("Activity: Notifying withdraw status - correlationId: {}, status: {}, userId: {}",
                correlationId, status, userId);
        
        // TODO: Implement notification logic
        // - Email notification
        // - SMS notification
        // - Push notification
        // - Kafka event
        
    } catch (Exception e) {
        log.error("Error notifying withdraw status", e);
        // Notification errors should not fail the workflow
    }
}
```

---

## Data Structures & DTOs

### Request DTOs

#### `WithdrawInitiationRequestDTO`
```java
public class WithdrawInitiationRequestDTO {
    private BigDecimal amount;              // Withdrawal amount
    private String currency;                // Product currency (AZN, USD)
    private String productNumber;           // Card/account number
    private String productType;             // CARD or ACCOUNT
    private BigDecimal calculatedRate;      // Exchange rate
    private boolean frontendSimaReady;      // SIMA pre-authorized
}
```

#### `TransferAuthoriseDTO`
```java
public class TransferAuthoriseDTO {
    private String otp;                     // One-time password
    private String simaCode;                // SIMA authorization code
}
```

### Response DTOs

#### `AuthorisationDTO`
```java
public class AuthorisationDTO {
    private UUID authorisationId;           // Transfer authorization ID
    private String correlationId;           // Tracking ID
    private boolean otpRequired;            // Needs OTP
    private boolean simaRequired;           // Needs SIMA
    private UUID simaAuthorisationId;       // SIMA auth ID
    private CommissionDTO commission;       // Fee details
}
```

#### `InvestmentReceiptDTO`
```java
public class InvestmentReceiptDTO {
    private String transactionId;           // Unique transaction ID
    private BigDecimal amount;              // Transaction amount
    private String status;                  // SUCCESS, FAILED
    private LocalDateTime transactionDate;  // When completed
    private String referenceNumber;         // Bank reference
}
```

### Internal DTOs

#### `WithdrawInitiateResponse`
```java
public class WithdrawInitiateResponse implements Serializable {
    private AuthorisationDTO authorisationDTO;
    private String correlationId;
    private boolean success;
    private String errorMessage;
}
```

#### `WithdrawFinalizeRequest`
```java
public class WithdrawFinalizeRequest implements Serializable {
    private UUID authorisationId;
    private String correlationId;
    private TransferAuthoriseDTO transferAuthoriseDTO;
}
```

#### `WithdrawFinalizeResponse`
```java
public class WithdrawFinalizeResponse implements Serializable {
    private InvestmentReceiptDTO receipt;
    private boolean success;
    private String errorMessage;
    private String workflowState;
}
```

---

## Configuration Details

### Temporal Configuration (`application-temporal.yml`)

```yaml
temporal:
  server:
    host: localhost                    # Temporal server address
    port: 7233                        # gRPC port
  namespace: default                  # Temporal namespace
  task-queue:
    withdraw: withdraw-task-queue     # Main queue for withdraw workflows
  worker:
    max-concurrent-activities: 10     # Max parallel activities
    max-concurrent-workflows: 10      # Max parallel workflows
  workflow:
    withdraw-timeout-hours: 24        # Workflow max lifetime
  enabled: true                       # Enable Temporal integration
```

### Activity Timeout Configuration

**Initiate Activities**:
```java
ActivityOptions.newBuilder()
    .setStartToCloseTimeout(Duration.ofMinutes(5))      // Total execution time
    .setScheduleToCloseTimeout(Duration.ofMinutes(10))  // Time from scheduled to complete
    .setRetryOptions(RetryOptions.newBuilder()
        .setInitialInterval(Duration.ofSeconds(1))      // First retry after 1s
        .setMaximumInterval(Duration.ofMinutes(1))      // Max retry interval 60s
        .setBackoffCoefficient(2.0)                     // Exponential: 1s, 2s, 4s, 8s...
        .setMaximumAttempts(5)                          // Up to 5 attempts
        .build())
```

**Finalize Activities**:
```java
ActivityOptions.newBuilder()
    .setStartToCloseTimeout(Duration.ofMinutes(5))
    .setScheduleToCloseTimeout(Duration.ofMinutes(15))
    .setRetryOptions(RetryOptions.newBuilder()
        .setInitialInterval(Duration.ofSeconds(2))
        .setMaximumInterval(Duration.ofMinutes(2))
        .setBackoffCoefficient(2.0)
        .setMaximumAttempts(10)                         // More retries for finalization
        .build())
```

**Timeout Meanings**:
- **StartToCloseTimeout**: Maximum time for single activity execution
- **ScheduleToCloseTimeout**: Maximum time from scheduling to completion (includes retries)
- **ScheduleToStartTimeout**: Maximum time waiting in queue before execution

---

## Error Handling & Retry Mechanisms

### Activity-Level Retries

**Automatic Retry Conditions**:
1. `Exception` thrown from activity (except `ApplicationFailure` with non-retryable flag)
2. Network timeouts
3. Service unavailability
4. Transient database errors

**Retry Sequence Example** (Initiate Activities):
```
Attempt 1: Immediate
Attempt 2: After 1 second
Attempt 3: After 2 seconds (1 + 1*2)
Attempt 4: After 4 seconds (1 + 3*2)
Attempt 5: After 8 seconds (1 + 7*2)
Max: 5 attempts, then ActivityFailure thrown
```

### Workflow-Level Error Handling

```java
try {
    // Main workflow logic
} catch (ActivityFailure e) {
    currentState = "ACTIVITY_FAILED";
    rollbackWithdraw();
    return buildFailureResponse(e);
} catch (Exception e) {
    currentState = "WORKFLOW_FAILED";
    rollbackWithdraw();
    return buildFailureResponse(e);
}
```

### Custom Exception Handling in Activities

```java
try {
    // Call external service
} catch (SocketTimeoutException e) {
    withdrawLog.setInternalWithdrawStatus(TIMEOUT_STATUS);
    throw e; // Temporal will retry
} catch (SpecificBusinessException e) {
    // Non-retryable business error
    throw ApplicationFailure.newNonRetryableFailure(
        "Business rule violation: " + e.getMessage(),
        "BUSINESS_ERROR"
    );
}
```

---

## State Management

### Workflow State Enum

```java
public enum WorkflowState {
    INITIALIZED,            // Workflow created
    INITIATING,            // Running initiation activities
    INITIATED,             // Initiation complete
    INITIATION_FAILED,     // Initiation failed
    WAITING_FINALIZATION,  // Awaiting user authorization
    FINALIZING,            // Running finalization activities
    FINALIZED,             // Successfully completed
    FINALIZATION_FAILED,   // Finalization failed
    FINALIZATION_TIMEOUT,  // User authorization timeout
    ACTIVITY_FAILED,       // Activity error after retries
    WORKFLOW_FAILED        // Unexpected workflow error
}
```

### Database Status Enum

```java
public enum InternalWithdrawStatusEnum {
    INITIAL,                           // Initial log created
    TRANSFER_INITIATION_TIMEOUT,       // Transfer service timeout
    TRANSFER_INITIATION_FAILED,        // Transfer service failed
    TRANSFER_FINALIZATION_TIMEOUT,     // Finalization timeout
    TRANSFER_FINALIZATION_FAILED,      // Finalization failed
    DW_AUTHORIZATION_TIMEOUT,          // DW auth timeout
    DW_AUTHORIZATION_FAILED,           // DW auth failed
    DW_WITHDRAW_TIMEOUT,               // DW withdraw timeout
    DW_WITHDRAW_FAILED,                // DW withdraw failed
    WITHDRAW_SUCCESS                   // Successfully completed
}
```

### State Transition Matrix

| From State | To State | Trigger | Activity |
|------------|----------|---------|----------|
| INITIALIZED | INITIATING | Workflow starts | - |
| INITIATING | INITIATED | All init activities succeed | saveWithdrawLog |
| INITIATING | INITIATION_FAILED | Activity fails | rollbackWithdraw |
| INITIATED | WAITING_FINALIZATION | Init complete | Workflow.await() |
| WAITING_FINALIZATION | FINALIZING | Signal received | finalizeWithdraw(signal) |
| WAITING_FINALIZATION | FINALIZATION_TIMEOUT | 20s timeout | rollbackWithdraw |
| FINALIZING | FINALIZED | All final activities succeed | notifyWithdrawStatus |
| FINALIZING | FINALIZATION_FAILED | Activity fails | rollbackWithdraw |
| Any | ACTIVITY_FAILED | Activity exception | rollbackWithdraw |
| Any | WORKFLOW_FAILED | Workflow exception | rollbackWithdraw |

---

## Database Operations

### Entity: `InvestmentWithdraw`

**Table**: `investment_withdraw`

**Key Fields**:
```java
@Entity
@Table(name = "investment_withdraw")
public class InvestmentWithdraw {
    
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private UUID id;
    
    @Column(unique = true, nullable = false)
    private String correlationId;           // Unique tracking ID
    
    private String userId;                  // User identifier
    private String dwAccountId;             // DriveWealth account ID
    private String dwAccountNo;             // DriveWealth account number
    
    private BigDecimal amount;              // Withdrawal amount
    private String currency;                // Currency (USD, AZN)
    
    private String productType;             // CARD or ACCOUNT
    private String productNumber;           // Masked number
    
    @Enumerated(EnumType.STRING)
    private InternalWithdrawStatusEnum internalWithdrawStatus;
    
    private String externalWithdrawStatus;  // External system status
    
    private String transferReferenceNumber; // Transfer service ref
    private Date transferDate;              // Transfer completion date
    
    private String dwWithdrawId;            // DriveWealth withdraw ID
    private String dwStatus;                // DW status
    private String dwPaymentRef;            // DW payment reference
    
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

### Repository Methods

```java
public interface WithdrawRepository extends JpaRepository<InvestmentWithdraw, UUID> {
    
    InvestmentWithdraw findFirstByCorrelationId(String correlationId);
    
    boolean existsByUserIdAndDwStatusInOrInternalWithdrawStatusIn(
        String dwAccountId,
        List<String> dwStatuses,
        List<InternalWithdrawStatusEnum> internalStatuses,
        LocalDateTime fromDateTime
    );
}
```

### History Entity

**Table**: `investment_withdraw_history`

**Purpose**: Immutable audit trail of all state changes.

```java
@Entity
@Table(name = "investment_withdraw_history")
public class InvestmentWithdrawHistory {
    
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private UUID id;
    
    private UUID withdrawId;                // FK to investment_withdraw
    private String correlationId;
    
    // Snapshot of all withdraw fields at this point in time
    private InternalWithdrawStatusEnum internalWithdrawStatus;
    private String externalWithdrawStatus;
    
    private LocalDateTime createdAt;        // When this history entry created
}
```

---

## External Service Integrations

### 1. Transfer Service

**Purpose**: Handles money transfers between accounts/cards.

**Base URL**: Configured in Feign client

**Endpoints Used**:

#### POST /transfer/initiate
```java
@PostMapping("/transfer/initiate")
ResponseDTO<AuthorisationDTO> initiateWithdraw(
    @RequestHeader Map<String, String> headers,
    @RequestBody WithdrawInitiationTransferRequestDTO request
);
```

#### PUT /transfer/finalize/{authorisationId}
```java
@PutMapping("/transfer/finalize/{authorisationId}")
ResponseDTO<TransferReceiptDTO> finalizeWithdraw(
    @PathVariable UUID authorisationId,
    @RequestBody TransferAuthoriseDTO transferAuthoriseDTO,
    @RequestHeader Map<String, String> headers
);
```

### 2. DriveWealth API

**Purpose**: US brokerage platform for investment operations.

**Authentication**: Bearer token (OAuth 2.0)

**Endpoints Used**:

#### POST /accounts/{accountNo}/withdrawals
```java
@PostMapping("/accounts/{accountNo}/withdrawals")
DriveWealthWithdrawResponseDto createWithdrawal(
    @RequestHeader("x-mysolomeo-api-key") String apiKey,
    @RequestBody DriveWealthWithdrawRequestDto request,
    @RequestHeader("Authorization") String bearerToken
);
```

**Headers**:
```
x-mysolomeo-api-key: <api-key>
Authorization: Bearer <token>
Content-Type: application/json
```

### 3. Transfer Checker Service

**Purpose**: Query transfer status for monitoring/timeout handling.

```java
@GetMapping("/transfer/status/{referenceNumber}")
ResponseDTO<TransferStatusDTO> getTransferStatus(
    @PathVariable String referenceNumber
);
```

---

## Temporal Implementation Details

### Worker Configuration

**Purpose**: Polls task queue and executes workflows/activities.

```java
@Configuration
@EnableConfigurationProperties(TemporalProperties.class)
public class TemporalConfig {
    
    @Bean
    public WorkflowClient workflowClient(TemporalProperties properties) {
        WorkflowServiceStubs service = WorkflowServiceStubs.newInstance(
            WorkflowServiceStubsOptions.newBuilder()
                .setTarget(properties.getServer().getHost() + ":" + 
                          properties.getServer().getPort())
                .build()
        );
        
        return WorkflowClient.newInstance(
            service,
            WorkflowClientOptions.newBuilder()
                .setNamespace(properties.getNamespace())
                .build()
        );
    }
    
    @Bean
    public WorkerFactory workerFactory(
            WorkflowClient workflowClient,
            WithdrawActivitiesImpl activities) {
        
        WorkerFactory factory = WorkerFactory.newInstance(workflowClient);
        
        Worker worker = factory.newWorker(withdrawTaskQueue);
        
        // Register workflows
        worker.registerWorkflowImplementationTypes(WithdrawWorkflowImpl.class);
        
        // Register activities
        worker.registerActivitiesImplementations(activities);
        
        factory.start();
        
        return factory;
    }
}
```

### Deterministic Execution

**Rules for Workflow Code**:

✅ **Allowed**:
- Calling activities
- Workflow.await()
- Workflow.sleep()
- Workflow.randomUUID()
- Workflow.currentTimeMillis()
- Pure functions
- Local state manipulation

❌ **NOT Allowed**:
- Direct database access
- Direct HTTP calls
- System.currentTimeMillis()
- UUID.randomUUID()
- Random number generation
- Thread.sleep()
- File I/O

**Why?** Temporal replays workflow from history. Non-deterministic code produces different results on replay.

### Event History

Every workflow action recorded as event:

```
1. WorkflowExecutionStarted
2. WorkflowTaskScheduled
3. WorkflowTaskStarted
4. WorkflowTaskCompleted
5. ActivityTaskScheduled (createWithdrawDbLog)
6. ActivityTaskStarted
7. ActivityTaskCompleted
8. ActivityTaskScheduled (checkWithdrawRestrictions)
9. ActivityTaskStarted
10. ActivityTaskCompleted
... (continues for all activities)
20. TimerStarted (for Workflow.await)
21. WorkflowSignaled (finalizeWithdraw)
22. TimerFired
... (finalization activities)
30. WorkflowExecutionCompleted
```

**History Size**: Typically 100-500 events for withdraw workflow.

### Workflow Replay

**When Replay Happens**:
1. Worker restart
2. Application deployment
3. New workflow task after timer/signal
4. After activity completion

**Replay Process**:
1. Temporal sends event history to worker
2. Worker executes workflow code
3. For activities already completed: Uses result from history
4. For new activities: Actually executes them
5. Workflow must produce same decisions

**Example**:
```java
// First execution:
String correlationId = Workflow.randomUUID().toString(); // abc-123

// Replay after activity completion:
String correlationId = Workflow.randomUUID().toString(); // abc-123 (same!)
```

---

## Performance Considerations

### Database Optimization

1. **Indexes**:
```sql
CREATE INDEX idx_withdraw_correlation_id ON investment_withdraw(correlation_id);
CREATE INDEX idx_withdraw_user_status ON investment_withdraw(user_id, internal_withdraw_status);
CREATE INDEX idx_withdraw_created_at ON investment_withdraw(created_at);
```

2. **Connection Pooling**:
```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
```

### Redis Optimization

1. **Connection Pooling**:
```yaml
spring:
  redis:
    lettuce:
      pool:
        max-active: 10
        max-idle: 5
        min-idle: 2
```

2. **TTL Strategy**:
- Workflow mappings: 48 hours
- Withdraw locks: 5 minutes
- Auth tokens: 1 hour

### Temporal Optimization

1. **Worker Configuration**:
```yaml
temporal:
  worker:
    max-concurrent-activities: 10    # Parallel activity execution
    max-concurrent-workflows: 10     # Parallel workflow execution
    max-activities-per-second: 100   # Rate limiting
```

2. **Task Queue Strategy**:
- Separate queues by operation type
- Prevents one operation from starving others
- Better resource allocation

---

## Monitoring & Observability

### Logging Strategy

**Structured Logging**:
```java
log.info("Withdraw initiated: userId={}, correlationId={}, amount={}", 
        userId, correlationId, amount);
```

**Log Levels**:
- `INFO`: Normal flow (initiation, finalization, state changes)
- `WARN`: Recoverable issues (retries, timeouts with fallback)
- `ERROR`: Failures requiring attention

### Metrics to Track

1. **Business Metrics**:
   - Withdraw success rate
   - Average processing time
   - Failure reasons distribution
   - Amount processed per day

2. **Technical Metrics**:
   - Workflow execution time
   - Activity retry count
   - Queue wait time
   - Worker utilization

3. **Temporal Metrics**:
   - Workflow start rate
   - Activity completion rate
   - Task queue lag
   - Worker health

### Temporal UI

**Access**: http://localhost:8080 (local) or configured URL

**Features**:
- View all workflow executions
- Filter by status, time, workflow type
- Inspect event history
- Query workflow state
- Terminate/cancel workflows
- View activity results
- Replay workflows for debugging

---

## Testing Strategies

### Unit Testing Activities

```java
@Test
void testCreateWithdrawDbLog() {
    // Arrange
    WithdrawInitiationRequestDTO request = createMockRequest();
    when(withdrawRepository.save(any())).thenReturn(mockEntity());
    
    // Act
    String result = activities.createWithdrawDbLog(request, "user-123", "corr-456");
    
    // Assert
    assertEquals("corr-456", result);
    verify(withdrawRepository).save(any());
    verify(withdrawHistoryRepository).save(any());
}
```

### Integration Testing Workflows

```java
@Test
void testCompleteWithdrawWorkflow() {
    // Start test environment
    TestWorkflowEnvironment testEnv = TestWorkflowEnvironment.newInstance();
    Worker worker = testEnv.newWorker(TASK_QUEUE);
    worker.registerWorkflowImplementationTypes(WithdrawWorkflowImpl.class);
    worker.registerActivitiesImplementations(mockActivities);
    
    // Execute workflow
    WithdrawWorkflow workflow = testEnv.newWorkflowStub(WithdrawWorkflow.class);
    WithdrawFinalizeResponse response = workflow.executeWithdraw(request, userId, dwAccountId);
    
    // Verify
    assertTrue(response.isSuccess());
}
```

### End-to-End Testing

```bash
# 1. Start workflow
curl -X POST http://localhost:8080/withdraw/temporal/initiate \
  -H "Content-Type: application/json" \
  -d '{"amount": 100, "currency": "USD", ...}'

# Response: {"data": {"correlationId": "abc-123", ...}}

# 2. Finalize workflow
curl -X PUT http://localhost:8080/withdraw/temporal/finalize/auth-id/abc-123 \
  -H "Content-Type: application/json" \
  -d '{"otp": "123456"}'

# Response: {"data": {"transactionId": "TXN123", "status": "SUCCESS", ...}}
```

---

## Security Considerations

### Authentication
- User authenticated via session
- Session validated in service layer
- User ID extracted from authenticated context

### Authorization
- User can only withdraw from own account
- Withdrawal limits enforced (in transfer service)
- Dual authorization (OTP + SIMA) supported

### Data Protection
- Sensitive data masked in logs
- Correlation ID for tracking (not personal data)
- Database encryption at rest
- TLS for all external communications

### Audit Trail
- All state changes logged to history table
- Temporal maintains complete event history
- Immutable audit records

---

## Troubleshooting Guide

### Issue: Workflow Not Found

**Symptoms**: 404 error on finalization

**Causes**:
1. Redis mapping expired (TTL: 48 hours)
2. Redis connection issue
3. Wrong correlation ID

**Solution**:
```java
// Check Redis
redis-cli GET "temporal:workflow:mapping:{correlationId}"

// Check Temporal UI
// Search for workflows by correlation ID in metadata
```

### Issue: Activity Timeout

**Symptoms**: Activities failing after max attempts

**Causes**:
1. External service slow/unavailable
2. Timeout too aggressive
3. Network issues

**Solution**:
- Increase timeout in `TemporalWorkflowConfig`
- Check external service health
- Review network connectivity

### Issue: Workflow Stuck in WAITING_FINALIZATION

**Symptoms**: Workflow not completing after signal

**Causes**:
1. Signal sent to wrong workflow ID
2. Workflow timed out (20 seconds)
3. Worker not running

**Solution**:
```java
// Check workflow state
String state = workflow.getCurrentState();

// Check if waiting
boolean waiting = workflow.isWaitingForFinalization();

// Verify worker running
// Check logs for "Worker started" message
```

---

## Best Practices Summary

### Code Organization
✅ Separate controllers, services, workflows, activities
✅ Use DTOs for data transfer
✅ Repository pattern for data access
✅ Mapper pattern for entity-DTO conversion

### Temporal Patterns
✅ One @WorkflowMethod per interface
✅ All I/O in activities
✅ Use Workflow.* methods for non-deterministic operations
✅ Query methods for state inspection
✅ Signal methods for external events

### Error Handling
✅ Automatic retries for transient failures
✅ Non-retryable for business errors
✅ Compensation/rollback on failures
✅ Graceful degradation for non-critical operations

### State Management
✅ Explicit state transitions
✅ State persisted automatically by Temporal
✅ Database status independent of workflow state
✅ History table for audit compliance

### Testing
✅ Unit test activities independently
✅ Integration test workflows with mock activities
✅ Use TestWorkflowEnvironment for deterministic tests
✅ End-to-end tests against real Temporal server

---

## Glossary

- **Workflow**: Long-running business process orchestration
- **Activity**: Single unit of business logic (can be retried)
- **Signal**: External event that triggers workflow continuation
- **Query**: Read-only inspection of workflow state
- **Task Queue**: Named queue for routing work to workers
- **Worker**: Process that executes workflows and activities
- **Correlation ID**: Unique identifier for tracking transactions
- **Workflow ID**: Unique identifier for workflow execution
- **Event History**: Complete log of workflow execution
- **Replay**: Re-executing workflow from history
- **Deterministic**: Always produces same result given same input

---

This documentation provides a complete code-level understanding of the withdraw process. For diagrams and visual representations, refer to `WITHDRAW_FLOW_DIAGRAM.md`.

