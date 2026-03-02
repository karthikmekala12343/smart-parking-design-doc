# Smart Parking System Design Documentation

Design documentation for a Smart Parking Lot Management System managing vehicle entry/exit, spot allocation, and fee calculation.

## Low Level Design

### Objective

Design the low-level architecture for a backend system of a smart parking lot. This system should handle vehicle entry and exit management, parking space allocation, and fee calculation with efficient algorithms and concurrent operation support.

### Problem Statement

Imagine a parking lot in an urban area with multiple floors and numerous parking spots. The system needs to:

- Automatically assign available parking spots to vehicles based on their size (motorcycle, car, bus)
- Track entry and exit times for accurate duration calculation
- Calculate parking fees based on stay duration and vehicle type
- Provide real-time updates of parking spot availability
- Handle multiple simultaneous vehicle entries and exits without conflicts

### Functional Requirements

### 1. Parking Spot Allocation
- Automatically assign an available parking spot to a vehicle when it enters
- Match vehicle size to appropriate spot sizes (motorcycle → SMALL, car → MEDIUM, bus → LARGE)
- Use optimal allocation strategy (e.g., assign smallest suitable spot to maximize availability)
- Handle reservation timeouts and release reserved spots if vehicle doesn't arrive

### 2. Check-In and Check-Out
- Record vehicle entry with entry timestamp, vehicle details, and assigned spot
- Record vehicle exit with exit timestamp
- Maintain historical records of all parking transactions
- Support queries for active parking sessions

### 3. Parking Fee Calculation
- Calculate fees based on parking duration and vehicle type
- Apply grace periods (free parking for short durations)
- Support different hourly rates for different vehicle types
- Implement daily/maximum caps on parking fees
- Generate invoices upon exit

### 4. Real-Time Availability Update
- Maintain current availability status of all parking spots
- Cache spot availability for fast queries
- Update spot status immediately on entry/exit
- Provide API endpoints to query available spots by size and floor
- Publish real-time updates via WebSocket or Pub/Sub mechanism

### Design Aspects to Consider

### 1. Data Model
- **ParkingSpot**: Represents physical parking spaces with attributes (floor, size, status)
- **Vehicle**: Stores vehicle information (license plate, type, size category)
- **EntryRecord**: Transaction records linking vehicles to spots with timestamps and fees
- **Rate**: Pricing configuration rules for different vehicle types
- Proper indexing for efficient queries on status and timestamps

### 2. Spot Allocation Algorithm
- Use database transactions with row-level locking (`SELECT ... FOR UPDATE`)
- Query for the smallest available spot matching vehicle size
- Prevent race conditions with optimistic or pessimistic locking strategies
- Support partitioning by floor/spot size for scalability
- Implement timeout mechanism for failed allocations
- Use event-driven architecture with message brokers (Kafka, RabbitMQ)

### 3. Fee Calculation Logic
- Retrieve entry record and calculate parking duration
- Apply vehicle-specific rates from rate table
- Implement grace period logic (free parking for X minutes)
- Calculate tiered or hourly charges
- Apply maximum daily fee caps
- Handle partial hour calculations (round up)

### 4. Concurrency Handling
- Ensure atomic operations for spot assignment and exit processing
- Use row-level locking to prevent race conditions when multiple vehicles compete for spots
- Implement stateless services for horizontal scalability
- Use message queues to handle burst entry/exit events
- Consider eventual consistency with cache updates
- Implement reconciliation jobs for spot status correction

### Architecture Components

1. **API Gateway** – Routes external requests to internal services
2. **Entry Service** – Validates vehicles and assigns parking spots
3. **Exit Service** – Calculates fees and releases parking spots
4. **Spot Manager** – Manages availability cache and real-time updates
5. **Billing Module** – Handles rate logic and fee calculations
6. **Database** – Persistent data storage with transactional support
7. **Messaging Layer** – Event queue and Pub/Sub for real-time updates
8. **Monitoring & Logging** – System health, occupancy tracking, and anomalies

## Documentation

- [SmartParkingSystemDesign_LLD.md](SmartParkingSystemDesign_LLD.md) - Complete low-level design documentation including database schema, algorithms, and implementation details
