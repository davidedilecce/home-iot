# Home IoT Platform

<img src="images/smart-home.png" alt="Smart Home" width="150px"/>

## Overview

**Home IoT** is a comprehensive, open-source Internet of Things (IoT) platform designed for intelligent home automation and telemetry management. It provides a modular microservices architecture that enables real-time collection, processing, and analysis of sensor data from Zigbee-enabled IoT devices. The platform supports seamless integration with smart home devices, asynchronous event processing, centralized user management, and a secure, scalable API layer.

Built on a modern cloud-native stack using Spring Boot and Spring Cloud, Home IoT is ideal for developers, system integrators, and IoT enthusiasts who want a flexible, production-ready solution to:

- Acquire telemetry data from Zigbee-based sensors in real-time
- Process and store events from distributed IoT devices
- Manage users and permissions with centralized authentication
- Query and retrieve the latest sensor readings with low latency
- Scale horizontally to support large deployments

---

## Technology Stack

The Home IoT platform is built using industry-leading open-source technologies:

| Component | Technology |
|-----------|-----------|
| **Language** | Java 21 |
| **Framework** | Spring Boot 4.0.1, Spring Cloud 2025.1.0 |
| **Build Tool** | Gradle (Kotlin DSL) |
| **Service Discovery** | Netflix Eureka |
| **Configuration Management** | Spring Cloud Config Server |
| **API Gateway** | Spring Cloud Gateway (WebFlux/WebMVC) |
| **Message Broker** | RabbitMQ (asynchronous event streaming) |
| **IoT Protocol** | MQTT (Zigbee2MQTT integration) |
| **Security** | Spring Security |
| **Project Grouping** | Git Submodules for independent versioning |

---

## Architecture Overview

Home IoT follows a **microservices architecture** pattern with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────────┐
│                     Client Applications                       │
│              (Mobile, Web, IoT Devices)                       │
└────────────────────────┬────────────────────────────────────┘
                         │
         ┌───────────────▼───────────────┐
         │   API Gateway (Spring Cloud)   │
         │   - Request Routing             │
         │   - Authentication              │
         │   - Rate Limiting               │
         └───────────────┬───────────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
    ┌───▼────┐   ┌──────▼──────┐   ┌─────▼────┐
    │ Auth   │   │  Domain     │   │Telemetry │
    │Service │   │ Microsvcs   │   │ Microsvcs│
    └────────┘   └─────────────┘   └──────────┘
        │                │                │
        └────────────────┼────────────────┘
                         │
        ┌────────────────▼────────────────┐
        │  Message Broker (RabbitMQ)      │
        │  - Asynchronous Event Streaming │
        │  - Service-to-Service Comm.     │
        └────────────────┬────────────────┘
                         │
        ┌────────────────▼────────────────┐
        │ Service Registry (Eureka)       │
        │ - Service Discovery             │
        │ - Health Monitoring             │
        └─────────────────────────────────┘
