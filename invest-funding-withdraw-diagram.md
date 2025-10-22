# Investment Withdraw Service - System Design & Sequence Flow

## Overview
This document provides a comprehensive system design and sequence flow for the Investment Withdraw functionality, covering the complete journey from API initiation to finalization.

## 📌 How to View Diagrams

This document contains **Mermaid diagrams** that render as visual flowcharts. To view them:

✅ **GitHub/GitLab**: Diagrams render automatically  
✅ **VS Code**: Install "Markdown Preview Mermaid Support" extension  
✅ **IntelliJ/WebStorm**: Built-in Mermaid support  
✅ **Online**: Copy to [mermaid.live](https://mermaid.live) for editing  

---

## Table of Contents
- [Architecture Overview](#architecture-overview)
- [System Architecture Diagram](#system-architecture-diagram)
- [Complete Withdraw Journey](#complete-withdraw-journey-high-level)
- [Flow Diagrams](#flow-diagrams)
  - [Initiate Withdraw Flow](#initiate-withdraw-flow)
  - [Finalize Withdraw Flow](#finalize-withdraw-flow)
  - [Retry Withdraw Flow](#retry-withdraw-flow)
- [Status State Diagram](#status-state-diagram)
- [Components](#components)
- [Data Flow](#data-flow)
- [Error Handling](#error-handling)
- [Status Tracking](#status-tracking)

---

## Architecture Overview

The Investment Withdraw service follows a microservices architecture with the following key layers:

1. **Controller Layer**: REST API endpoints
2. **Service Layer**: Business logic orchestration with retry mechanisms
3. **Client Layer**: Integration with external services (Feign clients)
4. **Repository Layer**: Data persistence
5. **Cache Layer**: Redis for locking mechanism
6. **External Services**: Transfer Service, DriveWealth, Transfer Checker

---

## System Architecture Diagram

```mermaid
flowchart LR
    Client([Mobile/Web Client])
    
    subgraph FundingMS[Investment Funding Microservice]
        Controller[WithdrawController<br/>REST API]
        Service[WithdrawService<br/>Business Logic]
        Repos[(WithdrawRepository<br/>WithdrawHistoryRepository)]
        JobScheduler[WithdrawTransferStatusUpdateJob<br/>Retry Job]
    end
    
    subgraph ExternalServices[External Services]
        Transfer[Transfer Service<br/>Bank Transfers]
        DW[DriveWealth API<br/>Broker Platform]
        Checker[Transfer Checker MS<br/>Status Verification]
        Flex[Flex System<br/>Customer Info]
    end
    
    subgraph Infrastructure[Infrastructure]
        Session[Session Service<br/>Auth & Context]
        Auth[Authorization Service<br/>OAuth Tokens]
        Redis[(Redis Cache<br/>Locks & Tokens)]
    end
    
    Client -->|1. POST /initiate| Controller
    Client -->|2. PUT /finalize| Controller
    Controller --> Service
    Service --> Repos
    Service -->|Check Lock| Redis
    Service -->|Initiate/Finalize Transfer| Transfer
    Service -->|Create Redemption| DW
    Service -->|Fetch Customer Info| Flex
    Service --> Session
    Service --> Auth
    Auth --> Redis
    JobScheduler -->|Retry Timeout Cases| Service
    Service -->|Verify Transfer Status| Checker
    
    style Client fill:#e1f5ff
    style Controller fill:#d4edda
    style Service fill:#fff3cd
    style Repos fill:#e7f3ff
    style Transfer fill:#f8d7da
    style DW fill:#f8d7da
    style Checker fill:#f8d7da
    style FundingMS fill:#f0f8ff
    style ExternalServices fill:#fff5f5
    style Infrastructure fill:#f5fff5
    style JobScheduler fill:#ffe6e6
```

---

## Complete Withdraw Journey (High-Level)

```mermaid
flowchart TB
    subgraph Phase1[" PHASE 1: INITIATE WITHDRAW "]
        Start1([Client sends request<br/>POST /withdraw/initiate]) --> V1[Get User Context]
        V1 --> R1{Check Withdraw<br/>Restrictions}
        R1 -->|Locked/Pending| E1([Return Error:<br/>WITHDRAW_IS_NOT_ALLOWED])
        R1 -->|Allowed| F1[Fetch Customer Info<br/>from Flex]
        F1 --> T1[Call Transfer Service<br/>Initiate Transfer]
        T1 --> D1[Save to Database]
        D1 --> RES1([Return AuthorisationID<br/>+ CorrelationID])
    end
    
    RES1 -.->|Client gets OTP<br/>and calls finalize| Start2
    
    subgraph Phase2[" PHASE 2: FINALIZE WITHDRAW "]
        Start2([Client sends OTP<br/>PUT /withdraw/finalize]) --> L2[Load Transaction<br/>by CorrelationID]
        L2 --> LOCK[Set Redis Lock<br/>WITHDRAW_LOCK_PREFIX]
        LOCK --> T2[Call Transfer Service<br/>Finalize with OTP]
        T2 --> C2{Transfer<br/>Success?}
        C2 -->|No| E2[Remove Lock]
        E2 --> ER([Return Error])
        C2 -->|Yes| TK2[Get OAuth Token]
        TK2 --> DW2[Create Redemption<br/>in DriveWealth]
        DW2 --> DWR{DW Redemption<br/>Success?}
        DWR -->|401 Unauthorized| RF[Refresh Token<br/>& Retry]
        RF --> DW2
        DWR -->|Success| S2[Save Final State]
        DWR -->|Failed| S2
        S2 --> UL[Remove Redis Lock]
        UL --> R2([Return Receipt])
    end
    
    subgraph Phase3[" PHASE 3: RETRY MECHANISM (Job) "]
        Job([Scheduled Job<br/>WithdrawTransferStatusUpdateJob]) --> Q1[Query TIMEOUT withdraws]
        Q1 --> CHK[TransferCheckerMS:<br/>Check Status]
        CHK --> ST{Transfer<br/>Status?}
        ST -->|Success| DWR2[Create DW Redemption]
        ST -->|Failed| INC[Increment Retry Count]
        DWR2 --> SAV[Save Updated State]
        INC --> SAV
    end
    
    style Start1 fill:#e1f5ff,stroke:#0066cc,stroke-width:3px
    style Start2 fill:#e1f5ff,stroke:#0066cc,stroke-width:3px
    style Job fill:#ffe6e6,stroke:#dc3545,stroke-width:3px
    style RES1 fill:#d4edda,stroke:#28a745,stroke-width:3px
    style R2 fill:#d4edda,stroke:#28a745,stroke-width:3px
    style E1 fill:#f8d7da,stroke:#dc3545,stroke-width:3px
    style ER fill:#f8d7da,stroke:#dc3545,stroke-width:3px
    style Phase1 fill:#f0f8ff
    style Phase2 fill:#fff5f5
    style Phase3 fill:#fff9e6
```

### Legend
- 🔵 **Blue** - Start/Entry points
- 🟢 **Green** - Success endpoints
- 🔴 **Red** - Error endpoints
- 🟡 **Yellow** - Warning/Timeout states
- ⚪ **Light Blue** - Data operations (DB, Cache)
- 🔶 **Orange** - Background jobs
- 💎 **Diamond** - Decision points

---

## Flow Diagrams

### Initiate Withdraw Flow

```mermaid
flowchart TD
    Start([Client: POST /withdraw/initiate]) --> A[WithdrawController receives request]
    A --> B[WithdrawService.initiateWithdraw]
    
    B --> C[SessionService: get current user context]
    C --> D[Get User Details:<br/>CIF, PIN, Phone, UserID, DwAccountNo]
    
    D --> E[Generate new UUID as correlationId]
    
    E --> F[FlexInfoService: fetch customer info]
    F --> G[Get Customer Name & Address]
    
    G --> H{Product Type?}
    H -->|CARD| I[FlexInfoService: fetchCardIban]
    H -->|ACCOUNT| J[FlexInfoService: fetchAccountIban]
    
    I --> K[Create InvestmentWithdraw Entity<br/>Status: INITIAL]
    J --> K
    
    K --> L[Check Withdraw Restrictions]
    L --> M{Is Withdraw<br/>Allowed?}
    
    M -->|Redis Lock Exists| N[WITHDRAW_IS_NOT_ALLOWED]
    M -->|Pending/Failed Exists| N
    
    N --> End2([Throw Exception:<br/>WITHDRAW_IS_NOT_ALLOWED])
    
    M -->|Allowed| O[Build WithdrawInitiationTransferRequestDTO<br/>amount, currency=USD, correlationId,<br/>productNumber]
    
    O --> P[TransferServiceClient: initiateWithdraw]
    
    P --> Q{Transfer Service<br/>Response?}
    Q -->|Success| R[Receive AuthorisationDTO<br/>authorisationId, expiryTime]
    Q -->|Timeout| S[Catch SocketTimeoutException<br/>Status: TRANSFER_INITIATION_TIMEOUT]
    Q -->|Failed| T[Catch Exception<br/>Status: TRANSFER_INITIATION_FAILED]
    
    R --> U[WithdrawRepository: save InvestmentWithdraw]
    S --> U
    T --> U
    
    U --> V[WithdrawHistoryRepository: save history]
    
    V --> W{Was Transfer<br/>Successful?}
    W -->|Yes| X[Add correlationId to response<br/>Return AuthorisationDTO]
    W -->|No| Y[Throw Exception to Client]
    
    X --> End1([Return 200 OK<br/>AuthorisationDTO + correlationId])
    Y --> End3([Return Error Response])
    
    style Start fill:#e1f5ff
    style End1 fill:#d4edda
    style End2 fill:#f8d7da
    style End3 fill:#f8d7da
    style N fill:#f8d7da
    style R fill:#d4edda
    style S fill:#fff3cd
    style T fill:#f8d7da
    style K fill:#e7f3ff
    style U fill:#e7f3ff
    style V fill:#e7f3ff
    style L fill:#ffe6cc
```

---

### Finalize Withdraw Flow

```mermaid
flowchart TD
    Start([Client: PUT /withdraw/finalize]) --> A[WithdrawController receives request<br/>authorisationId, correlationId, TransferAuthoriseDTO]
    A --> B[WithdrawService.finalizeWithdraw]
    
    B --> C[SessionService: get current user]
    C --> D[WithdrawRepository: find by correlationId]
    D --> E[Retrieved InvestmentWithdraw entity]
    
    E --> F[RedisService: Set Lock<br/>Key: WITHDRAW_LOCK_PREFIX + dwAccountId<br/>TTL: withdrawLockTimeSeconds]
    
    F --> G[TransferServiceClient: finalizeWithdraw<br/>authorisationId, OTP/SIMA]
    
    G --> H{Transfer<br/>Finalization<br/>Result?}
    
    H -->|Success=true| I[Update Transfer Details:<br/>transferDate, referenceNumber,<br/>flexContractNumber, RRN]
    H -->|Success=false| J[Status: TRANSFER_FINALIZATION_FAILED<br/>Remove Lock & Return Failure Receipt]
    H -->|Timeout| K[Status: TRANSFER_FINALIZATION_TIMEOUT<br/>Remove Lock & Throw Exception]
    H -->|Error| L[Status: TRANSFER_FINALIZATION_FAILED<br/>Remove Lock & Throw Exception]
    
    J --> SaveDB1[Save to DB & History]
    SaveDB1 --> UL1[RedisService: Delete Lock]
    UL1 --> End2([Return Failure Receipt])
    
    I --> M[AuthorizationService: getAuthorizationToken]
    
    M --> N{Get Token<br/>Result?}
    N -->|Success| O[Received Bearer Token]
    N -->|Timeout| P[Status: DW_AUTHORIZATION_TIMEOUT<br/>Remove Lock & Throw Exception]
    N -->|Failed| Q[Status: DW_AUTHORIZATION_FAILED<br/>Remove Lock & Throw Exception]
    
    O --> R[Build DriveWealthWithdrawRequestDto<br/>accountNo, type=REDEMPTION, amount, USD]
    
    R --> S[DriveWealthClient: createWithdrawal<br/>key, request, token]
    
    S --> T{DW Redemption<br/>Result?}
    
    T -->|Success| U[Receive DriveWealthWithdrawResponseDto<br/>Update DW fields in withdrawLog]
    T -->|401 Unauthorized| V[Token Expired]
    T -->|Timeout| W[Status: DW_WITHDRAW_TIMEOUT<br/>Attempt Retry]
    T -->|Business Error| X[Parse DW Error Response]
    T -->|Generic Error| Y[Catch Exception]
    
    V --> Z[AuthorizationEvictionService: evict token]
    Z --> AA[AuthorizationService: get new token]
    AA --> AB[DriveWealthClient: createWithdrawal<br/>RETRY with new token]
    
    AB --> AC{Retry<br/>Result?}
    AC -->|Success| U
    AC -->|Failed| AD[Handle in catchDwWithdrawException]
    
    W --> AE[RetryableService: doRetry]
    X --> AE
    Y --> AE
    
    AE --> AF[Status: DW_WITHDRAW_FAILED<br/>ExternalStatus: errorCode - message<br/>Throw DwWithdrawException]
    
    U --> AG{Check DW<br/>Status ID}
    AG -->|Success Status| AH[Status: WITHDRAW_SUCCESS<br/>Update all DW fields]
    AG -->|Failed Status| AI[Status: DW_WITHDRAW_FAILED<br/>Update all DW fields]
    
    AH --> AJ[WithdrawRepository: save final state]
    AI --> AJ
    
    AJ --> AK[WithdrawHistoryRepository: save history]
    
    AK --> AL[RedisService: Delete Lock]
    
    AL --> AM[Map to InvestmentReceiptDTO:<br/>TransferReceipt + InvestmentWithdraw]
    
    AM --> End1([Return 200 OK<br/>InvestmentReceiptDTO])
    
    K --> SaveDB2[Save to DB & History]
    L --> SaveDB2
    P --> SaveDB3[Save to DB & History]
    Q --> SaveDB3
    AD --> SaveDB4[Save to DB & History]
    AF --> SaveDB4
    
    SaveDB2 --> UL2[RedisService: Delete Lock]
    SaveDB3 --> UL2
    SaveDB4 --> UL2
    
    UL2 --> End3([Throw Exception])
    
    style Start fill:#e1f5ff
    style End1 fill:#d4edda
    style End2 fill:#f8d7da
    style End3 fill:#f8d7da
    style AH fill:#d4edda
    style AI fill:#fff3cd
    style J fill:#f8d7da
    style K fill:#fff3cd
    style L fill:#f8d7da
    style P fill:#fff3cd
    style Q fill:#f8d7da
    style W fill:#fff3cd
    style AF fill:#f8d7da
    style V fill:#fff3cd
    style U fill:#d4edda
    style F fill:#ffe6cc
    style UL1 fill:#ffe6cc
    style AL fill:#ffe6cc
    style UL2 fill:#ffe6cc
    style I fill:#e7f3ff
    style AJ fill:#e7f3ff
    style AK fill:#e7f3ff
```

---

### Retry Withdraw Flow (Background Job)

```mermaid
flowchart TD
    Start([Scheduled Job:<br/>WithdrawTransferStatusUpdateJob]) --> A[Query withdraws with status:<br/>TRANSFER_FINALIZATION_TIMEOUT]
    
    A --> B[Filter by withdrawRetryStartDate]
    
    B --> C{Any withdraws<br/>to retry?}
    C -->|No| End1([End])
    
    C -->|Yes| D[For each withdraw]
    
    D --> E[withdrawService.retryWithdraw]
    
    E --> F{Internal Status?}
    F -->|Not TIMEOUT| End2([Skip - Not Retryable])
    
    F -->|TIMEOUT| G{Has Transfer<br/>Reference Number?}
    
    G -->|No| H[TransferServiceClient:<br/>getTransferDetail by correlationId]
    H --> I[Get Reference Number]
    I --> J[Update withdraw.transferReferenceNumber]
    
    G -->|Yes| K[TransferCheckerMsClient:<br/>getTransferStatus by refNum]
    J --> K
    
    K --> L{Transfer<br/>Status?}
    
    L -->|Success| M[handleSuccessfulWithdraw]
    L -->|Failed/Other| N[handleFailedWithdraw]
    
    M --> O[doDwWithdraw:<br/>Create DriveWealth redemption]
    
    O --> P[AuthorizationService: get token]
    P --> Q[DriveWealthClient: createWithdrawal]
    
    Q --> R{DW Result?}
    R -->|Success| S[Update DW fields<br/>Status: WITHDRAW_SUCCESS]
    R -->|401| T[Refresh token & retry]
    T --> Q
    R -->|Failed| U[Status: DW_WITHDRAW_FAILED]
    
    S --> V[Increment retry attempt count]
    U --> V
    
    N --> W[Increment retry attempt count<br/>Keep TIMEOUT status]
    
    V --> X[WithdrawRepository: save]
    W --> X
    
    X --> Y{More withdraws<br/>to process?}
    Y -->|Yes| D
    Y -->|No| End3([End Job Execution])
    
    style Start fill:#ffe6e6
    style End1 fill:#d4edda
    style End2 fill:#fff3cd
    style End3 fill:#d4edda
    style S fill:#d4edda
    style U fill:#f8d7da
    style W fill:#fff3cd
    style K fill:#e7f3ff
    style P fill:#ffe6cc
    style X fill:#e7f3ff
```

**Key Points:**
- Job runs on schedule (configured via cron expression)
- Only processes withdraws with `TRANSFER_FINALIZATION_TIMEOUT` status
- Checks actual transfer status from TransferCheckerMS
- If transfer succeeded, completes DriveWealth redemption
- If transfer failed, increments retry count
- Has max retry count limit configured

---

## Status State Diagram

This diagram shows all possible internal status transitions throughout the withdraw lifecycle:

```mermaid
stateDiagram-v2
    [*] --> INITIAL: Initiate Withdraw Request
    
    INITIAL --> CheckRestrictions: Validate
    CheckRestrictions --> [*]: Locked/Pending (WITHDRAW_IS_NOT_ALLOWED)
    
    CheckRestrictions --> TRANSFER_INITIATION_TIMEOUT: Transfer Service Timeout
    CheckRestrictions --> TRANSFER_INITIATION_FAILED: Transfer Service Failed
    CheckRestrictions --> Authorized: Transfer Initiated Successfully
    
    Authorized --> TRANSFER_FINALIZATION_TIMEOUT: Finalize Timeout
    Authorized --> TRANSFER_FINALIZATION_FAILED: Finalize Failed/Rejected
    Authorized --> TransferSuccess: Transfer Finalized Successfully
    
    TransferSuccess --> DW_AUTHORIZATION_TIMEOUT: Get Token Timeout
    TransferSuccess --> DW_AUTHORIZATION_FAILED: Get Token Failed
    TransferSuccess --> TokenReady: Token Obtained
    
    TokenReady --> DW_WITHDRAW_TIMEOUT: Redemption API Timeout
    TokenReady --> DW_WITHDRAW_FAILED: Redemption Failed/Business Error
    TokenReady --> WITHDRAW_SUCCESS: Redemption Created Successfully
    
    TRANSFER_FINALIZATION_TIMEOUT --> RetryJob: Background Job Picks Up
    RetryJob --> CheckTransferStatus: Query TransferCheckerMS
    
    CheckTransferStatus --> TokenReady: Transfer Was Successful
    CheckTransferStatus --> TRANSFER_FINALIZATION_TIMEOUT: Transfer Still Pending (Retry Again)
    CheckTransferStatus --> TRANSFER_FINALIZATION_FAILED: Transfer Failed (Give Up)
    
    TRANSFER_INITIATION_TIMEOUT --> [*]: End (Retryable)
    TRANSFER_INITIATION_FAILED --> [*]: End (Retryable)
    TRANSFER_FINALIZATION_FAILED --> [*]: End (Not Retryable)
    DW_AUTHORIZATION_TIMEOUT --> [*]: End (Retryable)
    DW_AUTHORIZATION_FAILED --> [*]: End (Retryable)
    DW_WITHDRAW_TIMEOUT --> [*]: End (Retryable)
    DW_WITHDRAW_FAILED --> [*]: End (Retryable)
    WITHDRAW_SUCCESS --> [*]: End (Success ✓)
    
    note right of CheckRestrictions
        Redis lock or
        pending/failed withdraw
        blocks new requests
    end note
    
    note right of TRANSFER_FINALIZATION_TIMEOUT
        Timeout status triggers
        background job retry
        mechanism
    end note
    
    note right of WITHDRAW_SUCCESS
        Final success state
        Funds withdrawn from
        DriveWealth account
    end note
```

### Status Categories

#### ✅ Success States
- `WITHDRAW_SUCCESS` - Complete end-to-end success

#### ⏳ In-Progress States  
- `INITIAL` - Request received and validated
- `Authorized` - Transfer service authorization received
- `TransferSuccess` - Bank transfer initiated
- `TokenReady` - OAuth token obtained

#### ⚠️ Timeout States (Retryable via Job)
- `TRANSFER_INITIATION_TIMEOUT`
- `TRANSFER_FINALIZATION_TIMEOUT` - **Triggers retry job**
- `DW_AUTHORIZATION_TIMEOUT`
- `DW_WITHDRAW_TIMEOUT`

#### ❌ Failed States
- `TRANSFER_INITIATION_FAILED` (Retryable)
- `TRANSFER_FINALIZATION_FAILED` (Not Retryable - transfer rejected)
- `DW_AUTHORIZATION_FAILED` (Retryable)
- `DW_WITHDRAW_FAILED` (Retryable)

#### 🚫 Blocked States
- `WITHDRAW_IS_NOT_ALLOWED` - Redis lock exists or pending/failed withdraws found

---

## Components

### 1. WithdrawController
REST API controller exposing withdraw endpoints:
- `POST /withdraw/initiate` - Initiates a withdrawal transaction
- `PUT /withdraw/finalize/{authorisationId}/{correlationId}` - Finalizes the withdrawal

### 2. WithdrawService (WithdrawServiceImpl)
Core business logic orchestrating:
- Transaction validation and restriction checks
- Redis locking mechanism (prevents concurrent withdrawals)
- Transfer service integration
- DriveWealth redemption creation
- Database logging and history tracking
- Error handling and retry mechanisms
- Background job support for timeout recovery

### 3. External Clients

#### TransferServiceClient
Handles internal bank transfer operations:
- `initiateWithdraw()` - Initiates money transfer to customer account
- `finalizeWithdraw()` - Finalizes the transfer with OTP/authorization
- `getTransferDetail()` - Gets transfer details by correlationId

#### DriveWealthClient
Integrates with DriveWealth broker:
- `getAuthorizationToken()` - Gets OAuth token
- `createWithdrawal()` - Creates redemption (withdrawal) from investment account

#### TransferCheckerMsClient
Verifies transfer status (used in retry job):
- `getTransferStatus()` - Gets transfer status by reference number

### 4. Supporting Services

- **SessionService**: Manages user session context
- **FlexInfoService**: Fetches customer information from Flex system
- **AuthorizationService**: Manages DriveWealth OAuth tokens
- **AuthorizationEvictionService**: Handles token refresh
- **RetryableService**: Handles retry logic for failed operations
- **RedisService**: Manages Redis locks and caching

### 5. Background Jobs

- **WithdrawTransferStatusUpdateJob**: Scheduled job that retries timeout withdrawals by checking actual transfer status

### 6. Repositories

- **WithdrawRepository**: Stores withdrawal transactions
- **WithdrawHistoryRepository**: Maintains audit trail of all operations

---

## Data Flow

### Request/Response DTOs

#### WithdrawInitiationRequestDTO
```json
{
  "amount": "BigDecimal",
  "currency": "CurrencyEnum",
  "productNumber": "String",
  "productType": "CategoryProductTypeEnum",
  "calculatedRate": "BigDecimal",
  "frontendSimaReady": "boolean"
}
```

#### AuthorisationDTO (Response)
```json
{
  "authorisationId": "UUID",
  "correlationId": "String",
  "expiryTime": "LocalDateTime"
}
```

#### TransferAuthoriseDTO (Finalize Request)
```json
{
  "otpCode": "String",
  "simaStan": "String"
}
```

#### InvestmentReceiptDTO (Finalize Response)
```json
{
  "success": "boolean",
  "amount": "BigDecimal",
  "currency": "CurrencyEnum",
  "referenceNumber": "String",
  "transactionDate": "LocalDateTime",
  "statusText": "String",
  "correlationId": "String",
  "dwWithdrawId": "String",
  "internalStatus": "InternalWithdrawStatusEnum"
}
```

---

## Error Handling

### Internal Status Enumeration

The system tracks various internal statuses throughout the withdraw lifecycle:

#### Initiation Phase
- `INITIAL` - Transaction initiated
- `TRANSFER_INITIATION_TIMEOUT` - Transfer service timeout
- `TRANSFER_INITIATION_FAILED` - Transfer service failed

#### Finalization Phase - Transfer
- `TRANSFER_FINALIZATION_TIMEOUT` - Transfer finalization timeout (triggers retry job)
- `TRANSFER_FINALIZATION_FAILED` - Transfer finalization failed

#### Finalization Phase - DW Authorization
- `DW_AUTHORIZATION_TIMEOUT` - Authorization timeout
- `DW_AUTHORIZATION_FAILED` - Authorization failed

#### Finalization Phase - DW Redemption
- `DW_WITHDRAW_TIMEOUT` - Redemption creation timeout
- `DW_WITHDRAW_FAILED` - Redemption creation failed
- `WITHDRAW_SUCCESS` - Complete success

### Error Recovery Mechanisms

1. **Redis Lock**: Prevents concurrent withdraw requests from same user
2. **Pending Check**: Blocks new withdrawals if user has pending/failed ones
3. **Token Refresh**: Automatic retry with new token on 401 Unauthorized
4. **Retry Job**: Background job retries timeout cases by checking actual transfer status
5. **Retry Logic**: RetryableService handles retryable failures
6. **Timeout Handling**: Specific status codes for timeout scenarios
7. **Transaction History**: All operations logged to history table
8. **Idempotency**: Correlation ID prevents duplicate transactions

### Withdraw Restrictions

User cannot initiate new withdraw if:
- Redis lock exists (key: `WITHDRAW_LOCK_PREFIX + dwAccountId`)
- Has pending/failed withdrawals with status:
  - DriveWealth status: `PENDING` or `FAILED`
  - Internal status: `TRANSFER_FINALIZATION_TIMEOUT`
- Above conditions checked from `withdrawRetryStartDate` onwards

---

## Status Tracking

### Database Tables

#### investment_withdraw (Main Table)
Stores the current state of each withdraw transaction with fields:
- Transaction details (amount, currency, product info)
- User information (cif, fin, phone, userId)
- Transfer details (referenceNumber, transferDate, RRN)
- DriveWealth details (accountNo, accountId, withdrawId, status, paymentRef)
- Status tracking (internalWithdrawStatus, externalWithdrawStatus)
- Retry tracking (retryAttemptCount)
- Timestamps (createdDate, transferDate)

#### investment_withdraw_history (Audit Table)
Maintains complete audit trail of all state changes.

---

## Integration Points

### 1. Transfer Service
**Base URL**: Configured via `client.base-url.transfer`

**Endpoints**:
- `POST /investment/withdraw/initiate` - Initiates internal bank transfer
- `PUT /investment/withdraw/finalize/{authorisationId}` - Finalizes transfer with OTP
- `GET /transfer-detail/{correlationId}` - Gets transfer details

**Purpose**: Handles money transfer from investment account to customer's bank account

### 2. DriveWealth API
**Base URL**: Configured via `client.base-url.drive-wealth`

**Endpoints**:
- `POST /auth/tokens` - Gets OAuth bearer token
- `POST /funding/redemptions` - Creates redemption (withdrawal) from investment account

**Purpose**: Creates redemptions in broker investment accounts

### 3. Transfer Checker Microservice
**Purpose**: Verifies actual status of transfers (used in retry mechanism)

**Endpoints**:
- Gets transfer status by reference number

### 4. Flex System
**Service**: FlexInfoService (internal)

**Purpose**: Fetches customer information (name, address, IBAN) from legacy banking system

### 5. Session Management
**Service**: SessionService (internal)

**Purpose**: Manages user authentication and session context

### 6. Redis Cache
**Purpose**: 
- Stores withdraw locks to prevent concurrent operations
- Caches OAuth tokens

**Keys**:
- `WITHDRAW_LOCK_PREFIX + dwAccountId` - Lock key per user
- Lock TTL configured via `redis.withdraw-lock-time-seconds`

---

## Background Job Configuration

### WithdrawTransferStatusUpdateJob

**Purpose**: Automatically retries withdrawals that timed out during transfer finalization

**Configuration Properties**:
- `withdraw.started-date` - Start date for checking withdrawals
- `retry.timeout.withdraw-retry-start-date` - From which date to retry
- `retry.timeout.withdraw-max-retry-count` - Maximum retry attempts

**Behavior**:
1. Runs on schedule (cron configured)
2. Queries withdrawals with `TRANSFER_FINALIZATION_TIMEOUT` status
3. Checks actual transfer status via TransferCheckerMS
4. If transfer succeeded → Creates DriveWealth redemption
5. If transfer failed → Increments retry count
6. Gives up after max retry count reached

---

## Security Considerations

1. **Authentication**: 
   - User session validation via SessionService
   - DriveWealth OAuth 2.0 bearer tokens

2. **Authorization**:
   - User context verification
   - CIF/PIN validation

3. **Concurrency Control**:
   - Redis locks prevent concurrent withdrawals
   - Check for pending/failed withdrawals

4. **Data Protection**:
   - Sensitive data logged to audit trail
   - PII handling in database

5. **Token Management**:
   - Token caching in AuthorizationService
   - Automatic token refresh on expiry
   - Token eviction service

---

## Performance Optimizations

1. **Token Caching**: DriveWealth tokens cached to reduce auth calls
2. **Redis Locks**: Fast distributed locking mechanism
3. **Async Processing**: History logging doesn't block main flow
4. **Connection Pooling**: Feign clients use connection pools
5. **Database Indexing**: Correlation ID and userId indexed for fast lookups
6. **Retry Strategy**: Smart retry only for retryable errors
7. **Background Jobs**: Offload retry logic to scheduled jobs

---

## Monitoring & Observability

### Logging Points
- Request/response at controller level
- Service method entry/exit
- External service calls
- Redis lock operations
- Retry job executions
- Error scenarios with stack traces
- Status transitions

### Key Metrics to Monitor
- Withdraw initiation success rate
- Withdraw finalization success rate
- Average processing time
- DriveWealth API latency
- Transfer service latency
- Timeout occurrences
- Retry attempts and success rate
- Redis lock contention
- Background job execution time

### Health Checks
- Database connectivity
- Redis connectivity
- External service availability
- DriveWealth API health
- Transfer service health
- Transfer checker service health

---

## Key Differences from Top-Up

| Aspect | Top-Up | Withdraw |
|--------|--------|----------|
| **Direction** | Customer Account → DriveWealth | DriveWealth → Customer Account |
| **DW Operation** | Deposit | Redemption |
| **Onboarding Flow** | Supported | Not needed (user already exists) |
| **Restrictions** | None | Redis lock + pending/failed check |
| **Lock Mechanism** | No | Yes (Redis lock during finalization) |
| **Retry Job** | No | Yes (WithdrawTransferStatusUpdateJob) |
| **Transfer Verification** | No | Yes (TransferCheckerMS) |
| **Concurrency** | Allowed | Prevented via locks |

---

## Best Practices

1. **Idempotency**: Use correlation IDs to prevent duplicate processing
2. **Graceful Degradation**: Specific error codes for different failure scenarios
3. **Audit Trail**: Complete history of all operations
4. **Retry Logic**: Automatic retries for transient failures via background jobs
5. **Circuit Breaker**: Consider implementing for external services
6. **Timeout Configuration**: Proper timeout settings for all external calls
7. **Transaction Management**: Ensure database consistency
8. **Error Messages**: User-friendly error messages with technical details logged
9. **Lock Management**: Always release locks in finally blocks
10. **Restriction Checks**: Prevent problematic concurrent operations

---

## Conclusion

This Investment Withdraw service implements a robust, production-ready flow with:
- ✅ Comprehensive error handling
- ✅ Audit trail and history tracking
- ✅ Retry mechanisms via background jobs
- ✅ Token refresh handling
- ✅ Redis-based locking for concurrency control
- ✅ Restriction checks to prevent issues
- ✅ Integration with multiple external services
- ✅ Proper status tracking throughout the lifecycle
- ✅ Transfer status verification for timeout recovery

The sequence diagrams and documentation provide complete visibility into the system behavior from API entry to successful withdrawal completion.

