---
title: Relational Databases - II
date: 2022-11-13 16:54:43
tags: [EN, UCSB, CS291A, Database]
categories: Database
---

# Relational Databases - II

## Schedule

|   T1   |   T2   |   T3   |
| :----: | :----: | :----: |
|  R(X)  |        |        |
|  W(X)  |        |        |
| Commit |        |        |
|        |  R(Y)  |        |
|        |  W(Y)  |        |
|        | Commit |        |
|        |        |  R(Z)  |
|        |        |  W(Z)  |
|        |        | Commit |

Describe the execution of `transactions `that run in a database

### Schedule Types

A schedule is *serial* if

1. the transactions are executed non-interleaved

2. one transaction finishes before the next one starts

Two schedules are *conflict equivalent* if

1. they involve the same set of transactions

2. every pair of conflicting actions are ordered in the same way

A schedule is *conflict serializable* if

1. the schedule is **conflict equivalent** to a **serial** schedule

A schedule is *recoverable* if

1. transactions commit only after all transactions whose changes they read, commit

## Conflicting Actions

> Two actions are said to be in conflict if:
>
> - the actions belong to different transactions
> - at least one of the actions is a write operation
> - the actions access the same object (read or write)

eg. Conflicting Actions

|  T1  |  T2  |
| :--: | :--: |
| R(X) |      |
|      | W(X) |

eg. Non-Conflicting - All reads

|  T1  |  T2  |
| :--: | :--: |
| R(X) |      |
|      | R(X) |

eg. Non-Conflicting - Write to different object

|  T1  |  T2  |
| :--: | :--: |
| R(X) |      |
|      | W(Y) |

So we cannot blindly execute transactions ***in parallel***. Because: 

### Dirty Read Problem

A transaction (T2) reads a value written by another transaction (T1) that is later *rolled back*.

The result of the T2 transaction will put the database in an incorrect state.

|     T1     |   T2   |
| :--------: | :----: |
|    W(X)    |        |
|            |  R(X)  |
| **Cancel** |        |
|            |  W(Y)  |
|            | Commit |

### Incorrect Summary Problem

A transaction (T1) computes a summary over the values of all the instances of a `repeated data-item`. While that occurs, another transaction (T2) updates some instances of data-item.

The resulting summary will `not reflect a correct result` for any deterministic order of the transactions (T1 then T2, or T2 then T1).

|   T1   |   T2   |
| :----: | :----: |
| R(X*)  |        |
|        | W(X^n) |
|        | Commit |
| R(X*)  |        |
|  W(Y)  |        |
| Commit |        |

### Not Conflict Serializable

|  T1  |  T2  |
| :--: | :--: |
| R(A) |      |
| W(A) |      |
|      | R(A) |
|      | W(A) |
|      | R(B) |
|      | W(B) |
| R(B) |      |
| W(B) |      |

When transactions are serialized, the schedule on the top is conflict equivalent to neither of:

|  T1  |  T2  |
| :--: | :--: |
| R(A) |      |
| W(A) |      |
| R(B) |      |
| W(B) |      |
|      | R(A) |
|      | W(A) |
|      | R(B) |
|      | W(B) |

or

|  T1  |  T2  |
| :--: | :--: |
|      | R(A) |
|      | W(A) |
|      | R(B) |
|      | W(B) |
| R(A) |      |
| W(A) |      |
| R(B) |      |
| W(B) |      |

## Serializable Schedule v.s. Serial Execution

- Having a **serial schedule** is important for `consistent results`. For example it is good when you are keeping track of your bank balance in the database.

- A **serial execution** of transactions is safe but `slow`.

Most general purpose relational databases default to employing `conflict-serializable` and `recoverable schedules`. In order to implement this, locks are used.

## Locks

- A lock is a system object associated with a shared resource such as a data item, a row, or a page in memory.
- A database lock may need to be acquired by a transaction before accessing the object.
- Locks prevent undesired, incorrect, or inconsistent operations on shared resources by concurrent transactions.

