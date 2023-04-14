---
title: Non-Relational Data Stores (NoSQL)
date: 2022-11-14 23:00:45
tags: [EN, UCSB, CS291A, Database]
categories: Database
---

# Motivation

## Application Growth 

As our application experiences greater and greater popularity, the data layer proves difficult to scale horizontally. Without a scaling path for our data layer, our growing data set and/or our usage of the database will become a bottleneck limiting our application.

Relational Databases are great tools for our data layer. Unfortunately, there is no simple way to spread load across multiple RDBMSes. We've looked at a few techniques for scaling RDBMSes:

- Distinguishing Reads from Writes
- Service Oriented Architectures
- Sharding

## RDBMS Scaling Limitations

What if these techniques aren't sufficient for our application?

- We are already using read-replicas, and it's not enough.

- We've already broken our application out via SOA, and still have load hotspots.

- There's no good way to shard our application.

When relational databases fail to scale to our needs, we need to turn to non-relational solutions.

# NoSQL

Non-relational databases are often referred to as NoSQL databases. This is an umbrella term for many types of databases:

- Key-value stores
- Column-oriented data stores
- Document-oriented stores
- Graph databases
