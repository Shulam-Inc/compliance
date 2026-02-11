# Compliance Project Plan

## Overview

KYT/AML/OFAC screening service for transaction monitoring and regulatory compliance. Ensures all Shulam payments comply with sanctions requirements and AML regulations.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      COMPLIANCE                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────┐        │
│  │              Screening Engine                    │        │
│  │                                                  │        │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐        │        │
│  │  │   OFAC   │ │   KYT    │ │ Velocity │        │        │
│  │  │  Screen  │ │  Score   │ │  Check   │        │        │
│  │  └──────────┘ └──────────┘ └──────────┘        │        │
│  └─────────────────────────────────────────────────┘        │
│                          │                                   │
│                          ▼                                   │
│  ┌─────────────────────────────────────────────────┐        │
│  │              Decision Engine                     │        │
│  │         (approve / review / block)              │        │
│  └─────────────────────────────────────────────────┘        │
│                          │                                   │
│                          ▼                                   │
│  ┌─────────────────────────────────────────────────┐        │
│  │              Alert & Reporting                   │        │
│  │         (SAR generation, audit logs)            │        │
│  └─────────────────────────────────────────────────┘        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Dependencies

| Dependency | Type | Purpose |
|------------|------|---------|
| facilitator | Internal | Receives screening requests |
| Chainalysis KYT | External | Transaction risk scoring |
| TRM Labs | External | Wallet risk assessment |
| OFAC SDN List | External | Sanctions screening |

---

## Milestones

### M1: Foundation
- [ ] Service scaffold
- [ ] OFAC SDN list loader
- [ ] Local caching of lists
- [ ] Basic screening endpoint

### M2: OFAC Screening
- [ ] Exact address match
- [ ] Fuzzy name matching (for entities)
- [ ] Daily list updates
- [ ] Screening audit log

### M3: KYT Integration
- [ ] Chainalysis API integration
- [ ] Risk score thresholds
- [ ] Transaction monitoring rules
- [ ] Alert generation

### M4: Velocity Checks
- [ ] Transaction frequency limits
- [ ] Amount thresholds
- [ ] Geographic patterns
- [ ] Behavioral anomaly detection

### M5: Reporting
- [ ] SAR generation
- [ ] CTR generation (if needed)
- [ ] Compliance dashboard
- [ ] Regulatory exports

---

## User Stories (Gherkin)

### Epic 1: OFAC Screening

```gherkin
Feature: OFAC SDN List Screening
  As a compliance officer
  I want all transactions screened against OFAC lists
  So that we don't process payments involving sanctioned parties

  Background:
    Given the compliance service is running
    And the OFAC SDN list is loaded (updated daily)
    And the screening endpoint is available

  Scenario: Screen clean wallet address
    Given a wallet address "0xClean123..."
    And this address is not on any sanctions list
    When I POST to /screen with the address
    Then I receive a 200 response
    And the response contains:
      | field      | value   |
      | sanctioned | false   |
      | risk_level | low     |
      | cleared    | true    |

  Scenario: Block sanctioned wallet address
    Given a wallet address "0xSanctioned456..."
    And this address is on the OFAC SDN list
    When I POST to /screen with the address
    Then I receive a 200 response
    And the response contains:
      | field      | value     |
      | sanctioned | true      |
      | risk_level | blocked   |
      | list       | OFAC_SDN  |
      | cleared    | false     |
    And an alert is created for compliance review
    And the screening is logged for audit

  Scenario: Screen address associated with sanctioned entity
    Given a wallet address "0xEntityRelated..."
    And this address has transacted with known sanctioned addresses
    When I POST to /screen with the address
    Then I receive a 200 response
    And the response contains:
      | field      | value    |
      | sanctioned | false    |
      | risk_level | high     |
      | flags      | indirect_exposure |
    And a review alert is created

  Scenario: Daily OFAC list update
    Given the OFAC SDN list was last updated 24 hours ago
    When the daily update job runs
    Then the latest OFAC SDN list is downloaded
    And new entries are added to the screening database
    And removed entries are marked inactive
    And a summary is logged

  Scenario: Handle OFAC list unavailable
    Given the OFAC API is unavailable
    When a screening request arrives
    Then the cached list is used
    And a warning is logged
    And operations is alerted if cache is > 48 hours old
```