```

---

## UI - iOS

Below is an overview of the main screens in the Home IoT iOS application.

---

### 1. Login Page
<img src="images/login.png" alt="Login" width="200px"/>

---

### 2. Apartments List
<img src="images/apartments-list.png" alt="Apartments List" width="200px"/>

---

### 3. Apartment Detail
<img src="images/apartment-details-4.png" alt="Apartment Detail" width="200px"/>

---

### 4. Water Leak Alarm
<img src="images/apartment-details.png" alt="Water Leak Alarm" width="200px"/>

---

### 5. Smoke Alarm
<img src="images/apartment-details-3.png" alt="Smoke Alarm" width="200px"/>

---

### 6. No Alarm
<img src="images/apartment-details-2.png" alt="No Alarm" width="200px"/>

---

### 7. Events
<img src="images/events-list.png" alt="Events" width="200px"/>

---

### 8. Profile
<img src="images/profile.png" alt="Profile" width="200px"/>


## Microservices Architecture

### 1. **Infrastructure & Platform Services**

#### `home-iot-service-registry`
- **Purpose:** Central service discovery using Netflix Eureka
- **Responsibilities:**
  - Maintains a dynamic registry of all active microservices
  - Enables client-side service discovery
  - Provides health check monitoring for service instances
- **Technology:** Spring Boot, Eureka Server
- **Deployment:** Runs as a singleton service, accessed by all other services

#### `home-iot-config-server`
- **Purpose:** Centralized configuration management for the entire platform
- **Responsibilities:**
  - Distributes configuration profiles to all microservices
  - Supports environment-specific configurations (dev, staging, production)
  - Enables dynamic configuration refresh without service restart
  - Version-controlled configuration through Git backend
- **Technology:** Spring Cloud Config Server
- **Configuration Files:** Located in `home-iot-config/` directory
  - `application.yml` - Global logging and management settings
  - Service-specific configs: `auth-server.yml`, `ms-apartment.yml`, `ms-event.yml`, etc.

### 2. **API Gateway & Entry Point**

#### `home-iot-api-gateway`
- **Purpose:** Central API Gateway - the single entry point for all external client requests
- **Responsibilities:**
  - Route incoming API requests to appropriate microservices based on request path
  - Handle cross-cutting concerns: authentication, authorization, rate limiting
  - Support both synchronous (REST) and asynchronous (RabbitMQ-based) communication patterns
  - Aggregate responses from multiple services for complex queries
  - WebFlux for non-blocking, reactive request handling
- **Technology:** Spring Cloud Gateway, WebFlux/WebMVC, Eureka Client
- **Communication Patterns:**
  - **REST:** Direct synchronous calls to domain microservices
  - **Async:** Publishes telemetry queries to RabbitMQ for event-driven processing
- **Security:** Authenticates all incoming requests before routing

### 3. **Authentication & Security**

#### `home-iot-auth-server`
- **Purpose:** Centralized authentication and authorization service
- **Responsibilities:**
  - Issues and validates authentication tokens (JWT or session-based)
  - Manages user credentials securely
  - Enforces role-based access control (RBAC)
  - Integrates with all microservices for authorization checks
  - Manages password policies and security configurations
- **Technology:** Spring Boot, Spring Security, Eureka Client
- **Service Registration:** Registers with Eureka for discoverability
- **Notes:** Critical component for platform security and multi-tenant support

### 4. **Domain Microservices**

#### `home-iot-ms-apartment`
- **Purpose:** Manages apartment/property-level entities and telemetry aggregation
- **Responsibilities:**
  - CRUD operations for apartment/property entities
  - Aggregates telemetry data from devices within an apartment
  - Retrieves latest sensor readings for devices
  - Publishes apartment-related events asynchronously via RabbitMQ
  - Handles queries for apartment status and device mappings
- **Technology:** Spring Boot, Spring Security, RabbitMQ Client, Eureka Client
- **Data Model:** Apartments, rooms, device mappings, aggregated telemetry
- **Event Publishing:** Emits events when apartments or device states change

#### `home-iot-ms-user`
- **Purpose:** User account and profile management service
- **Responsibilities:**
  - User registration and account creation
  - User profile management (name, email, preferences)
  - User role and permission assignments
  - Integration with auth server for credential management
  - User data persistence and retrieval
- **Technology:** Spring Boot, Spring Security, JPA/Hibernate, Eureka Client
- **Service Registration:** Registers with Eureka for service discovery
- **Security:** Spring Security integration for role-based access

#### `home-iot-ms-events`
- **Purpose:** Event management and event sourcing service
- **Responsibilities:**
  - Consumes events from RabbitMQ published by other microservices
  - Stores events for audit trails and historical analysis
  - Provides event query and filtering APIs
  - Event deduplication and processing
  - Real-time event notification to subscribed clients
- **Technology:** Spring Boot, Spring Cloud Stream, RabbitMQ
- **Deployment:** Independent deployment as a Git submodule
- **Event Sources:** Receives events from telemetry services, apartment service, user service
- **Scalability:** Handles high-volume event streams with asynchronous processing

#### `home-iot-ms-gateway`
- **Purpose:** Additional gateway service for telemetry and sensor data routing
- **Responsibilities:**
  - Provides supplementary APIs for telemetry queries
  - Acts as a facade for complex multi-service queries
  - Publishes telemetry-related events to RabbitMQ
  - Handles device discovery and metadata queries
  - May serve as a secondary gateway for specific device types
- **Technology:** Spring Boot, RabbitMQ Client, Eureka Client
- **Deployment:** Independent module (Git submodule reference)

### 5. **Telemetry Microservices**

#### `home-iot-ms-telemetry-reader`
- **Purpose:** Real-time telemetry data ingestion and query service
- **Responsibilities:**
  - Read telemetry data from sensors (via MQTT, REST APIs, or message queues)
  - Validate and normalize sensor data formats
  - Store telemetry readings with timestamps
  - Provide REST APIs to query historical telemetry data
  - Support range queries, aggregations, and filtering
  - Publish telemetry events asynchronously to RabbitMQ for downstream processing
  - Expose APIs for retrieving latest sensor values
- **Technology:** Spring Boot, Spring Data, Time-series data support, RabbitMQ Client
- **Data Model:** Time-series sensor readings with metadata
- **Performance:** Optimized for high-volume read operations and time-range queries
- **Event Publishing:** Continuously emits sensor reading events

#### `home-iot-ms-telemetry-writer`
- **Purpose:** Telemetry data persistence and write optimization
- **Responsibilities:**
  - Accept telemetry data from external IoT devices or forwarders
  - Write telemetry data to persistent storage (database)
  - Handle data validation, transformation, and enrichment
  - Publish write confirmation events to RabbitMQ
  - Implement data buffering and batch operations for efficiency
  - Manage data retention policies
- **Technology:** Spring Boot, Spring Data, Database drivers, RabbitMQ Client
- **Deployment:** Independent microservice (Git submodule)
- **Data Persistence:** Direct database writes for durability
- **Event Publishing:** Emits events for write completions and errors

### 6. **IoT Device Integration**

#### `home-iot-zigbee2mqtt-forwarder`
- **Purpose:** Bridge between Zigbee devices and the Home IoT platform via MQTT
- **Responsibilities:**
  - Subscribe to Zigbee2MQTT MQTT broker topics
  - Receive real-time telemetry from Zigbee sensors and devices
  - Format and transform Zigbee device data to platform format
  - Forward telemetry data to the telemetry microservices
  - Handle device discovery announcements from Zigbee network
  - Manage device lifecycle events (join, leave, status updates)
- **Technology:** Python, MQTT Client, REST APIs
- **Deployment:** Standalone Python application (containerized via Docker)
- **Communication Protocol:** 
  - MQTT for Zigbee device communication
  - REST/HTTP to publish data to telemetry services
- **File:** `zigbee2mqtt_forwarder.py`
- **Docker:** Includes `Dockerfile` for containerization
- **Configuration:** Environment variables in `.env` file

---

## Communication Patterns

### Synchronous Communication
- **REST APIs:** Direct HTTP calls between services via API Gateway
- **Service-to-Service:** Using Eureka for service discovery
- **Use Cases:** User queries, device status checks, configuration retrieval

### Asynchronous Communication
- **RabbitMQ:** Event-driven messaging for decoupled services
- **Publish-Subscribe:** Services publish events, others subscribe and process
- **Use Cases:** Telemetry events, device alerts, audit logging, data aggregation

### Service Discovery
- **Eureka Client:** All services register their health and location
- **Load Balancing:** Spring Cloud provides client-side load balancing
- **Resilience:** Services can handle temporary unavailability of other services

---

## Data Flow Example: Telemetry Ingestion

```
1. Zigbee Device → Zigbee2MQTT Network
                      ↓
