---
layout: post
title: "Implementing an Order Processing System: Part 3 - Advanced Database Operations"
seo_title: "E-commerce Platform: Advanced Database Operations with sqlc"
seo_description: "Master advanced database operations using sqlc in a Golang-based e-commerce platform, including query optimization and database sharding."
date: 2024-08-03 12:00:00
categories: [Temporal, E-commerce Platform, Database Management]
tags: [Golang, PostgreSQL, sqlc, Database Sharding, Temporal, Query Optimization]
author: Hungai Amuhinda
excerpt: "Implement advanced database operations, including complex queries, database sharding, and ensuring data consistency in a distributed e-commerce system."
permalink: /e-commerce-platform/part-3-advanced-database-operations/
toc: true
comments: true
---


## 1. Introduction and Goals

Welcome to the third installment of our series on implementing a sophisticated order processing system! In our previous posts, we laid the foundation for our project and explored advanced Temporal workflows. Today, we're diving deep into the world of database operations using sqlc, a powerful tool that generates type-safe Go code from SQL.

### Recap of Previous Posts

In Part 1, we set up our project structure, implemented a basic CRUD API, and integrated with a Postgres database. In Part 2, we expanded our use of Temporal, implementing complex workflows, handling long-running processes, and exploring advanced concepts like the Saga pattern.

### Importance of Efficient Database Operations in Microservices

In a microservices architecture, especially one handling complex processes like order management, efficient database operations are crucial. They directly impact the performance, scalability, and reliability of our system. Poor database design or inefficient queries can become bottlenecks, leading to slow response times and poor user experience.

### Overview of sqlc and its Benefits

sqlc is a tool that generates type-safe Go code from SQL. Here are some key benefits:

1. **Type Safety**: sqlc generates Go code that is fully type-safe, catching many errors at compile-time rather than runtime.
2. **Performance**: The generated code is efficient and avoids unnecessary allocations.
3. **SQL-First**: You write standard SQL, which is then translated into Go code. This allows you to leverage the full power of SQL.
4. **Maintainability**: Changes to your schema or queries are immediately reflected in the generated Go code, ensuring your code and database stay in sync.

### Goals for this Part of the Series

By the end of this post, you'll be able to:

1. Implement complex database queries and transactions using sqlc
2. Optimize database performance through efficient indexing and query design
3. Implement batch operations for handling large datasets
4. Manage database migrations in a production environment
5. Implement database sharding for improved scalability
6. Ensure data consistency in a distributed system

Let's dive in!

## 2. Theoretical Background and Concepts

Before we start implementing, let's review some key concepts that will be crucial for our advanced database operations.

### SQL Performance Optimization Techniques

Optimizing SQL performance involves several techniques:

1. **Proper Indexing**: Creating the right indexes can dramatically speed up query execution.
2. **Query Optimization**: Structuring queries efficiently, using appropriate joins, and avoiding unnecessary subqueries.
3. **Data Denormalization**: In some cases, strategically duplicating data can improve read performance.
4. **Partitioning**: Dividing large tables into smaller, more manageable chunks.

### Database Transactions and Isolation Levels

Transactions ensure that a series of database operations are executed as a single unit of work. Isolation levels determine how transaction integrity is visible to other users and systems. Common isolation levels include:

1. **Read Uncommitted**: Lowest isolation level, allows dirty reads.
2. **Read Committed**: Prevents dirty reads, but non-repeatable reads can occur.
3. **Repeatable Read**: Prevents dirty and non-repeatable reads, but phantom reads can occur.
4. **Serializable**: Highest isolation level, prevents all above phenomena.

### Database Sharding and Partitioning

Sharding is a method of horizontally partitioning data across multiple databases. It's a key technique for scaling databases to handle large amounts of data and high traffic loads. Partitioning, on the other hand, is dividing a table into smaller pieces within the same database instance.

### Batch Operations

Batch operations allow us to perform multiple database operations in a single query. This can significantly improve performance when dealing with large datasets by reducing the number of round trips to the database.

### Database Migration Strategies

Database migrations are a way to manage changes to your database schema over time. Effective migration strategies allow you to evolve your schema while minimizing downtime and ensuring data integrity.

Now that we've covered these concepts, let's start implementing advanced database operations in our order processing system.

