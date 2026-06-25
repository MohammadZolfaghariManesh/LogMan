# Log Man

A production Java application built for a secure, air-gapped enterprise environment. It collects, filters, and migrates audit logs from SQL Server into MongoDB — running automatically on a schedule with no manual intervention.

---

## The Problem

The client ran two versions of their software simultaneously (a 2017 and a 2022 build), each with a different database structure. Both systems generated large volumes of raw audit logs in SQL Server, but only a small subset of those logs were meaningful — for example, out of dozens of entries that might exist for a single user registration flow, only the final "user created successfully" event actually mattered.

The goal was to extract only those meaningful events, enrich them with additional metadata, and store them as structured documents in MongoDB — reliably, automatically, and without ever touching the JAR file after deployment.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                   conf_file.txt                     │
│  (controls everything: sources, methods, IPs, etc.) │
└───────────────────────┬─────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│                  Main JAR (LogMan)                  │
│                                                     │
│  1. Read conf_file.txt via SettingWrapper           │
│  2. Check / create lock file                        │
│  3. Connect to SQL Server via JDBC                  │
│  4. Run selected query methods (2017 / 2022 / both) │
│  5. Filter & map results to LogObject POJOs         │
│  6. Insert into MongoDB (sync or async)             │
│  7. Optionally delete source records from SQL       │
│  8. Delete lock file on clean exit                  │
└──────────┬──────────────────────────────────────────┘
           │                        │
           ▼                        ▼
   SQL Server (2017)         SQL Server (2022)
   SQL Server (2024)
           │
           └──────────────► MongoDB Collection
```

---

## Key Design Decisions

### 1. Config-File Driven Behavior

The JAR itself never needs to be modified or redeployed for routine changes. A plain text `conf_file.txt` controls:

- Which SQL Server version to read from (2017, 2022, or both)
- Which log types to process (e.g. create user, modify user, delete user)
- Whether to delete source records after successful migration
- IP addresses and ports for both SQL Server and MongoDB
- Sync vs async insertion mode
- Query batch size and other runtime parameters

This made it easy for operators to adjust behavior in a restricted environment where redeploying a JAR was a significant process.

### 2. Intelligent Log Filtering

Raw SQL logs contain many intermediate entries for any given operation. The application queries only for records matching specific success criteria per log type — so instead of retrieving every row related to a user creation and then filtering in memory, the SQL queries themselves are targeted, keeping data transfer minimal.

### 3. Dual-Version SQL Support

The client ran two structurally different database versions side by side. Each query method has two implementations — one per schema version — selected at runtime based on the `appVersion` value in the config file. Both versions can be processed in the same run if needed.

### 4. POJO Remapping for MongoDB

Source SQL records contain raw data with field names and structures that don't match the required MongoDB document schema. Each log type is mapped into a `LogObject` POJO with:
- Renamed fields to match the target schema
- Additional metadata fields not present in SQL (org ID, department ID, app server IP, sensitivity level, action flags, timestamps)
- Normalized action type and subtype classifications

### 5. Sync & Async Insertion

The application supports two MongoDB insertion modes, selected via config:

- **Sync** — inserts are blocking, results are confirmed before moving on. Safer for smaller batches or when confirmation matters.
- **Async** — non-blocking insertion for higher throughput when processing large volumes.

Both modes are implemented as separate manager classes and injected via constructor overloading into the main process class.

### 6. Lock File Mechanism

Because the JAR runs every minute via Windows Task Scheduler, it needed a way to prevent overlapping executions — and to detect crashes.

**How it works:**
- On startup, the JAR looks for a `lockfile.txt` in its directory
- If none exists → it creates one, writes the current epoch timestamp, and proceeds
- If one exists → it reads the timestamp and calculates elapsed time
  - If elapsed time is **under the threshold** (e.g. 3 days) → another instance is likely running, so the JAR exits immediately
  - If elapsed time is **over the threshold** → the previous run likely crashed, the lock file is stale, so it is deleted and execution continues
- On clean exit, the lock file is deleted automatically
- On crash, the lock file remains — an operator must investigate and delete it manually before the next run resumes

This approach requires no external process manager or database — just the filesystem.

### 7. Credential Encryption

Database credentials are never stored in plain text. The application uses AES encryption:

- A separate admin JAR (with its own `main` class) accepts a plain-text password and outputs the encrypted string
- Admins run this once via a dedicated batch script and store the encrypted value in the config file
- The main JAR decrypts credentials at runtime before establishing connections
- The encryption key is compiled into the application and not exposed in config files

### 8. Batch Script Automation

Three batch scripts handle the full operational lifecycle:

| Script | Purpose |
|--------|---------|
| `run.bat` | Launches the JAR with the correct classpath (external libs) and passes the config file path as an argument |
| `schedule.bat` | Registers `run.bat` as a Windows Task Scheduler task running every 1 minute |
| `encrypt.bat` | Runs the admin JAR to encrypt credentials for use in the config file |

### 9. Obfuscation

The final JAR was obfuscated before delivery to the client to protect proprietary business logic and prevent reverse engineering of the implementation.

---

## Tech Stack

| Layer | Technology |
|-------|------------|
| Language | Java (pure, no framework) |
| SQL connectivity | JDBC |
| MongoDB connectivity | MongoDB Java Driver (sync & async) |
| Encryption | AES/ECB/PKCS5Padding via `javax.crypto` |
| Logging | Log4j2 (with external XML config) |
| Build | JAR with external dependency classpath |
| Deployment | Batch scripts + Windows Task Scheduler |
| Source control | Git + Azure DevOps |

---

## Why Pure Java (No Spring)?

This project was developed and deployed in a secure, air-gapped enterprise environment with no direct internet access on production machines. Keeping the dependency footprint minimal was a deliberate choice — pure Java with JDBC and the MongoDB driver meant fewer moving parts, easier deployment, and no framework overhead in a restricted environment.

---

## Companion Project

**[Log Forwarder](link-to-repo)** — reads the migrated documents from MongoDB and exports them to structured text files via socket-based communication, then removes the exported documents. Shares the same configuration, encryption, locking, and automation patterns as Log Man.

---

## Note on Source Code

This project was built for and deployed in a production enterprise environment. The source code is proprietary and not publicly available. This repository documents the architecture and design for portfolio purposes.