2. MQTT Forwarder → Consumes MQTT messages
                      ↓
3. Telemetry Writer → Receives and persists data
                      ↓
4. RabbitMQ → Publishes telemetry events
                      ↓
5. Event Service → Consumes and stores events
   Apartment Service → Aggregates data
                      ↓
6. API Gateway ← Query requests
                      ↓
7. Client Application ← Telemetry data
```

---

## Project Structure

```
home-iot/
├── README.md                                 # This file
├── home-iot-config/                         # Centralized configuration files
│   ├── application.yml                      # Global config
│   ├── auth-server.yml                      # Auth service config
│   ├── ms-apartment.yml                     # Apartment service config
│   ├── ms-event.yml                         # Events service config
│   ├── ms-gateway.yml                       # Gateway service config
│   └── ... (other service configs)
├── home-iot-service-registry/               # Eureka Service Registry
│   └── src/main/java/...
├── home-iot-config-server/                  # Spring Cloud Config Server
│   └── src/main/java/...
├── home-iot-api-gateway/                    # API Gateway (Spring Cloud Gateway)
│   └── src/main/java/...
├── home-iot-auth-server/                    # Authentication & Authorization
│   ├── src/main/java/...
│   └── src/test/java/...
├── home-iot-ms-apartment/                   # Apartment Domain Service
│   ├── src/main/java/...
│   └── src/test/java/...
├── home-iot-ms-user/                        # User Management Service
│   ├── src/main/java/...
│   └── src/test/java/...
├── home-iot-ms-events/                      # Event Management Service (Submodule)
│   └── src/main/java/...
├── home-iot-ms-gateway/                     # Telemetry Gateway Service (Submodule)
│   └── src/main/java/...
├── home-iot-ms-telemetry-reader/            # Telemetry Reader Service (Submodule)
│   └── src/main/java/...
├── home-iot-ms-telemetry-writer/            # Telemetry Writer Service (Submodule)
│   └── src/main/java/...
├── home-iot-zigbee2mqtt-forwarder/          # Zigbee2MQTT Forwarder (Python)
│   ├── zigbee2mqtt_forwarder.py             # Main forwarder application
│   ├── Dockerfile                           # Docker container configuration
│   └── env                                  # Environment variables
└── images/                                  # Documentation images
    ├── smart-home.png
    ├── apartments-list.png
    ├── apartment-details.png
    └── ... (UI screenshots)
