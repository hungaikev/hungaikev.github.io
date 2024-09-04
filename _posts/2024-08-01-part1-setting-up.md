---
layout: post
title: "Implementing an Order Processing System: Part 1 - Setting Up the Foundation"
seo_title: "E-commerce Platform: Setting Up the Foundation"
seo_description: "Learn how to set up the foundation for a robust e-commerce platform using Golang, Gin, Temporal, and PostgreSQL."
date: 2024-08-01 12:00:00
categories: [Temporal,E-commerce Platform, System Architecture]
tags: [Golang, Gin, Temporal, PostgreSQL, Docker]
author: Hungai Amuhinda
excerpt: "Set up the foundation for a sophisticated e-commerce platform, including project structure, basic API, database integration, and simple Temporal workflow."
permalink: /e-commerce-platform/part-1-setting-up-the-foundation/
toc: true
comments: true
---



## 1. Introduction and Goals

Welcome to the first part of our comprehensive blog series on implementing a sophisticated order processing system using Temporal for microservice orchestration. In this series, we'll explore the intricacies of building a robust, scalable, and maintainable system that can handle complex, long-running workflows.

Our journey begins with setting up the foundation for our project. By the end of this post, you'll have a fully functional CRUD REST API implemented in Golang, integrated with Temporal for workflow orchestration, and backed by a Postgres database. We'll use modern tools and best practices to ensure our codebase is clean, efficient, and easy to maintain.

### Goals for this post:

1. Set up a well-structured project using Go modules
2. Implement a basic CRUD API using Gin and oapi-codegen
3. Set up a Postgres database and implement migrations
4. Create a simple Temporal workflow with database interaction
5. Implement dependency injection for better testability and maintainability
6. Containerize our application using Docker
7. Provide a complete local development environment using docker-compose

Let's dive in and start building our order processing system!

## 2. Theoretical Background and Concepts

Before we start implementing, let's briefly review the key technologies and concepts we'll be using:

### Golang

Go is a statically typed, compiled language known for its simplicity, efficiency, and excellent support for concurrent programming. Its standard library and robust ecosystem make it an excellent choice for building microservices.

### Temporal

Temporal is a microservice orchestration platform that simplifies the development of distributed applications. It allows us to write complex, long-running workflows as simple procedural code, handling failures and retries automatically.

### Gin Web Framework

Gin is a high-performance HTTP web framework written in Go. It provides a martini-like API with much better performance and lower memory usage.

### OpenAPI and oapi-codegen

OpenAPI (formerly known as Swagger) is a specification for machine-readable interface files for describing, producing, consuming, and visualizing RESTful web services. oapi-codegen is a tool that generates Go code from OpenAPI 3.0 specifications, allowing us to define our API contract first and generate server stubs and client code.

### sqlc

sqlc generates type-safe Go code from SQL. It allows us to write plain SQL queries and generate fully type-safe Go code to interact with our database, reducing the likelihood of runtime errors and improving maintainability.

### Postgres

PostgreSQL is a powerful, open-source object-relational database system known for its reliability, feature robustness, and performance.

### Docker and docker-compose

Docker allows us to package our application and its dependencies into containers, ensuring consistency across different environments. docker-compose is a tool for defining and running multi-container Docker applications, which we'll use to set up our local development environment.

Now that we've covered the basics, let's start implementing our system.

## 3. Step-by-Step Implementation Guide

### 3.1 Setting Up the Project Structure

First, let's create our project directory and set up the basic structure:

```bash
mkdir order-processing-system
cd order-processing-system

# Create directory structure
mkdir -p cmd/api \
         internal/api \
         internal/db \
         internal/models \
         internal/service \
         internal/workflow \
         migrations \
         pkg/logger \
         scripts

# Initialize Go module
go mod init github.com/yourusername/order-processing-system

# Create main.go file
touch cmd/api/main.go
```

This structure follows the standard Go project layout:

- `cmd/api`: Contains the main application entry point
- `internal`: Houses packages that are specific to this project and not meant to be imported by other projects
- `migrations`: Stores database migration files
- `pkg`: Contains packages that can be imported by other projects
- `scripts`: Holds utility scripts for development and deployment

### 3.2 Creating the Makefile

Let's create a Makefile to simplify common tasks:

```bash
touch Makefile
```

Add the following content to the Makefile:

```makefile
.PHONY: generate build run test clean

generate:
	@echo "Generating code..."
	go generate ./...

build:
	@echo "Building..."
	go build -o bin/api cmd/api/main.go

run:
	@echo "Running..."
	go run cmd/api/main.go

test:
	@echo "Running tests..."
	go test -v ./...

clean:
	@echo "Cleaning..."
	rm -rf bin

.DEFAULT_GOAL := build
```

This Makefile provides targets for generating code, building the application, running it, running tests, and cleaning up build artifacts.

### 3.3 Implementing the Basic CRUD API

#### 3.3.1 Define the OpenAPI Specification

Create a file named `api/openapi.yaml` and define our API specification:

```yaml
openapi: 3.0.0
info:
  title: Order Processing API
  version: 1.0.0
  description: API for managing orders in our processing system

paths:
  /orders:
    get:
      summary: List all orders
      responses:
        '200':
          description: Successful response
          content:
            application/json:    
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/Order'
    post:
      summary: Create a new order
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateOrderRequest'
      responses:
        '201':
          description: Created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'

  /orders/{id}:
    get:
      summary: Get an order by ID
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'
        '404':
          description: Order not found
    put:
      summary: Update an order
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UpdateOrderRequest'
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'
        '404':
          description: Order not found
    delete:
      summary: Delete an order
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
      responses:
        '204':
          description: Successful response
        '404':
          description: Order not found

components:
  schemas:
    Order:
      type: object
      properties:
        id:
          type: integer
        customer_id:
          type: integer
        status:
          type: string
          enum: [pending, processing, completed, cancelled]
        total_amount:
          type: number
        created_at:
          type: string
          format: date-time
        updated_at:
          type: string
          format: date-time
    CreateOrderRequest:
      type: object
      required:
        - customer_id
        - total_amount
      properties:
        customer_id:
          type: integer
        total_amount:
          type: number
    UpdateOrderRequest:
      type: object
      properties:
        status:
          type: string
          enum: [pending, processing, completed, cancelled]
        total_amount:
          type: number
```

This specification defines our basic CRUD operations for orders.

#### 3.3.2 Generate API Code

Install oapi-codegen:

```bash
go install github.com/deepmap/oapi-codegen/cmd/oapi-codegen@latest
```

Generate the server code:

```bash
oapi-codegen -package api -generate types,server,spec api/openapi.yaml > internal/api/api.gen.go
```

This command generates the Go code for our API, including types, server interfaces, and the OpenAPI specification.

#### 3.3.3 Implement the API Handler

Create a new file `internal/api/handler.go`:

```go
package api

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

type Handler struct {
	// We'll add dependencies here later
}

func NewHandler() *Handler {
	return &Handler{}
}

func (h *Handler) RegisterRoutes(r *gin.Engine) {
	RegisterHandlers(r, h)
}

// Implement the ServerInterface methods

func (h *Handler) GetOrders(c *gin.Context) {
	// TODO: Implement
	c.JSON(http.StatusOK, []Order{})
}

func (h *Handler) CreateOrder(c *gin.Context) {
	var req CreateOrderRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	// TODO: Implement order creation logic
	order := Order{
		Id:         1,
		CustomerId: req.CustomerId,
		Status:     "pending",
		TotalAmount: req.TotalAmount,
	}

	c.JSON(http.StatusCreated, order)
}

func (h *Handler) GetOrder(c *gin.Context, id int) {
	// TODO: Implement
	c.JSON(http.StatusOK, Order{Id: id})
}

func (h *Handler) UpdateOrder(c *gin.Context, id int) {
	var req UpdateOrderRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	// TODO: Implement order update logic
	order := Order{
		Id:     id,
		Status: *req.Status,
	}

	c.JSON(http.StatusOK, order)
}

func (h *Handler) DeleteOrder(c *gin.Context, id int) {
	// TODO: Implement
	c.Status(http.StatusNoContent)
}
```

This implementation provides a basic structure for our API handlers. We'll flesh out the actual logic when we integrate with the database and Temporal workflows.

### 3.4 Setting Up the Postgres Database

#### 3.4.1 Create a docker-compose file

Create a `docker-compose.yml` file in the project root:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:13
    environment:
      POSTGRES_USER: orderuser
      POSTGRES_PASSWORD: orderpass
      POSTGRES_DB: orderdb
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

This sets up a Postgres container for our local development environment.

#### 3.4.2 Implement Database Migrations

Install golang-migrate:

```bash
go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest
```

Create our first migration:

```bash
migrate create -ext sql -dir migrations -seq create_orders_table
```

Edit the `migrations/000001_create_orders_table.up.sql` file:

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    status VARCHAR(20) NOT NULL,
    total_amount DECIMAL(10, 2) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_orders_status ON orders(status);
```

Edit the `migrations/000001_create_orders_table.down.sql` file:

```sql
DROP TABLE IF EXISTS orders;
```

#### 3.4.3 Run Migrations

Add a new target to our Makefile:

```makefile
migrate-up:
	@echo "Running migrations..."
	migrate -path migrations -database "postgresql://orderuser:orderpass@localhost:5432/orderdb?sslmode=disable" up

migrate-down:
	@echo "Reverting migrations..."
	migrate -path migrations -database "postgresql://orderuser:orderpass@localhost:5432/orderdb?sslmode=disable" down
```

Now we can run migrations with:

```bash
make migrate-up
```

### 3.5 Implementing Database Operations with sqlc

#### 3.5.1 Install sqlc

```bash
go install github.com/kyleconroy/sqlc/cmd/sqlc@latest
```

#### 3.5.2 Configure sqlc

Create a `sqlc.yaml` file in the project root:

```yaml
version: "2"
sql:
  - engine: "postgresql"
    queries: "internal/db/queries.sql"
    schema: "migrations"
    gen:
      go:
        package: "db"
        out: "internal/db"
        emit_json_tags: true
        emit_prepared_queries: false
        emit_interface: true
        emit_exact_table_names: false
```

#### 3.5.3 Write SQL Queries

Create a file `internal/db/queries.sql`:

```sql
-- name: GetOrder :one
SELECT * FROM orders
WHERE id = $1 LIMIT 1;

-- name: ListOrders :many
SELECT * FROM orders
ORDER BY id;

-- name: CreateOrder :one
INSERT INTO orders (
  customer_id, status, total_amount
) VALUES (
  $1, $2, $3
)
RETURNING *;

-- name: UpdateOrder :one
UPDATE orders
SET status = $2, total_amount = $3, updated_at = CURRENT_TIMESTAMP
WHERE id = $1
RETURNING *;

-- name: DeleteOrder :exec
DELETE FROM orders
WHERE id = $1;
```

#### 3.5.4 Generate Go Code

Add a new target to our Makefile:

```makefile
generate-sqlc:
	@echo "Generating sqlc code..."
	sqlc generate
```

Run the code generation:

```bash
make generate-sqlc
```

This will generate Go code for interacting with our database in the `internal/db` directory.

### 3.6 Integrating Temporal

#### 3.6.1 Set Up Temporal Server

Add Temporal to our `docker-compose.yml`:

```yaml
  temporal:
    image: temporalio/auto-setup:1.13.0
    ports:
      - "7233:7233"
    environment:
      - DB=postgresql
      - DB_PORT=5432
      - POSTGRES_USER=orderuser
      - POSTGRES_PWD=orderpass
      - POSTGRES_SEEDS=postgres
    depends_on:
      - postgres

  temporal-admin-tools:
    image: temporalio/admin-tools:1.13.0
    depends_on:
      - temporal
```

#### 3.6.2 Implement a Basic Workflow

Create a file `internal/workflow/order_workflow.go`:

```go
package workflow

import (
	"time"

	"go.temporal.io/sdk/workflow"
	"github.com/yourusername/order-processing-system/internal/db"
)

func OrderWorkflow(ctx workflow.Context, order db.Order) error {
	logger := workflow.GetLogger(ctx)
	logger.Info("OrderWorkflow started", "OrderID", order.ID)

	// Simulate order processing
	err := workflow.Sleep(ctx, 5*time.Second)
	if err != nil {
		return err
	}

	// Update order status
	err = workflow.ExecuteActivity(ctx, UpdateOrderStatus, workflow.ActivityOptions{
		StartToCloseTimeout: time.Minute,
	}, order.ID, "completed").Get(ctx, nil)
	if err != nil {
		return err
	}

	logger.Info("OrderWorkflow completed", "OrderID", order.ID)
	return nil
}

func UpdateOrderStatus(ctx workflow.Context, orderID int64, status string) error {
	// TODO: Implement database update
	return nil
}
```

This basic workflow simulates order processing by waiting for 5 seconds and then updating the order status to "completed".

#### 3.6.3 Integrate Workflow with API

Update the `internal/api/handler.go` file to include Temporal client and start the workflow:

```go
package api

