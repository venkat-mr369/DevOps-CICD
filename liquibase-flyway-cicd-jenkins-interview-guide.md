## Liquibase, Flyway & CI/CD Jenkins — Complete Interview Guide
### Database Change Management for PostgreSQL & MySQL

> **Installation Steps • In-Depth Explanations • Real Database Scripts • CI/CD Pipeline Setup**
> For Sr DBA / DevOps / Database Engineer Interviews

---
---

## SECTION 1: DATABASE CHANGE MANAGEMENT — CONCEPTS

---

### Q1. What is Database Change Management (DCM) and why is it needed?

**Answer:** Database Change Management is the practice of tracking, versioning, and deploying database schema changes (DDL) and reference data changes (DML) in a controlled, repeatable, and auditable manner — just like how Git manages application source code.

### The Problem Without DCM

```
  WITHOUT Database Change Management:

  Developer A:                Developer B:              DBA:
  "I added a column          "I renamed that            "Which script do I
   to the users table         column yesterday"          run in production?
   last week"                                            Which order?
                                                         Did QA already run it?"
  ┌──────────┐
  │ DEV DB   │ ── manually run scripts ──► nobody knows what state each DB is in
  │ QA DB    │ ── forgot to run script ──► QA has different schema than DEV
  │ STAGING  │ ── ran scripts out of order ──► staging is broken
  │ PROD DB  │ ── deployed wrong version ──► PRODUCTION IS DOWN!
  └──────────┘
```

### The Solution With DCM (Liquibase/Flyway)

```
  WITH Database Change Management:

  All changes are versioned files in Git:
  V1__create_users_table.sql
  V2__add_email_column.sql
  V3__create_orders_table.sql
  V4__add_index_on_email.sql

  Liquibase/Flyway:
  ├── Tracks which changes have been applied to each database
  ├── Applies only NEW changes (never re-runs old ones)
  ├── Runs changes in correct ORDER (V1 → V2 → V3 → V4)
  ├── Works identically in DEV → QA → STAGING → PROD
  ├── Provides ROLLBACK capability
  └── Integrates with Jenkins CI/CD pipeline

  Result: Every database environment is ALWAYS in the correct state!
```

---

### Q2. What is the difference between Liquibase and Flyway?

| Feature | Liquibase | Flyway |
|---------|-----------|--------|
| **Founded** | 2006 by Nathan Voas | 2010 by Axel Fontaine (Redgate) |
| **License** | Apache 2.0 (Community) + Pro | Apache 2.0 (Community) + Teams/Enterprise |
| **Change Format** | XML, YAML, JSON, SQL | SQL only (+ Java migrations) |
| **Tracking Table** | `DATABASECHANGELOG` | `flyway_schema_history` |
| **Rollback** | Built-in (Community) | Pro/Enterprise only |
| **Diff/Compare** | Built-in (generates changelog) | Not available |
| **Database Support** | 50+ databases | 30+ databases |
| **Changelog Approach** | Changelog file references changesets | Numbered SQL files (V1__, V2__) |
| **Preconditions** | Yes (check before running) | No |
| **Contexts/Labels** | Yes (run different changes per env) | No (Community) |
| **Spring Boot** | Excellent integration | Excellent integration |
| **Jenkins Plugin** | Yes (official) | Yes (community) |
| **Learning Curve** | Medium (XML/YAML to learn) | Easy (just SQL files) |
| **Best For** | Enterprise, multi-DB, complex workflows | Simple, SQL-first, fast start |

**Interview Summary:** Flyway is simpler (just numbered SQL files) — great for teams that want minimal overhead. Liquibase is more powerful (rollback, diff, preconditions, multiple formats) — great for enterprise environments with complex requirements.

---
---

## SECTION 2: FLYWAY — COMPLETE GUIDE

---

### Q3. What is Flyway and how does it work?

**Answer:** Flyway is a database migration tool that uses **versioned SQL scripts** to manage schema changes. It tracks which scripts have been applied using a metadata table called `flyway_schema_history`.

### How Flyway Works — Step by Step

```
  FLYWAY WORKFLOW:

  1. You write numbered SQL migration files:
     V1__create_users.sql
     V2__add_email.sql
     V3__create_orders.sql

  2. You run: flyway migrate

  3. Flyway checks flyway_schema_history table:
     "V1 and V2 already applied. Only V3 is new."

  4. Flyway runs ONLY V3__create_orders.sql

  5. Flyway records V3 in flyway_schema_history:
     | version | description    | script                  | checksum   | success |
     | 1       | create users   | V1__create_users.sql    | -12345678  | true    |
     | 2       | add email      | V2__add_email.sql       | -87654321  | true    |
     | 3       | create orders  | V3__create_orders.sql   | -11223344  | true    |  ← NEW

  6. Next time you run 'flyway migrate', Flyway skips V1, V2, V3
     and only runs V4+ (if any new files exist)
```

### Flyway Naming Convention (CRITICAL!)

```
  V{version}__{description}.sql        → Versioned migration (runs once)
  U{version}__{description}.sql        → Undo migration (Pro only)
  R__{description}.sql                  → Repeatable migration (runs every time checksum changes)

  Rules:
  ├── V = Versioned (most common), U = Undo, R = Repeatable
  ├── Version: numbers separated by dots or underscores (1, 2, 1.1, 2.0.1)
  ├── TWO underscores (__) between version and description
  ├── Description: underscores become spaces in the history table
  └── Extension: .sql

  Examples:
  V1__create_users_table.sql           ✓ Correct
  V2__add_email_column.sql             ✓ Correct
  V2.1__add_phone_column.sql           ✓ Correct (sub-version)
  V3__create_orders_table.sql          ✓ Correct
  R__refresh_materialized_view.sql     ✓ Repeatable (runs when content changes)
  
  V1_create_users.sql                  ✗ WRONG (single underscore)
  create_users.sql                     ✗ WRONG (no V prefix)
  V1-create-users.sql                  ✗ WRONG (hyphens not allowed in version)
```

---

### Q4. How to install Flyway? (Step-by-Step)

### Installation on Linux (Production DBA Server)

```bash
# ============================================================
# FLYWAY INSTALLATION — Linux (Ubuntu/RHEL)
# ============================================================

# Step 1: Download Flyway Community Edition
cd /opt
wget https://repo1.maven.org/maven2/org/flywaydb/flyway-commandline/10.15.0/flyway-commandline-10.15.0-linux-x64.tar.gz

# Step 2: Extract
tar -xzf flyway-commandline-10.15.0-linux-x64.tar.gz
mv flyway-10.15.0 flyway

# Step 3: Add to PATH
echo 'export PATH=$PATH:/opt/flyway' >> ~/.bashrc
source ~/.bashrc

# Step 4: Verify installation
flyway --version
# Output: Flyway Community Edition 10.15.0

# Step 5: Create project directory structure
mkdir -p /opt/flyway-projects/telecom/{sql,conf}

# Directory structure:
# /opt/flyway-projects/telecom/
# ├── conf/
# │   └── flyway.conf          ← database connection config
# └── sql/
#     ├── V1__create_schema.sql
#     ├── V2__create_tables.sql
#     └── V3__insert_seed_data.sql
```

### Installation on Windows

```powershell
# Step 1: Download from https://flywaydb.org/download
# Step 2: Extract to C:\flyway
# Step 3: Add C:\flyway to System PATH
# Step 4: Verify
flyway --version
```

### Installation via Docker (Recommended for CI/CD)

```bash
# Pull Flyway Docker image
docker pull flyway/flyway:10.15.0

# Run migration using Docker
docker run --rm \
  -v /opt/flyway-projects/telecom/sql:/flyway/sql \
  -v /opt/flyway-projects/telecom/conf:/flyway/conf \
  flyway/flyway:10.15.0 migrate
```

---

### Q5. Flyway Configuration for PostgreSQL and MySQL

### PostgreSQL Configuration

