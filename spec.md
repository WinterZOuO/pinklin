# PROJECT DEVELOPMENT PROMPT
# Phase 1 — Membership & Business Core

## 1. Project Objective

Develop Phase 1 of an integrated AI Video Creation & Short Video Platform.

The objective of this phase is to establish the platform's core business infrastructure, including:

1. User Registration & Authentication
2. Membership Tier Management
3. Referral Code System
4. Referral Relationship Management
5. AI Credit Wallet
6. Orders & Payment Management
7. Administrative Backend

This phase must deliver a functional, secure, extensible business foundation.

The system will eventually support three interconnected products:

- Short Video Platform
- AI One-Stop Video Creation Platform
- Membership Center

Phase 1 focuses exclusively on the shared membership and business infrastructure. Do not prematurely implement video streaming or AI video generation.

---

## 2. Technical Architecture

Use a Modular Monolith architecture.

### Recommended Technology Stack

Backend:
- Python 3.12+
- Django 5.x
- Django REST Framework
- PostgreSQL
- Redis
- Celery

Frontend:
- Next.js
- React
- TypeScript
- Tailwind CSS
- Responsive Design

Infrastructure:
- Docker
- Docker Compose
- Environment-based configuration
- RESTful API
- OpenAPI / Swagger documentation

Architecture requirements:

- Strict separation between business modules.
- Shared authentication and user identity.
- Centralized permission management.
- Independent business services.
- Transaction-safe financial operations.
- API-first architecture.
- Future compatibility with mobile applications.
- No hardcoded business rules.
- All sensitive configuration must use environment variables.

Do not introduce microservices at this stage.

---

## 3. Module A — User Registration & Authentication

Implement a complete account management system.

### Registration
- Email registration.
- Phone number registration.
- Email or SMS verification.
- Unique username.
- Password strength validation.
- Optional referral code during registration.
- Duplicate account prevention.

### Login
- Email / Phone / Username login.
- Password authentication.
- JWT access token.
- Refresh token rotation.
- Logout and token revocation.
- Password recovery.
- Login attempt rate limiting.

### User Profile
- User ID.
- Username.
- Avatar.
- Email.
- Phone number.
- Registration timestamp.
- Account status.
- Membership tier.
- Referral code.
- Last login timestamp.

### Account Status
- Active.
- Suspended.
- Disabled.
- Pending verification.

Security requirements:
- Use secure password hashing.
- Never store plaintext passwords.
- Protect against brute-force attacks.
- Implement authentication throttling.
- Enforce authorization on the backend.
- Maintain authentication and security audit logs.

---

## 4. Module B — Membership Tier Management

Create a configurable membership system.

### Default Membership Tiers

1. Free Member
2. Standard Member
3. VIP Member
4. Creator Member

These tiers are initial examples and must be configurable through the administration backend.

### Membership Features

Each membership tier must support:

- Tier name.
- Tier description.
- Monthly price.
- Annual price.
- Membership duration.
- AI credit allocation.
- AI generation privileges.
- Daily usage limits.
- Membership benefits.
- Status.
- Display priority.

### Membership Operations

- Upgrade membership.
- Downgrade membership.
- Renew membership.
- Expiration management.
- Membership purchase history.
- Automatic expiration processing.
- Membership entitlement validation.

Important:

Membership tier and account role must be separate concepts.

A user can simultaneously be:
- A VIP member.
- A content creator.
- A referral participant.

Do not use membership tiers as authorization roles.

---

## 5. Module C — Referral Code System

Build a referral system with unique referral codes for every eligible member.

### Referral Code Generation

Automatically generate a unique referral code after registration.

Requirements:
- Unique code.
- Non-sequential and difficult to guess.
- Case-insensitive validation.
- Referral link generation.
- QR code support.
- Copy-to-clipboard support.
- Referral code activation and suspension.

Example:

REF8K29X

### Referral Registration Flow

1. Member A shares a referral link.
2. Member B opens the registration page.
3. The referral code is automatically populated.
4. Member B completes registration.
5. The system validates the referral code.
6. The system binds Member B to Member A.
7. The referral relationship is recorded.

### Referral Rules

- One account can have only one direct referrer.
- Referral relationships cannot be modified arbitrarily after registration.
- Self-referral is prohibited.
- Circular referral relationships are prohibited.
- Referral ownership must be immutable after successful binding.
- Referral code changes must not rewrite historical referral relationships.
- Referral code validation must be transaction-safe.

