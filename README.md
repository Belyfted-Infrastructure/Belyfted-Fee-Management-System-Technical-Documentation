# Belyfted Fee Management System - Technical Documentation

## Table of Contents
1. [System Overview](#system-overview)
2. [System Architecture](#system-architecture)
3. [Database Schema](#database-schema)
4. [Service Fee Structure](#service-fee-structure)
5. [Auto-Upgrade Logic](#auto-upgrade-logic)
6. [API Endpoints](#api-endpoints)
7. [Admin Management](#admin-management)

---

## 1. System Overview

The Belyfted Fee Management System handles three distinct user categories with automatic plan upgrades based on monthly transaction thresholds. The system processes payments, tracks transaction limits, and seamlessly upgrades users to higher tiers when their activity exceeds current plan thresholds.

### Key Features
- Multi-tier subscription management for three user categories
- Automatic plan upgrades based on transaction volume
- Real-time transaction fee calculation
- Monthly limit tracking and enforcement
- Comprehensive admin dashboard for fee management

---

## 2. System Architecture

### 2.1 Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                  Frontend Layer                      │
│  (Admin Dashboard, User Portal, API Consumers)       │
└──────────────────┬──────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────┐
│              Application Layer (Laravel)             │
│  ┌──────────────┐ ┌──────────────┐ ┌─────────────┐ │
│  │ Controllers  │ │   Services   │ │   Events    │ │
│  └──────────────┘ └──────────────┘ └─────────────┘ │
│  ┌──────────────┐ ┌──────────────┐ ┌─────────────┐ │
│  │   Models     │ │  Middleware  │ │   Queues    │ │
│  └──────────────┘ └──────────────┘ └─────────────┘ │
└──────────────────┬──────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────┐
│                Database Layer (MySQL)                │
│  Users | Plans | Subscriptions | Transactions        │
└──────────────────────────────────────────────────────┘
```

### 2.2 Core Components

**Controllers:**
- `PlanController` - Manages plan CRUD operations
- `SubscriptionController` - Handles user subscriptions
- `TransactionController` - Processes transactions and fees
- `AdminFeeController` - Admin fee management interface

**Services:**
- `FeeCalculationService` - Calculates transaction fees
- `PlanUpgradeService` - Handles automatic upgrades
- `LimitTrackingService` - Monitors transaction limits
- `BillingService` - Processes monthly charges

**Events:**
- `TransactionProcessed` - Fired after each transaction
- `PlanUpgraded` - Fired when user is auto-upgraded
- `MonthlyLimitReached` - Fired when limit is reached
- `MonthlyBillingDue` - Fired for monthly subscription charges

**Jobs:**
- `ProcessMonthlyBilling` - Charges monthly subscription fees
- `CheckTransactionLimits` - Monitors and upgrades plans
- `GenerateMonthlyReports` - Creates financial reports

---

## 3. Database Schema

### 3.1 Plans Table
```sql
CREATE TABLE plans (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_category ENUM('high_risk', 'sme_enterprise', 'individual') NOT NULL,
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    monthly_fee DECIMAL(10, 2) NOT NULL DEFAULT 0.00,
    transaction_fee DECIMAL(10, 2) NOT NULL DEFAULT 0.00,
    monthly_transaction_limit INT UNSIGNED NOT NULL DEFAULT 0,
    monthly_amount_limit DECIMAL(15, 2) NULL,
    features JSON NULL,
    is_active BOOLEAN DEFAULT TRUE,
    sort_order INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_category (user_category),
    INDEX idx_active (is_active)
);
```

### 3.2 Users Table (Extended)
```sql
CREATE TABLE users (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    user_category ENUM('high_risk', 'sme_enterprise', 'individual') NOT NULL,
    current_plan_id BIGINT UNSIGNED NULL,
    account_status ENUM('active', 'suspended', 'closed') DEFAULT 'active',
    kyc_verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (current_plan_id) REFERENCES plans(id) ON DELETE SET NULL,
    INDEX idx_category (user_category),
    INDEX idx_plan (current_plan_id)
);
```

### 3.3 Subscriptions Table
```sql
CREATE TABLE subscriptions (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT UNSIGNED NOT NULL,
    plan_id BIGINT UNSIGNED NOT NULL,
    status ENUM('active', 'cancelled', 'expired', 'upgraded') DEFAULT 'active',
    started_at TIMESTAMP NOT NULL,
    ends_at TIMESTAMP NULL,
    cancelled_at TIMESTAMP NULL,
    upgraded_to_plan_id BIGINT UNSIGNED NULL,
    auto_upgraded BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (plan_id) REFERENCES plans(id) ON DELETE RESTRICT,
    FOREIGN KEY (upgraded_to_plan_id) REFERENCES plans(id) ON DELETE SET NULL,
    INDEX idx_user_status (user_id, status),
    INDEX idx_dates (started_at, ends_at)
);
```

### 3.4 Transactions Table
```sql
CREATE TABLE transactions (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT UNSIGNED NOT NULL,
    subscription_id BIGINT UNSIGNED NULL,
    transaction_type ENUM('payment', 'transfer', 'withdrawal', 'deposit') NOT NULL,
    amount DECIMAL(15, 2) NOT NULL,
    transaction_fee DECIMAL(10, 2) NOT NULL DEFAULT 0.00,
    currency VARCHAR(3) DEFAULT 'GBP',
    status ENUM('pending', 'completed', 'failed', 'refunded') DEFAULT 'pending',
    reference VARCHAR(100) UNIQUE NOT NULL,
    description TEXT NULL,
    metadata JSON NULL,
    processed_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (subscription_id) REFERENCES subscriptions(id) ON DELETE SET NULL,
    INDEX idx_user_date (user_id, created_at),
    INDEX idx_reference (reference),
    INDEX idx_status (status)
);
```

### 3.5 Monthly Usage Tracking Table
```sql
CREATE TABLE monthly_usage_tracking (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT UNSIGNED NOT NULL,
    subscription_id BIGINT UNSIGNED NOT NULL,
    month CHAR(7) NOT NULL, -- Format: YYYY-MM
    transaction_count INT UNSIGNED DEFAULT 0,
    total_amount DECIMAL(15, 2) DEFAULT 0.00,
    total_fees_paid DECIMAL(10, 2) DEFAULT 0.00,
    limit_reached_at TIMESTAMP NULL,
    upgrade_triggered BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (subscription_id) REFERENCES subscriptions(id) ON DELETE CASCADE,
    UNIQUE KEY unique_user_month (user_id, month),
    INDEX idx_month (month)
);
```

### 3.6 Plan Upgrades History Table
```sql
CREATE TABLE plan_upgrade_history (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT UNSIGNED NOT NULL,
    from_plan_id BIGINT UNSIGNED NOT NULL,
    to_plan_id BIGINT UNSIGNED NOT NULL,
    from_subscription_id BIGINT UNSIGNED NOT NULL,
    to_subscription_id BIGINT UNSIGNED NOT NULL,
    upgrade_reason ENUM('auto_limit_exceeded', 'manual_upgrade', 'admin_upgrade') NOT NULL,
    trigger_transaction_id BIGINT UNSIGNED NULL,
    monthly_usage_at_upgrade JSON NULL,
    upgraded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (from_plan_id) REFERENCES plans(id) ON DELETE RESTRICT,
    FOREIGN KEY (to_plan_id) REFERENCES plans(id) ON DELETE RESTRICT,
    FOREIGN KEY (trigger_transaction_id) REFERENCES transactions(id) ON DELETE SET NULL,
    INDEX idx_user (user_id),
    INDEX idx_upgrade_date (upgraded_at)
);
```

### 3.7 Monthly Billing Table
```sql
CREATE TABLE monthly_billings (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT UNSIGNED NOT NULL,
    subscription_id BIGINT UNSIGNED NOT NULL,
    billing_month CHAR(7) NOT NULL, -- Format: YYYY-MM
    monthly_fee DECIMAL(10, 2) NOT NULL,
    total_transaction_fees DECIMAL(10, 2) DEFAULT 0.00,
    total_amount DECIMAL(10, 2) NOT NULL,
    status ENUM('pending', 'paid', 'failed', 'refunded') DEFAULT 'pending',
    paid_at TIMESTAMP NULL,
    invoice_number VARCHAR(50) UNIQUE NULL,
    payment_reference VARCHAR(100) NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (subscription_id) REFERENCES subscriptions(id) ON DELETE CASCADE,
    UNIQUE KEY unique_user_billing_month (user_id, billing_month),
    INDEX idx_status (status),
    INDEX idx_billing_month (billing_month)
);
```

---

## 4. Service Fee Structure

### 4.1 High Risk Businesses
- **User Category**: `high_risk`
- **Plan**: MSB (Money Services Business)
- **Monthly Fee**: £500.00
- **Transaction Fee**: £0.40 per transaction
- **Monthly Transaction Limit**: 10,000 transactions (configurable)

### 4.2 SME and Enterprises
| Plan | Monthly Fee | Transaction Fee | Monthly Limit |
|------|-------------|----------------|---------------|
| Basic | £9.99 | £0.10 | 500 transactions |
| Grow | £25.99 | £0.08 | 2,000 transactions |
| Scale | £79.99 | £0.05 | 10,000 transactions |
| Enterprise | Custom | Custom | Unlimited |

### 4.3 Individual Users
| Plan | Monthly Fee | Transaction Fee | Monthly Limit |
|------|-------------|----------------|---------------|
| Basic Account | £0.00 | £0.00 | 100 transactions |
| Student/Graduate | £0.00 | £0.00 | 150 transactions |
| Standard Account | £4.99 | £0.05 | 500 transactions |
| Plus Account | £9.99 | £0.03 | 1,500 transactions |
| Premium Account | £19.99 | £0.02 | 5,000 transactions |
| Diamond Account | £48.00 | £0.00 | Unlimited |

---

## 5. Auto-Upgrade Logic

### 5.1 Upgrade Flow

```
┌─────────────────────────┐
│  Transaction Initiated  │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ Calculate Transaction   │
│ Fee Based on Current    │
│ Plan                    │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ Update Monthly Usage    │
│ Tracking (Count +       │
│ Amount)                 │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ Check if Monthly Limit  │
│ Exceeded?               │
└───────────┬─────────────┘
            │
    ┌───────┴────────┐
    │                │
   Yes              No
    │                │
    ▼                ▼
┌────────────┐  ┌─────────────┐
│ Trigger    │  │ Complete    │
│ Auto-      │  │ Transaction │
│ Upgrade    │  └─────────────┘
└─────┬──────┘
      │
      ▼
┌────────────────────────┐
│ Find Next Higher Plan  │
└─────┬──────────────────┘
      │
      ▼
┌────────────────────────┐
│ Create New             │
│ Subscription           │
└─────┬──────────────────┘
      │
      ▼
┌────────────────────────┐
│ Mark Old Subscription  │
│ as Upgraded            │
└─────┬──────────────────┘
      │
      ▼
┌────────────────────────┐
│ Log Upgrade History    │
└─────┬──────────────────┘
      │
      ▼
┌────────────────────────┐
│ Send Notification to   │
│ User                   │
└────────────────────────┘
```

### 5.2 Upgrade Rules

**Rule 1: Within Same Category**
- Upgrades only occur within the same user category
- High Risk → No auto-upgrade (single plan)
- SME: Basic → Grow → Scale → Enterprise
- Individual: Basic → Student → Standard → Plus → Premium → Diamond

**Rule 2: Immediate Effect**
- Upgrades take effect immediately
- New transaction fees apply to subsequent transactions
- Pro-rated billing for remaining month (optional)

**Rule 3: No Downgrade**
- Automatic downgrades are not performed
- Users can manually downgrade at month-end
- Admin can manually change plans anytime

---

## 6. API Endpoints

### 6.1 Plan Management (Admin)

**GET /api/admin/plans**
```json
Response:
{
    "success": true,
    "data": {
        "high_risk": [...],
        "sme_enterprise": [...],
        "individual": [...]
    }
}
```

**POST /api/admin/plans**
```json
Request:
{
    "user_category": "individual",
    "name": "Premium Account",
    "slug": "premium-account",
    "monthly_fee": 19.99,
    "transaction_fee": 0.02,
    "monthly_transaction_limit": 5000,
    "features": {
        "unlimited_exchange": true,
        "better_rates": true
    }
}
```

**PUT /api/admin/plans/{id}**
**DELETE /api/admin/plans/{id}**

### 6.2 Subscription Management

**POST /api/subscriptions/create**
```json
Request:
{
    "plan_id": 5,
    "payment_method": "card_ending_1234"
}

Response:
{
    "success": true,
    "data": {
        "subscription_id": 123,
        "plan": {...},
        "status": "active",
        "started_at": "2024-01-01T00:00:00Z"
    }
}
```

**GET /api/subscriptions/current**
**POST /api/subscriptions/cancel**
**POST /api/subscriptions/upgrade/{plan_id}**

### 6.3 Transaction Processing

**POST /api/transactions/process**
```json
Request:
{
    "type": "transfer",
    "amount": 100.00,
    "currency": "GBP",
    "recipient": "user@example.com",
    "description": "Payment for services"
}

Response:
{
    "success": true,
    "data": {
        "transaction_id": 456,
        "amount": 100.00,
        "fee": 0.03,
        "total": 100.03,
        "status": "completed",
        "usage": {
            "current_month_transactions": 145,
            "limit": 1500,
            "remaining": 1355
        },
        "auto_upgraded": false
    }
}
```

### 6.4 Usage Tracking

**GET /api/usage/current-month**
```json
Response:
{
    "success": true,
    "data": {
        "month": "2024-01",
        "plan": "Plus Account",
        "transaction_count": 145,
        "transaction_limit": 1500,
        "total_amount": 14500.00,
        "total_fees_paid": 4.35,
        "percentage_used": 9.67
    }
}
```

**GET /api/usage/history?months=6**

---

## 7. Admin Management

### 7.1 Admin Dashboard Features

**Fee Configuration Module**
- View all plans by category
- Create new plans with custom fees
- Edit existing plan details
- Enable/disable plans
- Set transaction limits
- Configure auto-upgrade thresholds

**User Management**
- View users by category and plan
- Manually upgrade/downgrade users
- Override automatic upgrades
- View user transaction history
- Generate user fee reports

**Financial Reports**
- Monthly revenue by plan
- Transaction fee collection
- Upgrade conversion rates
- User growth by plan tier
- Average revenue per user (ARPU)

**System Settings**
- Configure grace periods
- Set notification preferences
- Manage billing cycles
- Define upgrade policies
- Configure fee calculation rules

### 7.2 Admin Controllers Structure

```php
// AdminFeeController.php
- index() - Dashboard overview
- listPlans() - All plans by category
- createPlan() - Form to create plan
- storePlan() - Save new plan
- editPlan() - Edit plan form
- updatePlan() - Update plan
- deletePlan() - Soft delete plan
- togglePlanStatus() - Enable/disable

// AdminUserController.php
- listUsers() - All users with filters
- viewUser() - User detail with usage
- manualUpgrade() - Upgrade user plan
- viewTransactions() - User transactions
- exportReport() - Export user data

// AdminReportsController.php
- revenue() - Revenue reports
- upgrades() - Upgrade analytics
- usage() - Usage statistics
- billing() - Billing summaries
```