### Epic 2: KYT Risk Scoring

```gherkin
Feature: Know Your Transaction Risk Scoring
  As a compliance officer
  I want transactions scored for risk
  So that I can identify suspicious activity

  Background:
    Given the Chainalysis KYT integration is active
    And risk thresholds are configured:
      | level   | score_range |
      | low     | 0-30        |
      | medium  | 31-60       |
      | high    | 61-80       |
      | severe  | 81-100      |

  Scenario: Score low-risk transaction
    Given a transaction from wallet "0xClean..."
    And the wallet has no exposure to high-risk services
    And the transaction amount is 50 USDC
    When I POST to /screen/transaction
    Then the KYT score is 15
    And the decision is "approve"
    And no alerts are generated

  Scenario: Score medium-risk transaction
    Given a transaction from wallet "0xMixed..."
    And the wallet has minor exposure to mixing services
    When I POST to /screen/transaction
    Then the KYT score is 45
    And the decision is "approve"
    And the transaction is logged for monitoring

  Scenario: Flag high-risk transaction
    Given a transaction from wallet "0xRisky..."
    And the wallet has significant darknet market exposure
    When I POST to /screen/transaction
    Then the KYT score is 72
    And the decision is "review"
    And an alert is created for manual review
    And the transaction is held pending review

  Scenario: Block severe-risk transaction
    Given a transaction from wallet "0xDangerous..."
    And the wallet is directly linked to ransomware
    When I POST to /screen/transaction
    Then the KYT score is 95
    And the decision is "block"
    And an alert is created with priority "urgent"
    And the facilitator is notified to reject the payment

  Scenario: Handle KYT service unavailable
    Given the Chainalysis API is unavailable
    When a screening request arrives
    Then OFAC screening is still performed
    And the transaction is flagged for delayed KYT review
    And a warning is logged
```

### Epic 3: Velocity Checks

```gherkin
Feature: Transaction Velocity Monitoring
  As a compliance officer
  I want to detect unusual transaction patterns
  So that I can identify potential money laundering

  Background:
    Given velocity rules are configured:
      | rule                    | threshold      |
      | max_daily_amount        | 10,000 USDC    |
      | max_daily_count         | 50 transactions|
      | max_single_amount       | 5,000 USDC     |
      | min_time_between        | 60 seconds     |

  Scenario: Normal transaction velocity
    Given a buyer has made 5 transactions today
    And total daily volume is 500 USDC
    When they make a new 100 USDC transaction
    Then velocity check passes
    And no alerts are generated

  Scenario: Exceed daily transaction count
    Given a buyer has made 49 transactions today
    When they attempt transaction #50
    Then velocity check flags the transaction
    And an alert is created: "Daily transaction count exceeded"
    And the transaction requires manual approval

  Scenario: Exceed daily volume
    Given a buyer has transacted 9,500 USDC today
    When they attempt a 600 USDC transaction
    Then velocity check flags the transaction
    And an alert is created: "Daily volume limit exceeded"
    And the transaction requires manual approval

  Scenario: Rapid-fire transactions
    Given a buyer made a transaction 30 seconds ago
    When they attempt another transaction
    Then velocity check flags the transaction
    And an alert is created: "Transactions too rapid"
    And a brief cooldown is enforced

  Scenario: Structuring detection
    Given a buyer makes multiple transactions of $9,900 each
    And this is just below the $10,000 threshold
    And this pattern repeats over 3 days
    When the pattern analyzer runs
    Then a structuring alert is created
    And compliance officer is notified for review
```

### Epic 4: Alert Management