```properties
# File: /opt/flyway-projects/telecom/conf/flyway.conf
# ============================================================
# FLYWAY CONFIGURATION — PostgreSQL
# ============================================================

# Database Connection
flyway.url=jdbc:postgresql://pg-server:5432/telecom_db
flyway.user=flyway_admin
flyway.password=Str0ng_P@ssw0rd!
flyway.schemas=public,telecom

# Migration Settings
flyway.locations=filesystem:/opt/flyway-projects/telecom/sql
flyway.table=flyway_schema_history
flyway.baselineOnMigrate=true
flyway.baselineVersion=0
flyway.validateOnMigrate=true
flyway.cleanDisabled=true

# Encoding
flyway.encoding=UTF-8
flyway.placeholderReplacement=true

# PostgreSQL-specific
flyway.defaultSchema=telecom
flyway.createSchemas=true
```

### MySQL Configuration

```properties
# File: /opt/flyway-projects/telecom/conf/flyway-mysql.conf
# ============================================================
# FLYWAY CONFIGURATION — MySQL
# ============================================================

# Database Connection
flyway.url=jdbc:mysql://mysql-server:3306/telecom_db?useSSL=true&serverTimezone=UTC
flyway.user=flyway_admin
flyway.password=Str0ng_P@ssw0rd!
flyway.schemas=telecom_db

# Migration Settings
flyway.locations=filesystem:/opt/flyway-projects/telecom/sql-mysql
flyway.table=flyway_schema_history
flyway.baselineOnMigrate=true
flyway.baselineVersion=0
flyway.validateOnMigrate=true
flyway.cleanDisabled=true

# MySQL-specific
flyway.defaultSchema=telecom_db
```

---

### Q6. Flyway Database Scripts — PostgreSQL

### V1__create_schema_and_roles.sql

```sql
-- ============================================================
-- Flyway Migration V1: Create Schema, Roles, and Extensions
-- Database: PostgreSQL
-- ============================================================

-- Create schema
CREATE SCHEMA IF NOT EXISTS telecom;

-- Create application roles
DO $$
BEGIN
    IF NOT EXISTS (SELECT FROM pg_roles WHERE rolname = 'telecom_app') THEN
        CREATE ROLE telecom_app LOGIN PASSWORD 'AppP@ss123';
    END IF;
    IF NOT EXISTS (SELECT FROM pg_roles WHERE rolname = 'telecom_readonly') THEN
        CREATE ROLE telecom_readonly LOGIN PASSWORD 'ReadP@ss123';
    END IF;
END $$;

-- Grant schema usage
GRANT USAGE ON SCHEMA telecom TO telecom_app;
GRANT USAGE ON SCHEMA telecom TO telecom_readonly;

-- Enable extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";
```

### V2__create_core_tables.sql

```sql
-- ============================================================
-- Flyway Migration V2: Create Core Tables
-- Database: PostgreSQL
-- ============================================================

SET search_path TO telecom;

-- Subscribers table
CREATE TABLE subscribers (
    subscriber_id   UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    first_name      VARCHAR(50) NOT NULL,
    last_name       VARCHAR(50) NOT NULL,
    email           VARCHAR(100) UNIQUE,
    phone           VARCHAR(20) UNIQUE NOT NULL,
    plan_type       VARCHAR(20) DEFAULT 'prepaid' CHECK (plan_type IN ('prepaid', 'postpaid')),
    balance         NUMERIC(12,2) DEFAULT 0.00,
    status          VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'suspended')),
    region          VARCHAR(30) NOT NULL,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Call Detail Records (CDR)
CREATE TABLE call_records (
    cdr_id          UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    subscriber_id   UUID NOT NULL REFERENCES subscribers(subscriber_id),
    call_type       VARCHAR(10) NOT NULL CHECK (call_type IN ('voice', 'sms', 'data')),
    destination     VARCHAR(20),
    duration_sec    INTEGER DEFAULT 0,
    data_usage_mb   NUMERIC(10,2) DEFAULT 0.00,
    cost            NUMERIC(10,2) NOT NULL,
    call_timestamp  TIMESTAMPTZ DEFAULT NOW()
);

-- Plans table
CREATE TABLE plans (
    plan_id         SERIAL PRIMARY KEY,
    plan_name       VARCHAR(50) NOT NULL,
    monthly_cost    NUMERIC(10,2) NOT NULL,
    data_limit_gb   INTEGER,
    voice_minutes   INTEGER,
    sms_limit       INTEGER,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Payments table
CREATE TABLE payments (
    payment_id      UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    subscriber_id   UUID NOT NULL REFERENCES subscribers(subscriber_id),
    amount          NUMERIC(12,2) NOT NULL,
    payment_method  VARCHAR(30) CHECK (payment_method IN ('UPI', 'credit_card', 'debit_card', 'net_banking', 'cash')),
    transaction_ref VARCHAR(50) UNIQUE,
    status          VARCHAR(20) DEFAULT 'completed' CHECK (status IN ('completed', 'pending', 'failed', 'refunded')),
    payment_date    TIMESTAMPTZ DEFAULT NOW()
);

-- Create indexes
CREATE INDEX idx_subscribers_phone ON subscribers(phone);
CREATE INDEX idx_subscribers_region ON subscribers(region);
CREATE INDEX idx_subscribers_status ON subscribers(status);
CREATE INDEX idx_cdr_subscriber ON call_records(subscriber_id);
CREATE INDEX idx_cdr_timestamp ON call_records(call_timestamp DESC);
CREATE INDEX idx_cdr_type ON call_records(call_type);
CREATE INDEX idx_payments_subscriber ON payments(subscriber_id);
CREATE INDEX idx_payments_date ON payments(payment_date DESC);

-- Grant permissions
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA telecom TO telecom_app;
GRANT SELECT ON ALL TABLES IN SCHEMA telecom TO telecom_readonly;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA telecom TO telecom_app;

COMMENT ON TABLE subscribers IS 'Telecom subscriber master data';
COMMENT ON TABLE call_records IS 'Call Detail Records for billing';
COMMENT ON TABLE plans IS 'Available subscription plans';
COMMENT ON TABLE payments IS 'Payment transactions';
```

### V3__insert_seed_data.sql

```sql
-- ============================================================
-- Flyway Migration V3: Insert Seed/Reference Data
-- Database: PostgreSQL
-- ============================================================

SET search_path TO telecom;

-- Insert plans
INSERT INTO plans (plan_name, monthly_cost, data_limit_gb, voice_minutes, sms_limit) VALUES
    ('Basic',     199.00,   1,   100,  100),
    ('Standard',  499.00,   3,   500,  500),
    ('Premium',   999.00,  10,  1000, 1000),
    ('Unlimited', 1499.00, -1,    -1,   -1),
    ('Data Only', 299.00,   5,     0,    0);

-- Insert sample subscribers
INSERT INTO subscribers (first_name, last_name, email, phone, plan_type, balance, region) VALUES
    ('Venkat',  'Sarath',   'venkat@email.com',   '+91-9876543210', 'postpaid', 1500.00, 'ap-south-1'),
    ('Meena',   'Sharma',   'meena@email.com',    '+91-9876543211', 'prepaid',  350.00,  'ap-south-1'),
    ('Kishore', 'Reddy',    'kishore@email.com',  '+91-9876543212', 'postpaid', 2200.00, 'us-east-1'),
    ('Priya',   'Patel',    'priya@email.com',    '+91-9876543213', 'prepaid',  180.00,  'eu-west-3'),
    ('Arjun',   'Nair',     'arjun@email.com',    '+91-9876543214', 'postpaid', 950.00,  'ap-south-1');
```

### V4__add_audit_columns.sql

