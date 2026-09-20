# Uber Application

> **Backend API only.** This repository contains the backend microservices (APIs) for a simplified Uber-style ride-hailing system. It does not include a frontend (web or mobile UI). Interact with the APIs using Postman, curl, or by building your own client.

---

## Services Overview

| Service | Port | Responsibility |
|---|---|---|
| location-service | 8082 | Tracks real-time driver locations via Redis Geospatial |
| ride-service | 8083 | Manages ride lifecycle, publishes events to Kafka |
| matching-service | 8084 | Consumes ride events, finds & assigns the best driver |

---

## Architecture Flow

```
Driver Phone → Location Service → Redis (GEOADD)

Rider App → Ride Service → Kafka (ride.requested)
                                      ↓
                           Matching Service (consumer)
                                      ↓
                           Location Service (find nearby drivers)
                                      ↓
                           Matching Algorithm (score drivers)
                                      ↓
                           Kafka (ride.matched)
                                      ↓
                           Ride Service (update ride with driver)
```

---

## Prerequisites

- Java 21
- Maven
- Docker & Docker Compose

---

## How To Run

### Step 1: Start Infrastructure

```bash
docker-compose up -d
```

This starts Redis, MySQL, Zookeeper, and Kafka in containers. The Spring Boot services themselves are **not** containerized — run them locally with Maven (Steps 2–4).

Wait 30 seconds for Kafka to fully start before running the services.

### Step 2: Start Location Service

```bash
cd location-service
mvn spring-boot:run
```

### Step 3: Start Ride Service

```bash
cd ride-service
mvn spring-boot:run
```

Ride Service connects to MySQL on `localhost:3306`, database `uberapp`. Update the credentials in `src/main/resources/application.yaml` to match your local setup before running (see Configuration below).

### Step 4: Start Matching Service

```bash
cd matching-service
mvn spring-boot:run
```

---

## Configuration

`ride-service/src/main/resources/application.yaml` currently points at a local MySQL instance with a hardcoded username/password. Before running, either:

- update `spring.datasource.username` / `spring.datasource.password` to your own local MySQL credentials, or
- override them via environment variables / command-line args (e.g. `--spring.datasource.password=yourpassword`) so credentials aren't committed to source control.

The database itself (`uberapp` for local MySQL; `ride_db` is created by `docker-compose`) must exist before `ride-service` starts, unless you create it manually — `spring.jpa.hibernate.ddl-auto=update` will create/update tables but not the schema itself.

---

## Testing End-to-End Flow

### Step 1: Add Driver Locations (Location Service)

```
POST http://localhost:8082/api/v1/locations/drivers/update
{
    "driverId": "driver:1",
    "latitude": 12.9716,
    "longitude": 77.5946
}

POST http://localhost:8082/api/v1/locations/drivers/update
{
    "driverId": "driver:2",
    "latitude": 12.9800,
    "longitude": 77.5800
}

POST http://localhost:8082/api/v1/locations/drivers/update
{
    "driverId": "driver:3",
    "latitude": 12.9600,
    "longitude": 77.6100
}
```

### Step 2: Request a Ride (Ride Service)

```
POST http://localhost:8083/api/v1/rides/request
{
    "riderId": "rider:1",
    "pickupLatitude": 12.9716,
    "pickupLongitude": 77.5946,
    "pickupAddress": "MG Road, Bangalore",
    "dropLatitude": 12.9352,
    "dropLongitude": 77.6245,
    "dropAddress": "Koramangala, Bangalore"
}
```

### Step 3: Check Ride Status

```
GET http://localhost:8083/api/v1/rides/{rideId}
```

You should see `driverId` assigned and `status = ACCEPTED` once matching-service has processed the request (this happens asynchronously via Kafka, so allow a moment before checking).

### Step 4: Start the Ride

```
PUT http://localhost:8083/api/v1/rides/{rideId}/start
```

### Step 5: Complete the Ride

```
PUT http://localhost:8083/api/v1/rides/{rideId}/complete
```

### Step 6: Get Rider History

```
GET http://localhost:8083/api/v1/rides/rider/rider:1
```

---

## Verify in Redis CLI

```bash
docker exec -it redis-geo redis-cli

# See all stored drivers
ZRANGE drivers:locations 0 -1

# Check specific driver position
GEOPOS drivers:locations "driver:1"

# Distance between two drivers
GEODIST drivers:locations "driver:1" "driver:2" km
```

---

## Project Structure

```
├── location-service/     # Driver location tracking (Redis Geo)
├── ride-service/         # Ride lifecycle + Kafka producer (MySQL)
├── matching-service/     # Kafka consumer + driver matching (Feign → location-service)
└── docker-compose.yml    # Redis, MySQL, Zookeeper, Kafka
```

---

## Troubleshooting

- **matching-service can't reach location-service**: confirm `location.service.url` in `matching-service/src/main/resources/application.yaml` points to `http://localhost:8082` and that location-service is already running.
- **Ride stuck in `MATCHING` status**: check matching-service and ride-service logs — this usually means no drivers were found within the 5km search radius, or Kafka hasn't finished starting yet.
- **ride-service fails to start**: verify MySQL is up (`docker ps`) and the credentials in `application.yaml` match your database.

---

## Key Concepts Covered

- Redis Geospatial (`GEOADD`, `GEORADIUS`)
- Kafka event-driven architecture (Producer/Consumer)
- Ride state machine (`REQUESTED → MATCHING → ACCEPTED → RIDE_STARTED → COMPLETED`)
- Driver scoring algorithm (distance + rating weighted score)
- Service-to-service REST communication via Feign (Matching → Location Service)
- Docker Compose for infrastructure setup
