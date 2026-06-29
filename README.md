# Storyloom Service Registry

A **Eureka Service Registry** server for the Storyloom microservices platform. This service acts as the central discovery server, allowing microservices to register themselves and discover other services at runtime without hard-coded hostnames and ports.

---

## Overview

The Storyloom Service Registry is built with **Spring Boot 4.0.6** and **Spring Cloud Netflix Eureka Server**. It provides a REST-based service registry where all Storyloom microservices register on startup and query to locate their peers.

By decoupling service discovery from individual services, the registry enables:

- **Dynamic scaling** — new service instances register automatically
- **Load balancing** — clients discover all available instances of a service
- **Resilience** — the registry can detect and evict unhealthy instances via heartbeats
- **Loose coupling** — services don't need to know each other's network locations

---

## Tech Stack

| Technology          | Version         | Purpose                         |
| ------------------- | --------------- | ------------------------------- |
| Java                | 21              | Runtime language                |
| Spring Boot         | 4.0.6           | Application framework           |
| Spring Cloud        | 2025.1.1        | Microservices tooling           |
| Netflix Eureka      | (managed by Spring Cloud) | Service registration & discovery |
| Spring Web MVC      | (starter)       | Web endpoints for the Eureka UI |
| Maven               | —               | Build & dependency management   |

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│            Storyloom Platform                    │
│                                                  │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   │
│  │ Service A │   │ Service B │   │ Service C │   │
│  │ (Client)  │   │ (Client)  │   │ (Client)  │   │
│  └─────┬─────┘   └─────┬─────┘   └─────┬─────┘   │
│        │               │               │          │
│        └───────────────┼───────────────┘          │
│                        │ Register / Discover      │
│                        ▼                          │
│           ┌────────────────────────┐              │
│           │   EUREKA SERVER        │              │
│           │   (this project)       │              │
│           │   Port: 8761            │              │
│           └────────────────────────┘              │
│                                                  │
└─────────────────────────────────────────────────┘
```

---

## Project Structure

```
storyloom-service-registry/
├── src/
│   ├── main/
│   │   ├── java/com/example/storyloom_service_registry/
│   │   │   └── StoryloomServiceRegistryApplication.java   # Entry point with @EnableEurekaServer
│   │   └── resources/
│   │       └── application.properties                     # Eureka & server configuration
│   └── test/
│       └── java/com/example/storyloom_service_registry/
│           └── StoryloomServiceRegistryApplicationTests.java
├── .gitignore
├── .mvn/
│   └── wrapper/
│       └── maven-wrapper.properties
├── mvnw / mvnw.cmd            # Maven Wrapper scripts
└── pom.xml                     # Maven build configuration
```

---

## Configuration

The service is configured in `src/main/resources/application.properties`:

| Property                              | Value                          | Description                                                  |
| ------------------------------------- | ------------------------------ | ------------------------------------------------------------ |
| `spring.application.name`             | `storyloom-service-registry`   | Logical name of this service                                  |
| `server.port`                         | `8761`                         | HTTP port (standard Eureka default)                           |
| `eureka.instance.hostname`           | `localhost`                    | Hostname the registry advertises                             |
| `eureka.client.fetch-registry`        | `false`                        | This server does **not** fetch the registry from a peer       |
| `eureka.client.register-with-eureka` | `false`                        | This server does **not** register itself as a client          |

> **Note:** `fetch-registry` and `register-with-eureka` are set to `false` because this is a **standalone** registry. In a production HA setup with multiple Eureka peers, both would be set to `true` and each server would replicate with the others.

---

## Getting Started

### Prerequisites

- **Java 21** or higher installed
- **Maven** (or use the included Maven Wrapper)

### Running the Service

**Using Maven Wrapper (recommended):**

```bash
# Linux / macOS
./mvnw spring-boot:run

# Windows
mvnw.cmd spring-boot:run
```

**Using Maven directly:**

```bash
mvn spring-boot:run
```

**Building the JAR first, then running:**

```bash
mvn clean package
java -jar target/storyloom-service-registry-0.0.1-SNAPSHOT.jar
```

Once started, the Eureka dashboard will be available at:

👉 **http://localhost:8761**

---

## Eureka Dashboard

After starting the registry, open your browser and navigate to:

```
http://localhost:8761
```

You'll see the Eureka Server dashboard, which displays:

- **DS Replicas** — peer Eureka servers (if any)
- **Instances currently registered with Eureka** — a list of all microservices that have registered themselves
- **General Info** — server status, memory, and environment details

Initially the list will be empty. As you start other Storyloom microservices configured to use this registry, they will appear in the dashboard.

---

## Registering a Client Service

To register another Spring Boot microservice with this registry, add the following:

### 1. Dependency (in the client's `pom.xml`)

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

### 2. Configuration (in the client's `application.properties`)

```properties
spring.application.name=my-storyloom-service
eureka.client.service-url.defaultZone=http://localhost:8761/eureka
```

The client will automatically register with the Eureka server on startup.

---

## Running Tests

```bash
# Using Maven Wrapper
./mvnw test        # Linux / macOS
mvnw.cmd test      # Windows

# Using Maven directly
mvn test
```

---

## API Endpoints

The Eureka Server exposes the following REST endpoints:

| Endpoint                              | Method | Description                                    |
| ------------------------------------- | ------ | ---------------------------------------------- |
| `/eureka/apps`                        | GET    | List all registered applications               |
| `/eureka/apps/{appId}`                | GET    | Get instances of a specific application        |
| `/eureka/apps/{appId}/{instanceId}`   | GET    | Get details of a specific instance             |
| `/eureka/apps/{appId}/{instanceId}`   | DELETE | Deregister a specific instance                 |
| `/eureka/apps/{appId}/{instanceId}`   | PUT    | Send a heartbeat (renew) for an instance        |

---

## Production Considerations

For a production deployment, consider:

1. **High Availability** — Run multiple Eureka server instances in a peer-to-peer cluster so the registry itself is not a single point of failure.
2. **Securing the Registry** — Add Spring Security to protect the Eureka endpoints with authentication.
3. **Self-Preservation Mode** — Eureka enters self-preservation when too many instances expire at once (e.g., during a network partition) to prevent mass deregistration. Understand and tune this threshold for your environment.
4. **Heartbeat & Eviction** — Default heartbeat interval is 30 seconds; instances not sending heartbeats for 90 seconds are evicted. Adjust `eureka.server.lease-renewal-interval-in-seconds` and `eureka.server.lease-expiration-duration-in-seconds` as needed.

---

## Troubleshooting

| Issue                                           | Solution                                                                                   |
| ----------------------------------------------- | ------------------------------------------------------------------------------------------ |
| Port 8761 already in use                         | Another Eureka instance is running. Kill it or change `server.port` in `application.properties`. |
| Clients not appearing in the dashboard           | Verify the client's `eureka.client.service-url.defaultZone` points to `http://localhost:8761/eureka`. |
| Self-preservation mode warnings in logs          | Normal behavior when instances expire. If persistent, check network connectivity between services. |
| `com.example.storyloom_service_registry` error  | The original package `com.example.storyloom-service-registry` is invalid (hyphens are not allowed). The corrected package is used instead. |

---

## License

This project is part of the **Storyloom** platform. See the root project repository for license details.

---

## Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/my-feature`)
3. Commit your changes (`git commit -m 'Add my feature'`)
4. Push to the branch (`git push origin feature/my-feature`)
5. Open a Pull Request
