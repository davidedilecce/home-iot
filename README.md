# Home IoT Platform

![Home IoT architecture](images/smart-home.png)

Home IoT is an open-source platform designed for collecting and managing telemetry data from Zigbee sensors. 
It provides a modular microservices architecture with an API Gateway, a Backend-for-Frontend (BFF) for iOS, and asynchronous event handling via RabbitMQ. 
The platform enables real-time telemetry monitoring, event processing, user management, and seamless integration with IoT devices.

It is ideal for developers and IoT enthusiasts who want a flexible and scalable solution to acquire, store, and process data from Zigbee-based devices in a centralized and reliable way.

---

## Built with

The Home IoT platform is built using a modern stack for microservices and IoT integration:

- **Language:** Java 21
- **Frameworks:** Spring Boot 4, Spring Cloud (for service discovery, config, and API gateway)
- **Build Tool:** Gradle
- **Messaging / Event Streaming:** RabbitMQ (for asynchronous events), MQTT (for Zigbee sensor data)
- **Service Discovery & Config:** Eureka, Spring Cloud Config Server
- **API Gateway & BFF:** Spring Cloud Gateway, custom iOS Backend-for-Frontend (BFF)
- **IoT Integration:** Zigbee2MQTT forwarder for Zigbee device telemetry

This stack allows for real-time telemetry acquisition, robust event processing, and scalable IoT device management.

---

## Architecture

### Spring Cloud Core Services

These are the foundational services that enable the microservices architecture, service discovery, centralized configuration, and API routing.

**home-iot-api-gateway**
- **Functionality:** Serves as the primary entry point for all client requests. Routes API calls to the appropriate backend microservices, handles telemetry queries, retrieves latest sensor values, and processes events asynchronously via RabbitMQ.
- **Notes:** Acts as the central gateway for all clients (mobile, web) and simplifies request routing and security.

**home-iot-config-server**
- **Functionality:** Provides centralized configuration management for all microservices.
- **Service Registration:** Integrated with each microservice to allow dynamic configuration updates without redeployment.
- **Notes:** Ensures consistency of configuration across the platform.

**home-iot-service-registry**
- **Functionality:** Central service registry for all microservices using Eureka.
- **Notes:** Includes missing submodules. Ensures that services can dynamically discover each other.

---

### Authentication and Security

**home-iot-auth-server** 
- **Functionality:** Manages authentication endpoints and security for all clients and microservices.
- **Service Registration:** Registers with Eureka for discoverability.
- **Notes:** Provides centralized management of credentials and access control.

---

### BFF Microservices

**home-iot-ios-bff**
- **Functionality:** Backend-for-Frontend dedicated to iOS clients. Aggregates data from multiple microservices and adapts responses specifically for iOS applications.
- **Communication:** Interacts with the backend microservices through the API Gateway.
- **Notes:** Simplifies client logic and improves performance for mobile apps.

---

### Domain Microservices

These services manage the business logic and domain entities of the platform.

**home-iot-ms-apartment** 
- **Functionality:** Handles telemetry queries and retrieves the latest values for apartment-related devices. Publishes events asynchronously via RabbitMQ.
- **Submodule:** Included as a Git submodule for modular development.

**home-iot-ms-user**
- **Functionality:** Manages user endpoints and user-related data.
- **Service Registration:** Registers with Eureka.

**home-iot-ms-events**
- **Functionality:** Dedicated microservice for event management. Processes and stores events coming from various telemetry sources.
- **Submodule:** Added as a Git submodule for independent updates.

**home-iot-ms-gateway** (`@332dbca`)
- **Functionality:** Provides additional telemetry APIs and gateways for sensor data, handling both queries and event publication via RabbitMQ.

---

### Telemetry Microservices

These services are responsible for handling sensor telemetry, including reading, writing, and forwarding data.

**home-iot-ms-telemetry-reader** (`@2d8bf36`)
- **Functionality:** Reads telemetry data from sensors and exposes it via APIs. Publishes events asynchronously to RabbitMQ.

**home-iot-ms-telemetry-writer** (`@9f41fe1`)
- **Functionality:** Handles writing telemetry data into the system and broadcasting updates/events through RabbitMQ.


**home-iot-zigbee2mqtt-forwarder** (`@87abcbb`)
- **Functionality:** Forwards Zigbee2MQTT sensor data to the telemetry system via MQTT.

---