```sql
-- ============================================================
-- Flyway Migration V4: Add Audit Columns to All Tables
-- Database: PostgreSQL
-- ============================================================

SET search_path TO telecom;

-- Add audit columns to subscribers
ALTER TABLE subscribers ADD COLUMN IF NOT EXISTS created_by VARCHAR(50) DEFAULT CURRENT_USER;
ALTER TABLE subscribers ADD COLUMN IF NOT EXISTS updated_by VARCHAR(50);

-- Add audit columns to call_records
ALTER TABLE call_records ADD COLUMN IF NOT EXISTS created_by VARCHAR(50) DEFAULT CURRENT_USER;

-- Add audit columns to payments
ALTER TABLE payments ADD COLUMN IF NOT EXISTS created_by VARCHAR(50) DEFAULT CURRENT_USER;

-- Create updated_at trigger function
CREATE OR REPLACE FUNCTION update_modified_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    NEW.updated_by = CURRENT_USER;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Apply trigger to subscribers
CREATE TRIGGER trg_subscribers_updated
    BEFORE UPDATE ON subscribers
    FOR EACH ROW
    EXECUTE FUNCTION update_modified_column();
```

### V5__create_partitioned_cdr.sql

```sql
-- ============================================================
-- Flyway Migration V5: Partition CDR Table by Month
-- Database: PostgreSQL 12+
-- ============================================================

SET search_path TO telecom;

-- Create partitioned CDR table for high-volume data
CREATE TABLE call_records_partitioned (
    cdr_id          UUID DEFAULT uuid_generate_v4(),
    subscriber_id   UUID NOT NULL,
    call_type       VARCHAR(10) NOT NULL,
    destination     VARCHAR(20),
    duration_sec    INTEGER DEFAULT 0,
    data_usage_mb   NUMERIC(10,2) DEFAULT 0.00,
    cost            NUMERIC(10,2) NOT NULL,
    call_timestamp  TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (cdr_id, call_timestamp)
) PARTITION BY RANGE (call_timestamp);

-- Create monthly partitions for 2025
CREATE TABLE cdr_2025_01 PARTITION OF call_records_partitioned
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
CREATE TABLE cdr_2025_02 PARTITION OF call_records_partitioned
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');
CREATE TABLE cdr_2025_03 PARTITION OF call_records_partitioned
    FOR VALUES FROM ('2025-03-01') TO ('2025-04-01');
CREATE TABLE cdr_2025_04 PARTITION OF call_records_partitioned
    FOR VALUES FROM ('2025-04-01') TO ('2025-05-01');
CREATE TABLE cdr_2025_05 PARTITION OF call_records_partitioned
    FOR VALUES FROM ('2025-05-01') TO ('2025-06-01');
CREATE TABLE cdr_2025_06 PARTITION OF call_records_partitioned
    FOR VALUES FROM ('2025-06-01') TO ('2025-07-01');

-- Create indexes on partitioned table
CREATE INDEX idx_cdr_part_subscriber ON call_records_partitioned(subscriber_id);
CREATE INDEX idx_cdr_part_timestamp ON call_records_partitioned(call_timestamp DESC);
```

### R__refresh_views.sql (Repeatable Migration)

```sql
-- ============================================================
-- Flyway Repeatable Migration: Refresh Views
-- Runs every time the file content changes
-- Database: PostgreSQL
-- ============================================================

SET search_path TO telecom;

CREATE OR REPLACE VIEW vw_subscriber_summary AS
SELECT 
    s.subscriber_id,
    s.first_name || ' ' || s.last_name AS full_name,
    s.phone,
    s.plan_type,
    s.balance,
    s.region,
    s.status,
    COUNT(c.cdr_id) AS total_calls,
    COALESCE(SUM(c.cost), 0) AS total_spent,
    COALESCE(SUM(c.duration_sec), 0) AS total_duration_sec,
    COALESCE(SUM(c.data_usage_mb), 0) AS total_data_mb,
    MAX(c.call_timestamp) AS last_call_date
FROM subscribers s
LEFT JOIN call_records c ON s.subscriber_id = c.subscriber_id
GROUP BY s.subscriber_id, s.first_name, s.last_name, s.phone, 
         s.plan_type, s.balance, s.region, s.status;

CREATE OR REPLACE VIEW vw_monthly_revenue AS
SELECT 
    DATE_TRUNC('month', c.call_timestamp) AS month,
    c.call_type,
    COUNT(*) AS total_records,
    SUM(c.cost) AS total_revenue,
    AVG(c.cost) AS avg_cost_per_record
FROM call_records c
GROUP BY DATE_TRUNC('month', c.call_timestamp), c.call_type
ORDER BY month DESC, call_type;
```

---

### Q7. Flyway Database Scripts — MySQL

### V1__create_schema_mysql.sql

```sql
-- ============================================================
-- Flyway Migration V1: Create Schema and Tables
-- Database: MySQL 8.0+
-- ============================================================

-- Subscribers table
CREATE TABLE subscribers (
    subscriber_id   CHAR(36) PRIMARY KEY DEFAULT (UUID()),
    first_name      VARCHAR(50) NOT NULL,
    last_name       VARCHAR(50) NOT NULL,
    email           VARCHAR(100) UNIQUE,
    phone           VARCHAR(20) UNIQUE NOT NULL,
    plan_type       ENUM('prepaid', 'postpaid') DEFAULT 'prepaid',
    balance         DECIMAL(12,2) DEFAULT 0.00,
    status          ENUM('active', 'inactive', 'suspended') DEFAULT 'active',
    region          VARCHAR(30) NOT NULL,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_phone (phone),
    INDEX idx_region (region),
    INDEX idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Call Detail Records
CREATE TABLE call_records (
    cdr_id          CHAR(36) PRIMARY KEY DEFAULT (UUID()),
    subscriber_id   CHAR(36) NOT NULL,
    call_type       ENUM('voice', 'sms', 'data') NOT NULL,
    destination     VARCHAR(20),
    duration_sec    INT DEFAULT 0,
    data_usage_mb   DECIMAL(10,2) DEFAULT 0.00,
    cost            DECIMAL(10,2) NOT NULL,
    call_timestamp  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (subscriber_id) REFERENCES subscribers(subscriber_id),
    INDEX idx_cdr_subscriber (subscriber_id),
    INDEX idx_cdr_timestamp (call_timestamp DESC)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Plans table
CREATE TABLE plans (
    plan_id         INT AUTO_INCREMENT PRIMARY KEY,
    plan_name       VARCHAR(50) NOT NULL,
    monthly_cost    DECIMAL(10,2) NOT NULL,
    data_limit_gb   INT,
    voice_minutes   INT,
    sms_limit       INT,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Payments table
CREATE TABLE payments (
    payment_id      CHAR(36) PRIMARY KEY DEFAULT (UUID()),
    subscriber_id   CHAR(36) NOT NULL,
    amount          DECIMAL(12,2) NOT NULL,
    payment_method  ENUM('UPI', 'credit_card', 'debit_card', 'net_banking', 'cash'),
    transaction_ref VARCHAR(50) UNIQUE,
    status          ENUM('completed', 'pending', 'failed', 'refunded') DEFAULT 'completed',
    payment_date    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (subscriber_id) REFERENCES subscribers(subscriber_id),
    INDEX idx_payments_subscriber (subscriber_id),
    INDEX idx_payments_date (payment_date DESC)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### V2__insert_seed_data_mysql.sql

```sql
-- ============================================================
-- Flyway Migration V2: Seed Data — MySQL
-- ============================================================

INSERT INTO plans (plan_name, monthly_cost, data_limit_gb, voice_minutes, sms_limit) VALUES
    ('Basic',     199.00,   1,   100,  100),
    ('Standard',  499.00,   3,   500,  500),
    ('Premium',   999.00,  10,  1000, 1000),
    ('Unlimited', 1499.00, -1,    -1,   -1),
    ('Data Only', 299.00,   5,     0,    0);

INSERT INTO subscribers (first_name, last_name, email, phone, plan_type, balance, region) VALUES
    ('Venkat',  'Sarath',  'venkat@email.com',  '+91-9876543210', 'postpaid', 1500.00, 'ap-south-1'),
    ('Meena',   'Sharma',  'meena@email.com',   '+91-9876543211', 'prepaid',  350.00,  'ap-south-1'),
    ('Kishore', 'Reddy',   'kishore@email.com', '+91-9876543212', 'postpaid', 2200.00, 'us-east-1');
