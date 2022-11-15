---
title: Relational Database Management System (RDBMS) Scaling - I
date: 2022-11-14 17:48:10
tags: [EN, UCSB, CS291A, Database]
categories: Database
---

# Motivation

You have your web application running on EC2. It is becoming increasingly popular and performance is degrading. *What do you do?*

## Handling Concurrent Requests

You have deployed application servers that can serve many concurrent requests. But as your site's popularity continues to grow that is not sufficient.

![Concurrent Requests](https://yeyuan.pro/img/202211142225820.png)

## Vertical Scaling

![Vertical Scaling](https://yeyuan.pro/img/202211142225821.png)

You have increased your instance sizes and handled more load. However, as the popularity continues to grow you are no longer able to continue scaling vertically.

## Horizontal Scaling

You have introduced a load balancer that distributes traffic across a pool of application servers. Nevertheless, as the traffic continues to increase, additional horizontal scaling of the application servers does not solve the problem.

![Horizontal Scaling](https://yeyuan.pro/img/202211142225822.png)

## Caching

You have properly configured HTTP caching such that unnecessary requests never happen. You are also caching heavyweight database operations. But it is still not enough to handle the increased load of your popular site.

## SQL Structure and Query Optimization

You have reduced the number of queries your application servers makes to the database. You have used `EXPLAIN` to discover indexes to add, and to ensure your most common SQL queries are as optimal as possible. However, your database is still a bottleneck.

## Non-trivial DB Horizontal Scaling

> Can we horizontally scale the database by adding more database servers and accessing them through a load balancer?

Unfortunately, it's not that simple. Horizontal scaling works great for `stateless` services. However, the database contains the state of our application, thus is not trivial to horizontally scale.

Note: Horizontally scaling databases in this way works fine for read-only operations.

![Database Horizontal Scaling Problem](https://yeyuan.pro/img/202211142225823.png)

## Scaling Relational Databases

- Sharding
- Service Oriented Architectures (SOA)
- Separating Reads from Writes

# Partitioning (Sharding)

Take a single database and ***split/partition/shard*** it up into multiple smaller databases such that everything still works.

> How do we handle joins across partitioned data?

## Joins

Any particular database join connects a small part of your database. However, transitively, database joins could connect everything together.

> **E.g. Demo App:** 
>
> - Any comment is only related to its parent (if not top-level), its children (replies), and its submission.
> - Submissions relate to each other through communities.
> - Transitively, all of these relationships could be joined across.

## Separating Data

Find a separation of your data that ideally produces unrelated (not joined across) *partitions*. Once separated, your application cannot utilize the database to join across partitions. If you need to perform operations across sharded data, you will need to do it at the application level. Consider the performance trade-offs. Could you partition another way?

![Sharding](https://yeyuan.pro/img/202211142225824.png)

## Similar Data

Partitioning involves splitting data of the same type (e.g., the rows of the tables).

For instance if we wanted to split our `Comments` table into two partitions, we could store comments belonging to half the submissions in *partition1*, and those belonging to the other half in *partition2*.

> What is not partitioning?

Separating tables into their own databases is not partitioning. While this approach may work for some applications, the ability to join across tables is lost.

> How can we find what partition our data is on?

We need some sort of mapping to determine where to find that data.

### At the App Server

Each application server contains a configuration that informs it of where each database is (IP address, DNS name) and how to map data to the database. The mapping can be arbitrarily complex. The mapping itself may even be stored in a database.

![App Server Sharding](https://yeyuan.pro/img/202211142225825.png)

### At the load balancer

The load balancer could be configured to route requests to the app servers that are configured to *talk* to the right database. Such mappings are limited by knowledge that the load balancer can inspect:

- Resource URI
- Headers
- Request Payload

![Load Balancer Sharding](https://yeyuan.pro/img/202211142225826.png)

### Across Load Balancers

Host names (DNS) can be configured to point to the correct load balancer for a given request.

Examples:

- en.wikipedia.org vs. es.wikipedia.org (language based sharding)
- google.com vs. google.co.uk (location based sharding)
- na6.salesforce.com vs. naX.salesforce.com (customer based sharding)

**Note**: The above examples could involve only a single load balancer.

![Sharding Across Load Balancers](https://yeyuan.pro/img/202211142225827.png)

### Trade-offs

The approaches we just described are vary from providing more flexibility to providing more scalability.

- App Server (most flexible)
- Load Balancer
- DNS (most scalable)

## Partitioning and Growth

Ideally the number of partitions increase as the usage of your application increases.

> **Examples**:
>
> - If each customer's data can be partitioned from the others, then doubling the number of customers doubles the number of partitions.
> - The data that represent one user's email conceptually requires no relation to the data representing other users' email. When a request arrives associated with a particular user, the server applies some mapping function to determine which database the user's data are located in. Should the email provider need to take down a database, they can relocate the partitioned data to another database, and update the mapping with little disruption.

## Partitioning Demo App

### Settings

- Users can create and view communities.
- Users can create submissions in these communities.
- Each submission has a tree of comments.

> How can we partition this application?

### By User?

This would have difficulty as logged in users will want to see communities, submissions, and comments made by other logged in users.

### By Submission?

This would make generating submission lists for a single community difficult.

### By Community?

#### Methodology

Obtaining a community list will be more difficult, but cleanly partitioning comments and submissions by their community is doable.

> How could we make community based paritioning work?

We can use information in the url with any of the load balancing approaches.

- [http://ucsb.demo.com](http://ucsb.demo.com/) (community sub-domain)
- http://demo.com/ucsb (community path)

Either the application server connects to the right database for the `ucsb` community, or DNS/loadbalancer directs the request to an application server that always talks to the `ucsb` containing database.

#### Difficulty

- The global list of submissions.
- List of submissions by user.
- List of comments by user.

> What can we do to resolve these issues?

![Sharding Global Submission View](https://yeyuan.pro/img/202211142225828.png)

#### Solving Partitioning Problems

- Modify the user interface such that the difficult to partition page does not exist. only providing the list of communities
- Alternatively, periodically run an expensive background job to keep a semi-up-to-date global submission list aggregating results from across databases.

## Partitioning in Rails

Rails 6+ has built-in support for partitioning: https://guides.rubyonrails.org/active_record_multiple_databases.html

```ruby
def index
    ...
    ActiveRecord::Base.connected_to(database: :customer1)
        ...
    end
end
```

## Partitioning Trade-offs - Summary

### Strengths

- If you genuinely have zero relations across partitions, this scaling path is very powerful.
- Partitioning works best when partitions grow with usage.

### Weaknesses

- Partitioning can inhibit feature development. That is your application may be perfectly partitionable today, but future features may change that. **(Partitioning is scalable, but not flexible)**
- Not easy to retroactively add partitioning to an existing application.
- Transactions across partitions do not exist.
- Consistent DB snapshots across partitions do not exist.
