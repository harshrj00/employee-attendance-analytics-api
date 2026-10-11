# DECISIONS.md

### 1. Indexes

A unique index on `employees.emp_code` prevents duplicate employee codes. A unique compound index on `attendance_logs (emp_code, date)` enforces one attendance record per employee per day. Other indexes support department and joining-date filters, attendance listing, status filtering, punch-out lookup, and date-based late-minute queries. I did not add a unique email index because the contract guarantees uniqueness for `emp_code`, not email.

### 2. Punch-in race

Two identical requests can reach the insert concurrently. The unique compound index allows only one insert to succeed. The losing request raises `DuplicateKeyError`, which the API converts to HTTP 409. This database constraint, rather than a preliminary check, provides the concurrency guarantee.

### 3. Leaderboard ties

The leaderboard uses MongoDB's `$rank` based on total late minutes. It applies the rank cutoff after calculating ranks, so employees tied at the limit are included. Tied employees are then ordered by employee code.

### 4. Department headcount

The department summary starts with eligible employees who joined by the month's end, then looks up attendance logs. Employees with no matching logs remain in the aggregation and count toward headcount.

### 5. Scaling

For 100 times more data, I would profile query execution plans and consider pre-aggregated daily summaries with a refresh strategy, while retaining raw attendance records for auditing.