```

### V3__add_audit_trigger_mysql.sql

```sql
-- ============================================================
-- Flyway Migration V3: Audit Trigger — MySQL 8.0+
-- ============================================================

-- Audit log table
CREATE TABLE audit_log (
    log_id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    table_name      VARCHAR(50) NOT NULL,
    action          VARCHAR(10) NOT NULL,
    record_id       VARCHAR(36),
    old_values      JSON,
    new_values      JSON,
    changed_by      VARCHAR(100) DEFAULT (CURRENT_USER()),
    changed_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- Trigger: Log subscriber updates
DELIMITER //
CREATE TRIGGER trg_subscribers_after_update
AFTER UPDATE ON subscribers
FOR EACH ROW
BEGIN
    INSERT INTO audit_log (table_name, action, record_id, old_values, new_values)
    VALUES (
        'subscribers',
        'UPDATE',
        OLD.subscriber_id,
        JSON_OBJECT('balance', OLD.balance, 'status', OLD.status, 'plan_type', OLD.plan_type),
        JSON_OBJECT('balance', NEW.balance, 'status', NEW.status, 'plan_type', NEW.plan_type)
    );
END //
DELIMITER ;
```

---

### Q8. Flyway Commands — Complete Reference

```bash
# ============================================================
# FLYWAY CLI COMMANDS
# ============================================================

# --- MIGRATE: Apply pending migrations ---
flyway -configFiles=conf/flyway.conf migrate
# Applies all pending versioned migrations in order
# Most commonly used command in CI/CD

# --- INFO: Show migration status ---
flyway -configFiles=conf/flyway.conf info
# Shows which migrations have been applied and which are pending
# Output:
# +-----------+---------+---------------------+------+
# | Version   | State   | Description         | Type |
# +-----------+---------+---------------------+------+
# | 1         | Success | create schema       | SQL  |
# | 2         | Success | create tables       | SQL  |
# | 3         | Pending | insert seed data    | SQL  |
# +-----------+---------+---------------------+------+

# --- VALIDATE: Check applied migrations match local files ---
flyway -configFiles=conf/flyway.conf validate
# Ensures nobody modified already-applied SQL files
# Fails if checksums don't match (tamper detection)

# --- REPAIR: Fix failed migrations ---
flyway -configFiles=conf/flyway.conf repair
# Removes failed migration entries from history table
# Realigns checksums
# Use when: a migration failed halfway and you need to retry

# --- CLEAN: DROP everything (DANGEROUS!) ---
flyway -configFiles=conf/flyway.conf clean
# Drops ALL objects in configured schemas
# NEVER use in production! (set cleanDisabled=true)

# --- BASELINE: Mark existing DB as version 1 ---
flyway -configFiles=conf/flyway.conf baseline
# Use when: adopting Flyway on an existing database
# Creates flyway_schema_history with baseline entry
# Future migrations start from the next version

# --- UNDO: Rollback last migration (Pro only) ---
flyway -configFiles=conf/flyway.conf undo
# Runs the corresponding U{version}__ undo script

# ============================================================
# COMMAND WITH INLINE PARAMETERS (no config file)
# ============================================================
flyway -url=jdbc:postgresql://localhost:5432/telecom_db \
       -user=flyway_admin \
       -password=secret \
       -locations=filesystem:./sql \
       migrate

# ============================================================
# DOCKER COMMANDS
# ============================================================
# PostgreSQL migration via Docker
docker run --rm --network host \
  -v $(pwd)/sql:/flyway/sql \
  flyway/flyway:10.15.0 \
  -url=jdbc:postgresql://localhost:5432/telecom_db \
  -user=flyway_admin -password=secret \
  migrate

# MySQL migration via Docker
docker run --rm --network host \
  -v $(pwd)/sql-mysql:/flyway/sql \
  flyway/flyway:10.15.0 \
  -url=jdbc:mysql://localhost:3306/telecom_db \
  -user=flyway_admin -password=secret \
  migrate
```

---
---

## SECTION 3: LIQUIBASE — COMPLETE GUIDE

---

### Q9. What is Liquibase and how does it work?

**Answer:** Liquibase uses a **changelog** file (XML, YAML, JSON, or SQL) that contains **changesets**. Each changeset has a unique `id` and `author`. Liquibase tracks applied changesets in the `DATABASECHANGELOG` table.

### How Liquibase Works — Step by Step

```
  LIQUIBASE WORKFLOW:

  1. You create a master changelog file (db.changelog-master.xml)
     that references individual changelog files:
     ├── changelog-v1.0-schema.xml
     ├── changelog-v1.1-tables.xml
     └── changelog-v1.2-data.sql

  2. Each file contains CHANGESETS with unique id + author:
     <changeSet id="1" author="venkat">
         <createTable tableName="subscribers">
             <column name="id" type="UUID">...</column>
         </createTable>
     </changeSet>

  3. You run: liquibase update

  4. Liquibase checks DATABASECHANGELOG table:
     "Changeset 1-venkat already applied. 2-venkat is new."

  5. Liquibase applies only NEW changesets

  6. Records in DATABASECHANGELOG:
     | id | author  | filename          | dateExecuted        | md5sum     |
     | 1  | venkat  | changelog-v1.xml  | 2025-02-23 10:30:00 | abc123...  |
     | 2  | venkat  | changelog-v1.xml  | 2025-02-23 10:30:01 | def456...  | ← NEW
```

---

## Q10. How to install Liquibase? (Step-by-Step)

### Installation on Linux

```bash
# ============================================================
# LIQUIBASE INSTALLATION — Linux
# ============================================================

# Step 1: Download Liquibase
cd /opt
wget https://github.com/liquibase/liquibase/releases/download/v4.28.0/liquibase-4.28.0.tar.gz

# Step 2: Extract
mkdir liquibase && tar -xzf liquibase-4.28.0.tar.gz -C liquibase

# Step 3: Add to PATH
echo 'export PATH=$PATH:/opt/liquibase' >> ~/.bashrc
source ~/.bashrc

# Step 4: Verify
liquibase --version
# Output: Liquibase Community 4.28.0

# Step 5: Download JDBC drivers (REQUIRED!)

# PostgreSQL JDBC driver
wget https://jdbc.postgresql.org/download/postgresql-42.7.3.jar \
  -O /opt/liquibase/lib/postgresql-42.7.3.jar

# MySQL JDBC driver
wget https://repo1.maven.org/maven2/com/mysql/mysql-connector-j/8.4.0/mysql-connector-j-8.4.0.jar \
  -O /opt/liquibase/lib/mysql-connector-j-8.4.0.jar

# Step 6: Create project structure
mkdir -p /opt/liquibase-projects/telecom/{changelogs,sql}

# Directory structure:
# /opt/liquibase-projects/telecom/
# ├── liquibase.properties        ← connection config
# ├── changelogs/
# │   ├── db.changelog-master.xml ← master changelog
# │   ├── v1.0/
# │   │   ├── create-schema.xml
# │   │   └── create-tables.xml
# │   ├── v1.1/
# │   │   └── add-indexes.xml
# │   └── v1.2/
# │       └── seed-data.sql
# └── sql/
#     └── (SQL format changelogs)
```

### Installation via Docker

```bash
docker pull liquibase/liquibase:4.28.0

docker run --rm --network host \
  -v $(pwd)/changelogs:/liquibase/changelog \
  -v $(pwd)/liquibase.properties:/liquibase/liquibase.properties \
  liquibase/liquibase:4.28.0 update
```

---

### Q11. Liquibase Configuration for PostgreSQL and MySQL

### PostgreSQL — liquibase.properties

```properties
# ============================================================
# LIQUIBASE CONFIGURATION — PostgreSQL
# ============================================================
changeLogFile=changelogs/db.changelog-master.xml
url=jdbc:postgresql://pg-server:5432/telecom_db
username=liquibase_admin
password=Str0ng_P@ssw0rd!
driver=org.postgresql.Driver
defaultSchemaName=telecom
liquibase.hub.mode=off
logLevel=info
```

### MySQL — liquibase-mysql.properties

```properties
# ============================================================
# LIQUIBASE CONFIGURATION — MySQL
# ============================================================
changeLogFile=changelogs/db.changelog-master-mysql.xml
url=jdbc:mysql://mysql-server:3306/telecom_db?useSSL=true&serverTimezone=UTC
username=liquibase_admin
password=Str0ng_P@ssw0rd!
driver=com.mysql.cj.jdbc.Driver
defaultSchemaName=telecom_db
liquibase.hub.mode=off
logLevel=info
```

---

## Q12. Liquibase Changelog Files — PostgreSQL

### db.changelog-master.xml (Master File)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.28.xsd">

    <!-- Version 1.0: Schema and Core Tables -->
    <include file="changelogs/v1.0/create-schema.xml"/>
    <include file="changelogs/v1.0/create-tables.xml"/>
    
    <!-- Version 1.1: Indexes and Constraints -->
    <include file="changelogs/v1.1/add-indexes.xml"/>
    
    <!-- Version 1.2: Seed Data (SQL format) -->
    <include file="changelogs/v1.2/seed-data.sql"/>
    
    <!-- Version 1.3: Audit Trail -->
    <include file="changelogs/v1.3/audit-tables.xml"/>

</databaseChangeLog>
```

### v1.0/create-tables.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.28.xsd">

    <!-- ======================================================== -->
    <!-- Changeset 1: Create subscribers table                     -->
    <!-- ======================================================== -->
    <changeSet id="create-subscribers" author="venkat">
        <preConditions onFail="MARK_RAN">
            <not><tableExists tableName="subscribers" schemaName="telecom"/></not>
        </preConditions>
        
        <createTable tableName="subscribers" schemaName="telecom">
            <column name="subscriber_id" type="UUID" defaultValueComputed="uuid_generate_v4()">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="first_name" type="VARCHAR(50)">
                <constraints nullable="false"/>
            </column>
            <column name="last_name" type="VARCHAR(50)">
                <constraints nullable="false"/>
            </column>
            <column name="email" type="VARCHAR(100)">
                <constraints unique="true"/>
            </column>
            <column name="phone" type="VARCHAR(20)">
                <constraints unique="true" nullable="false"/>
            </column>
            <column name="plan_type" type="VARCHAR(20)" defaultValue="prepaid"/>
            <column name="balance" type="NUMERIC(12,2)" defaultValueNumeric="0.00"/>
            <column name="status" type="VARCHAR(20)" defaultValue="active"/>
            <column name="region" type="VARCHAR(30)">
                <constraints nullable="false"/>
            </column>
            <column name="created_at" type="TIMESTAMPTZ" defaultValueComputed="NOW()"/>
            <column name="updated_at" type="TIMESTAMPTZ" defaultValueComputed="NOW()"/>
        </createTable>
        
        <!-- ROLLBACK support -->
        <rollback>
            <dropTable tableName="subscribers" schemaName="telecom"/>
        </rollback>
    </changeSet>

    <!-- ======================================================== -->
    <!-- Changeset 2: Create call_records table                    -->
    <!-- ======================================================== -->
    <changeSet id="create-call-records" author="venkat">
        <createTable tableName="call_records" schemaName="telecom">
            <column name="cdr_id" type="UUID" defaultValueComputed="uuid_generate_v4()">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="subscriber_id" type="UUID">
                <constraints nullable="false" 
                    foreignKeyName="fk_cdr_subscriber"
                    referencedTableName="subscribers"
                    referencedColumnNames="subscriber_id"
                    referencedTableSchemaName="telecom"/>
            </column>
            <column name="call_type" type="VARCHAR(10)">
                <constraints nullable="false"/>
            </column>
            <column name="destination" type="VARCHAR(20)"/>
            <column name="duration_sec" type="INTEGER" defaultValueNumeric="0"/>
            <column name="data_usage_mb" type="NUMERIC(10,2)" defaultValueNumeric="0.00"/>
            <column name="cost" type="NUMERIC(10,2)">
                <constraints nullable="false"/>
            </column>
            <column name="call_timestamp" type="TIMESTAMPTZ" defaultValueComputed="NOW()"/>
        </createTable>
        
        <rollback>
            <dropTable tableName="call_records" schemaName="telecom"/>
        </rollback>
    </changeSet>

    <!-- ======================================================== -->
    <!-- Changeset 3: Create plans table                           -->
    <!-- ======================================================== -->
    <changeSet id="create-plans" author="venkat">
        <createTable tableName="plans" schemaName="telecom">
            <column name="plan_id" type="SERIAL" autoIncrement="true">
                <constraints primaryKey="true"/>
            </column>
            <column name="plan_name" type="VARCHAR(50)">
                <constraints nullable="false"/>
            </column>
            <column name="monthly_cost" type="NUMERIC(10,2)">
                <constraints nullable="false"/>
            </column>
            <column name="data_limit_gb" type="INTEGER"/>
            <column name="voice_minutes" type="INTEGER"/>
            <column name="sms_limit" type="INTEGER"/>
            <column name="is_active" type="BOOLEAN" defaultValueBoolean="true"/>
        </createTable>
        
        <rollback>
            <dropTable tableName="plans" schemaName="telecom"/>
        </rollback>
    </changeSet>

</databaseChangeLog>
```

### v1.2/seed-data.sql (SQL format changeset)

```sql
--liquibase formatted sql

--changeset venkat:insert-plans
--comment: Insert default telecom plans
INSERT INTO telecom.plans (plan_name, monthly_cost, data_limit_gb, voice_minutes, sms_limit) VALUES
    ('Basic',     199.00,   1,   100,  100),
    ('Standard',  499.00,   3,   500,  500),
    ('Premium',   999.00,  10,  1000, 1000),
    ('Unlimited', 1499.00, -1,    -1,   -1);
--rollback DELETE FROM telecom.plans WHERE plan_name IN ('Basic','Standard','Premium','Unlimited');

--changeset venkat:insert-subscribers
--comment: Insert sample subscribers
INSERT INTO telecom.subscribers (first_name, last_name, email, phone, plan_type, balance, region) VALUES
    ('Venkat',  'Sarath',  'venkat@email.com',  '+91-9876543210', 'postpaid', 1500.00, 'ap-south-1'),
    ('Meena',   'Sharma',  'meena@email.com',   '+91-9876543211', 'prepaid',  350.00,  'ap-south-1'),
    ('Kishore', 'Reddy',   'kishore@email.com', '+91-9876543212', 'postpaid', 2200.00, 'us-east-1');
--rollback DELETE FROM telecom.subscribers WHERE email IN ('venkat@email.com','meena@email.com','kishore@email.com');
```

---

### Q13. Liquibase Commands — Complete Reference

```bash
# ============================================================
# LIQUIBASE CLI COMMANDS
# ============================================================

# --- UPDATE: Apply all pending changesets ---
liquibase --defaults-file=liquibase.properties update
# Most common command. Applies all unapplied changesets.

# --- UPDATE-COUNT: Apply next N changesets ---
liquibase --defaults-file=liquibase.properties update-count 2
# Apply only the next 2 pending changesets

# --- UPDATE-SQL: Preview SQL without executing ---
liquibase --defaults-file=liquibase.properties update-sql > preview.sql
# Generates the SQL that WOULD be executed (dry run)
# CRITICAL: Always review this before running on production!

# --- STATUS: Show pending changesets ---
liquibase --defaults-file=liquibase.properties status
# Shows which changesets haven't been applied yet

# --- ROLLBACK-COUNT: Rollback last N changesets ---
liquibase --defaults-file=liquibase.properties rollback-count 1
# Rolls back the last applied changeset

# --- ROLLBACK-TO-DATE: Rollback to a specific date ---
liquibase --defaults-file=liquibase.properties rollback-to-date "2025-02-01 10:00:00"

# --- ROLLBACK (tag): Rollback to a named tag ---
liquibase --defaults-file=liquibase.properties tag v1.0-release
liquibase --defaults-file=liquibase.properties rollback v1.0-release

# --- DIFF: Compare two databases ---
liquibase --defaults-file=liquibase.properties \
  --referenceUrl=jdbc:postgresql://dev-server:5432/telecom_dev \
  --referenceUsername=admin --referencePassword=secret \
  diff
# Shows differences between dev and target databases

# --- GENERATE-CHANGELOG: Reverse-engineer existing DB ---
liquibase --defaults-file=liquibase.properties \
  generate-changelog --changelog-file=generated-changelog.xml
# Creates changelog from existing database schema
# Use when: adopting Liquibase on existing database

# --- VALIDATE: Check changelog for errors ---
liquibase --defaults-file=liquibase.properties validate

# --- HISTORY: Show applied changesets ---
liquibase --defaults-file=liquibase.properties history

# --- CLEAR-CHECKSUMS: Recompute checksums ---
liquibase --defaults-file=liquibase.properties clear-checksums
# Use when: you've legitimately modified an already-applied changeset
```

---
---

# SECTION 4: CI/CD WITH JENKINS — COMPLETE PIPELINE

---

## Q14. What is CI/CD for database changes?

**Answer:**

```
  CI/CD PIPELINE FOR DATABASE CHANGES:

  Developer                  Git                    Jenkins               Databases
  ─────────                  ───                    ───────               ─────────
  1. Write SQL migration  ──► Push to Git        
                                                    2. Jenkins detects
                                                       new commit (webhook)
                                                    
                                                    3. Jenkins runs
                                                       flyway validate
                                                       (check scripts OK)
                                                    
                                                    4. Jenkins runs
                                                       flyway migrate    ──► DEV DB
                                                       on DEV database
                                                    
                                                    5. Jenkins runs        
                                                       automated tests   ──► DEV DB
                                                       (schema validation)
                                                    
                                                    6. If tests pass,
                                                       Jenkins runs
                                                       flyway migrate    ──► QA DB
                                                       on QA database
                                                    
                                                    7. Manual approval
                                                       (DBA reviews)
                                                    
                                                    8. Jenkins runs
                                                       flyway migrate    ──► STAGING DB
                                                       on STAGING
                                                    
                                                    9. Final approval
                                                       (Change Advisory Board)
                                                    
                                                    10. Jenkins runs
                                                        flyway migrate   ──► PRODUCTION DB
                                                        on PRODUCTION
```

---

## Q15. Jenkins Installation and Setup (Step-by-Step)

```bash
# ============================================================
# JENKINS INSTALLATION — Linux (Ubuntu 24)
# ============================================================

# Step 1: Install Java (Jenkins requires Java 17+)
sudo apt update
sudo apt install -y openjdk-17-jdk
java --version

# Step 2: Add Jenkins repository
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

# Step 3: Install Jenkins
sudo apt update
sudo apt install -y jenkins

# Step 4: Start Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo systemctl status jenkins

# Step 5: Get initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

# Step 6: Open browser → http://your-server:8080
# Enter the initial admin password
# Install suggested plugins
# Create admin user

# Step 7: Install required Jenkins plugins
# Manage Jenkins → Manage Plugins → Install:
# - Pipeline
# - Git plugin
# - Credentials Binding
# - Docker Pipeline (if using Docker)
# - Slack Notification (optional)
# - Email Extension (optional)

# Step 8: Install Flyway/Liquibase on Jenkins server
# (same installation steps as before)
# OR use Docker images in pipeline
```

### Jenkins Credentials Setup

```
  Manage Jenkins → Manage Credentials → Global → Add Credentials

  1. DEV Database:
     ID: db-dev-credentials
     Username: flyway_admin
     Password: ********

  2. QA Database:
     ID: db-qa-credentials
     Username: flyway_admin
     Password: ********

  3. STAGING Database:
     ID: db-staging-credentials
     Username: flyway_admin
     Password: ********

  4. PRODUCTION Database:
     ID: db-prod-credentials
     Username: flyway_admin
     Password: ********

  5. Git Repository:
     ID: git-credentials
     Username: jenkins-bot
     Password/Token: ********
```

---

## Q16. Jenkins Pipeline — Flyway with PostgreSQL

### Jenkinsfile (Declarative Pipeline)

```groovy
// ============================================================
// JENKINSFILE — Database Migration Pipeline (Flyway + PostgreSQL)
// ============================================================

pipeline {
    agent any

    environment {
        FLYWAY_VERSION = '10.15.0'
        FLYWAY_HOME = '/opt/flyway'
        SQL_DIR = 'db/migrations/postgresql'
    }

    parameters {
        choice(
            name: 'TARGET_ENV',
            choices: ['dev', 'qa', 'staging', 'production'],
            description: 'Target environment for database migration'
        )
        booleanParam(
            name: 'DRY_RUN',
            defaultValue: true,
            description: 'Preview SQL only (do not execute)'
        )
    }

    stages {

        // ========================================
        // STAGE 1: Checkout Code from Git
        // ========================================
        stage('Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'git-credentials',
                    url: 'https://github.com/company/telecom-db-migrations.git'
                echo "Checked out migration scripts from Git"
            }
        }

        // ========================================
        // STAGE 2: Validate Migration Scripts
        // ========================================
        stage('Validate') {
            steps {
                script {
                    def dbUrl = getDbUrl(params.TARGET_ENV)
                    def credId = getCredId(params.TARGET_ENV)

                    withCredentials([usernamePassword(
                        credentialsId: credId,
                        usernameVariable: 'DB_USER',
                        passwordVariable: 'DB_PASS'
                    )]) {
                        sh """
                            ${FLYWAY_HOME}/flyway \
                                -url='${dbUrl}' \
                                -user='${DB_USER}' \
                                -password='${DB_PASS}' \
                                -locations='filesystem:${SQL_DIR}' \
                                -validateOnMigrate=true \
                                validate
                        """
                    }
                }
                echo "Migration scripts validated successfully"
            }
        }

        // ========================================
        // STAGE 3: Show Migration Info
        // ========================================
        stage('Info') {
            steps {
                script {
                    def dbUrl = getDbUrl(params.TARGET_ENV)
                    def credId = getCredId(params.TARGET_ENV)

                    withCredentials([usernamePassword(
                        credentialsId: credId,
                        usernameVariable: 'DB_USER',
                        passwordVariable: 'DB_PASS'
                    )]) {
                        sh """
                            ${FLYWAY_HOME}/flyway \
                                -url='${dbUrl}' \
                                -user='${DB_USER}' \
                                -password='${DB_PASS}' \
                                -locations='filesystem:${SQL_DIR}' \
                                info
                        """
                    }
                }
            }
        }

        // ========================================
        // STAGE 4: Dry Run (Preview SQL)
        // ========================================
        stage('Dry Run') {
            when { expression { params.DRY_RUN == true } }
            steps {
                script {
                    def dbUrl = getDbUrl(params.TARGET_ENV)
                    def credId = getCredId(params.TARGET_ENV)

                    withCredentials([usernamePassword(
                        credentialsId: credId,
                        usernameVariable: 'DB_USER',
                        passwordVariable: 'DB_PASS'
                    )]) {
                        sh """
                            ${FLYWAY_HOME}/flyway \
                                -url='${dbUrl}' \
                                -user='${DB_USER}' \
                                -password='${DB_PASS}' \
                                -locations='filesystem:${SQL_DIR}' \
                                migrate -dryRunOutput=dryrun.sql
                        """
                    }
                }
                archiveArtifacts artifacts: 'dryrun.sql', fingerprint: true
                echo "Dry run SQL saved. Review before proceeding."
            }
        }

        // ========================================
        // STAGE 5: Approval Gate (STAGING + PROD)
        // ========================================
        stage('Approval') {
            when {
                expression {
                    params.TARGET_ENV in ['staging', 'production'] && !params.DRY_RUN
                }
            }
            steps {
                script {
                    def approver = params.TARGET_ENV == 'production' ? 'DBA Lead + CAB' : 'DBA Lead'
                    input message: "Deploy to ${params.TARGET_ENV}?",
                          ok: "Approve & Deploy",
                          submitter: "dba-team,devops-lead"
                }
            }
        }

        // ========================================
        // STAGE 6: Execute Migration
        // ========================================
        stage('Migrate') {
            when { expression { params.DRY_RUN == false } }
            steps {
                script {
                    def dbUrl = getDbUrl(params.TARGET_ENV)
                    def credId = getCredId(params.TARGET_ENV)

                    withCredentials([usernamePassword(
                        credentialsId: credId,
                        usernameVariable: 'DB_USER',
                        passwordVariable: 'DB_PASS'
                    )]) {
                        sh """
                            echo "Starting migration on ${params.TARGET_ENV}..."
                            ${FLYWAY_HOME}/flyway \
                                -url='${dbUrl}' \
                                -user='${DB_USER}' \
                                -password='${DB_PASS}' \
                                -locations='filesystem:${SQL_DIR}' \
                                -baselineOnMigrate=true \
                                -cleanDisabled=true \
                                migrate
                            echo "Migration completed successfully!"
                        """
                    }
                }
            }
        }

        // ========================================
        // STAGE 7: Post-Migration Verification
        // ========================================
        stage('Verify') {
            when { expression { params.DRY_RUN == false } }
            steps {
                script {
                    def dbUrl = getDbUrl(params.TARGET_ENV)
                    def credId = getCredId(params.TARGET_ENV)

                    withCredentials([usernamePassword(
                        credentialsId: credId,
                        usernameVariable: 'DB_USER',
                        passwordVariable: 'DB_PASS'
                    )]) {
                        sh """
                            ${FLYWAY_HOME}/flyway \
                                -url='${dbUrl}' \
                                -user='${DB_USER}' \
                                -password='${DB_PASS}' \
                                -locations='filesystem:${SQL_DIR}' \
                                info
                        """
                    }
                }
                echo "Post-migration verification complete"
            }
        }
    }

    post {
        success {
            echo "Database migration pipeline completed successfully for ${params.TARGET_ENV}"
            // slackSend channel: '#dba-alerts', message: "DB Migration SUCCESS on ${params.TARGET_ENV}"
        }
        failure {
            echo "Database migration pipeline FAILED for ${params.TARGET_ENV}"
            // slackSend channel: '#dba-alerts', color: 'danger', message: "DB Migration FAILED on ${params.TARGET_ENV}!"
        }
    }
}

