# **UML / ER TO RELATIONAL SCHEMA MAPPING**  
Date: 26.02.2026  

# Relationship Mapping

## 1. CREATES (USER – CONTENT)

* **Type:** 1 : M
* **Rule:** Add foreign key in child table
* `CONTENT.user_id → USER(user_id)`
* `NOT NULL` (Total participation of CONTENT)

---

## 2. SUBMITS (USER – REPORT)

* **Type:** 1 : M
* **Rule:** Add foreign key in child table
* `REPORT.user_id → USER(user_id)`
* `NOT NULL`

---

## 3. IS_REPORTED_IN (CONTENT – REPORT)

* **Type:** 1 : M
* **Rule:** Add foreign key in child table
* `REPORT.content_id → CONTENT(content_id)`
* `NOT NULL`

---

## 4. IS_FLAGGED_AS (CONTENT – CONTENT_FLAG)

* **Type:** 1 : M (Identifying Relationship – Weak Entity)
* **Rule:** Composite Primary Key including foreign key

`CONTENT_FLAG` primary key:

```
(content_id, flag_id)
```

* `content_id → CONTENT(content_id)`

---

## 5. CONTRIBUTES_TO (REPORT – CONTENT_FLAG)

* **Type:** N : 1
* **Rule:** Add foreign key in N-side (REPORT)

Added in `REPORT`:

```
(flag_content_id, flag_id)
```

* References `CONTENT_FLAG(content_id, flag_id)`
* Nullable (Partial participation)

---

## 6. LEADS_TO (CONTENT_FLAG – VIOLATION)

* **Type:** 1 : M (Identifying – Weak Entity)
* **Rule:** Composite Primary Key

`VIOLATION` primary key:

```
(content_id, flag_id, violation_id)
```

* `(content_id, flag_id) → CONTENT_FLAG`

---

## 7. COMMITS (USER – VIOLATION)

* **Type:** 1 : M
* **Rule:** Add foreign key in child table

`VIOLATION.user_id → USER(user_id)`

* `NOT NULL`

---

## 8. HAS (USER – TRUST_SCORE)

* **Type:** 1 : 1 (Total, Total)
* **Rule:** Primary Key as Foreign Key

`TRUST_SCORE.user_id`

* Primary Key
* Foreign Key → `USER(user_id)`

Ensures exactly one trust score per user.

---

## 9. LOGS (AUDIT_LOG – System Entities)

* **Type:** Conceptual N : 1
* Polymorphic reference

Fields used:

```
reference_id
reference_type
```

No direct foreign key constraint (enforced at application level).

---

# Derived Attribute Mapping

## CONTENT.is_flagged (Derived)

Not stored physically.

Computed as:

```sql
EXISTS (
    SELECT 1 
    FROM CONTENT_FLAG
    WHERE CONTENT_FLAG.content_id = CONTENT.content_id
    AND resolved = FALSE
);
```

---

# Complete Relational Schema (SQL DDL)

---

## 1. USER

```sql
CREATE TABLE USER (
    user_id BIGINT PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(100) NOT NULL UNIQUE,
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    account_created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CHECK (status IN ('active','suspended','banned'))
);
```

---

## 2. CONTENT

```sql
CREATE TABLE CONTENT (
    content_id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    content_text TEXT NOT NULL,
    content_type VARCHAR(30) NOT NULL,
    content_hash VARCHAR(128) NOT NULL UNIQUE,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (user_id)
        REFERENCES USER(user_id)
        ON DELETE CASCADE,

    CHECK (content_type IN ('post','comment','review'))
);
```

---

## 3. REPORT

```sql
CREATE TABLE REPORT (
    report_id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    content_id BIGINT NOT NULL,
    report_reason VARCHAR(255) NOT NULL,
    reported_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',

    flag_content_id BIGINT,
    flag_id BIGINT,

    FOREIGN KEY (user_id)
        REFERENCES USER(user_id),

    FOREIGN KEY (content_id)
        REFERENCES CONTENT(content_id)
        ON DELETE CASCADE,

    FOREIGN KEY (flag_content_id, flag_id)
        REFERENCES CONTENT_FLAG(content_id, flag_id),

    CHECK (status IN ('pending','validated','rejected'))
);
```

---

## 4. CONTENT_FLAG

```sql
CREATE TABLE CONTENT_FLAG (
    content_id BIGINT NOT NULL,
    flag_id BIGINT NOT NULL,
    flag_type VARCHAR(50) NOT NULL,
    reason TEXT NOT NULL,
    flagged_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    resolved BOOLEAN NOT NULL DEFAULT FALSE,
    report_count INT NOT NULL DEFAULT 0 CHECK (report_count >= 0),

    PRIMARY KEY (content_id, flag_id),

    FOREIGN KEY (content_id)
        REFERENCES CONTENT(content_id)
        ON DELETE CASCADE
);
```

---

## 5. VIOLATION

```sql
CREATE TABLE VIOLATION (
    content_id BIGINT NOT NULL,
    flag_id BIGINT NOT NULL,
    violation_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    violation_type VARCHAR(50) NOT NULL,
    violation_date DATE NOT NULL,

    PRIMARY KEY (content_id, flag_id, violation_id),

    FOREIGN KEY (content_id, flag_id)
        REFERENCES CONTENT_FLAG(content_id, flag_id)
        ON DELETE CASCADE,

    FOREIGN KEY (user_id)
        REFERENCES USER(user_id),

    CHECK (violation_date <= CURRENT_DATE)
);
```

---

## 6. TRUST_SCORE

```sql
CREATE TABLE TRUST_SCORE (
    user_id BIGINT PRIMARY KEY,
    trust_score DECIMAL(5,2) NOT NULL DEFAULT 100.00,
    violation_count INT NOT NULL DEFAULT 0 CHECK (violation_count >= 0),
    accurate_reports INT NOT NULL DEFAULT 0 CHECK (accurate_reports >= 0),
    last_updated TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (user_id)
        REFERENCES USER(user_id)
        ON DELETE CASCADE
);
```

---

## 7. BANNED_WORD

```sql
CREATE TABLE BANNED_WORD (
    word_id BIGINT PRIMARY KEY,
    keyword VARCHAR(100) NOT NULL UNIQUE,
    severity_level INT NOT NULL CHECK (severity_level BETWEEN 1 AND 5),
    added_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    added_by VARCHAR(50) NOT NULL
);
```

---

## 8. AUDIT_LOG

```sql
CREATE TABLE AUDIT_LOG (
    audit_id BIGINT PRIMARY KEY,
    action_type VARCHAR(50) NOT NULL,
    action_details TEXT NOT NULL,
    action_timestamp TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    reference_id BIGINT NOT NULL,
    reference_type VARCHAR(50) NOT NULL
);
```

---

# Constraints Summary

## NOT NULL

* `USER.username`
* `CONTENT.content_text`
* `REPORT.user_id`
* Other mandatory attributes

---

## UNIQUE

* `USER.email`
* `USER.username`
* `CONTENT.content_hash`
* `BANNED_WORD.keyword`

---

## CHECK

* Valid `status` values
* `severity_level` between 1–5
* `violation_count ≥ 0`
* `violation_date ≤ CURRENT_DATE`

---

## DEFAULT

* `account_created_at`
* `reported_at`
* `resolved = FALSE`
* `trust_score = 100.00`

---
