# Comprehensive OLTP (Online Transaction Processing) Curriculum

## Module 1: Foundations of Transactional Systems
*   **Introduction to OLTP:** Core definitions, business importance, and real-world use cases (e-commerce, banking, inventory).
*   **OLTP vs. OLAP:** Deep dive into architectural differences, workload characteristics, row-oriented vs. column-oriented databases, and the ETL/ELT pipeline.
*   **Data Modeling for OLTP:** Normalization principles (1NF, 2NF, 3NF, BCNF) to reduce redundancy, ER diagramming, and schema design for rapid writes.

## Module 2: The ACID Model & Transaction Management
*   **Atomicity:** The "all-or-nothing" rule, rollback mechanisms, and write-ahead logging (WAL).
*   **Consistency:** System invariants, schema constraints, triggers, and maintaining data validity.
*   **Isolation:** The schedule of transactions, concurrency anomalies (Dirty Reads, Non-repeatable Reads, Phantom Reads).
*   **Durability:** Non-volatile storage, crash recovery, checkpoints, and buffer pool management.

## Module 3: Concurrency Control Mechanics
*   **Locking Protocol Mechanisms:** Shared (S) vs. Exclusive (X) locks, Intent locks, and Two-Phase Locking (2PL / Strict 2PL).
*   **Deadlocks:** Detection (Wait-For Graphs), prevention protocols, and resolution strategies (timeouts, victim selection).
*   **Snapshot Isolation & MVCC:** Multi-Version Concurrency Control internals, reader-writer non-blocking dynamics, and vacuuming/garbage collection.

## Module 4: Performance Tuning & Indexing
*   **B-Tree and B+ Tree Indexing:** Internal structure, search performance, and clustered vs. non-clustered indexes.
*   **Query Optimization:** How the parser, rewriter, and optimizer execute plans; analyzing `EXPLAIN` paths.
*   **Tuning Strategies:** Identifying slow queries, connection pooling, indexing strategies for compound keys, and minimizing lock contention.

## Module 5: High Availability, Scalability, & Distributed OLTP
*   **Replication Architecture:** Single-Primary vs. Multi-Primary, synchronous vs. asynchronous replication, and read-replica offloading.
*   **Sharding & Horizontal Scaling:** Partitioning strategies (range, hash, list), data distribution, and routing layers.
*   **Distributed Transactions:** The Two-Phase Commit (2PC) protocol, the CAP Theorem, PACELC theorem, and modern NewSQL database systems.