## 3. Implementing Complex Database Queries and Transactions

Let's start by implementing some complex queries and transactions using sqlc. We'll focus on our order processing system, adding some more advanced querying capabilities.

First, let's update our schema to include a new table for order items:

```sql
-- migrations/000002_add_order_items.up.sql
CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INTEGER NOT NULL REFERENCES orders(id),
    product_id INTEGER NOT NULL,
    quantity INTEGER NOT NULL,
    price DECIMAL(10, 2) NOT NULL
);
```

Now, let's define some complex queries in our sqlc query file:

```sql
-- queries/orders.sql

-- name: GetOrderWithItems :many
SELECT o.*, 
       json_agg(json_build_object(
           'id', oi.id,
           'product_id', oi.product_id,
           'quantity', oi.quantity,
           'price', oi.price
       )) AS items
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
WHERE o.id = $1
GROUP BY o.id;

-- name: CreateOrderWithItems :one
WITH new_order AS (
    INSERT INTO orders (customer_id, status, total_amount)
    VALUES ($1, $2, $3)
    RETURNING id
)
INSERT INTO order_items (order_id, product_id, quantity, price)
SELECT new_order.id, unnest($4::int[]), unnest($5::int[]), unnest($6::decimal[])
FROM new_order
RETURNING (SELECT id FROM new_order);

-- name: UpdateOrderStatus :exec
UPDATE orders
SET status = $2, updated_at = CURRENT_TIMESTAMP
WHERE id = $1;
```

These queries demonstrate some more advanced SQL techniques:

1. `GetOrderWithItems` uses a JOIN and json aggregation to fetch an order with all its items in a single query.
2. `CreateOrderWithItems` uses a CTE (Common Table Expression) and array unnesting to insert an order and its items in a single transaction.
3. `UpdateOrderStatus` is a simple update query, but we'll use it to demonstrate transaction handling.

Now, let's generate our Go code:

```bash
sqlc generate
```

This will create Go functions for each of our queries. Let's use these in our application:

```go
package db

import (
    "context"
    "database/sql"
)

type Store struct {
    *Queries
    db *sql.DB
}

func NewStore(db *sql.DB) *Store {
    return &Store{
        Queries: New(db),
        db:      db,
    }
}

func (s *Store) CreateOrderWithItemsTx(ctx context.Context, arg CreateOrderWithItemsParams) (int64, error) {
    tx, err := s.db.BeginTx(ctx, nil)
    if err != nil {
        return 0, err
    }
    defer tx.Rollback()

    qtx := s.WithTx(tx)
    orderId, err := qtx.CreateOrderWithItems(ctx, arg)
    if err != nil {
        return 0, err
    }

    if err := tx.Commit(); err != nil {
        return 0, err
    }

    return orderId, nil
}

func (s *Store) UpdateOrderStatusTx(ctx context.Context, id int64, status string) error {
    tx, err := s.db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback()

    qtx := s.WithTx(tx)
    if err := qtx.UpdateOrderStatus(ctx, UpdateOrderStatusParams{ID: id, Status: status}); err != nil {
        return err
    }

    // Simulate some additional operations that might be part of this transaction
    // For example, updating inventory, sending notifications, etc.

    if err := tx.Commit(); err != nil {
        return err
    }

    return nil
}
```

In this code:

1. We've created a `Store` struct that wraps our sqlc `Queries` and adds transaction support.
2. `CreateOrderWithItemsTx` demonstrates how to use a transaction to ensure that both the order and its items are created atomically.
3. `UpdateOrderStatusTx` shows how we might update an order's status as part of a larger transaction that could involve other operations.

These examples demonstrate how to use sqlc to implement complex queries and handle transactions effectively. In the next section, we'll look at how to optimize the performance of these database operations.

## 4. Optimizing Database Performance

Optimizing database performance is crucial for maintaining a responsive and scalable system. Let's explore some techniques to improve the performance of our order processing system.

### Analyzing Query Performance with EXPLAIN

PostgreSQL's EXPLAIN command is a powerful tool for understanding and optimizing query performance. Let's use it to analyze our `GetOrderWithItems` query:

```sql
EXPLAIN ANALYZE
SELECT o.*, 
       json_agg(json_build_object(
           'id', oi.id,
           'product_id', oi.product_id,
           'quantity', oi.quantity,
           'price', oi.price
       )) AS items
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
WHERE o.id = 1
GROUP BY o.id;
```

This will provide us with a query plan and execution statistics. Based on the results, we can identify potential bottlenecks and optimize our query.

### Implementing and Using Database Indexes Effectively

Indexes can dramatically improve query performance, especially for large tables. Let's add some indexes to our schema:

```sql
-- migrations/000003_add_indexes.up.sql
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_orders_status ON orders(status);
```

These indexes will speed up our JOIN operations and filtering by customer_id or status.

### Optimizing Data Types and Schema Design

Choosing the right data types can impact both storage efficiency and query performance. For example, using `BIGSERIAL` instead of `SERIAL` for `id` fields allows for a larger range of values, which can be important for high-volume systems.

### Handling Large Datasets Efficiently

When dealing with large datasets, it's important to implement pagination to avoid loading too much data at once. Let's add a paginated query for fetching orders:

```sql
-- name: ListOrdersPaginated :many
SELECT * FROM orders
ORDER BY created_at DESC
LIMIT $1 OFFSET $2;
```

In our Go code, we can use this query like this:

```go
func (s *Store) ListOrdersPaginated(ctx context.Context, limit, offset int32) ([]Order, error) {
    return s.Queries.ListOrdersPaginated(ctx, ListOrdersPaginatedParams{
        Limit: limit,
        Offset: offset,
    })
}
```

### Caching Strategies for Frequently Accessed Data

For data that's frequently accessed but doesn't change often, implementing a caching layer can significantly reduce database load. Here's a simple example using an in-memory cache:

```go
import (
    "context"
    "sync"
    "time"
)

type OrderCache struct {
    store *Store
    cache map[int64]*Order
    mutex sync.RWMutex
    ttl   time.Duration
}

func NewOrderCache(store *Store, ttl time.Duration) *OrderCache {
    return &OrderCache{
        store: store,
        cache: make(map[int64]*Order),
        ttl:   ttl,
    }
}

func (c *OrderCache) GetOrder(ctx context.Context, id int64) (*Order, error) {
    c.mutex.RLock()
    if order, ok := c.cache[id]; ok {
        c.mutex.RUnlock()
        return order, nil
    }
    c.mutex.RUnlock()

    order, err := c.store.GetOrder(ctx, id)
    if err != nil {
        return nil, err
    }

    c.mutex.Lock()
    c.cache[id] = &order
    c.mutex.Unlock()

    go func() {
        time.Sleep(c.ttl)
        c.mutex.Lock()
        delete(c.cache, id)
        c.mutex.Unlock()
    }()

    return &order, nil
}
```

This cache implementation stores orders in memory for a specified duration, reducing the need to query the database for frequently accessed orders.

## 5. Implementing Batch Operations

Batch operations can significantly improve performance when dealing with large datasets. Let's implement some batch operations for our order processing system.

### Designing Batch Insert Operations

First, let's add a batch insert operation for order items:

```sql
-- name: BatchCreateOrderItems :copyfrom
INSERT INTO order_items (
    order_id, product_id, quantity, price
) VALUES (
    $1, $2, $3, $4
);
```

In our Go code, we can use this to insert multiple order items efficiently:

```go
func (s *Store) BatchCreateOrderItems(ctx context.Context, items []OrderItem) error {
    return s.Queries.BatchCreateOrderItems(ctx, items)
}
```

### Handling Large Batch Operations Efficiently

When dealing with very large batches, it's important to process them in chunks to avoid overwhelming the database or running into memory issues. Here's an example of how we might do this:

```go
func (s *Store) BatchCreateOrderItemsChunked(ctx context.Context, items []OrderItem, chunkSize int) error {
    for i := 0; i < len(items); i += chunkSize {
        end := i + chunkSize
        if end > len(items) {
            end = len(items)
        }
        chunk := items[i:end]
        if err := s.BatchCreateOrderItems(ctx, chunk); err != nil {
            return err
        }
    }
    return nil
}
```

### Error Handling and Partial Failure in Batch Operations

When performing batch operations, it's important to handle partial failures gracefully. One approach is to use transactions and savepoints:

```go
func (s *Store) BatchCreateOrderItemsWithSavepoints(ctx context.Context, items []OrderItem, chunkSize int) error {
    tx, err := s.db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback()

    qtx := s.WithTx(tx)

    for i := 0; i < len(items); i += chunkSize {
        end := i + chunkSize
        if end > len(items) {
            end = len(items)
        }
        chunk := items[i:end]

        _, err := tx.ExecContext(ctx, "SAVEPOINT batch_insert")
        if err != nil {
            return err
        }

        err = qtx.BatchCreateOrderItems(ctx, chunk)
        if err != nil {
            _, rbErr := tx.ExecContext(ctx, "ROLLBACK TO SAVEPOINT batch_insert")
            if rbErr != nil {
                return fmt.Errorf("batch insert failed and unable to rollback: %v, %v", err, rbErr)
            }
            // Log the error or handle it as appropriate for your use case
            fmt.Printf("Failed to insert chunk %d-%d: %v\n", i, end, err)
        } else {
            _, err = tx.ExecContext(ctx, "RELEASE SAVEPOINT batch_insert")
            if err != nil {
                return err
            }
        }
    }

    return tx.Commit()
}
```

This approach allows us to rollback individual chunks if they fail, while still committing the successful chunks.

## 6. Handling Database Migrations in a Production Environment

As our system evolves, we'll need to make changes to our database schema. Managing these changes in a production environment requires careful planning and execution.

### Strategies for Zero-Downtime Migrations

To achieve zero-downtime migrations, we can follow these steps:

1. Make all schema changes backwards compatible
2. Deploy the new application version that supports both old and new schemas
3. Run the schema migration
4. Deploy the final application version that only supports the new schema

Let's look at an example of a backwards compatible migration:

```sql
-- migrations/000004_add_order_notes.up.sql
ALTER TABLE orders ADD COLUMN notes TEXT;

-- migrations/000004_add_order_notes.down.sql
ALTER TABLE orders DROP COLUMN notes;
```

This migration adds a new column, which is a backwards compatible change. Existing queries will continue to work, and we can update our application to start using the new column.

### Implementing and Managing Database Schema Versions

We're already using golang-migrate for our migrations, which keeps track of the current schema version. We can query this information to ensure our application is compatible with the current database schema:

```go
func (s *Store) GetDatabaseVersion(ctx context.Context) (int, error) {
    var version int
    err := s.db.QueryRowContext(ctx, "SELECT version FROM schema_migrations ORDER BY version DESC LIMIT 1").Scan(&version)
    if err != nil {
        return 0, err
    }
    return version, nil
}
```

### Handling Data Transformations During Migrations

Sometimes we need to not only change the schema but also transform existing data. Here's an example of a migration that does both:

```sql
-- migrations/000005_split_name.up.sql
ALTER TABLE customers ADD COLUMN first_name TEXT, ADD COLUMN last_name TEXT;
UPDATE customers SET 
    first_name = split_part(name, ' ', 1),
    last_name = split_part(name, ' ', 2)
WHERE name IS NOT NULL;
ALTER TABLE customers DROP COLUMN name;

-- migrations/000005_split_name.down.sql
ALTER TABLE customers ADD COLUMN name TEXT;
UPDATE customers SET name = concat(first_name, ' ', last_name)
WHERE first_name IS NOT NULL OR last_name IS NOT NULL;
ALTER TABLE customers DROP COLUMN first_name, DROP COLUMN last_name;
```

This migration splits the `name` column into `first_name` and `last_name`, transforming the existing data in the process.

### Rolling Back Migrations Safely

It's crucial to test both the up and down migrations thoroughly before applying them to a production database. Always have a rollback plan ready in case issues are discovered after a migration is applied.

In the next sections, we'll explore database sharding for scalability and ensuring data consistency in a distributed system.

## 7. Implementing Database Sharding for Scalability

As our order processing system grows, we may need to scale beyond what a single database instance can handle. Database sharding is a technique that can help us achieve horizontal scalability by distributing data across multiple database instances.

### Designing a Sharding Strategy for Our Order Processing System

For our order processing system, we'll implement a simple sharding strategy based on the customer ID. This approach ensures that all orders for a particular customer are on the same shard, which can simplify certain types of queries.

