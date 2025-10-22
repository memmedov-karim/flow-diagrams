# Investment Top-Up Service - System Design & Sequence Flow

## Overview
This document provides a comprehensive system design and sequence flow for the Investment Top-Up functionality, covering the complete journey from API initiation to finalization.

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
- [Complete Top-Up Journey](#complete-top-up-journey-high-level)
- [Flow Diagrams](#flow-diagrams)
  - [Initiate Top-Up Flow](#initiate-top-up-flow)
  - [Finalize Top-Up Flow](#finalize-top-up-flow)
- [Status State Diagram](#status-state-diagram)
- [Components](#components)
- [Data Flow](#data-flow)
- [Error Handling](#error-handling)
- [Status Tracking](#status-tracking)

---

## Architecture Overview

The Investment Top-Up service follows a microservices architecture with the following key layers:

1. **Controller Layer**: REST API endpoints
2. **Service Layer**: Business logic orchestration
3. **Client Layer**: Integration with external services (Feign clients)
4. **Repository Layer**: Data persistence
5. **Validation Layer**: Input validation
6. **External Services**: Transfer Service, DriveWealth, Onboarding Service

---

## Components

### 1. TopupController
REST API controller exposing top-up endpoints:
- `POST /top-up/initiate` - Initiates a top-up transaction
- `PUT /top-up/finalize/{authorisationId}/{correlationId}` - Finalizes the top-up
- `GET /top-up/{correlationId}` - Gets transaction details
- `POST /top-up/users` - Creates DriveWealth account

### 2. TopupService (TopupServiceImpl)
Core business logic orchestrating:
- Transaction validation
- Transfer service integration
- DriveWealth deposit creation
- Account onboarding (if needed)
- Database logging and history tracking
- Error handling and retry mechanisms

### 3. External Clients

#### TransferServiceClient
Handles internal bank transfer operations:
- `initiateTopup()` - Initiates money transfer from customer account
- `finalizeTopup()` - Finalizes the transfer with OTP/authorization

#### DriveWealthClient
Integrates with DriveWealth broker:
- `getAuthorizationToken()` - Gets OAuth token
- `createDeposit()` - Creates deposit in investment account

#### OnboardingServiceClient
Handles user onboarding:
- `createAccount()` - Creates DriveWealth account for new users

### 4. Supporting Services

- **TopupValidator**: Validates request parameters
- **SessionService**: Manages user session context
- **FlexInfoService**: Fetches customer information from Flex system
- **AuthorizationService**: Manages DriveWealth OAuth tokens
- **AuthorizationEvictionService**: Handles token refresh
- **RetryableService**: Handles retry logic for failed operations

### 5. Repositories

- **TopupRepository**: Stores top-up transactions
- **TopupHistoryRepository**: Maintains audit trail of all operations

---

## System Architecture Diagram

```mermaid
flowchart LR
    Client([Mobile/Web Client])
    
    subgraph FundingMS[Investment Funding Microservice]
        Controller[TopupController<br/>REST API]
        Service[TopupService<br/>Business Logic]
        Validator[TopupValidator]
        Repos[(TopupRepository<br/>TopupHistoryRepository)]
    end
    
    subgraph ExternalServices[External Services]
        Transfer[Transfer Service<br/>Bank Transfers]
        DW[DriveWealth API<br/>Broker Platform]
        Onboard[Onboarding Service<br/>Account Creation]
        Flex[Flex System<br/>Customer Info]
    end
    
    subgraph Infrastructure[Infrastructure]
        Session[Session Service<br/>Auth & Context]
        Auth[Authorization Service<br/>OAuth Tokens]
        Redis[(Redis Cache<br/>Tokens)]
    end
    
    Client -->|1. POST /initiate| Controller
    Client -->|2. PUT /finalize| Controller
    Controller --> Service
    Service --> Validator
    Service --> Session
    Service --> Repos
    Service -->|Initiate/Finalize Transfer| Transfer
    Service -->|Create Account| Onboard
    Service -->|Create Deposit| DW
    Service -->|Fetch Customer Info| Flex
    Service --> Auth
    Auth --> Redis
    
    style Client fill:#e1f5ff
    style Controller fill:#d4edda
    style Service fill:#fff3cd
    style Repos fill:#e7f3ff
    style Transfer fill:#f8d7da
    style DW fill:#f8d7da
    style Onboard fill:#f8d7da
    style FundingMS fill:#f0f8ff
    style ExternalServices fill:#fff5f5
    style Infrastructure fill:#f5fff5
```

---

## Complete Top-Up Journey (High-Level)

```mermaid
flowchart TB
    subgraph Phase1[" PHASE 1: INITIATE TOP-UP "]
        Start1([Client sends request<br/>POST /top-up/initiate]) --> V1[Validate Request]
        V1 --> U1[Get User Context]
        U1 --> F1[Fetch Customer Info<br/>from Flex]
        F1 --> T1[Call Transfer Service<br/>Initiate Transfer]
        T1 --> D1[Save to Database]
        D1 --> R1([Return AuthorisationID<br/>+ CorrelationID])
    end
    
    R1 -.->|Client gets OTP<br/>and calls finalize| Start2
    
    subgraph Phase2[" PHASE 2: FINALIZE TOP-UP "]
        Start2([Client sends OTP<br/>PUT /top-up/finalize]) --> L2[Load Transaction<br/>by CorrelationID]
        L2 --> T2[Call Transfer Service<br/>Finalize with OTP]
        T2 --> C2{Transfer<br/>Success?}
        C2 -->|No| E2([Return Error])
        C2 -->|Yes| F2{Is Onboarding<br/>Flow?}
        F2 -->|Yes| A2[Create DriveWealth<br/>Account]
        F2 -->|No| A3[Use Existing<br/>Account]
        A2 --> TK2[Get OAuth Token]
        A3 --> TK2
        TK2 --> DW2[Create Deposit<br/>in DriveWealth]
        DW2 --> DWR{DW Deposit<br/>Success?}
        DWR -->|401 Unauthorized| RF[Refresh Token<br/>& Retry]
        RF --> DW2
        DWR -->|Success| S2[Save Final State]
        DWR -->|Failed| S2
        S2 --> R2([Return Receipt])
    end
    
    style Start1 fill:#e1f5ff,stroke:#0066cc,stroke-width:3px
    style Start2 fill:#e1f5ff,stroke:#0066cc,stroke-width:3px
    style R1 fill:#d4edda,stroke:#28a745,stroke-width:3px
    style R2 fill:#d4edda,stroke:#28a745,stroke-width:3px
    style E2 fill:#f8d7da,stroke:#dc3545,stroke-width:3px
    style Phase1 fill:#f0f8ff
    style Phase2 fill:#fff5f5
```

### Legend
- 🔵 **Blue** - Start/Entry points
- 🟢 **Green** - Success endpoints
- 🔴 **Red** - Error endpoints
- 🟡 **Yellow** - Warning/Timeout states
- ⚪ **Light Blue** - Data operations (DB, Cache)
- 💎 **Diamond** - Decision points

---

## Flow Diagrams

### Initiate Top-Up Flow

```mermaid
flowchart TD
    Start([Client: POST /top-up/initiate]) --> A[TopupController receives request]
    A --> B[TopupService.initiateTopup]
    
    B --> C[TopupValidator: validate request]
    C --> D{Is correlationId<br/>valid for<br/>flow type?}
    D -->|No| E[Throw Validation Exception]
    D -->|Yes| F[SessionService: get current user context]
    
    F --> G[Get User Details:<br/>CIF, PIN, Phone, UserID, DwAccountNo]
    
    G --> H{Flow Type?}
    H -->|ONBOARDING| I[Use provided correlationId]
    H -->|DEFAULT| J[Generate new UUID as correlationId]
    
    I --> K[FlexInfoService: fetch customer info]
    J --> K
    
    K --> L[Get Customer Name & Address]
    L --> M{Product Type?}
    M -->|CARD| N[FlexInfoService: fetchCardIban]
    M -->|ACCOUNT| O[FlexInfoService: fetchAccountIban]
    
    N --> P[Create InvestmentTopup Entity<br/>Status: INITIAL]
    O --> P
    
    P --> Q[Build TopupInitiationTransferRequestDTO<br/>amount, currency=USD, correlationId,<br/>productNumber, calculatedRate]
    
    Q --> R[TransferServiceClient: initiateTopup]
    
    R --> S{Transfer Service<br/>Response?}
    S -->|Success| T[Receive AuthorisationDTO<br/>authorisationId, expiryTime]
    S -->|Timeout| U[Catch SocketTimeoutException<br/>Status: TRANSFER_INITIATION_TIMEOUT]
    S -->|Failed| V[Catch Exception<br/>Status: TRANSFER_INITIATION_FAILED]
    
    T --> W[TopupRepository: save InvestmentTopup]
    U --> W
    V --> W
    
    W --> X[TopupHistoryRepository: save history]
    
    X --> Y{Was Transfer<br/>Successful?}
    Y -->|Yes| Z[Add correlationId to response<br/>Return AuthorisationDTO]
    Y -->|No| AA[Throw Exception to Client]
    
    Z --> End1([Return 200 OK<br/>AuthorisationDTO + correlationId])
    AA --> End2([Return Error Response])
    
    style Start fill:#e1f5ff
    style End1 fill:#d4edda
    style End2 fill:#f8d7da
    style E fill:#f8d7da
    style T fill:#d4edda
    style U fill:#fff3cd
    style V fill:#f8d7da
    style P fill:#e7f3ff
    style W fill:#e7f3ff
    style X fill:#e7f3ff
```

---

### Finalize Top-Up Flow

```mermaid
flowchart TD
    Start([Client: PUT /top-up/finalize]) --> A[TopupController receives request<br/>authorisationId, correlationId, TransferAuthoriseDTO]
    A --> B[TopupService.finalizeTopup]
    
    B --> C[TopupRepository: find by correlationId]
    C --> D[Retrieved InvestmentTopup entity]
    
    D --> E[TransferServiceClient: finalizeTopup<br/>authorisationId, OTP/SIMA]
    
    E --> F{Transfer<br/>Finalization<br/>Result?}
    
    F -->|Success=true| G[Update Transfer Details:<br/>transferDate, referenceNumber,<br/>flexContractNumber, RRN]
    F -->|Success=false| H[Status: TRANSFER_FINALIZATION_FAILED<br/>Save & Return Failure Receipt]
    F -->|Timeout| I[Status: TRANSFER_FINALIZATION_TIMEOUT<br/>Throw Exception]
    F -->|Error| J[Status: TRANSFER_FINALIZATION_FAILED<br/>Throw Exception]
    
    H --> SaveDB1[Save to DB & History]
    SaveDB1 --> End2([Return Failure Receipt])
    
    G --> K[SessionService: get current user]
    K --> L[Get User: dwAccountNo, PIN, UserID]
    
    L --> M{Flow Type?}
    M -->|ONBOARDING| N[User doesn't have DW account yet]
    M -->|DEFAULT| O[User already has DW account]
    
    N --> P[OnboardingServiceClient: createAccount<br/>PIN, UserID]
    
    P --> Q{Account<br/>Creation<br/>Result?}
    Q -->|Success| R[Receive: accountNo, accountId, customerId<br/>Update topupLog]
    Q -->|Timeout| S[Status: DW_ACCOUNT_CREATION_TIMEOUT<br/>Throw Exception]
    Q -->|Failed| T[Status: DW_ACCOUNT_CREATION_FAILED<br/>Throw Exception]
    
    R --> U[Use new dwAccountNo]
    O --> V[Use existing dwAccountNo]
    
    U --> W[AuthorizationService: getAuthorizationToken]
    V --> W
    
    W --> X{Get Token<br/>Result?}
    X -->|Success| Y[Received Bearer Token]
    X -->|Timeout| Z[Status: DW_AUTHORIZATION_TIMEOUT<br/>Throw Exception]
    X -->|Failed| AA[Status: DW_AUTHORIZATION_FAILED<br/>Throw Exception]
    
    Y --> AB[Build DriveWealthTopupRequestDto<br/>accountNo, type=DEPOSIT, amount, USD]
    
    AB --> AC[DriveWealthClient: createDeposit<br/>key, request, token]
    
    AC --> AD{DW Deposit<br/>Result?}
    
    AD -->|Success| AE[Receive DriveWealthTopupResponseDto<br/>Update DW fields in topupLog]
    AD -->|401 Unauthorized| AF[Token Expired]
    AD -->|Timeout| AG[Status: DW_TOP_UP_TIMEOUT<br/>Throw Exception]
    AD -->|Business Error| AH[Parse DW Error Response]
    AD -->|Generic Error| AI[Catch Exception]
    
    AF --> AJ[AuthorizationEvictionService: evict token]
    AJ --> AK[AuthorizationService: get new token]
    AK --> AL[DriveWealthClient: createDeposit<br/>RETRY with new token]
    
    AL --> AM{Retry<br/>Result?}
    AM -->|Success| AE
    AM -->|Failed| AN[Handle in catchDwTopupException]
    
    AH --> AO[RetryableService: doRetry]
    AI --> AO
    
    AO --> AP[Status: DW_TOP_UP_FAILED<br/>ExternalStatus: errorCode - message<br/>Throw DwTopUpException]
    
    AE --> AQ{Check DW<br/>Status ID}
    AQ -->|Success Status| AR[Status: TOPUP_SUCCESS<br/>Update all DW fields]
    AQ -->|Failed Status| AS[Status: DW_TOP_UP_FAILED<br/>Update all DW fields]
    
    AR --> AT[TopupRepository: save final state]
    AS --> AT
    
    AT --> AU[TopupHistoryRepository: save history]
    
    AU --> AV[Map to InvestmentReceiptDTO:<br/>TransferReceipt + InvestmentTopup]
    
    AV --> End1([Return 200 OK<br/>InvestmentReceiptDTO])
    
    I --> SaveDB2[Save to DB & History]
    J --> SaveDB2
    S --> SaveDB3[Save to DB & History]
    T --> SaveDB3
    Z --> SaveDB4[Save to DB & History]
    AA --> SaveDB4
    AG --> SaveDB5[Save to DB & History]
    AN --> SaveDB5
    AP --> SaveDB5
    
    SaveDB2 --> End3([Throw Exception])
    SaveDB3 --> End3
    SaveDB4 --> End3
    SaveDB5 --> End3
    
    style Start fill:#e1f5ff
    style End1 fill:#d4edda
    style End2 fill:#f8d7da
    style End3 fill:#f8d7da
    style AR fill:#d4edda
    style AS fill:#fff3cd
    style H fill:#f8d7da
    style I fill:#fff3cd
    style J fill:#f8d7da
    style S fill:#fff3cd
    style T fill:#f8d7da
    style Z fill:#fff3cd
    style AA fill:#f8d7da
    style AG fill:#fff3cd
    style AP fill:#f8d7da
    style AF fill:#fff3cd
    style AE fill:#d4edda
    style G fill:#e7f3ff
    style R fill:#e7f3ff
    style Y fill:#e7f3ff
    style AT fill:#e7f3ff
    style AU fill:#e7f3ff
```

---

## Status State Diagram

This diagram shows all possible internal status transitions throughout the top-up lifecycle:

```mermaid
stateDiagram-v2
    [*] --> INITIAL: Initiate Top-Up Request
    
    INITIAL --> TRANSFER_INITIATION_TIMEOUT: Transfer Service Timeout
    INITIAL --> TRANSFER_INITIATION_FAILED: Transfer Service Failed
    INITIAL --> Authorized: Transfer Initiated Successfully
    
    Authorized --> TRANSFER_FINALIZATION_TIMEOUT: Finalize Timeout
    Authorized --> TRANSFER_FINALIZATION_FAILED: Finalize Failed/Rejected
    Authorized --> TransferSuccess: Transfer Finalized Successfully
    
    TransferSuccess --> DW_ACCOUNT_CREATION_TIMEOUT: Account Creation Timeout (ONBOARDING)
    TransferSuccess --> DW_ACCOUNT_CREATION_FAILED: Account Creation Failed (ONBOARDING)
    TransferSuccess --> AccountReady: Account Exists/Created
    
    AccountReady --> DW_AUTHORIZATION_TIMEOUT: Get Token Timeout
    AccountReady --> DW_AUTHORIZATION_FAILED: Get Token Failed
    AccountReady --> TokenReady: Token Obtained
    
    TokenReady --> DW_TOP_UP_TIMEOUT: Deposit API Timeout
    TokenReady --> DW_TOP_UP_FAILED: Deposit Failed/Business Error
    TokenReady --> TOPUP_SUCCESS: Deposit Created Successfully
    
    TRANSFER_INITIATION_TIMEOUT --> [*]: End (Retryable)
    TRANSFER_INITIATION_FAILED --> [*]: End (Retryable)
    TRANSFER_FINALIZATION_TIMEOUT --> [*]: End (Retryable)
    TRANSFER_FINALIZATION_FAILED --> [*]: End (Not Retryable)
    DW_ACCOUNT_CREATION_TIMEOUT --> [*]: End (Retryable)
    DW_ACCOUNT_CREATION_FAILED --> [*]: End (Retryable)
    DW_AUTHORIZATION_TIMEOUT --> [*]: End (Retryable)
    DW_AUTHORIZATION_FAILED --> [*]: End (Retryable)
    DW_TOP_UP_TIMEOUT --> [*]: End (Retryable)
    DW_TOP_UP_FAILED --> [*]: End (Retryable)
    TOPUP_SUCCESS --> [*]: End (Success ✓)
    
    note right of INITIAL
        Initial state when
        top-up request received
    end note
    
    note right of TOPUP_SUCCESS
        Final success state
        Funds deposited to
        DriveWealth account
    end note
    
    note right of DW_TOP_UP_FAILED
        All failed states are
        logged to history table
        for audit trail
    end note
```

### Status Categories

#### ✅ Success States
- `TOPUP_SUCCESS` - Complete end-to-end success

#### ⏳ In-Progress States  
- `INITIAL` - Request received and validated
- `Authorized` - Transfer service authorization received
- `TransferSuccess` - Bank transfer completed
- `AccountReady` - DriveWealth account ready
- `TokenReady` - OAuth token obtained

#### ⚠️ Timeout States (Retryable)
- `TRANSFER_INITIATION_TIMEOUT`
- `TRANSFER_FINALIZATION_TIMEOUT`
- `DW_ACCOUNT_CREATION_TIMEOUT`
- `DW_AUTHORIZATION_TIMEOUT`
- `DW_TOP_UP_TIMEOUT`

#### ❌ Failed States
- `TRANSFER_INITIATION_FAILED` (Retryable)
- `TRANSFER_FINALIZATION_FAILED` (Not Retryable - funds not debited)
- `DW_ACCOUNT_CREATION_FAILED` (Retryable)
- `DW_AUTHORIZATION_FAILED` (Retryable)
- `DW_TOP_UP_FAILED` (Retryable)

---

## Data Flow

### Request/Response DTOs

#### TopupInitiationRequestDTO
```json
{
  "amount": "BigDecimal",
  "currency": "CurrencyEnum",
  "productNumber": "String",
  "productType": "CategoryProductTypeEnum",
  "calculatedRate": "BigDecimal",
  "flowType": "FlowTypeEnum (DEFAULT | ONBOARDING)",
  "correlationId": "String (required for ONBOARDING)",
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
  "dwTopupId": "String",
  "internalStatus": "InternalTopupStatusEnum"
}
```

---

## Error Handling

### Internal Status Enumeration

The system tracks various internal statuses throughout the top-up lifecycle:

#### Initiation Phase
- `INITIAL` - Transaction initiated
- `TRANSFER_INITIATION_TIMEOUT` - Transfer service timeout
- `TRANSFER_INITIATION_FAILED` - Transfer service failed

#### Finalization Phase - Transfer
- `TRANSFER_FINALIZATION_TIMEOUT` - Transfer finalization timeout
- `TRANSFER_FINALIZATION_FAILED` - Transfer finalization failed

#### Finalization Phase - DW Account
- `DW_ACCOUNT_CREATION_TIMEOUT` - Account creation timeout
- `DW_ACCOUNT_CREATION_FAILED` - Account creation failed

#### Finalization Phase - DW Authorization
- `DW_AUTHORIZATION_TIMEOUT` - Authorization timeout
- `DW_AUTHORIZATION_FAILED` - Authorization failed

#### Finalization Phase - DW Deposit
- `DW_TOP_UP_TIMEOUT` - Deposit creation timeout
- `DW_TOP_UP_FAILED` - Deposit creation failed
- `TOPUP_SUCCESS` - Complete success

### Error Recovery Mechanisms

1. **Token Refresh**: Automatic retry with new token on 401 Unauthorized
2. **Retry Logic**: RetryableService handles retryable failures
3. **Timeout Handling**: Specific status codes for timeout scenarios
4. **Transaction History**: All operations logged to history table
5. **Idempotency**: Correlation ID prevents duplicate transactions

---

## Status Tracking

### Database Tables

#### investment_top_up (Main Table)
Stores the current state of each top-up transaction with fields:
- Transaction details (amount, currency, product info)
- User information (cif, fin, phone, userId)
- Transfer details (referenceNumber, transferDate, RRN)
- DriveWealth details (accountNo, accountId, topupId, status)
- Status tracking (internalTopupStatus, externalTopupStatus)
- Timestamps (createdDate, transferDate)

#### investment_top_up_history (Audit Table)
Maintains complete audit trail of all state changes.

### Flow Types

1. **DEFAULT Flow**: Standard top-up for existing investment users
   - User already has DriveWealth account
   - Skip account creation step

2. **ONBOARDING Flow**: First-time top-up during user onboarding
   - User doesn't have DriveWealth account yet
   - Requires account creation before deposit
   - Uses provided correlationId from onboarding process

---

## Integration Points

### 1. Transfer Service
**Base URL**: Configured via `client.base-url.transfer`

**Endpoints**:
- `POST /investment/top-up/initiate` - Initiates internal bank transfer
- `PUT /investment/top-up/finalize/{authorisationId}` - Finalizes transfer with OTP

**Purpose**: Handles money transfer from customer's bank account to investment account

### 2. DriveWealth API
**Base URL**: Configured via `client.base-url.drive-wealth`

**Endpoints**:
- `POST /auth/tokens` - Gets OAuth bearer token
- `POST /funding/deposits` - Creates deposit in investment account

**Purpose**: Creates deposits in broker investment accounts

### 3. Onboarding Service
**Base URL**: Configured via `client.base-url.onboarding`

**Endpoints**:
- `POST /broker/sign-in` - Creates DriveWealth account for new users

**Purpose**: Onboards new users to DriveWealth platform

### 4. Flex System
**Service**: FlexInfoService (internal)

**Purpose**: Fetches customer information (name, address, IBAN) from legacy banking system

### 5. Session Management
**Service**: SessionService (internal)

**Purpose**: Manages user authentication and session context

---

## Security Considerations

1. **Authentication**: 
   - User session validation via SessionService
   - DriveWealth OAuth 2.0 bearer tokens

2. **Authorization**:
   - User context verification
   - CIF/PIN validation

3. **Data Protection**:
   - Sensitive data logged to audit trail
   - PII handling in database

4. **Token Management**:
   - Token caching in AuthorizationService
   - Automatic token refresh on expiry
   - Token eviction service

---

## Performance Optimizations

1. **Token Caching**: DriveWealth tokens cached to reduce auth calls
2. **Async Processing**: History logging doesn't block main flow
3. **Connection Pooling**: Feign clients use connection pools
4. **Database Indexing**: Correlation ID indexed for fast lookups
5. **Retry Strategy**: Smart retry only for retryable errors

---

## Monitoring & Observability

### Logging Points
- Request/response at controller level
- Service method entry/exit
- External service calls
- Error scenarios with stack traces
- Status transitions

### Key Metrics to Monitor
- Top-up initiation success rate
- Top-up finalization success rate
- Average processing time
- DriveWealth API latency
- Transfer service latency
- Timeout occurrences
- Retry attempts

### Health Checks
- Database connectivity
- External service availability
- DriveWealth API health
- Transfer service health

---

## Best Practices

1. **Idempotency**: Use correlation IDs to prevent duplicate processing
2. **Graceful Degradation**: Specific error codes for different failure scenarios
3. **Audit Trail**: Complete history of all operations
4. **Retry Logic**: Automatic retries for transient failures
5. **Circuit Breaker**: Consider implementing for external services
6. **Timeout Configuration**: Proper timeout settings for all external calls
7. **Transaction Management**: Ensure database consistency
8. **Error Messages**: User-friendly error messages with technical details logged

---

## Conclusion

This Investment Top-Up service implements a robust, production-ready flow with:
- ✅ Comprehensive error handling
- ✅ Audit trail and history tracking
- ✅ Retry mechanisms
- ✅ Token refresh handling
- ✅ Support for both onboarding and standard flows
- ✅ Integration with multiple external services
- ✅ Proper status tracking throughout the lifecycle

The sequence diagrams and documentation provide complete visibility into the system behavior from API entry to successful top-up completion.

