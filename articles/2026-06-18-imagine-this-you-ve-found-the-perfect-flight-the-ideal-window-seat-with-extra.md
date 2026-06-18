Imagine this: you've found the perfect flight, the ideal window seat with extra legroom. You click "Book Now," confident in your choice. But two minutes later, a dreaded message flashes across the screen: "Seat Taken." Someone else snagged it. How did this happen, and how do engineers build systems that guarantee this never occurs?

This isn't just about an unlucky traveler; it's about the fundamental challenges of building robust, scalable, and consistent systems that millions rely on daily. Today, we're diving deep into the architecture of an airline reservation system, uncovering the complexities behind seemingly simple tasks like searching for flights or booking a seat.

### Understanding the Blueprint: Our Design Journey

Before we craft a solution, we must first understand the problem and its constraints. Our exploration will unfold in four key stages:

1.  **Defining the Mission: Requirements**
    We'll outline what our system *must do* (functional requirements) and *how well it must do it* (non-functional requirements).
2.  **Building Blocks: Entities & APIs**
    Next, we'll identify the core data structures and the interactions users (and other systems) will have with our reservation service.
3.  **The Grand Scheme: High-Level Design**
    Here, we'll sketch out the main components of our system, showing how they connect and collaborate.
4.  **Under the Hood: Deep Dives & Trade-Offs**
    Finally, we'll zoom in on the trickiest parts of the system, discussing critical design decisions and the compromises inherent in engineering at scale.

Our challenge: Design an airline reservation system that allows users to book flight seats and supports millions of daily active users. Let's begin.

## Defining the Mission: Requirements

When designing a system, especially in a high-stakes environment like an interview, it's crucial to identify the most important requirements. Focus on the core functionalities and critical performance aspects. For an airline reservation system, these boil down to a few key points:

### Functional Requirements

These define what the system *does*.

1.  **Users should be able to search for available flights.**
    This seems straightforward, but it's a beast. A user enters an origin, a destination, a date, maybe a return date, and the system must instantly query potentially thousands of flights. It needs to filter by availability, price, duration, number of stops, and return results within a second, even with thousands of concurrent searches. This is primarily a read-heavy operation demanding extreme speed.
2.  **Users should be able to book a flight and reserve a seat.**
    This is the heart of the system. A user picks a flight, selects a seat, and completes a booking. The critical challenge lies in the time window between a user selecting a seat and actually paying for it. During this period, another user might try to book the exact same seat. Our system must prevent **double bookings**.
3.  **Users should be able to view and manage their bookings.**
    After booking, users need to see their reservation details: flight info, seat assignment, booking status. They also need the flexibility to cancel or modify bookings if their plans change. This requirement impacts how we model data and the consistency guarantees needed. When a user cancels, that seat must immediately become available for others.
4.  **The system should process payments securely.**
    No booking is complete without payment. The system needs to charge the user and gracefully handle payment failures. We'll likely integrate with an external payment gateway (like Stripe or PayPal) to offload the complexity of payment processing itself, but we still need to coordinate between booking and payment states.

### Non-Functional Requirements

These define *how well* the system performs its functions.

1.  **High Availability for flight search.**
    When a user is searching for flights, it's okay if the data is slightly stale (e.g., a seat shows as available but gets taken a second later—that conflict will be handled at booking). What's *not* okay is the search system going down entirely. The system must always be responsive.
2.  **Low Latency for flight search (<500ms).**
    Users expect flight search results in under a second, ideally even faster. This is critical for a good user experience.
3.  **Strong Consistency for bookings.**
    When a user is booking and paying for a seat, we cannot afford stale data. Two people must *never* be able to book the same seat. This is paramount for system correctness.
4.  **High Throughput during peak times.**
    Consider a major airline opening bookings for a popular holiday route. Thousands of people will hit the system simultaneously. The system must scale smoothly under pressure, typically by adding more servers as demand grows.
