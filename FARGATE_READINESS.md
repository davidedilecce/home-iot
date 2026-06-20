# 🔍 Fargate Readiness Analysis — home-iot

> Analisi generata il 2026-04-08, basata sull'ispezione completa del codice sorgente.

---

## Verdetto Rapido

| Area | Stato | Note |
|------|-------|------|
| Dockerfiles | ✅ Pronto | Multi-stage, layer caching, non-root user, exec entrypoint |
| Health Check | ✅ Pronto | Actuator + probes liveness/readiness in tutti i 9 servizi |
| Graceful Shutdown | ✅ Pronto | `server.shutdown=graceful` + `lifecycle.timeout-per-shutdown-phase=15s` |
| Logging | ✅ Pronto | `logback-spring.xml` con JSON per profilo `aws` in tutti i 9 servizi |
| Sicurezza container | ✅ Pronto | `appuser:appgroup` non-root in tutti i Dockerfile |
| Configurazione env | ✅ Pronto | Profili `default`, `docker`, `aws` per ogni servizio |
| Actuator/Probes | ✅ Pronto | `spring-boot-starter-actuator` + probes in tutti i servizi |
| Immagini Docker | ✅ Pronto | Layer caching separato (dependencies vs source) |
| Resilienza | ✅ Buona | `bff-ios` ha circuit breaker + retry ben configurati |
| State/Storage | ✅ Stateless | Nessun file locale, nessuno stato in-process critico |

**Giudizio complessivo: ✅ GOOD TO GO.** Tutti i punti critici sono stati risolti.

---

## 1. 🔴 CRITICO — Health Check Assenti

### Problema

Fargate usa gli health check per decidere se un container è **sano**. Se non configurati:
- L'ALB non sa se il container è pronto → manda traffico a container ancora in boot
- ECS non sa quando riavviare un container bloccato → il servizio resta down silenziosamente

### Stato attuale

| Servizio | Ha `spring-boot-starter-actuator`? | Espone `/actuator/health`? |
|----------|-----------------------------------|----|
| api-gateway | ❌ No | ❌ No |
| auth-server | ❌ No | ❌ No |
| bff-ios | ❌ No | ❌ No |
| ms-apartment | ❌ No | ❌ No |
| ms-gateway | ❌ No | ❌ No |
| ms-user | ❌ No | ❌ No |
| ms-events | ❌ No | ❌ No |
| ms-telemetry-reader | ❌ No | ❌ No |
| ms-telemetry-writer | ✅ Sì | ✅ Sì (porta 8082) |

### Fix richiesto

**Per tutti gli 8 servizi web**, aggiungere in `build.gradle.kts`:
```kotlin
implementation(libs.spring.boot.starter.actuator)
```

E in ogni `application.yaml`:
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health
  endpoint:
    health:
      show-details: never
      probes:
        enabled: true   # abilita /actuator/health/liveness e /readiness
```

### Configurazione ECS Task Definition
```json
{
  "healthCheck": {
    "command": ["CMD-SHELL", "wget -qO- http://localhost:8080/actuator/health/liveness || exit 1"],
    "interval": 15,
    "timeout": 5,
    "retries": 3,
    "startPeriod": 60
  }
}
```

> **`startPeriod: 60`** è cruciale per Spring Boot: il JVM ha bisogno di tempo per avviarsi. Senza questo, Fargate uccide il container prima che finisca il boot.

### ALB Target Group Health Check
```
Path:     /actuator/health
Port:     8080
Interval: 15s
Timeout:  5s
Healthy:  2 consecutive
Unhealthy: 3 consecutive
```

---

## 2. 🔴 CRITICO — Graceful Shutdown Non Configurato

### Problema

Quando Fargate scala down o fa un rolling update, invia **SIGTERM** al container e aspetta `stopTimeout` secondi (default 30s) prima di SIGKILL. Senza graceful shutdown:
- Le richieste HTTP in-flight vengono troncate → errori 502 per i client
- `ms-events`: il consumer RabbitMQ potrebbe perdere messaggi in fase di processing (ACK manuale)
- `ms-telemetry-writer`: la connessione MQTT viene chiusa bruscamente

### Stato attuale

- Nessun servizio configura `server.shutdown=graceful`
- Nessun `@PreDestroy` o `DisposableBean` nel codice
- L'`ENTRYPOINT` usa `sh -c "java ..."` → **SIGTERM va a `sh`, non a Java** (PID 1 problem)

### Fix richiesto

**a) In ogni `application.yaml`:**
```yaml
spring:
  lifecycle:
    timeout-per-shutdown-phase: 15s