// ============================================================
// HELPER FUNCTIONS
// ============================================================
def getDbUrl(env) {
    switch(env) {
        case 'dev':        return 'jdbc:postgresql://dev-pg:5432/telecom_db'
        case 'qa':         return 'jdbc:postgresql://qa-pg:5432/telecom_db'
        case 'staging':    return 'jdbc:postgresql://staging-pg:5432/telecom_db'
        case 'production': return 'jdbc:postgresql://prod-pg:5432/telecom_db'
    }
}

def getCredId(env) {
    return "db-${env}-credentials"
}
```

---

## Q17. Jenkins Pipeline — Liquibase with MySQL

```groovy
// ============================================================
// JENKINSFILE — Database Migration Pipeline (Liquibase + MySQL)
// ============================================================

pipeline {
    agent any

    environment {
        LIQUIBASE_HOME = '/opt/liquibase'
        CHANGELOG_DIR = 'db/changelogs/mysql'
        CHANGELOG_FILE = 'db.changelog-master.xml'
    }

    parameters {
        choice(name: 'TARGET_ENV', choices: ['dev','qa','staging','production'])
        booleanParam(name: 'DRY_RUN', defaultValue: true)
        booleanParam(name: 'ROLLBACK', defaultValue: false, description: 'Rollback last changeset?')
        string(name: 'ROLLBACK_COUNT', defaultValue: '1', description: 'Number of changesets to rollback')
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-credentials',
                    url: 'https://github.com/company/telecom-db-migrations.git'
            }
        }

        stage('Validate') {
            steps {
                script {
                    def props = getProps(params.TARGET_ENV)
                    withCredentials([usernamePassword(credentialsId: props.credId,
                        usernameVariable: 'DB_USER', passwordVariable: 'DB_PASS')]) {
                        sh """
                            ${LIQUIBASE_HOME}/liquibase \
                                --url='${props.url}' \
                                --username='${DB_USER}' \
                                --password='${DB_PASS}' \
                                --changelog-file='${CHANGELOG_DIR}/${CHANGELOG_FILE}' \
                                validate
                        """
                    }
                }
            }
        }

        stage('Status') {
            steps {
                script {
                    def props = getProps(params.TARGET_ENV)
                    withCredentials([usernamePassword(credentialsId: props.credId,
                        usernameVariable: 'DB_USER', passwordVariable: 'DB_PASS')]) {
                        sh """
                            ${LIQUIBASE_HOME}/liquibase \
                                --url='${props.url}' \
                                --username='${DB_USER}' \
                                --password='${DB_PASS}' \
                                --changelog-file='${CHANGELOG_DIR}/${CHANGELOG_FILE}' \
                                status --verbose
                        """
                    }
                }
            }
        }

        stage('Preview SQL') {
            when { expression { params.DRY_RUN } }
            steps {
                script {
                    def props = getProps(params.TARGET_ENV)
                    withCredentials([usernamePassword(credentialsId: props.credId,
                        usernameVariable: 'DB_USER', passwordVariable: 'DB_PASS')]) {
                        sh """
                            ${LIQUIBASE_HOME}/liquibase \
                                --url='${props.url}' \
                                --username='${DB_USER}' \
                                --password='${DB_PASS}' \
                                --changelog-file='${CHANGELOG_DIR}/${CHANGELOG_FILE}' \
                                update-sql > preview-${params.TARGET_ENV}.sql
                        """
                    }
                }
                archiveArtifacts artifacts: "preview-${params.TARGET_ENV}.sql"
            }
        }

        stage('Approval') {
            when { expression { params.TARGET_ENV in ['staging','production'] && !params.DRY_RUN } }
            steps {
                input message: "Deploy to ${params.TARGET_ENV} MySQL?",
                      ok: "Approve", submitter: "dba-team"
            }
        }

        stage('Migrate') {
            when { expression { !params.DRY_RUN && !params.ROLLBACK } }
            steps {
                script {
                    def props = getProps(params.TARGET_ENV)
                    withCredentials([usernamePassword(credentialsId: props.credId,
                        usernameVariable: 'DB_USER', passwordVariable: 'DB_PASS')]) {
                        sh """
                            ${LIQUIBASE_HOME}/liquibase \
                                --url='${props.url}' \
                                --username='${DB_USER}' \
                                --password='${DB_PASS}' \
                                --changelog-file='${CHANGELOG_DIR}/${CHANGELOG_FILE}' \
                                update
                        """
                    }
                }
            }
        }

        stage('Rollback') {
            when { expression { params.ROLLBACK && !params.DRY_RUN } }
            steps {
                script {
                    def props = getProps(params.TARGET_ENV)
                    withCredentials([usernamePassword(credentialsId: props.credId,
                        usernameVariable: 'DB_USER', passwordVariable: 'DB_PASS')]) {
                        sh """
                            ${LIQUIBASE_HOME}/liquibase \
                                --url='${props.url}' \
                                --username='${DB_USER}' \
                                --password='${DB_PASS}' \
                                --changelog-file='${CHANGELOG_DIR}/${CHANGELOG_FILE}' \
                                rollback-count ${params.ROLLBACK_COUNT}
                        """
                    }
                }
            }
        }

        stage('Verify') {
            when { expression { !params.DRY_RUN } }
            steps {
                script {
                    def props = getProps(params.TARGET_ENV)
                    withCredentials([usernamePassword(credentialsId: props.credId,
                        usernameVariable: 'DB_USER', passwordVariable: 'DB_PASS')]) {
                        sh """
                            ${LIQUIBASE_HOME}/liquibase \
                                --url='${props.url}' \
                                --username='${DB_USER}' \
                                --password='${DB_PASS}' \
                                --changelog-file='${CHANGELOG_DIR}/${CHANGELOG_FILE}' \
                                history
                        """
                    }
                }
            }
        }
    }

    post {
        success { echo "Liquibase migration SUCCESS on ${params.TARGET_ENV}" }
        failure { echo "Liquibase migration FAILED on ${params.TARGET_ENV}" }
    }
}

