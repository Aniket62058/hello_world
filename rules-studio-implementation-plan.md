# Rules Studio — Implementation Plan
### Project: 1CATS | Service: `ui-orchestrator` (Spring Boot) + `UI Service` (React)

---

## Table of Contents

1. [Overview](#1-overview)
2. [Existing Architecture Summary](#2-existing-architecture-summary)
3. [Database Schema](#3-database-schema)
   - 3.1 [Existing Tables](#31-existing-tables)
   - 3.2 [New Table: `approval_request`](#32-new-table-approval_request)
4. [Package Structure (ui-orchestrator)](#4-package-structure-ui-orchestrator)
5. [Entity & Repository Layer](#5-entity--repository-layer)
6. [DTO Layer](#6-dto-layer)
7. [REST API Contracts](#7-rest-api-contracts)
   - 7.1 [ConfigSet APIs](#71-configset-apis)
   - 7.2 [RiskConfig APIs](#72-riskconfig-apis)
   - 7.3 [RulesConfig APIs](#73-rulesconfig-apis)
   - 7.4 [Approval APIs](#74-approval-apis)
8. [Weight Validation Logic](#8-weight-validation-logic)
   - 8.1 [Frontend (JavaScript)](#81-frontend-javascript)
   - 8.2 [Backend (Java)](#82-backend-java)
9. [Approval Workflow State Machine](#9-approval-workflow-state-machine)
10. [Mail Service](#10-mail-service)
11. [application.yml Configuration](#11-applicationyml-configuration)
12. [UI Routes (React)](#12-ui-routes-react)
13. [Wizard Flow — Component Breakdown](#13-wizard-flow--component-breakdown)
14. [Phased Delivery Plan](#14-phased-delivery-plan)
15. [Open Questions](#15-open-questions)

---

## 1. Overview

**Rules Studio** is a configuration management UI and backend workflow within 1CATS that allows users to create and manage `ConfigSet` entities — including their associated `RiskConfig` and `RulesConfig` — via a guided 3-step wizard.

Key behaviours:
- **Experimental configs** are saved directly with status `APPROVED`.
- **Official configs** are saved with status `PENDING_APPROVAL` and trigger an email to an approver with a token-based deep link.
- The rule engine requires that the sum of all `weight` values within a `ConfigSet` equals exactly `1.0` (validated with float tolerance `± 0.0001` on both frontend and backend).

---

## 2. Existing Architecture Summary

| Layer | Technology |
|---|---|
| Backend Service | `ui-orchestrator` — Spring Boot |
| Frontend Service | `UI Service` — React |
| API Gateway | Apigee (OAuth2 token-based auth) |
| Message Streaming | Confluent Kafka + Avro (Schema Registry) |
| Container Platform | OpenShift Container Platform (OCP) |
| Auth Provider | PingFederate (CSPing) — OAuth2 Resource Server |

The UI Service calls `ui-orchestrator` via REST APIs. `ui-orchestrator` owns persistence and all business logic.

---

## 3. Database Schema

### 3.1 Existing Tables

```sql
-- Already exists
CREATE TABLE config_set (
    id                BIGINT PRIMARY KEY AUTO_INCREMENT,
    name              VARCHAR(100) NOT NULL,
    config_set_version VARCHAR(20),
    sor               VARCHAR(50),
    business_unit     VARCHAR(50)
);

-- Already exists
CREATE TABLE config_artifact (
    artifact_id      BIGINT PRIMARY KEY AUTO_INCREMENT,
    config_type      VARCHAR(50) NOT NULL,  -- RULES / RISK_SCORE / ALERT_POLICY / ML_INFERENCE
    config_version   VARCHAR(20),
    effective_date   DATE,
    status           VARCHAR(30) NOT NULL,  -- APPROVED / PENDING_APPROVAL / REJECTED
    asset_class_code VARCHAR(20),
    name             VARCHAR(100),
    description      TEXT,
    created_by       VARCHAR(100),
    created_at       TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    modified_by      VARCHAR(100),
    modified_at      TIMESTAMP
);

-- Already exists
CREATE TABLE risk_config (
    artifact_id      BIGINT PRIMARY KEY,
    threshold_score  DECIMAL(5,4) NOT NULL,
    created_by       VARCHAR(100),
    created_at       TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    modified_by      VARCHAR(100),
    modified_at      TIMESTAMP,
    FOREIGN KEY (artifact_id) REFERENCES config_artifact(artifact_id)
);

-- Already exists
CREATE TABLE rules_config (
    id                    BIGINT PRIMARY KEY AUTO_INCREMENT,
    rule_indicator_id     VARCHAR(100),
    rules_artifact_id     BIGINT NOT NULL,
    asset_class_code      VARCHAR(20),
    name                  VARCHAR(100),
    version               VARCHAR(20),
    margin                DECIMAL(10,4),
    cap                   DECIMAL(10,4),
    rolling_max_look_back INT,
    weight                DECIMAL(5,4) NOT NULL,
    created_by            VARCHAR(100),
    created_at            TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    modified_by           VARCHAR(100),
    modified_at           TIMESTAMP,
    FOREIGN KEY (rules_artifact_id) REFERENCES config_artifact(artifact_id)
);
```

### 3.2 New Table: `approval_request`

```sql
CREATE TABLE approval_request (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    config_artifact_id  BIGINT NOT NULL,
    config_set_id       BIGINT NOT NULL,
    submitted_by        VARCHAR(100) NOT NULL,
    submitted_at        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    approver_email      VARCHAR(200) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'PENDING',  -- PENDING / APPROVED / REJECTED
    approver_comments   TEXT,
    reviewed_by         VARCHAR(100),
    reviewed_at         TIMESTAMP,
    approval_token      VARCHAR(36) NOT NULL UNIQUE,  -- UUID used in email deep link
    token_expires_at    TIMESTAMP NOT NULL,           -- submitted_at + 72 hours
    FOREIGN KEY (config_artifact_id) REFERENCES config_artifact(artifact_id),
    FOREIGN KEY (config_set_id) REFERENCES config_set(id)
);

CREATE INDEX idx_approval_token ON approval_request(approval_token);
CREATE INDEX idx_approval_config_artifact ON approval_request(config_artifact_id);
```

---

## 4. Package Structure (ui-orchestrator)

All new code lives under the existing base package, e.g. `com.company.cats.uiorchestrator`.

```
com.company.cats.uiorchestrator
│
├── controller
│   ├── ConfigSetController.java
│   ├── RiskConfigController.java
│   ├── RulesConfigController.java
│   └── ApprovalController.java
│
├── service
│   ├── ConfigSetService.java
│   ├── RiskConfigService.java
│   ├── RulesConfigService.java
│   ├── ApprovalService.java
│   └── MailService.java
│
├── repository
│   ├── ConfigSetRepository.java
│   ├── ConfigArtifactRepository.java
│   ├── RiskConfigRepository.java
│   ├── RulesConfigRepository.java
│   └── ApprovalRequestRepository.java
│
├── entity
│   ├── ConfigSet.java
│   ├── ConfigArtifact.java
│   ├── RiskConfig.java
│   ├── RulesConfig.java
│   └── ApprovalRequest.java
│
├── dto
│   ├── request
│   │   ├── ConfigSetRequestDto.java
│   │   ├── RiskConfigRequestDto.java
│   │   ├── RulesConfigRequestDto.java
│   │   ├── RulesConfigLineItemDto.java
│   │   └── ApprovalActionDto.java
│   └── response
│       ├── ConfigSetResponseDto.java
│       ├── RiskConfigResponseDto.java
│       ├── RulesConfigResponseDto.java
│       └── ApprovalRequestResponseDto.java
│
├── enums
│   ├── ConfigType.java          -- RULES, RISK_SCORE, ALERT_POLICY, ML_INFERENCE
│   ├── ArtifactStatus.java      -- APPROVED, PENDING_APPROVAL, REJECTED
│   └── ApprovalStatus.java      -- PENDING, APPROVED, REJECTED
│
└── exception
    ├── WeightSumValidationException.java
    ├── ApprovalTokenExpiredException.java
    └── ApprovalTokenNotFoundException.java
```

---

## 5. Entity & Repository Layer

### `ConfigArtifact.java`

```java
@Entity
@Table(name = "config_artifact")
public class ConfigArtifact {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long artifactId;

    @Enumerated(EnumType.STRING)
    private ConfigType configType;

    private String configVersion;
    private LocalDate effectiveDate;

    @Enumerated(EnumType.STRING)
    private ArtifactStatus status;

    private String assetClassCode;
    private String name;
    private String description;
    private String createdBy;
    private LocalDateTime createdAt;
    private String modifiedBy;
    private LocalDateTime modifiedAt;
}
```

### `ApprovalRequest.java`

```java
@Entity
@Table(name = "approval_request")
public class ApprovalRequest {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "config_artifact_id", nullable = false)
    private Long configArtifactId;

    @Column(name = "config_set_id", nullable = false)
    private Long configSetId;

    private String submittedBy;
    private LocalDateTime submittedAt;
    private String approverEmail;

    @Enumerated(EnumType.STRING)
    private ApprovalStatus status;

    private String approverComments;
    private String reviewedBy;
    private LocalDateTime reviewedAt;

    @Column(unique = true, nullable = false)
    private String approvalToken;   // UUID string

    private LocalDateTime tokenExpiresAt;
}
```

### `ApprovalRequestRepository.java`

```java
public interface ApprovalRequestRepository extends JpaRepository<ApprovalRequest, Long> {

    Optional<ApprovalRequest> findByApprovalToken(String token);

    List<ApprovalRequest> findByStatusOrderBySubmittedAtDesc(ApprovalStatus status);

    List<ApprovalRequest> findByConfigSetIdOrderBySubmittedAtDesc(Long configSetId);
}
```

---

## 6. DTO Layer

### `RulesConfigRequestDto.java`

```java
public class RulesConfigRequestDto {
    private Long configSetId;
    private String assetClassCode;
    private String configVersion;
    private LocalDate effectiveDate;
    private List<RulesConfigLineItemDto> rules;
}
```

### `RulesConfigLineItemDto.java`

```java
public class RulesConfigLineItemDto {
    private String ruleIndicatorId;
    private String name;
    private BigDecimal weight;       // Must collectively sum to 1.0 ± 0.0001
    private BigDecimal cap;
    private Integer rollingMaxLookBack;
    private String assetClassCode;
}
```

### `ApprovalActionDto.java`

```java
public class ApprovalActionDto {
    private String token;           // UUID from email deep link
    private String action;          // "APPROVE" or "REJECT"
    private String approverComments;
    private String reviewedBy;      // Approver's ID / email
}
```

---

## 7. REST API Contracts

All endpoints are prefixed with `/api/v1/rules-studio`.

### 7.1 ConfigSet APIs

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/config-sets` | List all ConfigSets (paginated) |
| `GET` | `/config-sets/{id}` | Get a single ConfigSet by ID |
| `POST` | `/config-sets` | Create a new ConfigSet |
| `PUT` | `/config-sets/{id}` | Update an existing ConfigSet |
| `GET` | `/config-sets/{id}/summary` | Full summary: ConfigSet + RiskConfig + RulesConfig |

**POST `/config-sets` — Request Body:**
```json
{
  "name": "Official-Q3-2025",
  "type": "OFFICIAL",
  "configSetVersion": "v1.0",
  "sor": "SOR_A",
  "businessUnit": "EQUITY"
}
```

**POST `/config-sets` — Response:**
```json
{
  "id": 101,
  "name": "Official-Q3-2025",
  "type": "OFFICIAL",
  "configSetVersion": "v1.0",
  "sor": "SOR_A",
  "businessUnit": "EQUITY",
  "createdAt": "2025-07-14T10:00:00"
}
```

---

### 7.2 RiskConfig APIs

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/config-sets/{configSetId}/risk-config` | Get RiskConfig for a ConfigSet |
| `POST` | `/config-sets/{configSetId}/risk-config` | Create RiskConfig (creates ConfigArtifact internally) |
| `PUT` | `/config-sets/{configSetId}/risk-config` | Update existing RiskConfig |

**POST `/config-sets/{configSetId}/risk-config` — Request Body:**
```json
{
  "thresholdScore": 0.75,
  "configVersion": "v1.0",
  "effectiveDate": "2025-08-01"
}
```

---

### 7.3 RulesConfig APIs

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/config-sets/{configSetId}/rules-config` | Get all rules for a ConfigSet |
| `POST` | `/config-sets/{configSetId}/rules-config` | Submit all rules (full replace, triggers approval workflow if OFFICIAL) |
| `PUT` | `/config-sets/{configSetId}/rules-config` | Update rules (same approval logic applies) |
| `GET` | `/config-sets/{configSetId}/rules-config/weight-sum` | Get current weight sum (used for UI live validation) |

**POST `/config-sets/{configSetId}/rules-config` — Request Body:**
```json
{
  "assetClassCode": "EQ",
  "configVersion": "v1.0",
  "effectiveDate": "2025-08-01",
  "rules": [
    {
      "ruleIndicatorId": "RULE_001",
      "name": "Velocity Check",
      "weight": 0.40,
      "cap": 100.0,
      "rollingMaxLookBack": 30,
      "assetClassCode": "EQ"
    },
    {
      "ruleIndicatorId": "RULE_002",
      "name": "Cancel Rate",
      "weight": 0.60,
      "cap": 50.0,
      "rollingMaxLookBack": 15,
      "assetClassCode": "EQ"
    }
  ]
}
```

**POST `/config-sets/{configSetId}/rules-config` — Response (OFFICIAL type):**
```json
{
  "artifactId": 205,
  "status": "PENDING_APPROVAL",
  "message": "Rules submitted for approval. Approval email sent to approver.",
  "approvalRequestId": 42
}
```

**POST `/config-sets/{configSetId}/rules-config` — Response (EXPERIMENTAL type):**
```json
{
  "artifactId": 206,
  "status": "APPROVED",
  "message": "Rules saved and approved automatically (Experimental config)."
}
```

---

### 7.4 Approval APIs

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/approvals/pending` | List all PENDING approval requests (Admin/Approver view) |
| `GET` | `/approvals/{approvalRequestId}` | Get approval request details |
| `GET` | `/approvals/token/{token}` | Resolve approval request by deep-link token (used by email link) |
| `POST` | `/approvals/action` | Approver takes action (APPROVE or REJECT) |
| `GET` | `/approvals/config-set/{configSetId}` | Approval history for a ConfigSet |

**POST `/approvals/action` — Request Body:**
```json
{
  "token": "550e8400-e29b-41d4-a716-446655440000",
  "action": "APPROVE",
  "approverComments": "Reviewed and approved for Q3 go-live.",
  "reviewedBy": "approver@company.com"
}
```

**POST `/approvals/action` — Response:**
```json
{
  "approvalRequestId": 42,
  "configArtifactId": 205,
  "newStatus": "APPROVED",
  "reviewedBy": "approver@company.com",
  "reviewedAt": "2025-07-15T14:30:00",
  "message": "Config artifact 205 has been APPROVED."
}
```

**Error Responses:**

| HTTP Status | Scenario |
|---|---|
| `400 Bad Request` | Weight sum != 1.0 or invalid action value |
| `404 Not Found` | Token or approval request not found |
| `409 Conflict` | Approval request already actioned |
| `410 Gone` | Approval token has expired (past 72h) |
| `422 Unprocessable Entity` | ConfigSet type mismatch or business rule violation |

---

## 8. Weight Validation Logic

The sum of all rule `weight` values within a `ConfigSet` must equal **exactly `1.0`**, validated with a float tolerance of `± 0.0001`.

### 8.1 Frontend (JavaScript)

```javascript
// constants.js
export const WEIGHT_TOLERANCE = 0.0001;
export const REQUIRED_WEIGHT_SUM = 1.0;

/**
 * Validates that the total weight of all rules sums to 1.0 within tolerance.
 * Uses integer arithmetic to avoid floating-point precision issues.
 *
 * @param {Array<{ weight: number }>} rules
 * @returns {{ valid: boolean, sum: number, message: string }}
 */
export function validateWeightSum(rules) {
  if (!rules || rules.length === 0) {
    return { valid: false, sum: 0, message: 'At least one rule is required.' };
  }

  // Round each weight to 4 decimal places before summing
  const sum = rules.reduce((acc, rule) => {
    const w = parseFloat(rule.weight) || 0;
    return acc + Math.round(w * 10000);
  }, 0) / 10000;

  const diff = Math.abs(sum - REQUIRED_WEIGHT_SUM);
  const valid = diff <= WEIGHT_TOLERANCE;

  return {
    valid,
    sum,
    message: valid
      ? 'Weight sum is valid.'
      : `Weight sum is ${sum.toFixed(4)}. It must equal exactly 1.0 (± ${WEIGHT_TOLERANCE}).`
  };
}

// Usage in RulesConfigPage.jsx
const { valid, sum, message } = validateWeightSum(rules);
// Disable submit button when !valid
```

**Live Weight Sum Display (React component snippet):**

```jsx
// RulesConfigPage.jsx
const weightResult = validateWeightSum(rules);

return (
  <div>
    <span className={weightResult.valid ? 'weight-valid' : 'weight-invalid'}>
      Total Weight: {weightResult.sum.toFixed(4)} / 1.0000
    </span>
    <button
      type="button"
      onClick={handleSubmit}
      disabled={!weightResult.valid}
    >
      Submit
    </button>
  </div>
);
```

---

### 8.2 Backend (Java)

```java
// WeightValidator.java
@Component
public class WeightValidator {

    private static final BigDecimal REQUIRED_SUM = BigDecimal.ONE;
    private static final BigDecimal TOLERANCE = new BigDecimal("0.0001");

    /**
     * Validates that the sum of all rule weights equals 1.0 within tolerance.
     * Uses BigDecimal to avoid floating-point precision errors.
     *
     * @param rules list of RulesConfigLineItemDto
     * @throws WeightSumValidationException if validation fails
     */
    public void validate(List<RulesConfigLineItemDto> rules) {
        if (rules == null || rules.isEmpty()) {
            throw new WeightSumValidationException("Rules list cannot be empty.");
        }

        BigDecimal sum = rules.stream()
                .map(RulesConfigLineItemDto::getWeight)
                .filter(Objects::nonNull)
                .reduce(BigDecimal.ZERO, BigDecimal::add);

        BigDecimal diff = REQUIRED_SUM.subtract(sum).abs();

        if (diff.compareTo(TOLERANCE) > 0) {
            throw new WeightSumValidationException(
                String.format(
                    "Weight sum validation failed. Sum = %s, required = 1.0 (tolerance ± %s).",
                    sum.toPlainString(),
                    TOLERANCE.toPlainString()
                )
            );
        }
    }
}
```

```java
// WeightSumValidationException.java
@ResponseStatus(HttpStatus.BAD_REQUEST)
public class WeightSumValidationException extends RuntimeException {
    public WeightSumValidationException(String message) {
        super(message);
    }
}
```

**Usage in `RulesConfigService.java`:**

```java
@Service
public class RulesConfigService {

    private final WeightValidator weightValidator;

    public RulesConfigResponseDto submitRules(Long configSetId, RulesConfigRequestDto dto) {
        // Step 1: Validate weight sum (throws if invalid)
        weightValidator.validate(dto.getRules());

        // Step 2: Determine ConfigSet type and route to approval or direct save
        // ...
    }
}
```

---

## 9. Approval Workflow State Machine

```
                        ┌──────────────┐
                        │   SUBMITTED  │
                        │  (Official)  │
                        └──────┬───────┘
                               │
                    Save with status =
                    PENDING_APPROVAL +
                    Send approval email
                               │
                               ▼
                  ┌────────────────────────┐
                  │     PENDING_APPROVAL   │◄──── Approver receives
                  │   (ConfigArtifact +    │      email with deep link
                  │   ApprovalRequest)     │      (token valid 72h)
                  └────────┬───────────────┘
                           │
             ┌─────────────┴─────────────┐
             │                           │
       action = APPROVE             action = REJECT
             │                           │
             ▼                           ▼
    ┌─────────────────┐       ┌──────────────────────┐
    │    APPROVED     │       │       REJECTED        │
    │  ConfigArtifact │       │  ConfigArtifact       │
    │  status updated │       │  status updated       │
    │  to APPROVED    │       │  to REJECTED          │
    └─────────────────┘       │  comments stored      │
                              └──────────────────────┘
                                         │
                                 User may re-submit
                                 (creates new artifact
                                  + new approval req)
```

### State Transition Table

| Current State | Action | Next State | Side Effect |
|---|---|---|---|
| — | Submit EXPERIMENTAL | `APPROVED` | Save directly, no email |
| — | Submit OFFICIAL | `PENDING_APPROVAL` | Create `ApprovalRequest`, send email |
| `PENDING_APPROVAL` | Approver approves | `APPROVED` | Update artifact status, update approval request |
| `PENDING_APPROVAL` | Approver rejects | `REJECTED` | Update artifact status, store comments |
| `PENDING_APPROVAL` | Token expired | `PENDING_APPROVAL` | `410 Gone` returned on token lookup; manual re-trigger required |

### `ApprovalService.java` — Core Logic

```java
@Service
@Transactional
public class ApprovalService {

    private final ApprovalRequestRepository approvalRequestRepository;
    private final ConfigArtifactRepository configArtifactRepository;

    public ApprovalRequestResponseDto processAction(ApprovalActionDto actionDto) {
        // 1. Resolve token
        ApprovalRequest request = approvalRequestRepository
                .findByApprovalToken(actionDto.getToken())
                .orElseThrow(() -> new ApprovalTokenNotFoundException(
                        "No approval request found for the provided token."));

        // 2. Check if already actioned
        if (request.getStatus() != ApprovalStatus.PENDING) {
            throw new IllegalStateException(
                    "This approval request has already been " + request.getStatus());
        }

        // 3. Check token expiry
        if (LocalDateTime.now().isAfter(request.getTokenExpiresAt())) {
            throw new ApprovalTokenExpiredException(
                    "Approval token has expired. Please request a new approval email.");
        }

        // 4. Apply action
        ApprovalStatus newStatus = ApprovalStatus.valueOf(actionDto.getAction());
        ArtifactStatus artifactStatus = newStatus == ApprovalStatus.APPROVED
                ? ArtifactStatus.APPROVED
                : ArtifactStatus.REJECTED;

        // 5. Update approval request
        request.setStatus(newStatus);
        request.setApproverComments(actionDto.getApproverComments());
        request.setReviewedBy(actionDto.getReviewedBy());
        request.setReviewedAt(LocalDateTime.now());

        // 6. Update config artifact
        ConfigArtifact artifact = configArtifactRepository
                .findById(request.getConfigArtifactId())
                .orElseThrow();
        artifact.setStatus(artifactStatus);
        artifact.setModifiedBy(actionDto.getReviewedBy());
        artifact.setModifiedAt(LocalDateTime.now());

        approvalRequestRepository.save(request);
        configArtifactRepository.save(artifact);

        return buildResponse(request, artifactStatus);
    }
}
```

---

## 10. Mail Service

### `MailService.java`

```java
@Service
public class MailService {

    private final JavaMailSender mailSender;

    @Value("${rules-studio.mail.approver-email}")
    private String approverEmail;

    @Value("${rules-studio.mail.deep-link-base-url}")
    private String deepLinkBaseUrl;

    @Value("${rules-studio.mail.from-address}")
    private String fromAddress;

    /**
     * Sends an approval request email to the configured approver.
     *
     * @param token         the UUID approval token
     * @param configSetName the name of the ConfigSet pending approval
     * @param submittedBy   the user who submitted
     */
    public void sendApprovalRequestEmail(String token, String configSetName, String submittedBy) {
        String deepLink = deepLinkBaseUrl + "/approval?token=" + token;

        SimpleMailMessage message = new SimpleMailMessage();
        message.setFrom(fromAddress);
        message.setTo(approverEmail);
        message.setSubject("[1CATS Rules Studio] Approval Required: " + configSetName);
        message.setText(buildEmailBody(configSetName, submittedBy, deepLink));

        mailSender.send(message);
    }

    private String buildEmailBody(String configSetName, String submittedBy, String deepLink) {
        return String.format("""
                Hello,

                A new Official ConfigSet has been submitted for your approval in 1CATS Rules Studio.

                ConfigSet Name : %s
                Submitted By   : %s

                Please review and take action within 72 hours:
                %s

                If the link above has expired, please contact the submitter to re-initiate the request.

                This is an automated email. Please do not reply.
                """, configSetName, submittedBy, deepLink);
    }
}
```

### Token Generation in `RulesConfigService.java`

```java
private ApprovalRequest createApprovalRequest(Long artifactId, Long configSetId,
                                               String submittedBy, int tokenExpiryHours) {
    ApprovalRequest request = new ApprovalRequest();
    request.setConfigArtifactId(artifactId);
    request.setConfigSetId(configSetId);
    request.setSubmittedBy(submittedBy);
    request.setSubmittedAt(LocalDateTime.now());
    request.setApproverEmail(approverEmail);
    request.setStatus(ApprovalStatus.PENDING);
    request.setApprovalToken(UUID.randomUUID().toString());
    request.setTokenExpiresAt(LocalDateTime.now().plusHours(tokenExpiryHours));
    return approvalRequestRepository.save(request);
}
```

---

## 11. application.yml Configuration

```yaml
# application.yml (ui-orchestrator)

spring:
  mail:
    host: smtp.company.com
    port: 587
    username: ${MAIL_USERNAME}
    password: ${MAIL_PASSWORD}
    properties:
      mail:
        smtp:
          auth: true
          starttls:
            enable: true
            required: true
        transport:
          protocol: smtp

rules-studio:
  mail:
    from-address: noreply-1cats@company.com
    approver-email: ${RULES_STUDIO_APPROVER_EMAIL:approver@company.com}
    deep-link-base-url: ${RULES_STUDIO_DEEP_LINK_BASE_URL:https://1cats.company.com}
  approval:
    token-expiry-hours: 72
  weight:
    tolerance: 0.0001
    required-sum: 1.0
```

> **Note:** `MAIL_USERNAME`, `MAIL_PASSWORD`, `RULES_STUDIO_APPROVER_EMAIL`, and `RULES_STUDIO_DEEP_LINK_BASE_URL` should be injected as environment variables or OCP Secrets — never hardcoded.

---

## 12. UI Routes (React)

```jsx
// App.jsx — React Router configuration
import { BrowserRouter, Routes, Route } from 'react-router-dom';

function App() {
  return (
    <BrowserRouter>
      <Routes>

        {/* Rules Studio Wizard */}
        <Route path="/rules-studio/new"            element={<ConfigSetPage mode="create" />} />
        <Route path="/rules-studio/edit/:id"       element={<ConfigSetPage mode="edit" />} />
        <Route path="/rules-studio/risk-config"    element={<RiskConfigPage />} />
        <Route path="/rules-studio/rules-config"   element={<RulesConfigPage />} />

        {/* Approval */}
        <Route path="/approval"                    element={<ApprovalPage />} />
        {/* ApprovalPage reads ?token=<UUID> from query params */}

        {/* Admin */}
        <Route path="/rules-studio/pending-approvals" element={<PendingApprovalsPage />} />

      </Routes>
    </BrowserRouter>
  );
}
```

### Route Summary

| Route | Component | Description |
|---|---|---|
| `/rules-studio/new` | `ConfigSetPage` | Wizard Step 1 — create new ConfigSet |
| `/rules-studio/edit/:id` | `ConfigSetPage` | Wizard Step 1 — edit existing ConfigSet |
| `/rules-studio/risk-config` | `RiskConfigPage` | Wizard Step 2 — set threshold score |
| `/rules-studio/rules-config` | `RulesConfigPage` | Wizard Step 3 — add/edit rules, live weight sum |
| `/approval?token=<UUID>` | `ApprovalPage` | Deep-link target — approver reviews and acts |
| `/rules-studio/pending-approvals` | `PendingApprovalsPage` | Admin/Approver list of all pending configs |

---

## 13. Wizard Flow — Component Breakdown

```
WizardShell (manages step state + shared data context)
├── Step 1: ConfigSetPage
│   ├── Form: name, type (Official/Experimental), SOR, businessUnit
│   ├── Action: POST /config-sets  OR  GET /config-sets/{id} + PUT /config-sets/{id}
│   └── On Success → navigate to /rules-studio/risk-config (pass configSetId in context)
│
├── Step 2: RiskConfigPage
│   ├── Form: thresholdScore
│   ├── Action: POST /config-sets/{configSetId}/risk-config
│   └── On Success → navigate to /rules-studio/rules-config
│
└── Step 3: RulesConfigPage
    ├── Dynamic rule rows: ruleIndicatorId, name, weight, cap, rollingMaxLookBack, assetClassCode
    ├── Live weight sum displayed (updates on every weight change)
    ├── Submit button disabled until weightSum ≈ 1.0 (tolerance ± 0.0001)
    ├── Action: POST /config-sets/{configSetId}/rules-config
    └── On Success:
        ├── EXPERIMENTAL → success toast "Config saved and approved"
        └── OFFICIAL     → info modal "Submitted for approval. Email sent to approver."
```

---

## 14. Phased Delivery Plan

### Phase 1 — Experimental Flow (Core CRUD)

**Goal:** End-to-end working wizard for Experimental ConfigSets. No approval workflow.

| # | Task | Owner |
|---|---|---|
| 1.1 | DB migration: create `approval_request` table (inactive in this phase) | Backend |
| 1.2 | Implement `ConfigSet` entity, repo, service, controller (CRUD) | Backend |
| 1.3 | Implement `ConfigArtifact` + `RiskConfig` entity, repo, service, controller | Backend |
| 1.4 | Implement `RulesConfig` entity, repo, service, controller | Backend |
| 1.5 | Implement `WeightValidator` with BigDecimal tolerance logic | Backend |
| 1.6 | Unit tests: `WeightValidator`, `RulesConfigService` | Backend |
| 1.7 | Implement `ConfigSetPage` (Step 1) in React | Frontend |
| 1.8 | Implement `RiskConfigPage` (Step 2) in React | Frontend |
| 1.9 | Implement `RulesConfigPage` (Step 3) with live weight counter | Frontend |
| 1.10 | Frontend weight validation utility (`validateWeightSum`) + unit tests | Frontend |
| 1.11 | Wizard state management (React context or Zustand) | Frontend |
| 1.12 | Integration testing: full Experimental wizard flow | QA/Dev |

**Exit Criteria:** User can complete the 3-step wizard for an Experimental ConfigSet; rules are saved with status `APPROVED`; weight validation enforced on both frontend and backend.

---

### Phase 2 — Official Config + Approval Workflow

**Goal:** Official ConfigSets trigger the approval email flow; approver can approve or reject via deep link.

| # | Task | Owner |
|---|---|---|
| 2.1 | Implement `ApprovalRequest` entity, repo | Backend |
| 2.2 | Implement `ApprovalService` (token creation, state machine, action handler) | Backend |
| 2.3 | Implement `MailService` using `JavaMailSender` | Backend |
| 2.4 | Wire Official submit path in `RulesConfigService` → create approval request + send email | Backend |
| 2.5 | Implement `ApprovalController` (token resolution, action endpoint) | Backend |
| 2.6 | `application.yml` mail config + OCP Secrets setup | DevOps/Backend |
| 2.7 | Implement `ApprovalPage` in React (token-based, loads request details, Approve/Reject UI) | Frontend |
| 2.8 | Handle expired token response (`410 Gone`) gracefully in UI | Frontend |
| 2.9 | Update `RulesConfigPage` to show post-submit messaging for Official flow | Frontend |
| 2.10 | Integration test: full Official wizard → email → approval deep link → status update | QA/Dev |
| 2.11 | Token expiry edge case testing (72h boundary) | QA |

**Exit Criteria:** Official ConfigSet submission sends email; approver can approve/reject via deep link; config artifact status transitions correctly; expired tokens return 410.

---

### Phase 3 — Admin, Audit & Pending Approvals

**Goal:** Give admins/approvers visibility into pending requests and a full audit trail.

| # | Task | Owner |
|---|---|---|
| 3.1 | Implement `GET /approvals/pending` endpoint | Backend |
| 3.2 | Implement approval history per ConfigSet (`GET /approvals/config-set/{id}`) | Backend |
| 3.3 | Implement `PendingApprovalsPage` in React (list of pending requests with actions) | Frontend |
| 3.4 | Add `modifiedBy` / `modifiedAt` audit logging to all service write paths | Backend |
| 3.5 | Add soft-delete flag (`is_deleted`) to `ConfigArtifact` and `ConfigSet` (pending decision on Open Questions) | Backend |
| 3.6 | Role-based access: only users with `APPROVER` role can access approval endpoints | Backend |
| 3.7 | Audit log table or structured logging for all state transitions | Backend |
| 3.8 | Re-submission flow for REJECTED configs (UI prompt + new artifact creation) | Frontend/Backend |

**Exit Criteria:** Approvers can see all pending requests in one view; full audit trail exists for all config lifecycle events; role-based access enforced.

---

### Phase 4 — ML Extensibility

**Goal:** Extend the Rules Studio configuration model to support ML-based inference configs alongside the existing rule-based configs.

| # | Task | Owner |
|---|---|---|
| 4.1 | Design `ML_INFERENCE` config artifact schema and additional metadata table | Backend/Arch |
| 4.2 | Extend `ConfigArtifact.configType` enum to include `ML_INFERENCE` (already in enum) | Backend |
| 4.3 | Add ML Config step to wizard (conditional — shown only when ML type selected) | Frontend |
| 4.4 | Extend approval workflow to handle ML configs (same state machine, different email template) | Backend |
| 4.5 | ML config versioning: link trained model artifact reference to `ConfigArtifact` | Backend/ML |
| 4.6 | Feature flag for ML config wizard step (enable only for approved tenants/BUs) | Backend/Frontend |

**Exit Criteria:** ML inference configs can be created, submitted for approval, and approved using the same Rules Studio wizard infrastructure.

---

## 15. Open Questions

The following items require explicit decisions before or during Phase 2/3 development:

| # | Question | Context | Options |
|---|---|---|---|
| OQ-1 | **Approver config strategy** | Currently a single `approverEmail` in `application.yml`. Is this always one fixed approver, or should it be per-BusinessUnit or per-SOR? | (a) Single global approver email — simplest; (b) Approver mapped per BusinessUnit in DB; (c) Approver group (distribution list) |
| OQ-2 | **Rejected config re-submission** | When a config is REJECTED, can the original submitter edit and re-submit? Does this create a new `ConfigArtifact` + new `ApprovalRequest`, or update the existing ones? | (a) New artifact + new request per re-submission (full history preserved); (b) Overwrite existing artifact (simpler but loses audit trail) |
| OQ-3 | **Token expiry handling** | After 72h, the token returns `410 Gone`. Can the submitter re-trigger the approval email (generating a new token) without resubmitting all rules? | (a) Allow re-send approval email endpoint; (b) Require full re-submission to generate new token |
| OQ-4 | **Soft vs hard delete** | Should `ConfigSet` and `ConfigArtifact` records support soft delete (`is_deleted` flag) or only hard delete? | (a) Soft delete — preserves history, required for audit; (b) Hard delete — simpler, but irreversible |
| OQ-5 | **Maker-checker enforcement** | Should the system enforce that the approver cannot be the same person as the submitter? | (a) Enforce at backend (compare `submittedBy` vs `reviewedBy`); (b) Trust process (no enforcement) |
| OQ-6 | **Versioning strategy** | When updating an existing APPROVED ConfigSet, should the new submission create a brand-new `ConfigArtifact` version or overwrite? How is rollback handled? | (a) Always create new artifact with incremented `configVersion` (recommended — full lineage); (b) Overwrite existing artifact |
| OQ-7 | **Concurrent approvals** | If two Official configs for the same `ConfigSet` are in `PENDING_APPROVAL` simultaneously, which takes precedence? | (a) Disallow concurrent pending approvals for same ConfigSet; (b) Allow — last approved wins; (c) Sequential queue |
| OQ-8 | **Weight sum scope** | Is the `weight == 1.0` constraint enforced per `assetClassCode` within a ConfigSet, or globally across all rules in the ConfigSet? | Needs clarification from product/domain team |

---

*Document version: 1.0 — Last updated: July 2025*
*Prepared for: 1CATS Rules Studio — ui-orchestrator + UI Service*