5.  **Durability of booking data.**
    Once a booking is confirmed, it cannot be lost. Ever. Even if servers crash, networks fail, or transactions are interrupted, a confirmed booking is a legal and financial commitment. This necessitates durable storage with replication and backups.

## Building Blocks: Core Entities and APIs

Now that we understand the requirements, let's define the fundamental pieces of data our system will manage and how external systems will interact with them.

### Core Entities

Our system revolves around several key entities:

*   **Users:** Represents the customer.
    *   `user_id` (Primary Key)
    *   `user_name`
    *   `email`
    *   `password_hash`
    *   `phone`
*   **Flights:** Represents a scheduled journey. This is relatively static data.
    *   `flight_id` (Primary Key)
    *   `flight_number`
    *   `airline`
    *   `departure_airport` (e.g., JFK)
    *   `arrival_airport` (e.g., CDG)
    *   `departure_time`
    *   `arrival_time`
*   **Seats:** Individual seats on a specific flight.
    *   `seat_id` (Primary Key)
    *   `flight_id` (Foreign Key to Flights)
    *   `seat_number` (e.g., 12A)
    *   `class` (e.g., Economy, Business, First Class)
    *   `status` (e.g., Available, Held, Booked) – *This is a critical field for concurrency.*
*   **Bookings:** A record created when a user successfully reserves a seat on a flight.
    *   `booking_id` (Primary Key)
    *   `user_id` (Foreign Key to Users)
    *   `flight_id` (Foreign Key to Flights)
    *   `seat_id` (Foreign Key to Seats)
    *   `booking_reference` (Unique identifier for the user)
    *   `status` (e.g., Pending, Confirmed, Cancelled)
    *   `total_price`
    *   `created_at`
*   **Payments:** Captures the details of a transaction.
    *   `payment_id` (Primary Key)
    *   `booking_id` (Foreign Key to Bookings)
    *   `amount`
    *   `payment_method` (e.g., Credit Card, PayPal)
    *   `status` (e.g., Succeeded, Failed, Refunded)
    *   `transaction_reference` (From external payment provider)
    *   `created_at`

### Core APIs

We'll design one API for each core functional requirement.

1.  **Search Flights**
    This is a read operation (`GET`). The client passes query parameters, and the server returns a list of available flights.

    ```
    GET /flights/search?origin=JFK&destination=CDG&date=2025-08-15&passengers=1
    ```

    **Response:**

    ```json
    [
      {
        "flightId": "FL123",
        "airline": "Air France",
        "departureTime": "2025-08-15T10:00:00Z",
        "arrivalTime": "2025-08-15T18:00:00Z",
        "duration": "8h",
        "stops": 0,
        "priceTiers": {
          "economy": { "price": 500, "seatsAvailable": 20 },
          "business": { "price": 1200, "seatsAvailable": 5 },
          "firstClass": { "price": 2500, "seatsAvailable": 2 }
        }
      }
    ]
    ```

2.  **Hold a Seat**
    To avoid race conditions during the booking process, we introduce a `hold` step. This temporarily reserves a seat for a user while they complete passenger and payment details.

    ```
    POST /bookings/hold
    ```

    **Request Body:**

    ```json
    {
      "flightId": "FL123",
      "seatId": "12A"
    }
    ```

    **Response:**

    ```json
    {
      "holdId": "HOLD456",
      "flightId": "FL123",
      "seatId": "12A",
      "status": "reserved",
      "expiresAt": "2025-08-15T10:10:00Z"
    }
    ```
    The `expiresAt` timestamp is crucial: if the user doesn't complete the booking within this time (e.g., 10 minutes), the hold automatically expires, and the seat becomes available again. This elegantly separates seat locking from payment processing concerns.

