---
title: Concurrency Control in Rails
date: 2022-11-13 19:16:34
tags: [EN, UCSB, CS291A, RoR]
categories: Ruby
---

# Scenario

![Database connectivity](https://yeyuan.pro/img/202211140013562.png)

We have many application servers running our application. We are using a relational database to ensure that each request observes a consistent view of the database.

# Locks in rails

Rails uses two types of concurrency control

- Optimistic Locking: Assume that database modifications are going to succeed. Throw an exception when they do not.
- Pessimistic Locking: Ensure that database modifications will succeed by explicitly avoiding conflict.

## Optimistic Lock

### Definition

Any ActiveRecord model will automatically utilize optimistic locking if an integer `lock_version` field is added to the object's table. Whenever such an ActiveRecord object is read from the database, that object contains its associated `lock_version`. When an update for the object occurs, Rails compares the object's `lock_version` to the most recent one in the database. If they differ a `StaleObjectException` is thrown. Otherwise, the data is written to the database, and the `lock_version` value is incremented. This optimistic locking is an application-level construct. The database does nothing more than storing the `lock_version` values.

### Example

```ruby
c1 = Person.find(1)
c1.name = "X Æ A-12"

c2 = Person.find(1)
c2.gender = "X-Ash-A-12"

c1.save!  # Succeeds
c2.save!  # throws StaleObjectException
```

### Strengths

- Predictable performance
- Lightweight

### Weaknesses

- Have to write error handling code
- (or) errors will propagate to your users

## Pessimistic Lock

### Definition

Easily done by calling `lock` along with ActiveRecord `find`. Whenever an ActiveRecord object is read from the database with that option an exclusive lock is acquired for the object. While this lock is held, the database prevents others from obtaining the lock, reading from, and writing to the object. The others are blocked until the object is unlocked. Implemented using the `SELECT FOR UPDATE` SQL.

### Example

#### Passing

Request 1

```ruby
transaction do
  person = Person.lock.find(1)
  person.name = "X Æ A-12"
  person.save!  # works fine
end
```

Request 2

```ruby
transaction do
  person = Person.lock.find(1)
  person.name = "X-Ash-A-12"
  person.save!  # works fine
end
```

#### Blocking

Request 1

```ruby
transaction do
  person = Person.lock.find(1)
  person.name = "X Æ A-12"
  my_long_procedure
  person.save!
end
```

Request 2

```ruby
transaction do
  person = Person.lock.find(1)
  person.name = "X-Ash-A-12"
  my_long_procedure
  person.save!
end
```

#### Issue: Deadlock

Request 1

```ruby
transaction do
  family = Family.lock.find(3)
  family.count += 1
  family.save!
  person = Person.lock.find(1)
  person.name = "X Æ A-12"
  person.save!
end
```

Request 2

```ruby
transaction do
  person = Person.lock.find(1)
  person.name = "X-Ash-A-12"
  person.save!
  family = Family.lock.find(3)
  family.count += 1
  family.save!
end
```

### Strengths

- Failed transactions are incredibly rare or nonexistent

### Weaknesses

- Have to worry about deadlocks and avoiding them
- Performance is less predictable

# DB Bottlenecks

By changing the way you interact with the Rails ORM (ActiveRecord) you can significantly improve performance.

## Development Mode in Rails

In *development* mode, Rails will output the SQL it generates and executes to the application server log.

To (temporarily) enable debugging in *production* mode, change `config/environments/production.rb` to contain:

```ruby
config.log_level = :debug
```

## Investigation

```sql
Processing by SubmissionsController#index as HTML
Submission Load (0.5ms)  SELECT `submissions`.* FROM `submissions`
Community Load (0.3ms)  SELECT  `communities`.* FROM `communities` WHERE `communities`.`id` = 1 LIMIT 1
SELECT COUNT(*) FROM `comments` WHERE `comments`.`submission_id` = 1
SELECT  `communities`.* FROM `communities` WHERE `communities`.`id` = 1 LIMIT 1  [["id", 1]]
SELECT COUNT(*) FROM `comments` WHERE `comments`.`submission_id` = 2
SELECT  `communities`.* FROM `communities` WHERE `communities`.`id` = 1 LIMIT 1  [["id", 1]]
SELECT COUNT(*) FROM `comments` WHERE `comments`.`submission_id` = 3
SELECT  `communities`.* FROM `communities` WHERE `communities`.`id` = 1 LIMIT 1  [["id", 1]]
...
SELECT  `communities`.* FROM `communities` WHERE `communities`.`id` = 20 LIMIT 1  [["id", 20]]
SELECT COUNT(*) FROM `comments` WHERE `comments`.`submission_id` = 400
```

>  That is a lot of `SELECT` queries!

We are issuing a ton of `SELECT` queries. The overhead associated with each is slowing us down.

## Issue fewer queries.

### Methodology

- Do not ask for the community each time
- Do not ask for the number of comments each time

### Optimization: Reducing (N+1) Queries in Rails

(**Before**) Without `includes`

```ruby
class SubmissionsController < ApplicationController
  def index
    @submissions = Submission.all
  end
end
```

(**After**) With `includes`

```ruby
class SubmissionsController < ApplicationController
  def index
    @submissions = Submission.includes(:comments)
                             .includes(:community).all
  end
end
```

Result

```sql
Result: ActiveRecord 39.6ms

Submission Load (0.9ms)  SELECT `submissions`.* FROM `submissions`
Comment Load (38.3ms)  SELECT `comments`.* FROM `comments` WHERE `comments`.`submission_id` IN (1, 2,... 399, 400)
Community Load (0.4ms)  SELECT `communities`.* FROM `communities` WHERE `communities`.`id` IN (1, 2, ...19, 20)
```

> https://guides.rubyonrails.org/active_record_querying.html#includes
>
> With `includes`, Active Record ensures that all of the specified associations are loaded using the minimum possible number of queries.
>
> Revisiting the above case using the `includes` method, we could rewrite `Book.limit(10)` to eager load authors:
>
> ```ruby
> books = Book.includes(:author).limit(10)
> 
> books.each do |book|
>   puts book.author.last_name
> end
> ```
>
> The above code will execute just **2** queries, as opposed to the **11** queries from the original case:
>
> ```ruby
> SELECT books.* FROM books LIMIT 10
> SELECT authors.* FROM authors
>   WHERE authors.book_id IN (1,2,3,4,5,6,7,8,9,10)
> ```

## Further optimization: SQL Explanation

Sometimes things are still slow even when the number of queries is minimized. SQL provides an `EXPLAIN` statement that can be used to analyze individual queries.

When a query starts with `EXPLAIN`...

- the query is not actually executed
- the produced output will help us identify potential improvements
- e.g. sequential vs index scan, startup vs total cost

## Optimizations

Three primary ways to optimize SQL queries:

- Add or modify indexes
- Modify the table structure
- Directly optimize the query

### SQL Indexes

An index is a fast, ordered, compact structure (often B-tree) for identifying row locations. When an index is provided on a column that is to be filtered (searching for a particular item), the database is able to quickly find that information. Indexes can exist on a single column, or across multiple columns. Multi-column indexes are useful when filtering on two columns (e.g., CS classes that are not full).

To add an index on the `name` field of the `Product` table, create a migration containing:

```ruby
class AddNameIndexProducts < ActiveRecord::Migration
  def change
    add_index :products, :name
  end
end
```

### Foreign Keys

By default, when dealing with relationships between `ActiveRecord` objects, Rails will validate the constraints in the **application layer**. For example, an `Employee` object should have a `Company` that they work for. Assuming the relationship is defined properly, Rails will enforce that when creating an `Employee`, the associated `Company` exists. Many databases, have built-in support for enforcing such constraints. With rails, one can also take advantage of the database's foreign key support via [add_foreign_key](http://guides.rubyonrails.org/active_record_migrations.html#foreign-keys).

```ruby
class AddForeignKeyToOrders < ActiveRecord::Migration
  def change
    add_foreign_key :employees, :companies
  end
end
```

### Optimize the Table Structure

Indexes work best when they can be kept in **memory**. Sometimes changing the field type, or index length can provide significant memory savings.

If appropriate some options are:

- Reduce the length of a VARCHAR index if appropriate
- Use a smaller unsigned integer type
- Use an integer or enum field for statuses rather than a text-based value

### Directly Optimize the Query

Before

```sql
explain select count(*) from txns where parent_id - 1600 = 16340;

select_type: SIMPLE
table: txns
type: index
key: index_txns_on_reverse_txn_id
rows: 439186
Extra: Using where; Using index
```

After

```sql
explain select count(*) from txns where parent_id = 16340 + 1600

select_type: SIMPLE
table: txns
type: const
key: index_txns_on_reverse_txn_id
rows: 1
Extra: Using index
```
