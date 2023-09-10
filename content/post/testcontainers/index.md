---
title: "Efficient Go Testing with Testcontainers"
description: "Learn how to efficiently do integration testing in Go using Testcontainers. This blog walks you through setting up a realistic testing environment, demonstrating usage of Testcontainers"
date: 2023-09-10
image: banner.png
categories:
  - Go
tags:
  - go
  - testcontainers
---

# Efficient Go Testing with Testcontainers

## Introduction
Testing is an essential part of developing a software, but it often involves setting up testing environment with dependencies like databases, message brokers, etc. making it very complex to test the applications. In this blog, we will explore how Testcontainers can help in testing applications in Golang without doing complex setup.

## What Are Testcontainers?
[Testcontainers](https://testcontainers.com/) is an open-source tool that simplifies the creation of isolated and reproducible testing environments. It leverages Docker to spin up containers within your tests, offering a consistent and controlled environment. This means you can easily test your code against dependencies like databases, message brokers, or other external services without the headache of complex setup and teardown procedures.

## When Should We Use Testcontainers?
Testcontainers are useful in scenarios where your application relies on external services or dependencies. Use Testcontainers when:

- Testing against a real database: Ensure that your database-related code works as expected without the need for a dedicated testing database.
- Testing microservices: Testcontainers can create isolated environments for each microservice, allowing you to validate their interactions.
- Running integration tests: Ensure that all components of your application work seamlessly together in a real-world environment.

You might be wondering why not use mocks of dependencies instead of testcontainers
## Testcontainers vs Mocks
Testcontainers offer a significant advantage over traditional mocking techniques. While mocks can simulate behavior to a certain extent, they often fall short in replicating the real-world scenarios and edge cases that an application might encounter. Testcontainers, on the other hand, provide an actual testing environment, allowing you to test against real dependencies like databases, message brokers, and other services. This results in higher test confidence as you can be more certain that your code will work as expected in a production setting.

## Testing a Sample Order Service using Testcontainers
To demonstrate the power of Testcontainers in Golang, let's create a simple "Order Service" and write tests for it. In our scenario, the Order Service interacts with a MySQL database for storing and retrieving order information.
We'll be using a Testcontainer for MySQL.
Testcontainers automatically downloads the required Docker images, ensuring a seamless setup process without manual intervention.

Here's a simplified code snippet to get you started:

First step is to install Testcontainers in your project,
```bash
go get github.com/testcontainers/testcontainers-go
```

Let's write a basic OrderService
```go
// order_service.go
package order

import (
	"database/sql"
	"fmt"
)

type OrderService struct {
	db *sql.DB
}

func NewOrderService(db *sql.DB) *OrderService {
	return &OrderService{db: db}
}

func (o *OrderService) AddOrder(productID int, quantity int) error {
	_, err := s.db.Exec("INSERT INTO order (product_id, quantity) VALUES (?, ?)", productID, quantity)
	return err
}

func (s *OrderService) GetOrder(id int) (int, int, error) {
	var productID, quantity int
	err := s.db.QueryRow("SELECT product_id, quantity FROM order WHERE id=?", id).Scan(&productID, &quantity)
	if err != nil {
		return 0, 0, err
	}
	return productID, quantity, nil
}
```
And the tests,
```go
// order_service.test.go
package order

import (
	"context"
	"database/sql"
	"fmt"
	"testing"

	"github.com/testcontainers/testcontainers-go"
	"github.com/testcontainers/testcontainers-go/wait"
	_ "github.com/go-sql-driver/mysql"
	"github.com/stretchr/testify/assert"
)

func TestOrderService(t *testing.T) {
	ctx := context.Background()

	// Start a MySQL container
	mysqlContainer, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
		Image:        "mysql:latest",
		ExposedPorts: []string{"3306/tcp"},
		Env: map[string]string{
			"MYSQL_ROOT_PASSWORD": "password",
		},
		WaitingFor: wait.ForLog("port: 3306  MySQL Community Server"),
	})
	assert.NoError(t, err)
	defer mysqlContainer.Terminate(ctx)

	// Get the MySQL container's host and port
	host, err := mysqlContainer.Host(ctx)
	assert.NoError(t, err)
	port, err := mysqlContainer.MappedPort(ctx, "3306")
	assert.NoError(t, err)

	// Create a database connection
	dsn := fmt.Sprintf("root:password@tcp(%s:%s)/order", host, port.Port())
	db, err := sql.Open("mysql", dsn)
	assert.NoError(t, err)
	defer db.Close()

	// Initialize and use the order service
	orderService := NewOrderService(db)

	// Add an order
	err = orderService.AddOrder(100, 2);
	assert.NoError(t, err)

	// Get the added order
	productID, quantity, err := orderService.GetOrder(1)
	assert.NoError(t, err)
	assert.Equal(t, 100, productID)
	assert.Equal(t, 2, quantity)
}

```

## Conclusion
Testcontainers are a game-changer in the world of testing, not only for databases but also for services like Kafka, Redis, Vault and more. They provide a consistent and efficient way to set up testing environments, making integration and end-to-end testing more manageable. By incorporating Testcontainers into your Golang testing toolkit, you can improve the reliability and quality of your code, ensuring that your applications run smoothly in real-world scenarios.

## References

- [Testcontainers Official Website](https://testcontainers.com/)
- [GitHub Repository for Testcontainers for Go](https://github.com/testcontainers/testcontainers-go)