3.  **Confirm Booking**
    Once the user has provided passenger details and payment information, the client sends a confirmation request.

    ```
    POST /bookings/confirm
    ```

    **Request Body:**

    ```json
    {
      "holdId": "HOLD456",
      "passengerDetails": {
        "name": "John Doe",
        "passportNumber": "A1234567",
        "dateOfBirth": "1990-01-01"
      },
      "paymentDetails": {
        "cardToken": "tok_visa_123"
      }
    }
    ```
    *Note: We send a `cardToken` from the payment provider (e.g., Stripe) instead of raw card numbers to avoid handling sensitive payment data directly on our servers, enhancing security.*

    **Response:**

    ```json
    {
      "bookingId": "BOOK789",
      "bookingReference": "XYZ123",
      "status": "confirmed",
      "flightDetails": { ... },
      "seatDetails": { ... },
      "totalAmount": 500,
      "confirmedAt": "2025-08-15T10:05:00Z"
    }
    ```

4.  **Get Booking Details**
    Users need to retrieve information about a specific booking.

    ```
    GET /bookings/:bookingId
    ```

    **Response:**

    ```json
    {
      "bookingId": "BOOK789",
      "status": "confirmed",
      "flightDetails": { ... },
      "seatDetails": { ... },
      "passengerDetails": { ... },
      "paymentStatus": "paid"
    }
    ```

5.  **Cancel Booking**
    Users can cancel a booking if plans change. We use `POST` instead of `DELETE` because cancellation is an action with side effects (triggering a refund, updating seat availability, changing booking status).

    ```
    POST /bookings/:bookingId/cancel
    ```

    **Response:**

    ```json
    {
      "bookingId": "BOOK789",
      "status": "cancelled",
      "refundAmount": 450
    }
    ```

## The Grand Scheme: High-Level Design

With requirements and APIs in hand, let's visualize the architecture. We'll build a microservices-based system to handle the scale and complexity.

### System Overview

Our system will consist of:

*   **Clients:** Mobile App, Web Browser, and potentially 3rd-Party Travel Platforms (like aggregators).
*   **API Gateway:** The entry point for all client requests.
*   **Microservices:** Dedicated services for specific functionalities.
*   **Data Stores:** Primary transactional database, message queue, and potentially a specialized search index.

Here's how the components connect:

```mermaid
graph TD
    subgraph CLIENTS
        C1[Mobile App] --> G
        C2[Web Browser] --> G
        C3[3rd-Party Travel Platform] --> G
    end

    G[API Gateway] --> MS1[Flight Search Service]
    G --> MS2[Booking Service]
    G --> MS3[Booking Management Service]

    subgraph MICROSERVICES
        MS1 --> DB1[PostgreSQL (Primary)]
        MS2 --> DB1
        MS3 --> DB1
        MS2 -- payment-request (async, dashed) --> KAFKA
        MS3 -- refund-request (async, dashed) --> KAFKA
        PAYMENT_SERVICE[Payment Service] -- transaction result --> KAFKA
        PAYMENT_SERVICE -- calls --> EXTERNAL_PAYMENT_PROVIDER[External Payment Provider (Stripe)]
        KAFKA -- booking-confirmed/failed --> MS2
        KAFKA -- refund-confirmed/failed --> MS3
        KAFKA --> NOTIFICATION_SERVICE[Notification Service (Email / SMS)]
    end

    subgraph KAFKA
        TOPIC1[topic: payment-request]
        TOPIC2[topic: booking-confirmed]
        TOPIC3[topic: booking-failed]
        TOPIC4[topic: refund-request]
        TOPIC5[topic: refund-confirmed]
        TOPIC6[topic: refund-failed]
    end

    DB1 -- reads --> MS1
    DB1 -- reads --> MS2
    DB1 -- reads --> MS3
```

### Component Breakdown

1.  **Clients:** Initiate requests.
2.  **API Gateway:**
    *   Acts as the **bouncer**: Handles **authentication** (ensuring valid tokens).
    *   Performs **rate limiting**: Protects backend services from being overwhelmed.
    *   Manages **routing**: Directs incoming requests to the appropriate microservice.