### Referral Dashboard

Display:
- Personal referral code.
- Referral URL.
- Total invited members.
- Registered members.
- Paying members.
- Referral reward history.
- Pending rewards.
- Confirmed rewards.

### Referral Reward Engine

Support configurable rewards:

- Fixed reward per qualifying purchase.
- Percentage-based reward on eligible purchases.
- Campaign-specific rewards.
- Minimum qualifying purchase amount.
- Reward settlement delay.
- Refund-related reward reversal.

Initial implementation must support direct, single-level referrals only.

Do not implement multi-level commissions.

Referral rewards must be recorded separately from AI credits and creator earnings.

---

## 6. Module D — AI Credit Wallet

Develop an independent AI credit wallet.

AI credits are platform consumption units, not cash.

### Wallet Functions

- View current balance.
- Purchase credits.
- Receive promotional credits.
- Consume credits.
- Refund credits.
- View transaction history.
- View credit expiration information.

### Credit Types

- Purchased Credits.
- Promotional Credits.
- Membership Credits.
- Administrative Credits.

### Credit Consumption Rules

Implement configurable consumption priorities.

Recommended default priority:

1. Promotional Credits.
2. Membership Credits.
3. Purchased Credits.

Each transaction must record:

- Transaction ID.
- User ID.
- Transaction type.
- Credit amount.
- Previous balance.
- New balance.
- Related order ID.
- Related generation job ID (future extension).
- Expiration date, if applicable.
- Timestamp.

### Financial Integrity

- Use database transactions.
- Prevent negative balances.
- Prevent duplicate deductions.
- Support idempotency keys.
- Maintain immutable transaction records.
- Support concurrent requests safely.
- Never modify historical ledger entries directly.

The wallet must be designed to support future AI generation services.

---

## 7. Module E — Orders & Payment Management

Create a centralized order management system.

### Supported Order Types

- Membership Subscription.
- AI Credit Purchase.
- Membership Renewal.
- Future AI Service Purchase.

### Order Lifecycle

Implement the following state machine:

CREATED
  -> PENDING_PAYMENT
  -> PAID
  -> PROCESSING
  -> COMPLETED

Additional states:

- CANCELLED
- EXPIRED
- REFUND_PENDING
- REFUNDED
- FAILED

State transitions must be validated by backend business logic.

### Order Information

- Order number.
- User ID.
- Product ID.
- Product type.
- Quantity.
- Unit price.
- Total amount.
- Currency.
- Payment method.
- Payment status.
- Order status.
- Created timestamp.
- Paid timestamp.
- Completed timestamp.

### Payment Architecture

Implement a provider-independent payment interface.

Payment providers must be replaceable without modifying core order logic.

Required components:

- Payment provider abstraction.
- Payment initiation.
- Payment callback handling.
- Payment signature verification.
- Idempotent payment processing.
- Payment timeout handling.
- Refund processing.
- Transaction records.

Do not assume a successful payment based solely on a frontend callback.

Only verified backend payment notifications can confirm successful payment.

Implement a mock payment provider for local development and automated testing.

Do not store bank card numbers or sensitive payment credentials.

### Order Processing

After successful payment:

Membership order:
- Activate or extend membership.
- Update membership entitlements.

AI credit order:
- Credit the user's AI wallet.
- Create corresponding ledger entries.

Referral order:
- Evaluate whether the purchase qualifies for referral rewards.
- Generate the corresponding reward record.

All related financial operations must be transaction-safe and idempotent.

---

## 8. Module F — Administrative Backend

Develop a complete administrative dashboard.

### Dashboard Overview

Display:
- Total registered members.
- Active members.
- New registrations.
- VIP members.
- Total orders.
- Paid orders.
- Gross revenue.
- AI credits issued.
- Referral statistics.

### Member Management

- Search members.
- View member profiles.
- Edit permitted profile fields.
- Suspend accounts.
- Reactivate accounts.
- View membership history.
- View referral relationships.
- View wallet transaction history.
- View order history.

### Membership Management

- Create membership tiers.
- Edit membership benefits.
- Set pricing.
- Set AI credit allocation.
- Configure duration.
- Enable or disable membership tiers.

### Referral Management

- Search referral codes.
- View referral relationships.
- View referral performance.
- Configure reward rules.
- Review referral rewards.
- Suspend abusive referral codes.
- Reverse eligible rewards through audited adjustment entries.

