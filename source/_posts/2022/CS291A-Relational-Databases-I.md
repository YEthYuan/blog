---
title: Relational Databases - I
date: 2022-11-13 16:10:27
tags: [EN, UCSB, CS291A, Database]
categories: Database
---

# Relational Databases - I

## SQL vs noSQL

### Relational (SQL)

**Examples:** 

- MySQL (MariaDB, Percona)
- PostgreSQL
- Oracle
- MS SQL

**Features:** 

- are a general-purpose persistence layer
- offer more features
- have a limited ability to scale horizontally

### Non-relational (NoSQL)

**Examples:** 

- Cassandra
- MongoDB
- Redis

**Features:** 

- often are more specialized
- require more from the application layer
- are better at scaling horizontally

> If your needs fit within an RDBMS's (relational database management system) ability to scale, they tend to be the best. If your scaling needs exceed RDBMS capabilities, go non-relational.

## Database Transactions

- Transactions are a database concept that allows a system to guarantee certain `semantic properties`.

- Transactions provide `control over concurrency`.

- Having rigorously defined guarantees means we can build `correct systems` using these databases.

## Database ACID Properties

**Atomicity**

- Complete everything, or do nothing
- No partial application of a transaction

**Consistency**

- The database should be consistent both at the beginning and end of a transaction
- Consistency is defined by the integrity constraints (e.g., foreign keys, `NOT NULL`, etc.)

**Isolation**

- A transaction should not see the effects of other uncommitted transactions

**Durability**

- Once committed, the transaction's effects should not disappear

> ***Overlapping Concerns:***
>
> **Atomicity** and **Durability** are related and are generally provided by *journaling*.
>
> **Consistency** and **Isolation** are provided by concurrency control (usually implemented via locking).