def getProps(env) {
    def urls = [
        dev:        'jdbc:mysql://dev-mysql:3306/telecom_db',
        qa:         'jdbc:mysql://qa-mysql:3306/telecom_db',
        staging:    'jdbc:mysql://staging-mysql:3306/telecom_db',
        production: 'jdbc:mysql://prod-mysql:3306/telecom_db'
    ]
    return [url: urls[env], credId: "db-${env}-credentials"]
}
```

---

### Q18. Jenkins Pipeline — Docker-Based (No Installation Needed)

```groovy
// ============================================================
// JENKINSFILE — Docker-based (no Flyway/Liquibase installation)
// ============================================================

pipeline {
    agent any
    
    stages {
        stage('Flyway Migrate via Docker') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'db-dev-credentials',
                        usernameVariable: 'DB_USER', passwordVariable: 'DB_PASS')]) {
                        sh """
                            docker run --rm --network host \
                                -v \$(pwd)/db/migrations/postgresql:/flyway/sql \
                                flyway/flyway:10.15.0 \
                                -url='jdbc:postgresql://dev-pg:5432/telecom_db' \
                                -user='${DB_USER}' \
                                -password='${DB_PASS}' \
                                -baselineOnMigrate=true \
                                migrate
                        """
                    }
                }
            }
        }

        stage('Liquibase Update via Docker') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'db-dev-mysql-cred',
                        usernameVariable: 'DB_USER', passwordVariable: 'DB_PASS')]) {
                        sh """
                            docker run --rm --network host \
                                -v \$(pwd)/db/changelogs:/liquibase/changelog \
                                liquibase/liquibase:4.28.0 \
                                --url='jdbc:mysql://dev-mysql:3306/telecom_db' \
                                --username='${DB_USER}' \
                                --password='${DB_PASS}' \
                                --changelog-file='changelog/db.changelog-master.xml' \
                                update
                        """
                    }
                }
            }
        }
    }
}
```

---
---

## SECTION 5: INTERVIEW QUESTIONS — ADVANCED

---

### Q19. How do you handle rollback in Flyway vs Liquibase?

**Answer:**

| Scenario | Flyway | Liquibase |
|----------|--------|-----------|
| Rollback support | Pro/Enterprise only | Free (Community) |
| Rollback method | U{version}__.sql undo files | `<rollback>` tag inside changeset |
| Rollback command | `flyway undo` | `liquibase rollback-count 1` |
| Rollback to tag | Not available (Community) | `liquibase rollback v1.0-release` |
| Rollback to date | Not available | `liquibase rollback-to-date "2025-02-01"` |
| Best practice | Write forward-only migrations (add column, never drop) | Define rollback blocks in every changeset |

---

### Q20. What happens if a migration fails halfway?

**Answer:**

**PostgreSQL:** Migrations are wrapped in a transaction by default. If a migration fails, the entire migration is rolled back. The `flyway_schema_history` / `DATABASECHANGELOG` entry is NOT created. You fix the script and re-run.

**MySQL:** DDL statements (CREATE TABLE, ALTER TABLE) cause an implicit COMMIT in MySQL. So if a migration has 3 DDL statements and fails on the 3rd, the first 2 are already committed. You need to manually clean up OR use `flyway repair` / `liquibase clear-checksums` to fix the history table.

**Best Practice:** For MySQL, keep each DDL change in a SEPARATE migration file. One migration = one DDL statement.

---

### Q21. How do you adopt Flyway/Liquibase on an existing database?

**Answer:**

```bash
# FLYWAY: Baseline an existing database
# This creates the flyway_schema_history table and marks
# the current state as "version 1" (baseline)
flyway -url=jdbc:postgresql://prod:5432/telecom_db \
  -user=admin -password=secret \
  -baselineVersion=1 \
  -baselineDescription="Existing production schema" \
  baseline