```

---

## Getting Started

### Prerequisites

- **Java 21** or higher
- **Gradle** (included via gradle wrapper)
- **Docker & Docker Compose** (for local development)
- **Git** (for cloning and submodule management)

### Building the Project

1. **Clone the repository with submodules:**
   ```bash
   git clone --recursive https://github.com/username/home-iot.git
   cd home-iot
   ```

2. **Build all microservices:**
   ```bash
   ./gradlew build
   ```

3. **Build a specific service:**
   ```bash
   cd home-iot-api-gateway
   ./gradlew build
   ```

### Running Services Locally

#### Option 1: Docker Compose (Recommended)
```bash
docker-compose up -d
```

#### Option 2: Running Services Individually
```bash
# Terminal 1: Service Registry
cd home-iot-service-registry
./gradlew bootRun

# Terminal 2: Config Server
cd home-iot-config-server
./gradlew bootRun

# Terminal 3: Auth Server
cd home-iot-auth-server
./gradlew bootRun

# Terminal 4: API Gateway
cd home-iot-api-gateway
./gradlew bootRun

# Terminal 5+: Domain services
cd home-iot-ms-apartment
./gradlew bootRun
```

### Accessing the Platform

- **API Gateway:** http://localhost:8080
- **Service Registry (Eureka):** http://localhost:8761
- **Config Server:** http://localhost:8888

---

## API Endpoints

### Authentication
- `POST /auth/login` - User login
- `POST /auth/logout` - User logout
- `POST /auth/refresh` - Refresh authentication token

### Users
- `GET /users` - List all users
- `GET /users/{id}` - Get user details
- `POST /users` - Create new user
- `PUT /users/{id}` - Update user
- `DELETE /users/{id}` - Delete user

### Apartments
- `GET /apartments` - List apartments
- `GET /apartments/{id}` - Get apartment details
- `GET /apartments/{id}/devices` - Get devices in apartment
- `GET /apartments/{id}/telemetry` - Get apartment telemetry

### Telemetry
- `GET /telemetry/{deviceId}` - Get device telemetry
- `GET /telemetry/{deviceId}/latest` - Get latest reading
- `GET /telemetry/range` - Query telemetry by time range
- `POST /telemetry` - Submit telemetry data

### Events
- `GET /events` - List events
- `GET /events/{id}` - Get event details
- `GET /events/device/{deviceId}` - Get device events

---

## Development & Deployment

### Environment Variables

Key environment variables for configuration:

```bash
EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://localhost:8761/eureka/
SPRING_CLOUD_CONFIG_URI=http://localhost:8888
RABBITMQ_HOST=localhost
RABBITMQ_PORT=5672
MQTT_BROKER_URL=tcp://localhost:1883
DATABASE_URL=jdbc:mysql://localhost:3306/home_iot
DATABASE_USERNAME=root
DATABASE_PASSWORD=password
```

### Database Setup

```sql
CREATE DATABASE home_iot;
CREATE USER 'iot_user'@'localhost' IDENTIFIED BY 'secure_password';
GRANT ALL PRIVILEGES ON home_iot.* TO 'iot_user'@'localhost';
FLUSH PRIVILEGES;
```

### Testing

Run unit and integration tests:
```bash
./gradlew test
```

Run tests for a specific service:
```bash
cd home-iot-ms-apartment
./gradlew test
```

---

## Monitoring & Logging

- **Service Health:** Access Eureka dashboard at `http://localhost:8761`
- **Logs:** Configured via `application.yml` with logging level set to INFO
- **Actuator Endpoints:** All services expose `/actuator/health` and `/actuator/metrics`
- **RabbitMQ Management:** Access at `http://localhost:15672` (default credentials: guest/guest)

---

## Git Submodules

Several microservices are maintained as Git submodules for independent versioning:

- `home-iot-ms-events`
- `home-iot-ms-gateway`
- `home-iot-ms-telemetry-reader`
- `home-iot-ms-telemetry-writer`

To update submodules:
```bash
git submodule update --remote
```

---

## Contributing

We welcome contributions! Please follow these guidelines:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

---

## License

This project is licensed under the MIT License - see the LICENSE file for details.

---

## Support & Contact

For issues, questions, or suggestions:
- Open an issue on GitHub
- Contact the development team
- Check documentation in the `docs/` directory

---

## Roadmap

- [ ] Mobile app for iOS and Android
- [ ] Advanced analytics and dashboards
- [ ] Machine learning for anomaly detection
- [ ] Support for additional IoT protocols (Thread, Matter)
- [ ] GraphQL API support
- [ ] Real-time WebSocket notifications
- [ ] Advanced security with OAuth2/OIDC

---

## Acknowledgments

Built with ❤️ using Spring Boot, Spring Cloud, and the IoT community.