server:
  shutdown: graceful
```

**b) In ogni Dockerfile, cambiare l'ENTRYPOINT:**

```dockerfile
# ❌ PRIMA (SIGTERM non arriva a Java)
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]

# ✅ DOPO (Java riceve SIGTERM come PID 1)
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Per mantenere `JAVA_OPTS` dinamici, usare la forma `exec`:
```dockerfile
ENTRYPOINT ["sh", "-c", "exec java $JAVA_OPTS -jar app.jar"]
```

> Il `exec` sostituisce il processo `sh` con `java`, che diventa PID 1 e riceve SIGTERM.

**c) ECS Task Definition — stopTimeout:**
```json
{
  "stopTimeout": 30
}
```
Deve essere > del `timeout-per-shutdown-phase` di Spring (15s), così Spring ha tempo di drenare le connessioni.

---

## 3. 🔴 CRITICO — Logging Non Strutturato

### Problema

Fargate invia lo stdout/stderr a **CloudWatch Logs** tramite il log driver `awslogs`. Con il formato di log default di Spring Boot (multi-riga, stacktrace spezzate), CloudWatch:
- Spezza gli stacktrace Java in decine di log entry separate
- Rende impossibile filtrare/cercare per livello (ERROR, WARN)
- Non supporta query strutturate (CloudWatch Insights)

### Stato attuale

- Nessun `logback-spring.xml` in nessun servizio
- Nessuna configurazione logging in nessun `application.yaml`
- Output: formato Spring Boot default (testo libero)

### Fix richiesto

Creare `src/main/resources/logback-spring.xml` **in ogni servizio** (o in `home-iot-common`):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

  <!-- Profilo locale / docker: log leggibili -->
  <springProfile name="default,docker">
    <include resource="org/springframework/boot/logging/logback/console-appender.xml"/>
    <root level="INFO">
      <appender-ref ref="CONSOLE"/>
    </root>
  </springProfile>

  <!-- Profilo AWS: JSON strutturato per CloudWatch Insights -->
  <springProfile name="aws">
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
      <encoder class="ch.qos.logback.classic.encoder.JsonEncoder"/>
    </appender>
    <root level="INFO">
      <appender-ref ref="CONSOLE"/>
    </root>
  </springProfile>
</configuration>
```

> Con log JSON, su CloudWatch Insights si può fare:
> `fields @timestamp, message | filter level = "ERROR" | sort @timestamp desc`

---

## 4. 🔴 IMPORTANTE — Container Girano come Root

### Problema

Tutti i Dockerfile usano l'utente `root` di default. Questo è una **violazione di sicurezza** in qualsiasi ambiente cloud:
- Se un attaccante sfrutta una vulnerabilità nel JVM/app, ha accesso root nel container
- AWS Fargate isola i container, ma è comunque una best practice fondamentale
- Molte policy di compliance (SOC2, ISO27001) lo richiedono

### Stato attuale

Nessun Dockerfile contiene `USER`, `adduser`, o `addgroup`.

### Fix richiesto

In ogni Dockerfile, nello stage di runtime:

```dockerfile
# ===== Stage 2: Runtime =====
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

# Utente non-root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

