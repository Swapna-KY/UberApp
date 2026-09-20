# Uber Application

> [!NOTE]
> **Backend API Only:** This repository contains the backend microservices (APIs) for the Uber application. It does not include a frontend (web or mobile UI). You can interact with the APIs using Postman, PowerShell, or by building your own frontend client.

## Services Overview

| Service | Port | Responsibility |
|---|---|---|
| location-service | 8082 | Tracks real-time driver locations via Redis Geospatial |
| ride-service | 8083 | Manages ride lifecycle, publishes events to Kafka |
| matching-service | 8084 | Consumes ride events, finds & assigns best driver |

---

## Architecture Flow

`
Driver Phone -+' Location Service -+' Redis (GEOADD)

Rider App -+' Ride Service -+' Kafka (ride.requested)
                                      -+"
                           Matching Service (consumer)
                                      -+"
                           Location Service (find nearby drivers)
                                      -+"
                           Matching Algorithm (score drivers)
                                      -+"
                           Kafka (ride.matched)
                                      -+"
                           Ride Service (update ride with driver)
`

---

## How To Run

### Start the Entire Stack (Infrastructure + Microservices)
The project is fully containerized. You can start all the databases (MySQL, Redis), message brokers (Kafka, Zookeeper), and the Spring Boot microservices with a single command:

`ash
docker-compose up --build -d
`

> [!TIP]
> The MySQL database is mapped to port **3307** on your local machine to avoid conflicts with existing MySQL installations.

Wait about 30-60 seconds for Kafka and the microservices to fully boot up and connect. You do not need to run mvn spring-boot:run manually.

---

## Testing End-to-End Flow

### Step 1: Add Driver Locations (Location Service)
`
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
`

### Step 2: Request a Ride (Ride Service)
`
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
`

### Step 3: Check Ride Status
`
GET http://localhost:8083/api/v1/rides/{rideId}
`
You will see driverId assigned and status = ACCEPTED

### Step 4: Start the Ride
`
PUT http://localhost:8083/api/v1/rides/{rideId}/start
`

### Step 5: Complete the Ride
`
PUT http://localhost:8083/api/v1/rides/{rideId}/complete
`

### Step 6: Get Rider History
`
GET http://localhost:8083/api/v1/rides/rider/rider:1
`

---

## Verify in Redis CLI
`ash
docker exec -it redis-geo redis-cli

# See all stored drivers
ZRANGE drivers:locations 0 -1

# Check specific driver position
GEOPOS drivers:locations "driver:1"

# Distance between two drivers
GEODIST drivers:locations "driver:1" "driver:2" km
`

---

## Key Concepts Covered
- Redis Geospatial (GEOADD, GEORADIUS)
- Kafka event-driven architecture (Producer/Consumer)
- Ride state machine (REQUESTED -+' MATCHING -+' ACCEPTED -+' STARTED -+' COMPLETED)
- Driver scoring algorithm (distance + rating weighted score)
- Service-to-service REST communication (Matching -+' Location Service)
- Docker Compose for infrastructure setup
