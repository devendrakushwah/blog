---
title: "Efficient Go Testing with Testcontainers"
description: "Learn how to efficiently do integration testing in Go using Testcontainers. This blog walks you through setting up realistic testing environments, demonstrating power of Testcontainers"
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
Testing is an essential part of software development, but it often involves complex setups to ensure the reliability of our applications. In this guide, we will explore how Testcontainers can revolutionize testing in Golang. Whether you are a seasoned developer or just getting started with Go, Testcontainers can simplify and enhance your testing workflow.

## What Are Testcontainers?
Testcontainers is a powerful open-source tool that simplifies the creation of isolated and reproducible testing environments. It leverages Docker to spin up containers within your tests, offering a consistent and controlled environment. This means you can easily test your code against dependencies like databases, message brokers, or other external services without the headache of complex setup and teardown procedures.

## When Should We Use Testcontainers?
Testcontainers are invaluable in scenarios where your application relies on external services or dependencies that can impact the functionality of your code. Use Testcontainers when:

- Testing against a real database: Ensure that your database-related code works as expected without the need for a dedicated testing database.
- Testing microservices: Testcontainers can create isolated environments for each microservice, allowing you to validate their interactions.
- Running integration tests: Ensure that all components of your application work seamlessly together in a real-world environment.

## Testcontainers vs Mocks
Testcontainers offer a significant advantage over traditional mocking techniques. While mocks can simulate behavior to a certain extent, they often fall short in replicating the real-world scenarios and edge cases that an application might encounter. Testcontainers, on the other hand, provide an actual testing environment, allowing you to test against real dependencies like databases, message brokers, and other services. This results in higher test confidence as you can be more certain that your code will work as expected in a production setting.


## Building a Sample Book Service
To demonstrate the power of Testcontainers in Golang, let's create a simple "Book Service" and write tests for it. In our scenario, the Book Service interacts with a MySQL database for storing and retrieving book information.
Notably, Testcontainers automatically downloads the required Docker images, ensuring a seamless setup process without manual intervention.

Here's a simplified code snippet to get you started:

```go
// book_service.go
package service

import (
	"database/sql"
	"fmt"
)

type BookService struct {
	db *sql.DB
}

func NewBookService(db *sql.DB) *BookService {
	return &BookService{db: db}
}

func (s *BookService) AddBook(title string, author string) error {
	_, err := s.db.Exec("INSERT INTO books (title, author) VALUES (?, ?)", title, author)
	return err
}

func (s *BookService) GetBook(id int) (string, string, error) {
	var title, author string
	err := s.db.QueryRow("SELECT title, author FROM books WHERE id=?", id).Scan(&title, &author)
	if err != nil {
		return "", "", err
	}
	return title, author, nil
}
```
And the tests,
```go
// book_service.test.go
package service

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

func TestBookService(t *testing.T) {
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
	dsn := fmt.Sprintf("root:password@tcp(%s:%s)/books", host, port.Port())
	db, err := sql.Open("mysql", dsn)
	assert.NoError(t, err)
	defer db.Close()

	// Initialize and use the book service
	bookService := NewBookService(db)

	// Add a book
	err = bookService.AddBook("The Catcher in the Rye", "J.D. Salinger")
	assert.NoError(t, err)

	// Get the added book
	title, author, err := bookService.GetBook(1)
	assert.NoError(t, err)
	assert.Equal(t, "The Catcher in the Rye", title)
	assert.Equal(t, "J.D. Salinger", author)
}

```

## Conclusion
Testcontainers are a game-changer in the world of testing, not only for databases but also for services like Kafka, MongoDB, and more. They provide a consistent and efficient way to set up testing environments, making integration and end-to-end testing more manageable. By incorporating Testcontainers into your Golang testing toolkit, you can improve the reliability and quality of your code, ensuring that your applications run smoothly in real-world scenarios.