### Wallet Management

- Search wallet accounts.
- View credit transactions.
- Issue promotional credits.
- Perform authorized adjustments.
- View credit expiration.
- Export transaction records.

Every administrative balance adjustment must require:
- Explicit reason.
- Administrator identity.
- Transaction reference.
- Audit log entry.

### Order Management

- Search orders.
- View payment records.
- View order lifecycle.
- Review refunds.
- Export order reports.

### System Management

- Administrator accounts.
- Role-based access control.
- Permission management.
- System configuration.
- Audit logs.
- API configuration.

Implement role-based permissions, at minimum:

- Super Administrator.
- Finance Administrator.
- Member Support.
- Operations Administrator.

Administrative permissions must be enforced server-side.

---

## 9. Database Design

Design a normalized PostgreSQL database.

At minimum, implement the following entities:

### Identity
- users
- user_profiles
- user_sessions
- verification_codes

### Authorization
- roles
- permissions
- user_roles
- role_permissions

### Membership
- membership_tiers
- membership_entitlements
- user_memberships
- membership_orders

### Referral
- referral_codes
- referral_bindings
- referral_reward_rules
- referral_rewards

### AI Wallet
- credit_wallets
- credit_lots
- credit_ledger

### Commerce
- products
- orders
- order_items
- payment_transactions
- refund_records

### Administration
- admin_audit_logs
- system_settings

Requirements:

- Define primary keys and foreign keys.
- Add appropriate indexes.
- Enforce uniqueness constraints.
- Define database transaction boundaries.
- Support historical records.
- Use UTC timestamps.
- Define appropriate monetary precision.
- Use database constraints wherever possible.

Separate cash-based financial records from non-cash AI credit records.

Do not use a single balance field to represent all financial assets.

---

## 10. API Development

Create versioned REST APIs.

Use the following API namespace:

/api/v1/

Suggested endpoints:

### Authentication
POST /auth/register
POST /auth/login
POST /auth/logout
POST /auth/refresh
POST /auth/password/reset

### Member
GET /members/me
PATCH /members/me
GET /members/me/membership

### Referral
GET /referrals/me
GET /referrals/me/statistics
GET /referrals/me/members
GET /referrals/me/rewards
POST /referrals/validate

### Wallet
GET /wallet/credits
GET /wallet/transactions
POST /wallet/credits/purchase

### Membership
GET /memberships/tiers
GET /memberships/me
POST /memberships/purchase
POST /memberships/renew

### Orders
POST /orders
GET /orders
GET /orders/{order_id}
POST /orders/{order_id}/cancel

### Payment
POST /payments/initiate
POST /payments/webhooks/{provider}

### Administration
/api/v1/admin/members/
/api/v1/admin/memberships/
/api/v1/admin/referrals/
/api/v1/admin/wallets/
/api/v1/admin/orders/
/api/v1/admin/reports/
/api/v1/admin/settings/

Document all APIs using OpenAPI.

Define request validation, response formats, error codes and authorization requirements.

---

## 11. Security & Financial Integrity

Security is a mandatory requirement.

Implement:

- Authentication and authorization.
- CSRF protection where applicable.
- CORS configuration.
- Input validation.
- SQL injection prevention.
- Rate limiting.
- Secure password storage.
- Payment signature verification.
- Webhook idempotency.
- Audit logging.
- Sensitive data protection.
- Administrative action tracking.

Financial requirements:

- All monetary operations must be transaction-safe.
- All payment callbacks must be idempotent.
- Refunds must create reversal records.
- Wallet ledger records must be immutable.
- No direct balance manipulation outside wallet services.
- Use database locking where concurrent balance changes may occur.
- Maintain reconciliation capability.

---

## 12. Testing Requirements

Implement automated tests for all critical business logic.

### Authentication Tests
- Successful registration.
- Duplicate registration.
- Invalid credentials.
- Token refresh.
- Suspended account login.

### Membership Tests
- Membership purchase.
- Membership upgrade.
- Membership expiration.
- Membership renewal.

### Referral Tests
- Unique referral code generation.
- Successful referral binding.
- Invalid referral code.
- Self-referral rejection.
- Duplicate referral binding.
- Reward calculation.
- Refund-related reward reversal.

### Wallet Tests
- Credit purchase.
- Credit consumption.
- Insufficient balance.
- Concurrent deductions.
- Duplicate transaction prevention.
- Refund crediting.