First, let's create a sharding function:

```go
const NUM_SHARDS = 4

func getShardForCustomer(customerID int64) int {
    return int(customerID % NUM_SHARDS)
}
```

This function will distribute customers (and their orders) evenly across our shards.

### Implementing a Sharding Layer with sqlc

Now, let's implement a sharding layer that will route queries to the appropriate shard:

```go
type ShardedStore struct {
    stores [NUM_SHARDS]*Store
}

func NewShardedStore(connStrings [NUM_SHARDS]string) (*ShardedStore, error) {
    var stores [NUM_SHARDS]*Store
    for i, connString := range connStrings {
        db, err := sql.Open("postgres", connString)
        if err != nil {
            return nil, err
        }
        stores[i] = NewStore(db)
    }
    return &ShardedStore{stores: stores}, nil
}

func (s *ShardedStore) GetOrder(ctx context.Context, customerID, orderID int64) (Order, error) {
    shard := getShardForCustomer(customerID)
    return s.stores[shard].GetOrder(ctx, orderID)
}

func (s *ShardedStore) CreateOrder(ctx context.Context, arg CreateOrderParams) (Order, error) {
    shard := getShardForCustomer(arg.CustomerID)
    return s.stores[shard].CreateOrder(ctx, arg)
}
```

This `ShardedStore` maintains connections to all of our database shards and routes queries to the appropriate shard based on the customer ID.

### Handling Cross-Shard Queries and Transactions

Cross-shard queries can be challenging in a sharded database setup. For example, if we need to get all orders across all shards, we'd need to query each shard and combine the results:

```go
func (s *ShardedStore) GetAllOrders(ctx context.Context) ([]Order, error) {
    var allOrders []Order
    for _, store := range s.stores {
        orders, err := store.ListOrders(ctx)
        if err != nil {
            return nil, err
        }
        allOrders = append(allOrders, orders...)
    }
    return allOrders, nil
}
```

Cross-shard transactions are even more complex and often require a two-phase commit protocol or a distributed transaction manager. In many cases, it's better to design your system to avoid the need for cross-shard transactions if possible.

### Rebalancing Shards and Handling Shard Growth

As your data grows, you may need to add new shards or rebalance existing ones. This process can be complex and typically involves:

1. Adding new shards to the system
2. Gradually migrating data from existing shards to new ones
3. Updating the sharding function to incorporate the new shards

Here's a simple example of how we might update our sharding function to handle a growing number of shards:

```go
var NUM_SHARDS = 4

func updateNumShards(newNumShards int) {
    NUM_SHARDS = newNumShards
}

func getShardForCustomer(customerID int64) int {
    return int(customerID % int64(NUM_SHARDS))
}
```

In a production system, you'd want to implement a more sophisticated approach, possibly using a consistent hashing algorithm to minimize data movement when adding or removing shards.

## 8. Ensuring Data Consistency in a Distributed System

Maintaining data consistency in a distributed system like our sharded database setup can be challenging. Let's explore some strategies to ensure consistency.

### Implementing Distributed Transactions with sqlc

While sqlc doesn't directly support distributed transactions, we can implement a simple two-phase commit protocol for operations that need to span multiple shards. Here's a basic example:

```go
func (s *ShardedStore) CreateOrderAcrossShards(ctx context.Context, arg CreateOrderParams, items []CreateOrderItemParams) error {
    // Phase 1: Prepare
    var preparedTxs []*sql.Tx
    for _, store := range s.stores {
        tx, err := store.db.BeginTx(ctx, nil)
        if err != nil {
            // Rollback any prepared transactions
            for _, preparedTx := range preparedTxs {
                preparedTx.Rollback()
            }
            return err
        }
        preparedTxs = append(preparedTxs, tx)
    }

    // Phase 2: Commit
    for _, tx := range preparedTxs {
        if err := tx.Commit(); err != nil {
            // If any commit fails, we're in an inconsistent state
            // In a real system, we'd need a way to recover from this
            return err
        }
    }

    return nil
}
```

This is a simplified example and doesn't handle many edge cases. In a production system, you'd need more sophisticated error handling and recovery mechanisms.

### Handling Eventual Consistency in Database Operations