# All future migrations must be V2 or higher
# V1 is considered "already applied" (baseline)

# LIQUIBASE: Generate changelog from existing database
liquibase --url=jdbc:postgresql://prod:5432/telecom_db \
  --username=admin --password=secret \
  generate-changelog --changelog-file=existing-schema.xml

# Then mark all existing changesets as already run:
liquibase changelog-sync
```

---

### Q22. Git Repository Structure for Database Migrations

```
  telecom-db-migrations/
  ├── README.md
  ├── Jenkinsfile                          ← CI/CD pipeline
  │
  ├── db/
  │   ├── migrations/
  │   │   ├── postgresql/                  ← Flyway SQL files for PostgreSQL
  │   │   │   ├── V1__create_schema.sql
  │   │   │   ├── V2__create_tables.sql
  │   │   │   ├── V3__seed_data.sql
  │   │   │   ├── V4__add_audit_columns.sql
  │   │   │   ├── V5__create_partitions.sql
  │   │   │   └── R__refresh_views.sql     ← Repeatable
  │   │   │
  │   │   └── mysql/                       ← Flyway SQL files for MySQL
  │   │       ├── V1__create_tables.sql
  │   │       ├── V2__seed_data.sql
  │   │       └── V3__add_triggers.sql
  │   │
  │   ├── changelogs/                      ← Liquibase changelogs
  │   │   ├── db.changelog-master.xml
  │   │   ├── v1.0/
  │   │   │   ├── create-schema.xml
  │   │   │   └── create-tables.xml
  │   │   ├── v1.1/
  │   │   │   └── add-indexes.xml
  │   │   └── v1.2/
  │   │       └── seed-data.sql
  │   │
  │   └── rollback/                        ← Emergency rollback scripts
  │       ├── rollback-v5.sql
  │       └── rollback-v4.sql
  │
  ├── conf/
  │   ├── flyway-dev.conf
  │   ├── flyway-qa.conf
  │   ├── flyway-staging.conf
  │   └── flyway-prod.conf                ← password from Jenkins credentials
  │
  └── tests/
      ├── validate-schema.sql              ← Post-migration validation queries
      └── smoke-test.sql                   ← Quick data verification
```

---

### Q23. Best Practices for Database CI/CD

1. **Never modify already-applied migrations** — Create new migration files instead
2. **One DDL per migration file** (especially for MySQL) — Easier to troubleshoot failures
3. **Always include rollback** — Liquibase: `<rollback>` tags. Flyway: separate rollback scripts
4. **Use DRY RUN first** — `flyway migrate -dryRun` or `liquibase update-sql` before production
5. **Approval gates for staging/production** — Manual approval in Jenkins pipeline
6. **Separate credentials per environment** — Never hardcode passwords in config files
7. **Tag releases** — `liquibase tag v2.0` or Git tags before production deployment
8. **Validate before migrate** — `flyway validate` catches tampered scripts
9. **Test on a copy first** — Restore prod backup → run migration → verify → then deploy to prod
10. **Keep migrations small and focused** — One logical change per migration file
11. **Use repeatable migrations for views/functions** — `R__` prefix in Flyway
12. **Monitor migration duration** — Large ALTER TABLE can lock tables; plan maintenance windows

---

*Liquibase, Flyway & CI/CD Jenkins — Complete Interview Guide*
*Database Change Management for PostgreSQL & MySQL*
