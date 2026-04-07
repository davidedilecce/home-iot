# 🚀 Analisi di Fattibilità — Dockerizzazione e Deploy su AWS

## Indice

1. [Riepilogo Architettura Attuale](#1-riepilogo-architettura-attuale)
2. [Fattibilità Dockerizzazione per Sottomodulo](#2-fattibilità-dockerizzazione-per-sottomodulo)
3. [Strategia per home-iot-common nel Docker Build](#3-strategia-per-home-iot-common-nel-docker-build)
4. [Mappa Porte e Networking](#4-mappa-porte-e-networking)
5. [Servizi Infrastrutturali su AWS](#5-servizi-infrastrutturali-su-aws)
6. [Strategia di Deploy AWS Consigliata](#6-strategia-di-deploy-aws-consigliata)
7. [Service Discovery e Configurazione senza Eureka/Config Server](#7-service-discovery-e-configurazione-senza-eurekaconfig-server)
8. [CI/CD Pipeline](#8-cicd-pipeline)
9. [Gestione Secrets e Variabili d'Ambiente](#9-gestione-secrets-e-variabili-dambiente)
10. [Ordine di Avvio dei Container](#10-ordine-di-avvio-dei-container)
11. [Zigbee2mqtt-forwarder — Strategia Ibrida](#11-zigbee2mqtt-forwarder--strategia-ibrida)
12. [Considerazioni Aggiuntive e Criticità](#12-considerazioni-aggiuntive-e-criticità)
13. [Stima Costi AWS](#13-stima-costi-aws)
14. [Piano di Implementazione a Fasi](#14-piano-di-implementazione-a-fasi)
15. [TODO List — Da Zero alla Produzione su AWS](#15-todo-list--da-zero-alla-produzione-su-aws)

---

## 1. Riepilogo Architettura Attuale

### Stack tecnologico
- **Java 21**, Spring Boot 4.0.1, Spring Cloud 2025.1.0
- **Build tool**: Gradle (Kotlin DSL) con version catalog condiviso (`gradle/libs.versions.toml`)
- **Service Discovery**: ~~Netflix Eureka~~ → **URL diretti via variabili d'ambiente**
- **Config Management**: ~~Spring Cloud Config Server~~ → **`application.yaml` locali + env vars**
- **API Gateway**: Spring Cloud Gateway (WebFlux) con routing statico verso hostname configurabili
- **Messaging**: RabbitMQ (AMQP 0.9.1)
- **IoT Protocol**: MQTT (Paho + Spring Integration MQTT)
- **Database**: MongoDB (4 database), InfluxDB 2 (time-series)

### Mappa dei Sottomoduli

| # | Sottomodulo | Tipo | Porta | Dipendenze Infra | Dipendenze Servizio |
|---|-------------|------|-------|-------------------|---------------------|
| 1 | `home-iot-api-gateway` | API Gateway (WebFlux) | **8080** | nessuna | ms-apartment, ms-user, ms-gateway, ms-events, ms-telemetry-reader (via URL diretti) |
| 2 | `home-iot-auth-server` | Auth + JWT | **8889** | nessuna | ms-user (via URL diretto) |
| 3 | `home-iot-bff-ios` | BFF reattivo | **8010** | nessuna | ms-events, ms-apartment, ms-gateway, ms-telemetry-reader (via URL diretti) |
| 4 | `home-iot-ms-apartment` | CRUD Apartment | **9000** | MongoDB (`apartment_db`) | nessuna |
| 5 | `home-iot-ms-gateway` | CRUD Gateway/Device | **9001** | MongoDB (`gateway_db`) | nessuna |
| 6 | `home-iot-ms-user` | CRUD User | **9002** | MongoDB (`user_db`) | nessuna |
| 7 | `home-iot-ms-events` | Event processing (reactive) | **9003** | MongoDB (`events_db`), RabbitMQ | ms-apartment, ms-gateway (via URL diretti) |
| 8 | `home-iot-ms-telemetry-reader` | Telemetry read (WebMVC) | **9010** | InfluxDB 2 | nessuna |
| 9 | `home-iot-ms-telemetry-writer` | Telemetry write (headless) | **nessuna** | InfluxDB 2, RabbitMQ, MQTT broker | nessuna |
| 10 | `home-iot-zigbee2mqtt-forwarder` | MQTT bridge (Python) | **nessuna** | MQTT locale + MQTT remoto | nessuna |
| — | `home-iot-common` | Libreria condivisa | N/A | — | Usato da tutti i servizi JVM |

> ℹ️ `service-registry` (Eureka) e `config-server` sono stati **rimossi**. I servizi ora si trovano tramite URL configurabili via variabili d'ambiente e non dipendono più da alcun componente di infrastruttura Spring Cloud.

### Variabili d'Ambiente già Esternalizzate

| Variabile | Servizio |
|-----------|----------|
| `HOME_IOT_JWT_SECRET` | auth-server |
| `HOME_IOT_ADMIN_USERNAME` / `HOME_IOT_ADMIN_PASSWORD` | ms-user |
| `HOME_IOT_INFLUX_HOST` / `HOME_IOT_INFLUX_API_TOKEN` | ms-telemetry-reader, ms-telemetry-writer |
| `HOME_IOT_MQTT_HOST` / `HOME_IOT_MQTT_CLIENT_ID` / `HOME_IOT_MQTT_USERNAME` / `HOME_IOT_MQTT_PASSWORD` / `HOME_IOT_MQTT_TOPIC` | ms-telemetry-writer |
| `GATEWAY_ID`, `LOCAL_BROKER`, `EXTERNAL_BROKER`, etc. | zigbee2mqtt-forwarder |

### 🔴 URL da Parametrizzare

Con la rimozione di Eureka non esistono più route `lb://SERVICE-NAME`. Tutti i servizi comunicano tramite **URL HTTP diretti** che devono essere configurabili via env var. Ecco i file `application.yaml` che richiedono intervento:

| File | URL da parametrizzare |
|------|-----------------------|
| `api-gateway/application.yaml` | Route URI verso ogni microservizio (es. `http://ms-apartment:9000`) |
| `auth-server/application.yaml` | URL ms-user |
| `bff-ios/application.yaml` | URL ms-events, ms-apartment, ms-gateway, ms-telemetry-reader |
| `ms-events/application.yaml` | URL ms-apartment, ms-gateway + `spring.rabbitmq.host` |
| `ms-telemetry-writer/application.yaml` | `spring.rabbitmq.host` |

**Pattern raccomandato** (`application.yaml`):
```yaml
# Esempio in api-gateway
spring:
  cloud:
    gateway:
      routes:
        - id: ms-apartment
          uri: ${MS_APARTMENT_URL:http://ms-apartment:9000}
          predicates:
            - Path=/api/apartments/**

# Esempio in ms-events
spring:
  rabbitmq:
    host: ${RABBITMQ_HOST:rabbitmq}
    port: ${RABBITMQ_PORT:5672}

ms-apartment:
  base-url: ${MS_APARTMENT_URL:http://ms-apartment:9000}
```

---

## 2. Fattibilità Dockerizzazione per Sottomodulo

| Sottomodulo | Fattibilità | Difficoltà | Dockerfile esistente? | Note |
|---|---|---|---|---|
| **api-gateway** | ✅ **Alta** | 🟡 Media | ✅ Esistente | Dipende da `home-iot-common`. Route da parametrizzare con env vars. Nessuna dipendenza da Eureka. |
| **auth-server** | ✅ **Alta** | 🟡 Media | ✅ Esistente | Dipende da `home-iot-common`. URL ms-user da parametrizzare. |
| **bff-ios** | ✅ **Alta** | 🟡 Media | ✅ Esistente | Dipende da `home-iot-common`. Resilience4j già pronto. URL servizi da parametrizzare. |
| **ms-apartment** | ✅ **Alta** | 🟡 Media | ✅ Esistente | MongoDB URI da parametrizzare. Nessuna dipendenza su altri servizi. |
| **ms-user** | ✅ **Alta** | 🟡 Media | ✅ Esistente | Come ms-apartment. Env vars admin già esternalizzate. |
| **ms-events** | ✅ **Media-Alta** | 🟠 Media-Alta | ✅ Esistente | 3 dipendenze infra (MongoDB, RabbitMQ, ms-apartment/ms-gateway). URL e RabbitMQ host hardcoded da parametrizzare. |
| **ms-gateway** | ✅ **Alta** | 🟡 Media | ✅ Esistente | Come ms-apartment. |
| **ms-telemetry-reader** | ✅ **Alta** | 🟡 Media | ✅ Esistente | InfluxDB già parametrizzato via env vars. |
| **ms-telemetry-writer** | ⚠️ **Media** | 🟠 Media-Alta | ✅ Esistente (da correggere) | Dockerfile non include `home-iot-common`. `web-application-type: none` → no health check HTTP. RabbitMQ host hardcoded. |
| **zigbee2mqtt-forwarder** | ✅ **Alta** | 🟢 Bassa | ✅ Funzionante | Già dockerizzato. Deploy on-premises, non su AWS. |
| **home-iot-common** | N/A | — | — | Libreria condivisa, non si dockerizza. |

**Verdetto**: La dockerizzazione è **fattibile al 100%** per tutti i moduli. Il lavoro principale è:
1. Correggere/completare i Dockerfile esistenti (in particolare ms-telemetry-writer)
2. Parametrizzare gli URL hardcoded rimasti con env vars
3. Assicurarsi che il build context monorepo includa `home-iot-common`

---

## 3. Strategia per home-iot-common nel Docker Build

### Problema
Ogni `settings.gradle.kts` referenzia:
```kotlin
include(":home-iot-common")
project(":home-iot-common").projectDir = file("../home-iot-common")
```
E il version catalog:
```kotlin
from(files("../gradle/libs.versions.toml"))
```

Il Docker build context di un singolo modulo **non include** i path relativi `../`.

### Soluzione Consigliata: Build context alla radice del monorepo

```
home-iot/                           ← Docker build context (.)
├── docker/
│   ├── api-gateway/Dockerfile
│   ├── auth-server/Dockerfile
│   ├── bff-ios/Dockerfile
│   ├── ms-apartment/Dockerfile
│   ├── ms-events/Dockerfile
│   ├── ms-gateway/Dockerfile
│   ├── ms-telemetry-reader/Dockerfile
│   ├── ms-telemetry-writer/Dockerfile
│   └── ms-user/Dockerfile
├── gradle/
│   └── libs.versions.toml
├── home-iot-common/
├── home-iot-api-gateway/
├── home-iot-auth-server/
└── ...
```

### Dockerfile Template (Multi-Stage)

```dockerfile
# ===== Stage 1: Build =====
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /workspace

# Copia Gradle wrapper e configurazione
COPY gradle/libs.versions.toml gradle/libs.versions.toml

# Copia la libreria comune
COPY home-iot-common/ home-iot-common/

# Copia il modulo target (es. api-gateway)
COPY home-iot-api-gateway/ home-iot-api-gateway/

# Build
WORKDIR /workspace/home-iot-api-gateway
RUN chmod +x gradlew && ./gradlew bootJar --no-daemon -x test

# ===== Stage 2: Runtime =====
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=build /workspace/home-iot-api-gateway/build/libs/*.jar app.jar

EXPOSE 8080
ENV JAVA_OPTS=""
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

**Comando build** (dalla radice del monorepo):
```bash
docker build -f docker/api-gateway/Dockerfile -t home-iot/api-gateway:latest .
```

### Alternativa: Pre-build JAR + Dockerfile semplice
Nella CI, compilare prima il JAR, poi dockerizzare:
```bash
# Step 1: Build (nella CI)
cd home-iot-api-gateway && ./gradlew bootJar

# Step 2: Docker (Dockerfile single-stage)
# FROM eclipse-temurin:21-jre-alpine
# COPY build/libs/*.jar app.jar
# ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## 4. Mappa Porte e Networking

### Docker Compose (sviluppo locale)

```
┌────────────────────── Docker Network: home-iot-net ──────────────────────┐
│                                                                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐   │
│  │   mongodb    │  │  rabbitmq    │  │   influxdb   │  │ mosquitto  │   │
│  │  :27017      │  │  :5672       │  │  :8086       │  │  :1883     │   │
│  │              │  │  :15672 (UI) │  │              │  │            │   │
│  └──────────────┘  └──────────────┘  └──────────────┘  └────────────┘   │
│                                                                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                   │
│  │ api-gateway  │  │ auth-server  │  │  bff-ios     │                   │
│  │ :8080        │  │ :8889        │  │  :8010       │                   │
│  └──────────────┘  └──────────────┘  └──────────────┘                   │
│                                                                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐   │
│  │ ms-apartment │  │ ms-gateway   │  │  ms-user     │  │ ms-events  │   │
│  │ :9000        │  │ :9001        │  │  :9002       │  │  :9003     │   │
│  └──────────────┘  └──────────────┘  └──────────────┘  └────────────┘   │
│                                                                           │
│  ┌──────────────────────┐  ┌──────────────────────┐                      │
│  │ ms-telemetry-reader  │  │ ms-telemetry-writer  │  ← No porta HTTP     │
│  │ :9010                │  │                      │                      │
│  └──────────────────────┘  └──────────────────────┘                      │
└───────────────────────────────────────────────────────────────────────────┘
```

I servizi si raggiungono via **hostname del container** (es. `http://ms-apartment:9000`), impostato tramite env var.

### AWS (ECS Fargate)

| Service | Porta container | Esposizione | Raggiungibilità |
|---------|----------------|-------------|-----------------|
| **api-gateway** | **8080** | ✅ **Pubblico via ALB** | `https://api.tuodominio.com` → ALB → :8080 |
| auth-server | 8889 | ❌ Interno VPC | DNS privato Cloud Map |
| bff-ios | 8010 | ❌ Interno VPC | DNS privato Cloud Map |
| ms-apartment | 9000 | ❌ Interno VPC | DNS privato Cloud Map |
| ms-gateway | 9001 | ❌ Interno VPC | DNS privato Cloud Map |
| ms-user | 9002 | ❌ Interno VPC | DNS privato Cloud Map |
| ms-events | 9003 | ❌ Interno VPC | DNS privato Cloud Map |
| ms-telemetry-reader | 9010 | ❌ Interno VPC | DNS privato Cloud Map |
| ms-telemetry-writer | — | ❌ Interno VPC | Solo outbound (RabbitMQ, InfluxDB, MQTT) |
| zigbee2mqtt-forwarder | — | ❌ **On-premises** | Solo outbound verso MQTT broker cloud |

**Solo l'API Gateway è esposto pubblicamente**. Tutti gli altri comunicano nella VPC privata tramite DNS Cloud Map (es. `ms-apartment.home-iot.local`).

---

## 5. Servizi Infrastrutturali su AWS

| Servizio Locale | Opzione AWS Consigliata | Alternativa | Costo indicativo | Note |
|-----------------|------------------------|-------------|-------------------|------|
| **MongoDB** (4 DB) | **Amazon DocumentDB** | MongoDB Atlas su AWS | ~$110/mese (db.t3.medium) | DocumentDB compatibile MongoDB 5.0. Verificare le aggregation pipeline usate nei microservizi (DocumentDB ha limitazioni). Atlas offre compatibilità 100% ma costi più alti. |
| **RabbitMQ** | **Amazon MQ for RabbitMQ** | Self-hosted su EC2 | ~$25/mese (mq.t3.micro) | Managed, supporta AMQP 0.9.1 nativo. Exchange/queue/routing-key funzionano **senza modifiche al codice**. |
| **InfluxDB 2** | **InfluxDB su EC2** (con EBS) | Amazon Timestream | ~$25-50/mese (t3.small + EBS) | ⚠️ **Non esiste un servizio managed InfluxDB su AWS**. Timestream richiederebbe riscrittura completa del data access layer (`influxdb-client-java` → AWS SDK). Consigliato restare con InfluxDB self-hosted su EC2 o ECS. |
| **MQTT Broker** (esterno) | **AWS IoT Core** | Mosquitto su EC2 | ~$1/milione msg | Scalabile, pay-per-message. Supporta MQTT 3.1.1/5.0. Richiede autenticazione via certificati X.509 → modificare config `paho-mqtt` nel telemetry-writer e zigbee2mqtt-forwarder per TLS. |

### Diagramma Infrastruttura AWS

```
┌─────────────────────────────── AWS VPC ────────────────────────────────┐
│                                                                         │
│  ┌─── Public Subnet ───┐    ┌─── Private Subnet ────────────────────┐  │
│  │                      │    │                                       │  │
│  │  ┌──────────────┐   │    │  ┌──────────────────────────────────┐ │  │
│  │  │     ALB      │───┼────┼─▶│       ECS Fargate Cluster        │ │  │
│  │  │  (HTTPS:443) │   │    │  │                                  │ │  │
│  │  └──────────────┘   │    │  │  api-gateway, auth-server,       │ │  │
│  │                      │    │  │  bff-ios, ms-apartment,          │ │  │
│  │  ┌──────────────┐   │    │  │  ms-user, ms-events, ms-gateway, │ │  │
│  │  │  NAT Gateway │   │    │  │  ms-telemetry-reader/writer      │ │  │
│  │  └──────────────┘   │    │  └──────────────────────────────────┘ │  │
│  └──────────────────────┘    │                                       │  │
│                               │  ┌──────────────┐ ┌──────────────┐   │  │
│                               │  │  DocumentDB   │ │  Amazon MQ   │   │  │
│                               │  │  (MongoDB)    │ │  (RabbitMQ)  │   │  │
│                               │  └──────────────┘ └──────────────┘   │  │
│                               │                                       │  │
│                               │  ┌──────────────┐ ┌──────────────┐   │  │
│                               │  │ EC2 InfluxDB  │ │ AWS IoT Core │   │  │
│                               │  │  (t3.small)   │ │  (MQTT)      │   │  │
│                               │  └──────────────┘ └──────────────┘   │  │
│                               └───────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘

┌─── On-Premises (Casa) ───┐
│  ┌──────────────────────┐ │
│  │ zigbee2mqtt-forwarder │─── MQTT/TLS ──▶ AWS IoT Core
│  │ (Docker / Raspberry)  │ │
│  └──────────────────────┘ │
│  ┌──────────────────────┐ │
│  │ Zigbee2MQTT + Broker  │ │
│  └──────────────────────┘ │
└───────────────────────────┘
```

---

## 6. Strategia di Deploy AWS Consigliata

### Confronto Opzioni

| Criterio | **ECS Fargate** ✅ | EKS (Kubernetes) | Elastic Beanstalk |
|----------|-------------------|-------------------|-------------------|
| Complessità operativa | 🟢 **Bassa** | 🔴 Alta | 🟡 Media |
| Costo (9 servizi) | 🟡 Pay-per-use (~$100-200/mese) | 🔴 Control plane $72/mese + nodi | 🟡 EC2 always-on |
| Scaling | 🟢 Auto per servizio | 🟢 HPA/VPA | 🟡 Per ambiente |
| Service Discovery | 🟢 Cloud Map nativo | 🟢 CoreDNS | 🔴 Limitato |
| Container nativi | 🟢 Sì | 🟢 Sì | 🟡 Supporto Docker |
| Serverless | 🟢 Sì (no server da gestire) | ❌ No (gestisci nodi) | ❌ No |
| Deploy speed | 🟢 ~2-5 min | 🟡 ~5-10 min | 🟡 ~5-10 min |
| Learning curve | 🟢 Bassa | 🔴 Alta | 🟡 Media |

### ✅ Raccomandazione: **Amazon ECS Fargate**

**Motivazioni**:
1. **9 microservizi** = dimensione ideale per ECS (EKS è overkill)
2. **Nessun server da gestire** (serverless containers)
3. **AWS Cloud Map** integrato per service discovery DNS-based
4. **Pay-per-use**: paghi solo il tempo CPU/RAM effettivo
5. **Integrazione nativa** con ALB, ECR, CloudWatch, Secrets Manager
6. **Semplicità**: definisci Task Definition + Service e sei operativo
7. **Routing diretto**: senza Eureka, ogni servizio risolve gli altri via hostname Cloud Map → configurazione trasparente

### Architettura ECS

```
ECR (Registry immagini)
  ├── home-iot/api-gateway:latest
  ├── home-iot/auth-server:latest
  ├── home-iot/bff-ios:latest
  ├── home-iot/ms-apartment:latest
  ├── home-iot/ms-user:latest
  ├── home-iot/ms-events:latest
  ├── home-iot/ms-gateway:latest
  ├── home-iot/ms-telemetry-reader:latest
  └── home-iot/ms-telemetry-writer:latest

ECS Cluster: home-iot-cluster
  ├── 9 Task Definitions (1 per servizio)
  ├── 9 ECS Services (desiredCount: 1, scaling auto)
  └── Cloud Map namespace: home-iot.local
       ├── api-gateway.home-iot.local         → :8080
       ├── auth-server.home-iot.local         → :8889
       ├── bff-ios.home-iot.local             → :8010
       ├── ms-apartment.home-iot.local        → :9000
       ├── ms-gateway.home-iot.local          → :9001
       ├── ms-user.home-iot.local             → :9002
       ├── ms-events.home-iot.local           → :9003
       ├── ms-telemetry-reader.home-iot.local → :9010
       └── ms-telemetry-writer.home-iot.local → (no HTTP)
```

### Sizing raccomandato per servizio

| Servizio | vCPU | Memory | Note |
|----------|------|--------|------|
| api-gateway | 0.5 | 1 GB | Tutto il traffico passa di qui |
| auth-server | 0.25 | 512 MB | Carico moderato |
| bff-ios | 0.25 | 512 MB | Aggregazione reattiva |
| ms-apartment | 0.25 | 512 MB | CRUD semplice |
| ms-user | 0.25 | 512 MB | CRUD semplice |
| ms-events | 0.5 | 1 GB | Consumer RabbitMQ + MongoDB |
| ms-gateway | 0.25 | 512 MB | CRUD semplice |
| ms-telemetry-reader | 0.25 | 512 MB | Query InfluxDB |
| ms-telemetry-writer | 0.5 | 1 GB | MQTT listener + InfluxDB writer + RabbitMQ publisher |

---

## 7. Service Discovery e Configurazione senza Eureka/Config Server

### Come funziona il routing senza Eureka

Con la rimozione di `service-registry` (Eureka) e `config-server`, l'architettura diventa **più semplice e diretta**:

| Prima (con Eureka) | Adesso (senza Eureka) |
|--------------------|----------------------|
| Route `lb://MS-APARTMENT` → Eureka → IP dinamico | Route `${MS_APARTMENT_URL:http://ms-apartment:9000}` → DNS diretto |
| Ogni servizio si registra a Eureka all'avvio | Nessuna registrazione necessaria |
| Config Server distribuisce `application.yaml` centralmente | Ogni servizio ha il proprio `application.yaml` completo |
| 2 container aggiuntivi sempre attivi | Zero overhead infrastrutturale Spring Cloud |

### Service Discovery in Docker Compose (locale)

In Docker Compose i servizi si trovano tramite il **nome del container** nella rete interna:
```yaml
# docker-compose.yml
services:
  api-gateway:
    environment:
      - MS_APARTMENT_URL=http://ms-apartment:9000
      - MS_USER_URL=http://ms-user:9002
      - MS_EVENTS_URL=http://ms-events:9003
      - MS_GATEWAY_URL=http://ms-gateway:9001
      - MS_TELEMETRY_READER_URL=http://ms-telemetry-reader:9010
      - AUTH_SERVER_URL=http://auth-server:8889

  ms-events:
    environment:
      - RABBITMQ_HOST=rabbitmq
      - MS_APARTMENT_URL=http://ms-apartment:9000
      - MS_GATEWAY_URL=http://ms-gateway:9001
```

### Service Discovery in AWS ECS (produzione)

In ECS + Cloud Map ogni servizio ottiene un **hostname DNS privato** nella VPC:
```
ms-apartment.home-iot.local   → IP privato ECS task (aggiornato automaticamente)
ms-user.home-iot.local        → IP privato ECS task
ms-events.home-iot.local      → IP privato ECS task
...
```

Le env vars in ECS diventano:
```yaml
environment:
  - MS_APARTMENT_URL=http://ms-apartment.home-iot.local:9000
  - RABBITMQ_HOST=b-xxxx.mq.eu-west-1.amazonaws.com
```

### Configurazione distribuita senza Config Server

Senza Config Server, ogni servizio porta la propria configurazione completa:

```
Locale      → application.yaml           (valori di sviluppo, default)
Docker      → application-docker.yaml    (override per Docker Compose)
AWS         → env vars (Secrets Manager + Parameter Store)
```

**Profilo `docker`** da attivare con `SPRING_PROFILES_ACTIVE=docker`:
```yaml
# application-docker.yaml (in ogni servizio)
spring:
  data:
    mongodb:
      uri: ${MONGODB_URI:mongodb://mongodb:27017/db_name}
  rabbitmq:
    host: ${RABBITMQ_HOST:rabbitmq}
    port: 5672
```

### Vantaggi della nuova architettura

| Aspetto | Beneficio |
|---------|-----------|
| **Avvio** | Nessuna dipendenza da container di infrastruttura Spring Cloud → avvio più veloce e robusto |
| **Debugging** | Errori di connessione immediati e leggibili (no "Config Server unreachable") |
| **Resilienza** | Nessun single point of failure (Eureka / Config Server non possono far crashare l'intero sistema) |
| **Costo** | 2 container in meno → risparmio ~$20-30/mese su Fargate |
| **Semplicità** | Dipendenze tra servizi espresse solo tramite URL → facilmente ispezionabili |

---

## 8. CI/CD Pipeline

### Strumento: **GitHub Actions** (il codice è già su GitHub)

### Pipeline Architettura

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Git Push    │────▶│  Build & Test │────▶│ Docker Build  │────▶│  Push to ECR  │
│   (main/tag)  │     │  (Gradle)    │     │  (Multi-stage)│     │              │
└──────────────┘     └──────────────┘     └──────────────┘     └──────┬───────┘
                                                                       │
                                                                       ▼
                                                              ┌──────────────┐
                                                              │  ECS Deploy   │
                                                              │  (Force new   │
                                                              │   deployment) │
                                                              └──────────────┘
```

### Workflow GitHub Actions (struttura)

```yaml
# .github/workflows/deploy-service.yml (reusable workflow)
name: Build & Deploy Microservice

on:
  workflow_call:
    inputs:
      service-name:
        required: true
        type: string
      service-port:
        required: true
        type: string

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 21

      - name: Build JAR
        run: |
          cd home-iot-${{ inputs.service-name }}
          ./gradlew bootJar -x test

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: eu-west-1

      - name: Login to ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build & Push Docker Image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        run: |
          docker build -f docker/${{ inputs.service-name }}/Dockerfile \
            -t $ECR_REGISTRY/home-iot/${{ inputs.service-name }}:${{ github.sha }} \
            -t $ECR_REGISTRY/home-iot/${{ inputs.service-name }}:latest .
          docker push $ECR_REGISTRY/home-iot/${{ inputs.service-name }}:${{ github.sha }}
          docker push $ECR_REGISTRY/home-iot/${{ inputs.service-name }}:latest

      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster home-iot-cluster \
            --service ${{ inputs.service-name }} \
            --force-new-deployment
```

### Trigger per modulo con change detection

```yaml
# .github/workflows/ci.yml
on:
  push:
    branches: [main]
    paths:
      - 'home-iot-common/**'               # Se cambia common → rebuild TUTTI
      - 'home-iot-api-gateway/**'
      - 'home-iot-auth-server/**'
      - 'home-iot-bff-ios/**'
      - 'home-iot-ms-apartment/**'
      - 'home-iot-ms-gateway/**'
      - 'home-iot-ms-user/**'
      - 'home-iot-ms-events/**'
      - 'home-iot-ms-telemetry-reader/**'
      - 'home-iot-ms-telemetry-writer/**'
```

**Regola critica**: Se `home-iot-common` cambia → ricostruire **tutti** i servizi dipendenti (matrix build).

---

## 9. Gestione Secrets e Variabili d'Ambiente

### Mapping su AWS

| Secret / Variabile | Servizio | AWS Service | Tipo |
|---------------------|----------|-------------|------|
| `HOME_IOT_JWT_SECRET` | auth-server | **Secrets Manager** | Secret |
| `HOME_IOT_ADMIN_USERNAME` | ms-user | **Secrets Manager** | Secret |
| `HOME_IOT_ADMIN_PASSWORD` | ms-user | **Secrets Manager** | Secret |
| `HOME_IOT_INFLUX_API_TOKEN` | telemetry-reader/writer | **Secrets Manager** | Secret |
| `HOME_IOT_MQTT_USERNAME` | telemetry-writer | **Secrets Manager** | Secret |
| `HOME_IOT_MQTT_PASSWORD` | telemetry-writer | **Secrets Manager** | Secret |
| MongoDB connection strings | ms-apartment, ms-user, ms-events, ms-gateway | **Secrets Manager** | Secret |
| RabbitMQ credentials | ms-events, ms-telemetry-writer | **Secrets Manager** | Secret |
| `HOME_IOT_INFLUX_HOST` | telemetry-reader/writer | **Parameter Store** | Config |
| `HOME_IOT_MQTT_HOST` | telemetry-writer | **Parameter Store** | Config |
| `HOME_IOT_MQTT_CLIENT_ID` | telemetry-writer | **Parameter Store** | Config |
| `HOME_IOT_MQTT_TOPIC` | telemetry-writer | **Parameter Store** | Config |
| `MS_APARTMENT_URL` | api-gateway, bff-ios, ms-events | **Parameter Store** | Config |
| `MS_USER_URL` | api-gateway, auth-server | **Parameter Store** | Config |
| `MS_EVENTS_URL` | api-gateway, bff-ios | **Parameter Store** | Config |
| `MS_GATEWAY_URL` | api-gateway, bff-ios, ms-events | **Parameter Store** | Config |
| `MS_TELEMETRY_READER_URL` | api-gateway, bff-ios | **Parameter Store** | Config |
| `RABBITMQ_HOST` | ms-events, ms-telemetry-writer | **Parameter Store** | Config |

### Esempio Task Definition ECS

```json
{
  "containerDefinitions": [{
    "name": "ms-apartment",
    "image": "123456789.dkr.ecr.eu-west-1.amazonaws.com/home-iot/ms-apartment:latest",
    "portMappings": [{ "containerPort": 9000 }],
    "environment": [
      { "name": "SPRING_PROFILES_ACTIVE", "value": "docker" }
    ],
    "secrets": [
      {
        "name": "SPRING_DATA_MONGODB_URI",
        "valueFrom": "arn:aws:secretsmanager:eu-west-1:123456789:secret:home-iot/mongodb-apartment-uri"
      }
    ],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/home-iot/ms-apartment",
        "awslogs-region": "eu-west-1",
        "awslogs-stream-prefix": "ecs"
      }
    }
  }]
}
```

---

## 10. Ordine di Avvio dei Container

```
╔══════════════════════════════════════════════════════════════════╗
║  FASE 0 — Infrastruttura (managed, sempre attiva)              ║
║  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐  ║
║  │ DocumentDB │ │ Amazon MQ  │ │ EC2 Influx │ │ IoT Core   │  ║
║  │ (MongoDB)  │ │ (RabbitMQ) │ │ (InfluxDB) │ │ (MQTT)     │  ║
║  └────────────┘ └────────────┘ └────────────┘ └────────────┘  ║
╚══════════════════════════════════════════════════════════════════╝
                              │
                              ▼
╔══════════════════════════════════════════════════════════════════╗
║  FASE 1 — Core Business Microservices (indipendenti)           ║
║  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐           ║
║  │ ms-apartment │ │ ms-user      │ │ ms-gateway   │           ║
║  └──────────────┘ └──────────────┘ └──────────────┘           ║
║  ┌────────────────────┐ ┌────────────────────┐                 ║
║  │ ms-telemetry-reader│ │ ms-telemetry-writer│                 ║
║  └────────────────────┘ └────────────────────┘                 ║
╚══════════════════════════════════════════════════════════════════╝
                              │
                              ▼
╔══════════════════════════════════════════════════════════════════╗
║  FASE 2 — Orchestration / Aggregation Layer                    ║
║  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐           ║
║  │ ms-events    │ │ auth-server  │ │ bff-ios      │           ║
║  │ (→ms-apt,gw) │ │ (→ms-user)  │ │ (→tutti)     │           ║
║  └──────────────┘ └──────────────┘ └──────────────┘           ║
╚══════════════════════════════════════════════════════════════════╝
                              │
                              ▼
╔══════════════════════════════════════════════════════════════════╗
║  FASE 3 — Entry Point                                          ║
║  ┌──────────────────────────────────────────────────────┐      ║
║  │              api-gateway (→ tutti i servizi)          │      ║
║  └──────────────────────────────────────────────────────┘      ║
╚══════════════════════════════════════════════════════════════════╝

╔══════════════════════════════════════════════════════════════════╗
║  SEPARATO — On-Premises                                        ║
║  ┌──────────────────────────────────────────────────────┐      ║
║  │ zigbee2mqtt-forwarder (MQTT locale → AWS IoT Core)   │      ║
║  └──────────────────────────────────────────────────────┘      ║
╚══════════════════════════════════════════════════════════════════╝
```

> ℹ️ Con la rimozione di Eureka e Config Server scompare la fase di bootstrap Spring Cloud. L'ordine di avvio è ora più semplice: prima l'infrastruttura managed, poi i microservizi core (indipendenti), poi gli aggregatori.

**In ECS Fargate**: l'ordine di avvio non è garantito tra servizi diversi. Gestire con:
- **Health checks** con retry + backoff nei client Spring (es. `spring.retry.*`)
- **ECS Service Dependencies**: configurare dependency ordering nella task definition
- **Startup probes** per attendere che le dipendenze siano disponibili

---

## 11. Zigbee2mqtt-forwarder — Strategia Ibrida

### Perché non può andare su AWS
Il forwarder deve connettersi al **broker MQTT locale** (Zigbee2MQTT / Mosquitto) che gira accanto ai dispositivi Zigbee fisici in casa.

### Deploy consigliato: On-Premises via Docker

```
┌─── Raspberry Pi / Mini-PC (Casa) ─────────────┐
│                                                  │
│  docker-compose.yml                             │
│  ┌────────────────────────────────────┐         │
│  │ zigbee2mqtt-forwarder              │         │
│  │ ┌──────────────────────────────┐   │         │
│  │ │ LOCAL_BROKER=mosquitto:1883  │   │         │
│  │ │ EXTERNAL_BROKER=xxx.iot.aws │   │         │
│  │ │ EXTERNAL_PORT=8883 (TLS)    │   │         │
│  │ └──────────────────────────────┘   │         │
│  └────────────────────────────────────┘         │
│       │                         │               │
│       ▼ (LAN)                   ▼ (Internet)    │
│  ┌──────────┐          ┌──────────────────┐     │
│  │Mosquitto │          │ AWS IoT Core     │     │
│  │(Zigbee2  │          │ (MQTT endpoint)  │     │
│  │ MQTT)    │          │                  │     │
│  └──────────┘          └──────────────────┘     │
└──────────────────────────────────────────────────┘
```

### Modifiche necessarie per AWS IoT Core
Il `zigbee2mqtt_forwarder.py` va modificato per supportare TLS con certificati X.509:

```python
# Aggiungere al client esterno:
external_client.tls_set(
    ca_certs="/certs/AmazonRootCA1.pem",
    certfile="/certs/certificate.pem.crt",
    keyfile="/certs/private.pem.key"
)
```

### Monitoring remoto
- Inviare log/heartbeat a **CloudWatch Logs** via `awslogs` Docker driver
- Oppure: creare un endpoint `/health` leggero (aggiungere Flask/FastAPI minimo) per monitoring esterno

---

## 12. Considerazioni Aggiuntive e Criticità

### 🔴 Criticità Alta

1. **URL diretti non parametrizzati** — Con la rimozione di Eureka i servizi devono conoscere gli URL degli altri via env vars. Verificare e completare la parametrizzazione in `api-gateway`, `bff-ios`, `auth-server`, `ms-events`.

2. **Dockerfile di ms-telemetry-writer da correggere** — Non include `home-iot-common` nel build context. `EXPOSE 8082` sbagliato (il servizio non ha web server). Da rifare con build context monorepo.

3. **Health check per ms-telemetry-writer** — `web-application-type: none` significa nessun endpoint HTTP. Opzioni:
    - Aggiungere Spring Boot Actuator con `management.server.port=8082` dedicato
    - Usare ECS **command health check**: `CMD-SHELL, pgrep -f app.jar || exit 1`

### 🟡 Criticità Media

4. **DocumentDB vs MongoDB** — DocumentDB ha limitazioni su alcune aggregation pipeline. Verificare le query usate (specialmente in ms-events). In caso di incompatibilità, usare MongoDB Atlas.

5. **AWS IoT Core autenticazione** — Richiede certificati X.509 anziché username/password. Modifica al forwarder Python e al telemetry-writer (Spring Integration MQTT config).

6. **Client-side load balancing assente** — Senza Eureka non c'è più `lb://` e Spring Cloud LoadBalancer. In produzione con più istanze di uno stesso servizio, l'ALB gestisce il bilanciamento per i servizi esposti; per la comunicazione interna, Cloud Map aggiorna automaticamente i record DNS con IP dei task attivi.

### 🟢 Criticità Bassa

7. **Profili Spring per ambiente** — Creare `application-docker.yaml` per ogni servizio che sovrascrive solo gli URL, attivato con `SPRING_PROFILES_ACTIVE=docker`.

8. **Logging centralizzato** — Configurare `awslogs` log driver in ECS per inviare tutti i log a CloudWatch.

---

## 13. Stima Costi AWS (mensili, regione eu-west-1)

| Componente | Spec | Costo stimato |
|-----------|------|---------------|
| **ECS Fargate** (9 servizi) | ~3.0 vCPU, ~6 GB RAM totali | ~$100-150/mese |
| **ALB** (Application Load Balancer) | 1 ALB + regole | ~$25/mese |
| **ECR** (Container Registry) | 9 immagini, ~4 GB | ~$1/mese |
| **Amazon DocumentDB** | db.t3.medium (1 istanza) | ~$110/mese |
| **Amazon MQ (RabbitMQ)** | mq.t3.micro (1 broker) | ~$25/mese |
| **EC2 (InfluxDB)** | t3.small + 50 GB EBS | ~$25/mese |
| **AWS IoT Core** | Pay-per-message | ~$1-5/mese |
| **CloudWatch** | Log + metriche | ~$10-20/mese |
| **NAT Gateway** | 1 per AZ | ~$35/mese |
| **Secrets Manager** | ~10 secrets | ~$4/mese |
| **Route 53** (DNS) | 1 hosted zone | ~$1/mese |
| | | |
| **TOTALE STIMATO** | | **~$330-500/mese** |

> 💡 Risparmio rispetto all'architettura con Eureka + Config Server: **~$20-30/mese** (2 task Fargate in meno) + maggiore stabilità.

### Ottimizzazioni costo
- Usare **Fargate Spot** per servizi non critici (bff-ios, ms-telemetry-reader) → risparmio ~70%
- Usare **Reserved Capacity** per DocumentDB → risparmio ~30%
- Iniziare con istanze minime e scalare in base al traffico

---

## 14. Piano di Implementazione a Fasi

### Fase 1 — Preparazione Codice (1-2 settimane)
- [ ] Parametrizzare tutti gli URL diretti con env vars (api-gateway, bff-ios, auth-server, ms-events)
- [ ] Creare profili `application-docker.yaml` per ogni servizio
- [ ] Correggere health check per ms-telemetry-writer (Actuator su porta dedicata o pgrep)
- [ ] Correggere Dockerfile di ms-telemetry-writer (build context monorepo)
- [ ] Verificare che tutti i Dockerfile usino la **build context monorepo** (root `/`) e includano `home-iot-common`
- [ ] Creare un **`docker-compose.yml` completo** con tutti i 9 servizi + MongoDB + RabbitMQ + InfluxDB + Mosquitto
- [ ] **Test locale con Docker Compose**: avviare tutto e verificare ogni servizio con `curl`

### Fase 2 — Infrastruttura AWS (1 settimana)
- [ ] Creare VPC con subnet pubbliche/private e NAT Gateway
- [ ] Provisioning DocumentDB, Amazon MQ, EC2 InfluxDB
- [ ] Configurare AWS IoT Core (MQTT broker)
- [ ] Creare ECR repository per ogni servizio
- [ ] Configurare Secrets Manager e Parameter Store con tutti i valori
- [ ] Setup ALB con certificato SSL (ACM) e dominio Route 53
- [ ] Creare namespace Cloud Map (`home-iot.local`)

### Fase 3 — Deploy ECS (1 settimana)
- [ ] Creare ECS Cluster Fargate
- [ ] Creare IAM roles per i task ECS (accesso a Secrets Manager, CloudWatch)
- [ ] Creare Task Definitions per ogni servizio con env vars e secrets
- [ ] Creare ECS Services con service discovery Cloud Map
- [ ] Configurare health checks e target groups ALB
- [ ] Deploy progressivo: core (ms-*) → auth/bff → api-gateway
- [ ] Verificare connettività inter-servizi tramite DNS Cloud Map

### Fase 4 — CI/CD + Monitoring (1 settimana)
- [ ] Setup GitHub Actions pipeline (build, push ECR, deploy ECS)
- [ ] Configurare CloudWatch dashboards e alarms
- [ ] Setup zigbee2mqtt-forwarder on-premises con connessione TLS a IoT Core
- [ ] Test end-to-end completo (dal dispositivo Zigbee all'app iOS)

### Fase 5 (Opzionale) — Evoluzione
- [ ] Implementare Fargate Spot per ottimizzare costi
- [ ] Aggiungere WAF (Web Application Firewall) all'ALB
- [ ] Configurare auto-scaling per servizi ad alto traffico
- [ ] Aggiungere X-Ray distributed tracing

---

## 15. TODO List — Da Zero alla Produzione su AWS

> Lista operativa completa per chi parte da zero. Seguire nell'ordine indicato.

---

### 🔧 STEP 1 — Prerequisiti locali

- [ ] Installare e configurare **AWS CLI** (`aws configure` con account di deploy)
- [ ] Installare **Docker** e verificare il build dalla radice del monorepo
- [ ] Creare un account **AWS** dedicato (o usare un account esistente con IAM User/Role dedicato)
- [ ] Abilitare **MFA** sull'account root AWS
- [ ] Creare un **IAM User** per la CI/CD con permessi minimi: `ECS`, `ECR`, `SecretsManager`, `SSM`, `CloudWatch`
- [ ] Configurare `aws configure` in locale con le credenziali del IAM User

---

### 🏗️ STEP 2 — Preparazione del codice

- [ ] **Parametrizzare URL diretti** in tutti gli `application.yaml` con hostname hardcoded:
    - `api-gateway` → route URI verso ogni microservizio
    - `auth-server` → URL di ms-user
    - `bff-ios` → URL di ms-apartment, ms-events, ms-gateway, ms-telemetry-reader
    - `ms-events` → URL di ms-apartment, ms-gateway + `rabbitmq.host`
    - `ms-telemetry-writer` → `rabbitmq.host`
- [ ] Creare **`application-docker.yaml`** in ogni servizio che sovrascrive gli URL con i nomi container docker-compose (es. `http://ms-apartment:9000`)
- [ ] **Correggere il Dockerfile di ms-telemetry-writer**:
    - Usare build context monorepo (includere `home-iot-common`)
    - Rimuovere `EXPOSE 8082` (nessun web server)
    - Aggiungere health check via `pgrep` oppure Actuator su porta separata
- [ ] Verificare che tutti gli altri Dockerfile usino la **build context monorepo** (root `/`) e includano `home-iot-common`
- [ ] Testare il build di ogni immagine Docker dalla radice:
  ```bash
  docker build -f docker/api-gateway/Dockerfile -t home-iot/api-gateway:test .
  ```
- [ ] Creare un **`docker-compose.yml` completo** con tutti i 9 servizi + MongoDB + RabbitMQ + InfluxDB + Mosquitto
- [ ] **Test locale con Docker Compose**: avviare tutto e verificare ogni servizio con `curl`

---

### ☁️ STEP 3 — Networking AWS (VPC)

- [ ] Aprire la **console AWS → VPC** (regione: `eu-west-1` o quella scelta)
- [ ] Creare una **VPC** dedicata: `home-iot-vpc` — CIDR: `10.0.0.0/16`
- [ ] Creare **2 subnet pubbliche** (una per AZ):
    - `home-iot-public-1a`: `10.0.1.0/24` — AZ: `eu-west-1a`
    - `home-iot-public-1b`: `10.0.2.0/24` — AZ: `eu-west-1b`
- [ ] Creare **2 subnet private** (una per AZ):
    - `home-iot-private-1a`: `10.0.10.0/24` — AZ: `eu-west-1a`
    - `home-iot-private-1b`: `10.0.11.0/24` — AZ: `eu-west-1b`
- [ ] Creare e attaccare un **Internet Gateway** alla VPC
- [ ] Creare un **NAT Gateway** nella subnet pubblica (con Elastic IP)
- [ ] Configurare **route tables**:
    - Subnet pubbliche → route `0.0.0.0/0` → Internet Gateway
    - Subnet private → route `0.0.0.0/0` → NAT Gateway
- [ ] Creare **Security Groups**:
    - `sg-alb`: ingress 80/443 da `0.0.0.0/0`
    - `sg-ecs`: ingress 8080/8889/8010/9000-9003/9010 da `sg-alb` e da se stesso (comunicazione interna)
    - `sg-documentdb`: ingress 27017 da `sg-ecs`
    - `sg-mq`: ingress 5671/5672 da `sg-ecs`
    - `sg-influxdb`: ingress 8086 da `sg-ecs`

---

### 🗄️ STEP 4 — Database e Broker (Infrastruttura Managed)

#### DocumentDB (MongoDB)
- [ ] Console AWS → **DocumentDB** → Create cluster
    - Istanza: `db.t3.medium`
    - Versione engine: `5.0`
    - VPC: `home-iot-vpc`, subnet private, Security Group: `sg-documentdb`
    - Abilitare deletion protection in produzione
- [ ] Creare **4 database**: `apartment_db`, `gateway_db`, `user_db`, `events_db`
- [ ] Salvare le connection string in **Secrets Manager** (una per microservizio)

#### Amazon MQ (RabbitMQ)
- [ ] Console AWS → **Amazon MQ** → Create broker
    - Engine: `RabbitMQ`
    - Tipo: `Single-instance broker` (dev/staging) o `Cluster deployment` (produzione)
    - Istanza: `mq.t3.micro`
    - VPC: subnet private, Security Group: `sg-mq`
- [ ] Annotare l'endpoint AMQP: `b-xxxx.mq.eu-west-1.amazonaws.com:5671`
- [ ] Salvare host e credenziali in **Secrets Manager** / Parameter Store

#### EC2 InfluxDB
- [ ] Console AWS → **EC2** → Launch Instance
    - AMI: Amazon Linux 2023
    - Tipo: `t3.small`
    - VPC: subnet privata, Security Group: `sg-influxdb`
    - Storage: 50 GB EBS gp3
- [ ] SSH nell'istanza e installare InfluxDB 2 (seguire docs ufficiali)
- [ ] Configurare bucket, organizzazione e API token
- [ ] Salvare host e API token in **Secrets Manager**

#### AWS IoT Core (MQTT)
- [ ] Console AWS → **IoT Core** → Settings → annotare l'endpoint: `xxxx.iot.eu-west-1.amazonaws.com`
- [ ] Creare un **Thing** per il telemetry-writer
- [ ] Creare un **Thing** per il zigbee2mqtt-forwarder
- [ ] Generare **certificati X.509** per ognuno e scaricare:
    - `certificate.pem.crt`, `private.pem.key`, `AmazonRootCA1.pem`
- [ ] Creare e associare **IoT Policy** con permessi `iot:Connect`, `iot:Publish`, `iot:Subscribe`, `iot:Receive`
- [ ] Salvare i certificati in **Secrets Manager** (o montarli come file nei container)

---

### 🔐 STEP 5 — Secrets Manager e Parameter Store

- [ ] Console AWS → **Secrets Manager** → creare i seguenti secret:
    - `home-iot/jwt-secret` → `HOME_IOT_JWT_SECRET`
    - `home-iot/admin-credentials` → `{ "username": "...", "password": "..." }`
    - `home-iot/influx-token` → `HOME_IOT_INFLUX_API_TOKEN`
    - `home-iot/mqtt-credentials` → `{ "username": "...", "password": "..." }`
    - `home-iot/mongodb/apartment-uri` → stringa connessione DocumentDB
    - `home-iot/mongodb/user-uri` → stringa connessione DocumentDB
    - `home-iot/mongodb/events-uri` → stringa connessione DocumentDB
    - `home-iot/mongodb/gateway-uri` → stringa connessione DocumentDB
    - `home-iot/rabbitmq-credentials` → `{ "username": "...", "password": "..." }`
    - `home-iot/iot-core/telemetry-writer-cert` → contenuto certificato X.509
    - `home-iot/iot-core/telemetry-writer-key` → chiave privata
- [ ] Console AWS → **Systems Manager → Parameter Store** → creare:
    - `/home-iot/influx-host` → hostname EC2 InfluxDB
    - `/home-iot/mqtt-host` → endpoint AWS IoT Core
    - `/home-iot/mqtt-client-id` → client ID telemetry-writer
    - `/home-iot/mqtt-topic` → topic MQTT
    - `/home-iot/rabbitmq-host` → endpoint Amazon MQ
    - `/home-iot/ms-apartment-url` → `http://ms-apartment.home-iot.local:9000`
    - `/home-iot/ms-user-url` → `http://ms-user.home-iot.local:9002`
    - `/home-iot/ms-events-url` → `http://ms-events.home-iot.local:9003`
    - `/home-iot/ms-gateway-url` → `http://ms-gateway.home-iot.local:9001`
    - `/home-iot/ms-telemetry-reader-url` → `http://ms-telemetry-reader.home-iot.local:9010`

---

### 📦 STEP 6 — Container Registry (ECR)

- [ ] Console AWS → **ECR** → Create repository per ognuno:
    - `home-iot/api-gateway`, `home-iot/auth-server`, `home-iot/bff-ios`
    - `home-iot/ms-apartment`, `home-iot/ms-user`, `home-iot/ms-events`
    - `home-iot/ms-gateway`, `home-iot/ms-telemetry-reader`, `home-iot/ms-telemetry-writer`
- [ ] Abilitare **image scanning on push** su ogni repository
- [ ] Abilitare **lifecycle policy** per mantenere solo le ultime N immagini
- [ ] Build e push manuale iniziale per verificare:
  ```bash
  aws ecr get-login-password --region eu-west-1 | \
    docker login --username AWS --password-stdin <account_id>.dkr.ecr.eu-west-1.amazonaws.com

  docker build -f docker/api-gateway/Dockerfile \
    -t <account_id>.dkr.ecr.eu-west-1.amazonaws.com/home-iot/api-gateway:latest .
  docker push <account_id>.dkr.ecr.eu-west-1.amazonaws.com/home-iot/api-gateway:latest
  ```

---

### 🌐 STEP 7 — Load Balancer e DNS

- [ ] Console AWS → **EC2 → Load Balancers** → Create ALB
    - Nome: `home-iot-alb`, schema: `internet-facing`
    - VPC: `home-iot-vpc`, subnet pubbliche
    - Security Group: `sg-alb`
- [ ] Creare **Target Group** per api-gateway:
    - Tipo: `IP`, protocollo: HTTP, porta: 8080
    - Health check: `GET /actuator/health`
- [ ] Configurare **Listener**:
    - HTTP :80 → redirect a HTTPS :443
    - HTTPS :443 → forward a Target Group api-gateway
- [ ] Richiedere certificato SSL in **AWS Certificate Manager (ACM)**:
    - Dominio: `api.tuodominio.com`, validazione: DNS (CNAME su Route 53)
- [ ] Console AWS → **Route 53** → aggiungere record:
    - `api.tuodominio.com` → Alias → ALB DNS name

---

### 🔍 STEP 8 — Service Discovery (Cloud Map)

- [ ] Console AWS → **Cloud Map** → Create namespace
    - Nome: `home-iot.local`, tipo: `DNS private` nella VPC `home-iot-vpc`
- [ ] Creare un **service** Cloud Map per ogni microservizio:
    - `api-gateway`, `auth-server`, `bff-ios`, `ms-apartment`, `ms-user`,
      `ms-events`, `ms-gateway`, `ms-telemetry-reader`, `ms-telemetry-writer`
    - DNS record type: `A`, TTL: `10`

---

### 🚢 STEP 9 — ECS Fargate

#### Cluster
- [ ] Console AWS → **ECS** → Create Cluster
    - Nome: `home-iot-cluster`, infrastruttura: `AWS Fargate`

#### IAM Roles
- [ ] Creare **Task Execution Role** (`ecsTaskExecutionRole`) con policy:
    - `AmazonECSTaskExecutionRolePolicy`
    - Inline policy per accesso a Secrets Manager (ARN specifici)
    - Inline policy per accesso a Parameter Store (`/home-iot/*`)
- [ ] Creare **Task Role** per accesso opzionale a CloudWatch, S3, etc.

#### Task Definitions (una per servizio)
Per ogni servizio, creare una Task Definition con:
- [ ] Launch type: `FARGATE`
- [ ] CPU/Memory come da tabella sizing (sezione 6)
- [ ] Container image: URI ECR
- [ ] Environment variables + secrets (da Secrets Manager / Parameter Store)
- [ ] Log driver: `awslogs` → CloudWatch group `/ecs/home-iot/<service>`
- [ ] Health check appropriato
- [ ] Ordine di creazione consigliato:
  `ms-apartment` → `ms-user` → `ms-gateway` → `ms-telemetry-reader` → `ms-telemetry-writer` → `ms-events` → `auth-server` → `bff-ios` → `api-gateway`

#### ECS Services
Per ogni servizio, creare un ECS Service:
- [ ] Task Definition: quella creata sopra
- [ ] `desiredCount: 1`
- [ ] VPC: subnet private, Security Group: `sg-ecs`
- [ ] Service Discovery: abilitare e collegare al service Cloud Map corrispondente
- [ ] Solo per api-gateway: collegare al Target Group ALB

---

### 📊 STEP 10 — Monitoring e Logging

- [ ] Console AWS → **CloudWatch** → creare Log Groups:
    - `/ecs/home-iot/api-gateway`, `/ecs/home-iot/auth-server`, etc.
- [ ] Creare una **CloudWatch Dashboard** con:
    - CPU e Memory utilization per ogni ECS Service
    - Numero di request all'ALB (RequestCount, TargetResponseTime)
    - Error rate (HTTPCode_Target_5XX_Count)
- [ ] Creare **CloudWatch Alarms**:
    - CPU > 80% per 5 minuti → notifica SNS
    - HealthyHostCount ALB = 0 → notifica SNS (critico)
    - DocumentDB FreeStorageSpace < 1 GB → notifica SNS
- [ ] Configurare **SNS Topic** per ricevere le notifiche via email

---

### 🤖 STEP 11 — CI/CD Pipeline (GitHub Actions)

- [ ] Repository GitHub → **Settings → Secrets and variables → Actions** → aggiungere:
    - `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_ACCOUNT_ID`, `AWS_REGION`
- [ ] Creare `.github/workflows/ci.yml` con change detection per modulo
- [ ] Creare `.github/workflows/deploy-service.yml` come reusable workflow
- [ ] Aggiungere workflow per ogni servizio che chiama il reusable workflow
- [ ] Testare una prima pipeline manuale con `workflow_dispatch`
- [ ] Verificare che il cambio a `home-iot-common` triggheri il rebuild di tutti i servizi

---

### 🏠 STEP 12 — On-Premises (zigbee2mqtt-forwarder)

- [ ] Aggiornare `zigbee2mqtt_forwarder.py` per supportare TLS con certificati X.509 verso AWS IoT Core
- [ ] Copiare i certificati (Step 4) nel container via volume Docker o secret
- [ ] Aggiornare il `docker-compose.yml` on-premises:
    - `EXTERNAL_BROKER=xxxx.iot.eu-west-1.amazonaws.com`
    - `EXTERNAL_PORT=8883`
    - `CA_CERT=/certs/AmazonRootCA1.pem`
    - `CERTFILE=/certs/certificate.pem.crt`
    - `KEYFILE=/certs/private.pem.key`
- [ ] Avviare il forwarder e verificare i messaggi su IoT Core (console → MQTT test client)

---

### ✅ STEP 13 — Test End-to-End e Go-Live

- [ ] Verificare che l'API Gateway risponda: `curl https://api.tuodominio.com/actuator/health`
- [ ] Testare il flusso di autenticazione: login → JWT → chiamata protetta
- [ ] Testare il flusso IoT: sensore Zigbee → forwarder → IoT Core → telemetry-writer → InfluxDB → telemetry-reader → bff-ios → app iOS
- [ ] Testare il flusso eventi: dispositivo → ms-events → RabbitMQ → ms-apartment/ms-gateway
- [ ] Verificare i log su CloudWatch per tutti i servizi
- [ ] Eseguire un test di carico leggero sull'ALB
- [ ] Verificare che gli alarms CloudWatch scattino correttamente in caso di errore simulato
- [ ] **Documentare** gli endpoint, le credenziali (in Secrets Manager) e il runbook operativo

---
