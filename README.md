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

### Docker image for broker service

```yaml
# base go image
FROM golang:1.18-alpine as builder

RUN mkdir /app

COPY . /app

WORKDIR /app

RUN CGO_ENABLED=0 go build -o brokerApp ./cmd/api

RUN chmod +x /app/brokerApp

# build a tiny docker image
FROM alpine:latest

RUN mkdir /app

COPY --from=builder /app/brokerApp /app

CMD [ "/app/brokerApp" ]
```

```yaml
services:
  broker-service:
    build:
      context: ./../broker-service
      dockerfile: ./../broker-service/broker-service.dockerfile
    restart: always
    ports:
      - "8080:80"
    deploy:
      mode: replicated
      replicas: 1
```

```sh
docker-compose up
```

### Json helper

```go
package main

import (
	"encoding/json"
	"errors"
	"io"
	"net/http"
)

type jsonResponse struct {
	Error   bool   `json:"error"`
	Message string `json:"message"`
	Data    any    `json:"data,omitempty"`
}

// readJSON tries to read the body of a request and converts it into JSON
func (app *Config) readJSON(w http.ResponseWriter, r *http.Request, data any) error {
	maxBytes := 1048576 // one megabyte

	r.Body = http.MaxBytesReader(w, r.Body, int64(maxBytes))

	dec := json.NewDecoder(r.Body)
	err := dec.Decode(data)
	if err != nil {
		return err
	}

	err = dec.Decode(&struct{}{})
	if err != io.EOF {
		return errors.New("body must have only a single JSON value")
	}

	return nil
}

// writeJSON takes a response status code and arbitrary data and writes a json response to the client
func (app *Config) writeJSON(w http.ResponseWriter, status int, data any, headers ...http.Header) error {
	out, err := json.Marshal(data)
	if err != nil {
		return err
	}

	if len(headers) > 0 {
		for key, value := range headers[0] {
			w.Header()[key] = value
		}
	}

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	_, err = w.Write(out)
	if err != nil {
		return err
	}

	return nil
}

// errorJSON takes an error, and optionally a response status code, and generates and sends
// a json error response
func (app *Config) errorJSON(w http.ResponseWriter, err error, status ...int) error {
	statusCode := http.StatusBadRequest

	if len(status) > 0 {
		statusCode = status[0]
	}

	var payload jsonResponse
	payload.Error = true
	payload.Message = err.Error()

	return app.writeJSON(w, statusCode, payload)
}
```

## Authentication service

### Stub authentication service