import (
	"context"
	"net/http"

	"github.com/gin-gonic/gin"
	"go.temporal.io/sdk/client"
	"github.com/yourusername/order-processing-system/internal/db"
	"github.com/yourusername/order-processing-system/internal/workflow"
)

type Handler struct {
	queries    *db.Queries
	temporalClient client.Client
}

func NewHandler(queries *db.Queries, temporalClient client.Client) *Handler {
	return &Handler{
		queries:    queries,
		temporalClient: temporalClient,
	}
}

// ... (previous handler methods)

func (h *Handler) CreateOrder(c *gin.Context) {
	var req CreateOrderRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	order, err := h.queries.CreateOrder(c, db.CreateOrderParams{
		CustomerID:  req.CustomerId,
		Status:     "pending",
		TotalAmount: req.TotalAmount,
	})
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}

	// Start Temporal workflow
	workflowOptions := client.StartWorkflowOptions{
		ID:        "order-" + order.ID,
		TaskQueue: "order-processing",
	}
	_, err = h.temporalClient.ExecuteWorkflow(context.Background(), workflowOptions, workflow.OrderWorkflow, order)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to start workflow"})
		return
	}

	c.JSON(http.StatusCreated, order)
}

// ... (implement other handler methods)
```

### 3.7 Implementing Dependency Injection

Create a new file `internal/service/service.go`:

```go
package service

import (
	"database/sql"

	"github.com/yourusername/order-processing-system/internal/api"
	"github.com/yourusername/order-processing-system/internal/db"
	"go.temporal.io/sdk/client"
)

type Service struct {
	DB            *sql.DB
	Queries       *db.Queries
	TemporalClient client.Client
	Handler       *api.Handler
}

func NewService() (*Service, error) {
	// Initialize database connection
	db, err := sql.Open("postgres", "postgresql://orderuser:orderpass@localhost:5432/orderdb?sslmode=disable")
	if err != nil {
		return nil, err
	}

	// Initialize Temporal client
	temporalClient, err := client.NewClient(client.Options{
		HostPort: "localhost:7233",
	})
	if err != nil {
		return nil, err
	}

	// Initialize queries
	queries := db.New(db)

	// Initialize handler
	handler := api.NewHandler(queries, temporalClient)

	return &Service{
		DB:            db,
		Queries:       queries,
		TemporalClient: temporalClient,
		Handler:       handler,
	}, nil
}

func (s *Service) Close() {
	s.DB.Close()
	s.TemporalClient.Close()
}
```

### 3.8 Update Main Function

Update the `cmd/api/main.go` file:

```go
package main

import (
	"log"

	"github.com/gin-gonic/gin"
	_ "github.com/lib/pq"
	"github.com/yourusername/order-processing-system/internal/service"
)

func main() {
	svc, err := service.NewService()
	if err != nil {
		log.Fatalf("Failed to initialize service: %v", err)
	}
	defer svc.Close()

	r := gin.Default()
	svc.Handler.RegisterRoutes(r)

	if err := r.Run(":8080"); err != nil {
		log.Fatalf("Failed to run server: %v", err)
	}
}
```

### 3.9 Dockerize the Application

Create a `Dockerfile` in the project root:

```Dockerfile
# Build stage
FROM golang:1.17-alpine AS build

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /order-processing-system ./cmd/api

# Run stage
FROM alpine:latest

WORKDIR /

COPY --from=build /order-processing-system /order-processing-system

EXPOSE 8080

ENTRYPOINT ["/order-processing-system"]
```

Update the `docker-compose.yml` file to include our application:

```yaml
version: '3.8'

services:
  postgres:
    # ... (previous postgres configuration)

  temporal:
    # ... (previous temporal configuration)

  temporal-admin-tools:
    # ... (previous temporal-admin-tools configuration)

  app:
    build: .
    ports:
      - "8080:8080"
    depends_on:
      - postgres
      - temporal
    environment:
      - DB_HOST=postgres
      - DB_USER=orderuser
      - DB_PASSWORD=orderpass
      - DB_NAME=orderdb
      - TEMPORAL_HOST=temporal:7233
```

## 4. Code Examples with Detailed Comments

Throughout the implementation guide, we've provided code snippets with explanations. Here's a more detailed look at a key part of our system: the Order Workflow.

```go
package workflow

