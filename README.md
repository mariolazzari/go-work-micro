# Working with microservices in Go

## Introduction

### Monolithic

- Client
- Server with all logic
  - Auth
  - Mail
  - Logs
  - Logic
- DB + cache

### Microservices

Braking monolith in complete separated smaller packages and communicate between theme via Json/Rest, gRpc, Rpc: easier to mantain, harder to write

- Auth -> Postgres
- Logs -> Mongo -> RabbitMQ
- Mail -> Logs
- Front1
- Fornt2

## Simple frontend with one microservice

### Frontend setup

```go
package main

import (
	"fmt"
	"html/template"
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		render(w, "test.page.gohtml")
	})

	fmt.Println("Starting front end service on port 80")
	err := http.ListenAndServe(":80", nil)
	if err != nil {
		log.Panic(err)
	}
}

func render(w http.ResponseWriter, t string) {
	partials := []string{
		"./cmd/web/templates/base.layout.gohtml",
		"./cmd/web/templates/header.partial.gohtml",
		"./cmd/web/templates/footer.partial.gohtml",
	}

	var templateSlice []string
	templateSlice = append(templateSlice, fmt.Sprintf("./cmd/web/templates/%s", t))

	for _, x := range partials {
		templateSlice = append(templateSlice, x)
	}

	tmpl, err := template.ParseFiles(templateSlice...)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	if err := tmpl.Execute(w, nil); err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
	}
}
```

### Broker service

```sh
go mod init github.com/mariolazzari/go-work-micro/broker-service
go get github.com/go-chi/chi/v5
go get github.com/go-chi/chi/v5/middleware
go get github.com/go-chi/chi/cors
```