### Payment Tests
- Successful payment callback.
- Invalid signature.
- Duplicate callback.
- Payment timeout.
- Refund processing.

### Administration Tests
- Role permissions.
- Unauthorized access rejection.
- Administrative balance adjustment.
- Audit log generation.

Testing requirements:

- Unit tests.
- Integration tests.
- API tests.
- Database transaction tests.
- Permission tests.
- Critical concurrency tests.

Use pytest.

Financial integrity tests must not be omitted.

---

## 13. Development Workflow

Follow a structured development process.

### Step 1 — Repository Inspection

Before modifying anything:

- Inspect the existing repository.
- Identify the existing architecture.
- Inspect current dependencies.
- Identify existing reusable components.
- Do not overwrite existing functionality without justification.

### Step 2 — Technical Design

Create:

docs/
  PRD.md
  ARCHITECTURE.md
  DATABASE-ERD.md
  API-SPECIFICATION.md
  BUSINESS-RULES.md
  SECURITY-DESIGN.md
  PAYMENT-FLOW.md
  REFERRAL-FLOW.md
  WALLET-LEDGER.md

### Step 3 — Implementation

Implement modules in this order:

1. Project infrastructure.
2. User identity and authentication.
3. Membership management.
4. Referral code and relationships.
5. AI credit wallet.
6. Orders and payments.
7. Administrative backend.
8. Integration and business workflow testing.

### Step 4 — Quality Assurance

Before marking Phase 1 complete:

- Run database migrations.
- Run all automated tests.
- Verify financial transaction integrity.
- Verify permissions.
- Verify API documentation.
- Verify Docker deployment.
- Verify frontend/backend integration.

### Step 5 — Delivery

Produce:

- Complete source code.
- Database migrations.
- API documentation.
- ERD diagrams.
- Docker Compose configuration.
- Environment variable example.
- Automated test suite.
- Local deployment guide.
- Administrator setup guide.
- Initial super administrator creation command.

---

## 14. Future Expansion Requirements

Although future modules are not part of Phase 1, the architecture must support them.

### Future Module A — AI Video Studio

Will use:
- Existing user identity.
- AI credit wallet.
- Membership entitlements.
- Centralized order system.

Future entities:
- ai_projects
- ai_templates
- generation_jobs
- model_providers
- media_assets

### Future Module B — Short Video Platform

Will use:
- Existing member accounts.
- Creator profiles.
- Video uploads.
- Watch statistics.
- Creator earnings.
- Revenue allocation.
- Payout management.

Future entities:
- creator_profiles
- videos
- video_series
- episodes
- watch_events
- revenue_batches
- revenue_allocations
- payout_requests

Do not implement these modules now.

Ensure that current architecture does not prevent their integration.

---

## 15. Development Rules

You are acting as a Senior Software Architect and Full-Stack Engineer.

Follow these rules:

1. Do not attempt to implement the entire project in one uncontrolled operation.
2. Design before coding.
3. Use clear module boundaries.
4. Prefer reusable services over duplicated business logic.
5. Never place financial logic directly inside frontend components.
6. Never trust frontend payment status.
7. Never modify immutable financial ledger history.
8. Use database transactions for financial operations.
9. Make all payment and wallet operations idempotent.
10. Write tests alongside implementation.
11. Keep documentation synchronized with code.
12. Avoid unnecessary dependencies and premature microservices.
13. Do not use placeholder implementations for critical financial logic.
14. Do not claim functionality is complete without test evidence.

If an architectural decision is unclear, document the options, trade-offs and recommended approach before proceeding.

---

## 16. Definition of Done

Phase 1 is complete only when:

- A user can register and log in.
- A user can manage their profile.
- Membership tiers can be configured.
- Users can purchase and manage memberships.
- Every eligible user can obtain a referral code.
- Referral relationships are correctly recorded.
- Referral rewards can be calculated and audited.
- Users can purchase AI credits.
- AI credits can be consumed and refunded safely.
- Orders can be created and tracked.
- Payment callbacks are securely verified.
- Administrators can manage members.
- Administrators can manage membership tiers.
- Administrators can manage referral rules.
- Administrators can inspect wallet transactions.
- Administrators can manage orders and refunds.
- All critical business workflows have automated tests.
- The complete system can be deployed using Docker Compose.

**Final Objective: Deliver a production-oriented Phase 1 Membership & Business Core that can serve as the shared foundation for the future AI Video Studio and Short Video Platform.**