3.  **Microservices:**
    *   **Flight Search Service:** Handles flight search queries.
    *   **Booking Service:** Manages seat holds and initial booking creation.
    *   **Booking Management Service:** Handles confirmed bookings, cancellations, and modifications.
    *   **Payment Service:** Processes payments and refunds via an external provider.
    *   **Notification Service:** Sends email/SMS confirmations and updates.
4.  **PostgreSQL (Primary):**
    *   Our primary relational database, chosen for its **ACID properties**, **row-level locking**, and **strong consistency**, which are essential for critical transactional data like bookings and seats.
5.  **Kafka:**
    *   A distributed streaming platform used as a **message queue**. It decouples services, allowing asynchronous communication and improving system resilience. Different topics handle various event types (payment requests, booking confirmations, etc.).

### High-Level Workflow Walkthrough

Let's trace the core user flows:

*   **Searching for Flights:**
    1.  A user searches for flights via a client.
    2.  The API Gateway routes the request to the Flight Search Service.
    3.  The Flight Search Service queries the PostgreSQL database (and later, Elasticsearch, as we'll see in the deep dive) for matching flights and available seats.
    4.  Results are returned to the client quickly.

*   **Booking a Flight and Reserving a Seat:**
    1.  The user selects a flight and a specific seat.
    2.  The API Gateway routes the request to the Booking Service.
    3.  The Booking Service attempts to acquire a "hold" on that seat (temporarily making it unavailable). This is a critical step for concurrency.
    4.  The client fills in passenger details and payment information.
    5.  The client sends a "confirm booking" request to the Booking Service.
    6.  The Booking Service creates a booking record in a **pending** state in PostgreSQL and publishes a `payment-request` event to Kafka. It doesn't wait for payment to complete synchronously.

*   **Processing Payment (Asynchronously):**
    1.  The Payment Service consumes the `payment-request` event from Kafka.
    2.  It calls the External Payment Provider (e.g., Stripe) to process the transaction.
    3.  Upon success or failure, the Payment Service publishes a `booking-confirmed` or `booking-failed` event back to Kafka.
    4.  The Booking Service consumes these events:
        *   If `booking-confirmed`, it updates the booking status in PostgreSQL to **confirmed** and marks the seat as **booked**.
        *   If `booking-failed`, it cancels the pending booking, releases the seat (updates status back to `available`), and triggers any necessary compensation.
    5.  The Notification Service also consumes `booking-confirmed` / `booking-failed` events and sends email/SMS to the user.

*   **Viewing/Managing Bookings:**
    1.  A user requests their booking history or details of a specific booking.
    2.  The API Gateway routes the request to the Booking Management Service.
    3.  The Booking Management Service queries PostgreSQL for the booking data.
    4.  The user can also initiate a cancellation, which triggers a `refund-request` event to Kafka, similar to the payment process.

This high-level design demonstrates a clear separation of concerns using microservices and leverages a message queue for resilient asynchronous communication. Now, let's tackle the toughest challenges.

## Under the Hood: Deep Dives

These deep dives expose the subtle but crucial decisions that differentiate a robust, scalable system from one prone to failure.

### Deep Dive 1: Preventing Double Bookings with Atomic Operations

**Problem:** Two users, Alice and Bob, simultaneously try to book the same seat (e.g., 12A on Flight FL123). If the system simply checks if the seat is available and then proceeds to book it, both Alice and Bob might see it as "available" before either transaction commits. This creates a race condition, leading to two bookings for one seat—a catastrophic outcome for an airline.

**Struggle with Naive Approach:**
A naive approach might look like this:
1.  User A checks `Seat 12A` -> `Available`
2.  User B checks `Seat 12A` -> `Available`
3.  User A proceeds to book `Seat 12A` -> `Success`
4.  User B proceeds to book `Seat 12A` -> `Success` (Double Booking!)

**Insight & Solution: Atomic Update with Guard Condition**

The key insight is that the "check and set" operation must be **atomic**. Relational databases like PostgreSQL provide mechanisms for this. Instead of separate `SELECT` and `UPDATE` statements, we combine them into a single, atomic `UPDATE` with a strong guard condition:

```sql
UPDATE seats
SET status = 'booked',
    booked_by = $user_id
WHERE flight_id = 'FL123'
  AND seat_id = '12A'
  AND status = 'available'; -- This is the crucial guard condition!
```

**How it works:**

1.  **Alice's Request:** Alice's booking service sends the `UPDATE` query. PostgreSQL initiates a transaction and places a **row-level lock** on `Seat 12A`.
2.  **Bob's Request:** Bob's booking service sends the exact same `UPDATE` query. When it tries to acquire a lock on `Seat 12A`, it finds that Alice's transaction already holds the lock. Bob's transaction is **forced to wait**.
3.  **Alice's Commit:** Alice's transaction successfully commits. The `status` of `Seat 12A` in the database is changed to `'booked'`. The row-level lock is **released**.
4.  **Bob's Resumption:** Bob's waiting transaction resumes. PostgreSQL re-evaluates the `WHERE` clause. Now, `status = 'available'` is `FALSE` because Alice just changed it to `'booked'`.
5.  **Bob's Failure:** The `UPDATE` query finds no rows matching the `WHERE` clause (since the status is no longer 'available'). No rows are updated. The booking service detects this (e.g., `rows_affected == 0`) and returns an error message: "Sorry, this seat has already been booked. Please choose another seat."

**Trade-offs:**
*   **Blocking:** There is brief blocking, as one transaction waits for the other. However, the lock duration is extremely short (only for the single `UPDATE` statement, not the entire booking workflow), making this efficient even under high contention.
*   **No external locking:** We avoid introducing complex distributed locking mechanisms (like ZooKeeper or Consul) for this specific problem, leveraging the database's inherent capabilities. This keeps the design simpler and reduces operational overhead.

### Deep Dive 2: Graceful Seat Hold Expiration

**Problem:** A user selects and "holds" a seat (preventing others from booking it) but then abandons the booking process (e.g., closes the tab, navigates away, phone dies). Without intervention, this seat remains perpetually held, even though no one intends to purchase it, leading to lost revenue and a poor experience for other potential buyers. The problem is how to release these held seats automatically and precisely after a defined timeout (e.g., 10 minutes).

**Struggle with Cron Job Expiration:**
A common naive approach is to use a background **cron job**:

1.  When a seat is held, store an `expires_at` timestamp in the database.
2.  A cron job runs every `X` minutes (e.g., 10 minutes).
3.  The cron job queries the database for all seats in `'held'` status where `expires_at` is in the past.
4.  It updates these seats to `'available'` and cleans up any related pending booking records.

**Trade-offs of Cron Job:**
*   **Imprecision:** If the cron job runs every 10 minutes, a seat held at 12:01 PM with a 10-minute expiry (12:11 PM) might not be released until the cron job next runs at 12:15 PM, meaning the seat is unnecessarily unavailable for up to 14 minutes. In a high-demand scenario, this lost availability can be significant.
*   **Scalability:** Scanning large tables frequently can be resource-intensive, especially with millions of seats and frequent updates.

**Insight & Solution: Distributed Locking with Redis TTL**

A more elegant and scalable solution uses a **distributed locking service** like Redis, leveraging its **Time-To-Live (TTL)** feature. This moves the temporary hold state out of the primary transactional database and into a highly performant, in-memory data store designed for ephemeral data.

```mermaid
graph TD
    subgraph MICROSERVICES
        BS[Booking Service]
        DLS[Distributed Locking Service (Redis)]
    end

    BS -- attempts to acquire lock --> DLS
    DLS -- returns lock status + TTL --> BS

    DLS --- C1[Key: "f47ac10b"]
    C1 --- C2[Value: "locked"]
    C1 --- C3[TTL_seconds: 600]
    C1 --- C4[TTL_readable: "10 minutes"]
```

**How it works:**

1.  **Acquire Lock:** When a user attempts to hold a seat, the Booking Service first tries to acquire a lock in the Distributed Locking Service (Redis). This lock is typically a key-value pair where the key is derived from the `flightId` and `seatId`, and the value indicates it's locked.
2.  **Time-To-Live (TTL):** Crucially, this Redis key is set with a **TTL** (e.g., 600 seconds for 10 minutes). Redis itself is responsible for automatically deleting this key once the TTL expires.
3.  **Lock Success/Failure:**
    *   If the lock is successfully acquired (the key didn't exist), the seat is considered temporarily reserved. The Booking Service returns the `holdId` and `expiresAt` timestamp to the client.
    *   If the lock cannot be acquired (another user already holds the lock), the seat is shown as unavailable.
4.  **Automatic Expiration:** If the user abandons the booking process, Redis automatically deletes the lock after its TTL, making the seat immediately available again without any explicit cleanup logic from our application or cron jobs.

**Trade-offs:**
*   **Eventual Consistency (Read Path):** While Redis provides atomic operations for locking, it's a separate system. There's a tiny window where a seat might be freed in Redis but not immediately reflected in the primary database (if seat status was also tracked there). However, for the *read path* (search), we typically prioritize availability over strict consistency for temporary holds. The primary database only stores confirmed bookings.
*   **Additional Infrastructure:** Introduces Redis, adding a component to manage.

**Benefits:**
*   **Precision:** Seats become available immediately upon expiration, maximizing availability.
*   **Scalability:** Redis is highly optimized for fast, ephemeral key-value operations at scale.
*   **Simplicity:** No complex cron jobs or manual cleanup logic. Redis handles expiration natively.
*   **Decoupling:** The temporary hold state is separated from the durable booking state.

### Deep Dive 3: Resilient Payment Processing with the Saga Pattern

**Problem:** Payment processing is often slow, unreliable, and involves external services (e.g., Stripe, PayPal). In a distributed microservices architecture, a single logical transaction (booking + payment) spans multiple services. If the payment fails or the Payment Service crashes mid-transaction, how do we ensure the entire system returns to a consistent state? We cannot simply roll back a successful payment that has already occurred with an external provider.

**Struggle with Distributed Transactions (2PC):**
Traditional distributed transaction protocols like **Two-Phase Commit (2PC)** are complex, can suffer from blocking, and are often not practical across disparate services, especially when external services are involved. We can't tell Stripe to "rollback" a payment.

**Insight & Solution: Asynchronous Communication and the Saga Pattern**

The **Saga pattern** addresses distributed transactions by breaking them down into a sequence of local, atomic transactions. Each local transaction updates its own database and publishes an event to a message queue (Kafka). Other services react to these events, performing their own local transactions. If any step fails, compensating transactions are triggered to undo prior actions.

```mermaid
graph TD
    subgraph MICROSERVICES
        BS[Booking Service]
        PS[Payment Service]
        MS[Booking Management Service]
    end

    subgraph KAFKA
        PQ[topic: payment-request]
        BC[topic: booking-confirmed]
        BF[topic: booking-failed]
        RQ[topic: refund-request]
        RC[topic: refund-confirmed]
        RF[topic: refund-failed]
    end

    EXTERNAL_PAYMENT_PROVIDER[External Payment Provider (Stripe)]

    BS --1. publish payment-request--> PQ
    PS --2. consume payment-request & call external provider--> EXTERNAL_PAYMENT_PROVIDER
    EXTERNAL_PAYMENT_PROVIDER --3. payment result --> PS
    PS --4a. publish booking-confirmed--> BC
    PS --4b. publish booking-failed--> BF

    BS --5a. consume booking-confirmed & finalize booking--> DB_BOOKING[PostgreSQL (Bookings)]
    BS --5b. consume booking-failed & undo booking--> DB_BOOKING

    MS --6. consume booking-failed (for compensation)--> MS
    MS --7. publish refund-request (if needed)--> RQ
    PS --8. consume refund-request & call external provider--> EXTERNAL_PAYMENT_PROVIDER
    EXTERNAL_PAYMENT_PROVIDER --9. refund result --> PS
    PS --10. publish refund-confirmed/failed--> RC & RF

    NOTIFICATION_SERVICE[Notification Service (Email / SMS)]
    BC --> NOTIFICATION_SERVICE
    BF --> NOTIFICATION_SERVICE
    RC --> NOTIFICATION_SERVICE
    RF --> NOTIFICATION_SERVICE
```

**How it works (Simplified Booking Saga):**

1.  **Step 1: Create Pending Booking (Booking Service)**
    *   Booking Service creates a booking record in `PENDING` status and places a hold on the seat.
    *   **Compensation:** If later steps fail, cancel the `PENDING` booking and release the seat.
2.  **Step 2: Process Payment (Payment Service)**
    *   Booking Service publishes a `payment-request` event to Kafka.
    *   Payment Service consumes this event, processes payment via Stripe.
    *   **If Payment Fails:** Payment Service publishes `booking-failed` to Kafka. Booking Service consumes it, triggers compensation for Step 1. User is notified.
    *   **If Payment Succeeds:** Payment Service publishes `booking-confirmed` to Kafka.
    *   **Compensation:** If Payment Service crashes *after* successful payment but *before* publishing `booking-confirmed`, a separate process (or the saga orchestrator) can detect the payment in Stripe and reconcile the booking, or trigger a refund.
3.  **Step 3: Confirm Booking (Booking Service)**
    *   Booking Service consumes `booking-confirmed` from Kafka.
    *   It updates the booking status to `CONFIRMED` and finalizes the seat assignment in PostgreSQL.
    *   **Compensation:** If Booking Service crashes *after* confirming but *before* notifying, a reconciliation process reverts to `PENDING` and triggers a refund if needed (this prevents charging a user for an unconfirmed booking).

**Failure Handling (Example):**

*   **Payment fails at Step 2:** The saga triggers compensation for Step 1. The booking is cancelled, the seat is released, and the user is notified of the payment failure.
*   **System crashes after payment, before confirmation:** The saga detects the incomplete state. It refunds the payment (compensating Step 2) and releases the seat (compensating Step 1), preventing an invalid charge.

**Trade-offs:**
*   **Complexity:** Saga patterns introduce significant complexity. Designing compensating actions for every step and handling out-of-order messages or failures during compensation is challenging.
*   **Eventual Consistency:** The user doesn't get an *instant* confirmation. They see a "pending" state and receive a notification later. For airline bookings, this is generally acceptable.

**Benefits:**
*   **Resilience:** The system can recover from failures in any service without data corruption.
*   **Decoupling:** Services operate independently, improving scalability and maintainability.
*   **Scalability:** Asynchronous processing allows the system to handle high throughput without bottlenecks from slow external calls.
*   **Correctness:** Avoids scenarios like users being charged for unconfirmed bookings.

### Deep Dive 4: Achieving Low-Latency Flight Search with CDC

**Problem:** The Flight Search Service needs to be extremely fast (<500ms) and highly available. Continuously querying the primary PostgreSQL database for complex searches (filtering by routes, dates, prices, stops, and *real-time* seat availability) would overwhelm it, impacting transactional operations (like booking).

**Struggle with Direct Primary DB Queries:**
While PostgreSQL is great for transactional consistency, it's not optimized for millions of complex, analytical-style search queries per day. These heavy reads would create latency spikes and bottleneck the system. Simply adding read replicas might help with read load but won't fundamentally solve the complex query performance unless highly optimized indexes are in place, and keeping eventual consistency in mind for seat availability is crucial.

**Insight & Solution: Dedicated Search Index (Elasticsearch) via Change Data Capture (CDC)**

To achieve low-latency, scalable search, we introduce a dedicated search engine like Elasticsearch, which is optimized for complex queries over large datasets. To keep Elasticsearch synchronized with the primary database, we use **Change Data Capture (CDC)**.

```mermaid
graph TD
    subgraph MICROSERVICES
        FS[Flight Search Service]
    end

    POSTGRESQL[PostgreSQL (Primary)]
    ELASTICSEARCH[Elasticsearch (Search Index)]
    DEBEZIUM[Debezium (CDC)]
    SYNC_SERVICE[Sync Service]
    KAFKA[Kafka]

    FS -- queries --> ELASTICSEARCH
    POSTGRESQL -- reads WAL --> DEBEZIUM
    DEBEZIUM -- publishes change events --> KAFKA
    KAFKA -- consumes change events --> SYNC_SERVICE
    SYNC_SERVICE -- writes updates --> ELASTICSEARCH
```

**How it works:**

1.  **Primary Data Source:** PostgreSQL remains the **source of truth** for flight and seat data due to its strong transactional guarantees.
2.  **Change Data Capture (Debezium):**
    *   A CDC tool like **Debezium** (an open-source connector) attaches itself to PostgreSQL's **Write-Ahead Log (WAL)**.
    *   Debezium continuously monitors the WAL for any inserts, updates, or deletes to the `flights` and `seats` tables.
    *   Whenever a change occurs, Debezium captures it as a **change event** and publishes it to a Kafka topic.
3.  **Asynchronous Synchronization (Sync Service):**
    *   A **Sync Service** consumes these change events from Kafka.
    *   It then applies these changes to Elasticsearch, updating the search index.
4.  **Fast Search:**
    *   The Flight Search Service now queries Elasticsearch instead of PostgreSQL. Elasticsearch, with its inverted index, can filter and rank millions of records in milliseconds.
    *   For seat availability, the Flight Search Service will query Elasticsearch for available flights based on *stale* data (which is acceptable for search, as established in NFRs). Then, when a user attempts to *hold* a seat, the Booking Service will perform a precise, atomic check against the *primary* database and Redis lock, as discussed in Deep Dive 1 & 2.

**Trade-offs:**
*   **Eventual Consistency:** The search index (Elasticsearch) is **eventually consistent** with the primary database. There's a small, acceptable delay (typically milliseconds to a few seconds) between a change in PostgreSQL and its reflection in Elasticsearch. For flight search, users don't expect absolute real-time accuracy for every available seat on the initial search results page; the critical consistency check happens at the booking stage.
*   **Increased Infrastructure:** Adds Elasticsearch, Debezium, and Kafka topics for CDC, increasing operational complexity.

**Benefits:**
*   **Blazing Fast Search:** Achieves the low-latency requirement even under massive query loads.
*   **Scalable Reads:** Elasticsearch can scale horizontally to handle millions of search queries.
*   **Primary DB Protection:** Offloads heavy read queries from the transactional database, allowing it to focus on critical writes.
*   **Clean Application Logic:** The Flight Search Service remains simple; it just queries Elasticsearch. The complexity of synchronization is handled by the CDC pipeline.

## Conclusion

Designing a robust airline reservation system is a masterclass in balancing performance, consistency, and resilience in a distributed environment. We've journeyed from high-level requirements to the intricate details of handling concurrent bookings and payment failures. The insights gained—from atomic updates for concurrency to distributed locks for temporary holds and CDC for scalable search—are fundamental patterns that extend far beyond airline systems.

Ultimately, successful system design isn't just about knowing the answers; it's about asking the right questions, understanding the trade-offs, and strategically applying architectural patterns to build a reliable and enjoyable user experience. The digital world is full of complexities, but with careful thought and the right tools, we can build systems that reliably deliver, even when thousands are clamoring for that same perfect window seat.