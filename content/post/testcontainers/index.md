---
title: "Level Up Your Go Testing : Dive into Test Containers"
description: "Learn how to efficiently do integration testing in Go using Testcontainers and increase tests confidence. This blog walks you through setting up a realistic testing environment, demonstrating usage of Testcontainers"
date: 2023-09-10
image: banner.png
categories:
  - Go
tags:
  - go
  - testcontainers
---

# Level Up Your Go Testing : Dive into Test Containers

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
package testcontainers

import "database/sql"

type OrderService struct {
	db *sql.DB
}

func NewOrderService(db *sql.DB) *OrderService {
	return &OrderService{db: db}
}

func (s *OrderService) AddOrder(productID int, quantity int) error {
	_, err := s.db.Exec("INSERT INTO orders (product_id, quantity) VALUES (?, ?)", productID, quantity)
	return err
}

func (s *OrderService) GetOrder(id int) (int, int, error) {
	var productID, quantity int
	err := s.db.QueryRow("SELECT product_id, quantity FROM orders WHERE id=?", id).Scan(&productID, &quantity)
	if err != nil {
		return 0, 0, err
	}
	return productID, quantity, nil
}

```
And the tests,
```go
// order_service_test.go
package testcontainers

import (
	"context"
	"database/sql"
	"fmt"
	"testing"

	_ "github.com/go-sql-driver/mysql" // mysql driver import needed
	"github.com/stretchr/testify/assert"
	"github.com/testcontainers/testcontainers-go"
	"github.com/testcontainers/testcontainers-go/wait"
)

func TestOrderService(t *testing.T) {
	ctx := context.Background()

	// Start a MySQL container
	request := testcontainers.ContainerRequest{
		Image:        "mysql:5.7.43",
		ExposedPorts: []string{"3306/tcp"},
		Env: map[string]string{
			"MYSQL_ROOT_PASSWORD": "password",
		},
		WaitingFor: wait.ForLog("port: 3306  MySQL Community Server"),
	}
	mysqlContainer, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
		ContainerRequest: request,
		Started:          true,
	})

	assert.NoError(t, err)
	defer mysqlContainer.Terminate(ctx)

	// Get the MySQL container's host and port
	host, err := mysqlContainer.Host(ctx)
	assert.NoError(t, err)
	port, err := mysqlContainer.MappedPort(ctx, "3306")
	assert.NoError(t, err)

	// Create a database connection
	dsn := fmt.Sprintf("root:password@tcp(%s:%s)/", host, port.Port())
	db, err := sql.Open("mysql", dsn)
	assert.NoError(t, err)
	defer db.Close()

	// Create database and table
	createDatabaseAndTable(db)

	// Initialize and use the order service
	orderService := NewOrderService(db)

	// Add an order
	err = orderService.AddOrder(100, 2)
	assert.NoError(t, err)

	// Get the added order
	productID, quantity, err := orderService.GetOrder(1)
	assert.NoError(t, err)
	assert.Equal(t, 100, productID)
	assert.Equal(t, 2, quantity)
}

func createDatabaseAndTable(db *sql.DB) {
	db.Exec(`CREATE DATABASE IF NOT EXISTS orders;`)
	db.Exec(`USE orders;`)
	db.Exec(`CREATE TABLE IF NOT EXISTS orders ( id INT AUTO_INCREMENT PRIMARY KEY, product_id INT, quantity INT);`)
}
```
Test output
```bash
=== RUN   TestOrderService
2023/09/12 23:41:58 github.com/testcontainers/testcontainers-go - Connected to docker: 
  Server Version: 24.0.6
  API Version: 1.43
  Operating System: Docker Desktop
  Total Memory: 7854 MB
  Resolved Docker Host: unix:///var/run/docker.sock
  Resolved Docker Socket Path: /var/run/docker.sock
2023/09/12 23:42:10 🐳 Creating container for image docker.io/testcontainers/ryuk:0.5.1
2023/09/12 23:42:10 ✅ Container created: f8adf3d96d20
2023/09/12 23:42:10 🐳 Starting container: f8adf3d96d20
2023/09/12 23:42:11 ✅ Container started: f8adf3d96d20
2023/09/12 23:42:11 🚧 Waiting for container id f8adf3d96d20 image: docker.io/testcontainers/ryuk:0.5.1. Waiting for: &{Port:8080/tcp timeout:<nil> PollInterval:100ms}
2023/09/12 23:42:11 🐳 Creating container for image mysql:5.7.43
2023/09/12 23:42:11 ✅ Container created: 327036e1fd9b
2023/09/12 23:42:11 🐳 Starting container: 327036e1fd9b
2023/09/12 23:42:11 ✅ Container started: 327036e1fd9b
2023/09/12 23:42:11 🚧 Waiting for container id 327036e1fd9b image: mysql:5.7.43. Waiting for: &{timeout:<nil> Log:port: 3306  MySQL Community Server Occurrence:1 PollInterval:100ms}
2023/09/12 23:42:23 🐳 Terminating container: 327036e1fd9b
2023/09/12 23:42:24 🚫 Container terminated: 327036e1fd9b
--- PASS: TestOrderService (25.62s)
PASS
```
## Conclusion
Testcontainers are a game-changer in the world of testing, not only for databases but also for services like Kafka, Redis, Vault and more. They provide a consistent and efficient way to set up testing environments, making integration and end-to-end testing more manageable. By incorporating Testcontainers into your Golang testing toolkit, you can improve the reliability and quality of your code, ensuring that your applications run smoothly in real-world scenarios.

Happy testing! 💣💥

## References

- [Testcontainers Official Website](https://testcontainers.com/)
- [GitHub Repository for Testcontainers for Go](https://github.com/testcontainers/testcontainers-go)