```gherkin
Feature: Compliance Alerts
  As a compliance officer
  I want to manage and resolve alerts
  So that I can maintain regulatory compliance

  Scenario: View pending alerts
    Given there are 5 unresolved compliance alerts
    When I GET /alerts?status=pending
    Then I receive a list of 5 alerts
    And each alert contains:
      | field       | present |
      | id          | yes     |
      | type        | yes     |
      | severity    | yes     |
      | created_at  | yes     |
      | transaction | yes     |

  Scenario: Resolve alert - clear transaction
    Given an alert "alert_123" for a flagged transaction
    And I have reviewed the transaction
    And it is legitimate
    When I POST to /alerts/alert_123/resolve with:
      | field      | value              |
      | decision   | clear              |
      | notes      | Verified merchant  |
    Then the alert status changes to "resolved"
    And the transaction is approved for processing
    And the resolution is logged for audit

  Scenario: Resolve alert - file SAR
    Given an alert "alert_456" for suspicious activity
    And I have reviewed the transaction
    And it appears to be money laundering
    When I POST to /alerts/alert_456/resolve with:
      | field      | value         |
      | decision   | file_sar      |
      | notes      | Structuring   |
    Then the alert status changes to "sar_filed"
    And a SAR draft is generated
    And the transaction is blocked
    And the wallet is added to internal watchlist
```

### Epic 5: Reporting

```gherkin
Feature: Compliance Reporting
  As a compliance officer
  I want to generate regulatory reports
  So that I can meet filing obligations

  Scenario: Generate SAR draft
    Given a suspicious transaction has been identified
    And all required data is available
    When I POST to /reports/sar/generate
    Then a SAR draft is created with:
      | section             | populated |
      | subject_information | yes       |
      | suspicious_activity | yes       |
      | transaction_details | yes       |
    And the draft is saved for review

  Scenario: Export audit log
    Given I need compliance records for date range
    When I GET /reports/audit?from=2026-01-01&to=2026-01-31
    Then I receive a downloadable report
    And the report includes all screenings
    And the report includes all alerts
    And the report includes all resolutions

  Scenario: Monthly compliance summary
    Given it is the first of the month
    When the monthly summary job runs
    Then a summary report is generated with:
      | metric                  | included |
      | total_screenings        | yes      |
      | blocked_transactions    | yes      |
      | alerts_generated        | yes      |
      | alerts_resolved         | yes      |
      | average_resolution_time | yes      |
    And the report is emailed to compliance team
```

---

## API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | /screen | Screen wallet address |
| POST | /screen/transaction | Full transaction screening |
| GET | /alerts | List compliance alerts |
| POST | /alerts/:id/resolve | Resolve an alert |
| GET | /wallet/:address | Get wallet risk profile |
| POST | /reports/sar/generate | Generate SAR draft |
| GET | /reports/audit | Export audit log |
| GET | /health | Service health check |

---

## Environment Variables

```bash
# External Services
CHAINALYSIS_API_KEY=
TRM_API_KEY=
OFAC_LIST_URL=https://sanctionslist.ofac.treas.gov/api/PublicationPreview/exports/SDN.XML

# Thresholds
RISK_THRESHOLD_REVIEW=60
RISK_THRESHOLD_BLOCK=80
DAILY_VOLUME_LIMIT=10000
DAILY_COUNT_LIMIT=50

# Service
PORT=3001
NODE_ENV=development
COMPLIANCE_OFFICER_EMAIL=compliance@shulam.io

# Storage
DATABASE_URL=
REDIS_URL=
```

---

## Agent Instructions

### For Timothy (Compliance)
1. Own all compliance logic and thresholds
2. Review every SAR before filing
3. Update velocity rules as patterns emerge
4. Ensure audit logs are complete

### For Andrew (Developer)
1. Implement screening endpoints
2. Build Chainalysis integration
3. Create alert management system
4. Ensure sub-100ms response times

### For Thomas (QA)
1. Test with known sanctioned addresses
2. Verify alert generation triggers
3. Test edge cases in velocity checks
4. Audit log completeness testing