import (
	"time"

	"go.temporal.io/sdk/workflow"
	"github.com/yourusername/order-processing-system/internal/db"
)

// OrderWorkflow defines the workflow for processing an order
func OrderWorkflow(ctx workflow.Context, order db.Order) error {
	logger := workflow.GetLogger(ctx)
	logger.Info("OrderWorkflow started", "OrderID", order.ID)

	// Simulate order processing
	// In a real-world scenario, this could involve multiple activities such as
	// inventory check, payment processing, shipping arrangement, etc.
	err := workflow.Sleep(ctx, 5*time.Second)
	if err != nil {
		return err
	}

	// Update order status
	// We use ExecuteActivity to run the status update as an activity
	// This allows for automatic retries and error handling
	err = workflow.ExecuteActivity(ctx, UpdateOrderStatus, workflow.ActivityOptions{
		StartToCloseTimeout: time.Minute,
	}, order.ID, "completed").Get(ctx, nil)
	if err != nil {
		return err
	}

	logger.Info("OrderWorkflow completed", "OrderID", order.ID)
	return nil
}

// UpdateOrderStatus is an activity that updates the status of an order
func UpdateOrderStatus(ctx workflow.Context, orderID int64, status string) error {
	// TODO: Implement database update
	// In a real implementation, this would use the db.Queries to update the order status
	return nil
}
```

This workflow demonstrates several key concepts:

1. Use of Temporal's `workflow.Context` for managing the workflow lifecycle.
2. Logging within workflows using `workflow.GetLogger`.
3. Simulating long-running processes with `workflow.Sleep`.
4. Executing activities within a workflow using `workflow.ExecuteActivity`.
5. Handling errors and returning them to be managed by Temporal.

## 5. Testing and Validation

For this initial setup, we'll focus on manual testing to ensure our system is working as expected. In future posts, we'll dive into unit testing, integration testing, and end-to-end testing strategies.

To manually test our system:

1. Start the services:
   ```
   docker-compose up
   ```

2. Use a tool like cURL or Postman to send requests to our API:

   Create an order:
   ```
   curl -X POST http://localhost:8080/orders -H "Content-Type: application/json" -d '{"customer_id": 1, "total_amount": 100.50}'
   ```

   List orders:
   ```
   curl http://localhost:8080/orders
   ```

   Get a specific order:
   ```
   curl http://localhost:8080/orders/1
   ```

   Update an order:
   ```
   curl -X PUT http://localhost:8080/orders/1 -H "Content-Type: application/json" -d '{"status": "processing", "total_amount": 150.75}'
   ```

   Delete an order:
   ```
   curl -X DELETE http://localhost:8080/orders/1
   ```

3. Check the logs to ensure the Temporal workflow is being triggered and completed successfully.

## 6. Challenges and Considerations

While setting up this initial version of our order processing system, we encountered several challenges and considerations:

1. **Database Schema Design**: Designing a flexible yet efficient schema for orders is crucial. We kept it simple for now, but in a real-world scenario, we might need to consider additional tables for order items, customer information, etc.

2. **Error Handling**: Our current implementation has basic error handling. In a production system, we'd need more robust error handling and logging, especially for the Temporal workflows.

3. **Configuration Management**: We hardcoded configuration values for simplicity. In a real-world scenario, we'd use environment variables or a configuration management system.

4. **Security**: Our current setup doesn't include any authentication or authorization. In a production system, we'd need to implement proper security measures.

5. **Scalability**: While Temporal helps with workflow scalability, we'd need to consider database scalability and API performance for a high-traffic system.

6. **Monitoring and Observability**: We haven't implemented any monitoring or observability tools yet. In a production system, these would be crucial for maintaining and troubleshooting the application.

## 7. Next Steps and Preview of Part 2

In this first part of our series, we've set up the foundation for our order processing system. We have a basic CRUD API, database integration, and a simple Temporal workflow. 

In the next part, we'll dive deeper into Temporal workflows and activities. We'll explore:

1. Implementing more complex order processing logic
2. Handling long-running workflows with Temporal
3. Implementing retry logic and error handling in workflows
4. Versioning workflows for safe updates
5. Implementing saga patterns for distributed transactions
6. Monitoring and observability for Temporal workflows

We'll also start to flesh out our API with more realistic order processing logic and explore patterns for maintaining clean, maintainable code as our system grows in complexity.

Stay tuned for Part 2, where we'll take our order processing system to the next level!


{% include jd-course.html %}





