In some cases, it may be acceptable (or necessary) to have eventual consistency rather than strong consistency. For example, if we're generating reports across all shards, we might be okay with slightly out-of-date data:

```go
func (s *ShardedStore) GetOrderCountsEventuallyConsistent(ctx context.Context) (map[string]int, error) {
    counts := make(map[string]int)
    var wg sync.WaitGroup
    var mu sync.Mutex
    errCh := make(chan error, NUM_SHARDS)

    for _, store := range s.stores {
        wg.Add(1)
        go func(store *Store) {
            defer wg.Done()
            localCounts, err := store.GetOrderCounts(ctx)
            if err != nil {
                errCh <- err
                return
            }
            mu.Lock()
            for status, count := range localCounts {
                counts[status] += count
            }
            mu.Unlock()
        }(store)
    }

    wg.Wait()
    close(errCh)

    if err := <-errCh; err != nil {
        return nil, err
    }

    return counts, nil
}
```

This function aggregates order counts across all shards concurrently, providing a eventually consistent view of the data.

### Implementing Compensating Transactions for Failure Scenarios

In distributed systems, it's important to have mechanisms to handle partial failures. Compensating transactions can help restore the system to a consistent state when a distributed operation fails partway through.

Here's an example of how we might implement a compensating transaction for a failed order creation:

```go
func (s *ShardedStore) CreateOrderWithCompensation(ctx context.Context, arg CreateOrderParams) (Order, error) {
    shard := getShardForCustomer(arg.CustomerID)
    order, err := s.stores[shard].CreateOrder(ctx, arg)
    if err != nil {
        return Order{}, err
    }

    // Simulate some additional processing that might fail
    if err := someProcessingThatMightFail(); err != nil {
        // If processing fails, we need to compensate by deleting the order
        if err := s.stores[shard].DeleteOrder(ctx, order.ID); err != nil {
            // Log the error, as we're now in an inconsistent state
            log.Printf("Failed to compensate for failed order creation: %v", err)
        }
        return Order{}, err
    }

    return order, nil
}
```

This function creates an order and then performs some additional processing. If the processing fails, it attempts to delete the order as a compensating action.

### Strategies for Maintaining Referential Integrity Across Shards

Maintaining referential integrity across shards can be challenging. One approach is to denormalize data to keep related entities on the same shard. For example, we might store a copy of customer information with each order:

```go
type Order struct {
    ID         int64
    CustomerID int64
    // Denormalized customer data
    CustomerName  string
    CustomerEmail string
    // Other order fields...
}
```

This approach trades some data redundancy for easier maintenance of consistency within a shard.

## 9. Testing and Validation

Thorough testing is crucial when working with complex database operations and distributed systems. Let's explore some strategies for testing our sharded database system.

### Unit Testing Database Operations with sqlc

sqlc generates code that's easy to unit test. Here's an example of how we might test our `GetOrder` function:

```go
func TestGetOrder(t *testing.T) {
    // Set up a test database
    db, err := sql.Open("postgres", "postgresql://testuser:testpass@localhost:5432/testdb")
    if err != nil {
        t.Fatalf("Failed to connect to test database: %v", err)
    }
    defer db.Close()

    store := NewStore(db)

    // Create a test order
    order, err := store.CreateOrder(context.Background(), CreateOrderParams{
        CustomerID: 1,
        Status:     "pending",
        TotalAmount: 100.00,
    })
    if err != nil {
        t.Fatalf("Failed to create test order: %v", err)
    }

    // Test GetOrder
    retrievedOrder, err := store.GetOrder(context.Background(), order.ID)
    if err != nil {
        t.Fatalf("Failed to get order: %v", err)
    }

    if retrievedOrder.ID != order.ID {
        t.Errorf("Expected order ID %d, got %d", order.ID, retrievedOrder.ID)
    }
    // Add more assertions as needed...
}
```

### Implementing Integration Tests for Database Functionality

Integration tests can help ensure that our sharding logic works correctly with real database instances. Here's an example:

```go
func TestShardedStore(t *testing.T) {
    // Set up test database instances for each shard
    connStrings := [NUM_SHARDS]string{
        "postgresql://testuser:testpass@localhost:5432/testdb1",
        "postgresql://testuser:testpass@localhost:5432/testdb2",
        "postgresql://testuser:testpass@localhost:5432/testdb3",
        "postgresql://testuser:testpass@localhost:5432/testdb4",
    }

    shardedStore, err := NewShardedStore(connStrings)
    if err != nil {
        t.Fatalf("Failed to create sharded store: %v", err)
    }

    // Test creating orders on different shards
    order1, err := shardedStore.CreateOrder(context.Background(), CreateOrderParams{CustomerID: 1, Status: "pending", TotalAmount: 100.00})
    if err != nil {
        t.Fatalf("Failed to create order on shard 1: %v", err)
    }

    order2, err := shardedStore.CreateOrder(context.Background(), CreateOrderParams{CustomerID: 2, Status: "pending", TotalAmount: 200.00})
    if err != nil {
        t.Fatalf("Failed to create order on shard 2: %v", err)
    }

    // Test retrieving orders from different shards
    retrievedOrder1, err := shardedStore.GetOrder(context.Background(), 1, order1.ID)
    if err != nil {
        t.Fatalf("Failed to get order from shard 1: %v", err)
    }

    retrievedOrder2, err := shardedStore.GetOrder(context.Background(), 2, order2.ID)
    if err != nil {
        t.Fatalf("Failed to get order from shard 2: %v", err)
    }

    // Add assertions to check the retrieved orders...
}
```

### Performance Testing and Benchmarking Database Operations

Performance testing is crucial, especially when working with sharded databases. Here's an example of how to benchmark our `GetOrder` function:

```go
func BenchmarkGetOrder(b *testing.B) {
    // Set up your database connection
    db, err := sql.Open("postgres", "postgresql://testuser:testpass@localhost:5432/testdb")
    if err != nil {
        b.Fatalf("Failed to connect to test database: %v", err)
    }
    defer db.Close()

    store := NewStore(db)

    // Create a test order
    order, err := store.CreateOrder(context.Background(), CreateOrderParams{
        CustomerID: 1,
        Status:     "pending",
        TotalAmount: 100.00,
    })
    if err != nil {
        b.Fatalf("Failed to create test order: %v", err)
    }

    // Run the benchmark
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        _, err := store.GetOrder(context.Background(), order.ID)
        if err != nil {
            b.Fatalf("Benchmark failed: %v", err)
        }
    }
}
```

This benchmark will help you understand the performance characteristics of your `GetOrder` function and can be used to compare different implementations or optimizations.

## 10. Challenges and Considerations

As we implement and operate our sharded database system, there are several challenges and considerations to keep in mind:

1. **Managing Database Connection Pools**: With multiple database instances, it's crucial to manage connection pools efficiently to avoid overwhelming any single database or running out of connections.

2. **Handling Database Failover and High Availability**: In a sharded setup, you need to consider what happens if one of your database instances fails. Implementing read replicas and automatic failover can help ensure high availability.

3. **Consistent Backups Across Shards**: Backing up a sharded database system requires careful coordination to ensure consistency across all shards.

4. **Query Routing and Optimization**: As your sharding scheme evolves, you may need to implement more sophisticated query routing to optimize performance.

5. **Data Rebalancing**: As some shards grow faster than others, you may need to periodically rebalance data across shards.

6. **Cross-Shard Joins and Aggregations**: These operations can be particularly challenging in a sharded system and may require implementation at the application level.

7. **Maintaining Data Integrity**: Ensuring data integrity across shards, especially for operations that span multiple shards, requires careful design and implementation.

8. **Monitoring and Alerting**: With a distributed database system, comprehensive monitoring and alerting become even more critical to quickly identify and respond to issues.

## 11. Next Steps and Preview of Part 4

In this post, we've delved deep into advanced database operations using sqlc, covering everything from optimizing queries and implementing batch operations to managing database migrations and implementing sharding for scalability.

In the next part of our series, we'll focus on monitoring and alerting with Prometheus. We'll cover:

1. Setting up Prometheus for monitoring our order processing system
2. Defining and implementing custom metrics
3. Creating dashboards with Grafana
4. Implementing alerting rules
5. Monitoring database performance
6. Monitoring Temporal workflows

Stay tuned as we continue to build out our sophisticated order processing system, focusing next on ensuring we can effectively monitor and maintain our system in a production environment!


{% include jd-course.html %}
