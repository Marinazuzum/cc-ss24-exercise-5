# Build stage
FROM golang:1.22 AS builder
WORKDIR /app

COPY . .
RUN go mod tidy

# Force static build for amd64 architecture
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o api_post_books_service ./main.go

# Final image
FROM debian:bullseye-slim
WORKDIR /app

# Copy only the built binary from the builder stage
COPY --from=builder /app/api_post_books_service .

EXPOSE 3002
CMD ["./api_post_books_service"]