COPY --from=build /workspace/.../build/libs/*.jar app.jar

EXPOSE 8080
ENTRYPOINT ["sh", "-c", "exec java $JAVA_OPTS -jar app.jar"]
```

---

## 5. 🟡 MEDIO — Manca il Profilo `aws`

### Problema

Hai i profili `default` e `docker`. Su Fargate servirà un profilo `aws` (o almeno passare tutto via env vars) per:
- Connessione DocumentDB con TLS + `retryWrites=false`
- Connessione Amazon MQ con TLS (`ssl.enabled=true`, porta 5671)
- Log JSON (vedi punto 3)
- Eventuali tuning specifici

### Stato attuale

La configurazione è quasi tutta parametrizzata via env vars ✅. Ma ci sono gap:

| Servizio | Parametro mancante per AWS |
|----------|---------------------------|
| Tutti con MongoDB | `spring.data.mongodb.uri` (oggi usa `host` separato — DocumentDB richiede URI completa con `?tls=true&retryWrites=false`) |
| ms-events, ms-telemetry-writer | `spring.rabbitmq.ssl.enabled`, porta 5671 |
| ms-telemetry-writer | Supporto TLS per MQTT (AWS IoT Core richiede certificati X.509) |

### Fix richiesto

Creare `application-aws.yaml` in ogni servizio con MongoDB:
```yaml
spring:
  data:
    mongodb:
      uri: ${MONGODB_URI}
      # sovrascrive host/database → usa URI completa con TLS
  rabbitmq:
    ssl:
      enabled: true
    port: 5671
  lifecycle:
    timeout-per-shutdown-phase: 15s

server:
  shutdown: graceful

management:
  endpoints:
    web:
      exposure:
        include: health
  endpoint:
    health:
      probes:
        enabled: true
```

Su Fargate: `SPRING_PROFILES_ACTIVE=aws`

---

## 6. 🟡 MEDIO — Immagini Docker Grandi e Lente a Buildare

### Problema

Ogni Dockerfile scarica le dipendenze Gradle da zero ad ogni build perché le `COPY` non sono stratificate per sfruttare il layer caching.

### Stato attuale

Il pattern attuale:
```dockerfile
COPY home-iot-ms-xxx/gradle/wrapper ...
COPY home-iot-ms-xxx/build.gradle.kts ...
COPY home-iot-ms-xxx/src ...          # ← invalida la cache ad ogni cambio di codice
RUN ./gradlew bootJar                  # ← ri-scarica TUTTE le dipendenze
```

### Fix consigliato

Separare il download delle dipendenze dal build del codice:
```dockerfile
# Stage 1: Dependencies (cached se build.gradle.kts non cambia)
COPY home-iot-ms-xxx/build.gradle.kts ...
COPY home-iot-ms-xxx/settings.gradle.kts ...
RUN ./gradlew dependencies --no-daemon  # ← scarica dipendenze (layer cached)

# Stage 2: Build (solo se il codice sorgente cambia)
COPY home-iot-ms-xxx/src ...
RUN ./gradlew bootJar --no-daemon -x test
```

Questo riduce il tempo di build CI da ~5min a ~1min quando cambia solo il codice sorgente.

---

## 7. 🟡 MEDIO — ENTRYPOINT Non Ottimale per Fargate

### Problema già coperto in §2, ma con dettaglio aggiuntivo:

L'attuale `ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]`:
1. **PID 1 problem** — `sh` è PID 1, Java è figlio → SIGTERM non arriva a Java
2. **Nessun default per `-XX:MaxRAMPercentage`** — su Fargate il container ha un memory limit fisso; senza questo flag il JVM può allocare troppa heap e essere OOM-killed

### Fix consigliato

```dockerfile
ENV JAVA_OPTS="-XX:MaxRAMPercentage=75.0 -XX:+UseG1GC"
ENTRYPOINT ["sh", "-c", "exec java $JAVA_OPTS -jar app.jar"]
```

> `-XX:MaxRAMPercentage=75.0` dice al JVM di usare al massimo il 75% della RAM visibile nel container. Il restante 25% serve per metaspace, thread stacks, buffer nativi, etc.

---

## 8. 🟢 OK — Cose Che Già Funzionano

### ✅ Stateless
- Nessun file scritto su disco
- Nessun stato in-memory critico (le cache in `ms-events` sono rigenerate ogni 5 min via `@Scheduled`)
- Perfetto per Fargate (container effimeri)

### ✅ Configurazione esternalizzata
- Tutti gli URL inter-servizio sono parametrizzati con env vars
- Secrets (JWT, admin password, InfluxDB token, MQTT credentials) già in env vars
- Pronto per Secrets Manager + Parameter Store

### ✅ Docker multi-stage build
- Tutti i Dockerfile usano multi-stage (build + runtime)
- Immagine base: `eclipse-temurin:21-jre-alpine` (leggera)
- Build context monorepo con `home-iot-common` incluso

### ✅ Resilience4j (bff-ios)
- Circuit breaker su 5 servizi downstream
- Retry con backoff su tutti i servizi
- Pronto per un ambiente distribuito

### ✅ Nessun localhost hardcoded
- Zero occorrenze di `localhost` o `127.0.0.1` nel codice Java
- Tutti i default `localhost` sono nei YAML e vengono sovrascitti da env vars

### ✅ `.dockerignore` presente
- Esclude `build/`, `.gradle/`, `.idea/`, `.git/`, immagini, markdown
- Le immagini Docker non conterranno file inutili

---

## 9. 📋 Checklist Riepilogativa — Da Fare Prima di Fargate

### Bloccanti (senza questi, il deploy sarà instabile)

- [x] **Aggiungere `spring-boot-starter-actuator`** a tutti gli 8 servizi web
- [x] **Configurare health check** (`/actuator/health` con probes liveness/readiness) in ogni servizio
- [x] **Abilitare graceful shutdown** (`server.shutdown=graceful` + `lifecycle.timeout-per-shutdown-phase=15s`)
- [x] **Fixare ENTRYPOINT** con `exec` in tutti i Dockerfile → `["sh", "-c", "exec java $JAVA_OPTS -jar app.jar"]`
- [x] **Creare utente non-root** in tutti i Dockerfile
- [x] **Aggiungere `-XX:MaxRAMPercentage=75.0`** al default `JAVA_OPTS`

### Importanti (per operatività in produzione)

- [x] **Creare `logback-spring.xml`** con output JSON per il profilo `aws`
- [x] **Creare profilo `application-aws.yaml`** con: MongoDB URI (TLS), RabbitMQ SSL, graceful shutdown
- [x] **Configurare connessione TLS per Amazon MQ** (`spring.rabbitmq.ssl.enabled=true`)
- [x] **Configurare connessione TLS per DocumentDB** (URI con `?tls=true&retryWrites=false` + CA cert)
- [x] **Ottimizzare layer caching** nei Dockerfile (separare dependencies da source)

### Consigliati (miglioramenti futuri)

- [ ] Aggiungere tracing distribuito (Micrometer + X-Ray o OpenTelemetry)
- [ ] Configurare auto-scaling ECS basato su CPU/RequestCount
- [ ] Aggiungere CloudWatch alarms su HealthyHostCount = 0 e CPU > 80%
- [ ] Passare a `HEALTHCHECK` anche dentro il Dockerfile (fallback se ECS health check non configurato)

---

## 10. Sizing Fargate Consigliato

| Servizio | vCPU | Memoria | Note |
|----------|------|---------|------|
| api-gateway | 0.25 | 512 MB | Proxy leggero, WebFlux |
| auth-server | 0.25 | 512 MB | Solo validazione JWT |
| bff-ios | 0.25 | 512 MB | WebFlux reattivo |
| ms-apartment | 0.25 | 512 MB | CRUD semplice |
| ms-gateway | 0.25 | 512 MB | CRUD semplice |
| ms-user | 0.25 | 512 MB | CRUD semplice |
| ms-events | 0.5 | 1024 MB | Consumer RabbitMQ + MongoDB + @Scheduled + cache in-memory |
| ms-telemetry-reader | 0.25 | 512 MB | Query InfluxDB |
| ms-telemetry-writer | 0.5 | 1024 MB | MQTT consumer + InfluxDB write + RabbitMQ producer |
| **TOTALE** | **2.75 vCPU** | **5.5 GB** | ~$90-120/mese Fargate |

> Con `JAVA_OPTS="-XX:MaxRAMPercentage=75.0"`:
> - 512 MB container → ~384 MB heap
> - 1024 MB container → ~768 MB heap



