# 📘 E-TOUR System Documentation

## PART 7: DATA FLOW & DATABASE DESIGN

---

# 💾 Why Relational Database (MySQL)?

| Requirement | Why Relational DB |
|-------------|------------------|
| **Structured data** | Tours, bookings, customers have fixed schemas |
| **Relationships** | Booking → Customer, Booking → Tour, etc. |
| **Data integrity** | Foreign keys prevent orphan records |
| **ACID transactions** | Payment updates must be atomic |
| **Complex queries** | JOIN across multiple tables for reports |

---

# 📊 Core Tables Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           E-TOUR DATABASE SCHEMA                            │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│ customer_master  │     │ category_master  │     │ booking_status   │
├──────────────────┤     ├──────────────────┤     │ _master          │
│ customer_id (PK) │     │ category_id (PK) │     ├──────────────────┤
│ first_name       │     │ category_name    │     │ status_id (PK)   │
│ last_name        │     │ destination      │     │ status_name      │
│ email (unique)   │     │ description      │     └────────┬─────────┘
│ password         │     │ image_url        │              │
│ phone            │     │ no_of_days       │              │
│ auth_provider    │     └────────┬─────────┘              │
│ customer_role    │              │                        │
└────────┬─────────┘              │                        │
         │                        │                        │
         │         ┌──────────────┴───────────────────┐    │
         │         │                                  │    │
         │         ▼                                  ▼    │
         │  ┌──────────────────┐            ┌──────────────────┐
         │  │   tour_master    │            │  cost_master     │
         │  ├──────────────────┤            ├──────────────────┤
         │  │ tour_id (PK)     │            │ cost_id (PK)     │
         │  │ category_id (FK) │            │ category_id (FK) │
         │  │ departure_id (FK)│            │ single_person_cost│
         │  │ tour_guide_id    │            │ extra_person_cost │
         │  └────────┬─────────┘            │ child_with_bed    │
         │           │                      │ child_without_bed │
         │           │                      └──────────────────┘
         │           │
         │  ┌────────┴─────────┐     ┌──────────────────┐
         │  │departure_master  │     │ itinerary_master │
         │  ├──────────────────┤     ├──────────────────┤
         │  │ departure_id (PK)│     │ itinerary_id (PK)│
         │  │ category_id (FK) │     │ category_id (FK) │
         │  │ departure_date   │     │ day_number       │
         │  │ return_date      │     │ title            │
         │  │ seats_available  │     │ description      │
         │  └──────────────────┘     └──────────────────┘
         │
         │
         ▼
┌──────────────────┐                    ┌──────────────────┐
│  booking_header  │◄───────────────────│  payment_master  │
├──────────────────┤     payment links  ├──────────────────┤
│ booking_id (PK)  │     to booking     │ payment_id (PK)  │
│ customer_id (FK) │                    │ booking_id (FK)  │
│ tour_id (FK)     │                    │ payment_date     │
│ departure_id (FK)│                    │ payment_amount   │
│ number_of_pax    │                    │ payment_status   │
│ total_amount     │                    │ transaction_ref  │
│ status_id (FK)   │                    │ payment_mode     │
│ booking_date     │                    └──────────────────┘
└────────┬─────────┘
         │
         │ booking has
         │ passengers
         ▼
┌──────────────────┐
│    passenger     │
├──────────────────┤
│ passenger_id (PK)│
│ booking_id (FK)  │
│ first_name       │
│ last_name        │
│ age              │
│ pax_type         │
│ id_proof_type    │
│ id_proof_number  │
└──────────────────┘
```

---

# 🔗 Key Relationships (Foreign Keys)

| Parent Table | Child Table | Relationship | FK Column |
|--------------|-------------|--------------|-----------|
| `customer_master` | `booking_header` | One customer → Many bookings | `customer_id` |
| `category_master` | `tour_master` | One category → Many tours | `category_id` |
| `category_master` | `departure_master` | One category → Many departures | `category_id` |
| `category_master` | `cost_master` | One category → One cost | `category_id` |
| `category_master` | `itinerary_master` | One category → Many days | `category_id` |
| `booking_header` | `passenger` | One booking → Many passengers | `booking_id` |
| `booking_header` | `payment_master` | One booking → Many payments | `booking_id` |
| `booking_status_master` | `booking_header` | One status → Many bookings | `status_id` |

---

# 🏛️ Entity-to-Table Mapping

## How JPA Maps Entities:

| Java Entity | Database Table | Mapping Annotation |
|-------------|----------------|-------------------|
| `CustomerMaster` | `customer_master` | `@Table(name = "customer_master")` |
| `BookingHeader` | `booking_header` | `@Table(name = "booking_header")` |
| `PaymentMaster` | `payment_master` | `@Table(name = "payment_master")` |
| `CategoryMaster` | `category_master` | `@Table(name = "category_master")` |
| `Passenger` | `passenger` | `@Table(name = "passenger")` |

## Relationship Mapping:

```java
// BookingHeader.java
@Entity
@Table(name = "booking_header")
public class BookingHeader {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;
    
    // Many bookings belong to one customer
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")
    private CustomerMaster customer;
    
    // One booking has many passengers
    @OneToMany(mappedBy = "booking", cascade = CascadeType.ALL)
    private List<Passenger> passengers;
    
    // One booking has many payment attempts
    @OneToMany(mappedBy = "booking")
    private List<PaymentMaster> payments;
}
```

---

# 🔄 Data Flow: Create Booking Example

## Step-by-Step Data Movement:

```
1. FRONTEND
   User fills booking form → sends JSON
   
   {
     "customerId": 5,
     "categoryId": 2,
     "departureId": 10,
     "passengers": [
       { "firstName": "John", "paxType": "adult" },
       { "firstName": "Jane", "paxType": "child_with_bed" }
     ]
   }

2. CONTROLLER
   @PostMapping("/api/bookings")
   Receives DTO, passes to service

3. SERVICE
   - Fetches CustomerMaster (id=5)
   - Fetches TourMaster by category/departure
   - Fetches CostMaster for pricing
   - Calculates total amount
   - Creates BookingHeader entity
   - Creates Passenger entities
   - Sets status = PENDING

4. REPOSITORY
   bookingRepository.save(bookingHeader)
   
   JPA generates:
   INSERT INTO booking_header (...) VALUES (...)
   INSERT INTO passenger (...) VALUES (...)
   INSERT INTO passenger (...) VALUES (...)

5. DATABASE
   booking_header: new row with booking_id = 123
   passenger: 2 new rows linked to booking_id = 123

6. RESPONSE
   Service converts entity to DTO
   Controller returns:
   { "bookingId": 123, "status": "PENDING", "totalAmount": 35000 }
```

---

# 🐢 Lazy Loading Concept

## What is Lazy Loading?

When you fetch an entity, JPA doesn't immediately load related entities.

```java
BookingHeader booking = bookingRepository.findById(123);
// At this point, passengers are NOT loaded from DB

List<Passenger> passengers = booking.getPassengers();
// NOW a second query runs to fetch passengers
```

## Why Lazy Loading?

| Benefit | Explanation |
|---------|-------------|
| **Performance** | Don't load data you don't need |
| **Memory** | Less data in memory |
| **Speed** | Faster initial query |

## E-TOUR Configuration:

```java
@ManyToOne(fetch = FetchType.LAZY)  // ← Lazy by default for @ManyToOne
@JoinColumn(name = "customer_id")
private CustomerMaster customer;

@OneToMany(mappedBy = "booking", fetch = FetchType.LAZY)  // ← Explicit lazy
private List<Passenger> passengers;
```

## Lazy Loading Pitfall:

```java
// IN Controller (outside transaction):
BookingHeader booking = bookingService.getBooking(123);
booking.getPassengers();  // ❌ LazyInitializationException!

// SOLUTION: Use DTOs that are fully populated in service layer
```

---

# 📐 Generated/Calculated Columns

## Example: Total Amount Calculation

E-TOUR calculates `total_amount` in the service layer, not database:

```java
// BookingServiceImpl
BigDecimal total = passengers.stream()
    .map(p -> getPriceForPaxType(p.getPaxType(), costMaster))
    .reduce(BigDecimal.ZERO, BigDecimal::add);

booking.setTotalAmount(total);
```

## Why Application-Level Calculation?

| Approach | Pros | Cons |
|----------|------|------|
| **DB Generated Column** | Always consistent | Requires migration for formula changes |
| **Application Calculation** | Flexible, easy to change | Must ensure consistency |

---

# 🛡️ How DB Design Supports Business Rules

## Business Rule 1: "One customer can have multiple bookings"

```sql
-- Foreign key allows multiple bookings per customer
booking_header.customer_id → customer_master.customer_id
-- No UNIQUE constraint on customer_id
```

## Business Rule 2: "Booking must have at least one passenger"

```java
// Enforced in service layer, not DB
if (passengers.isEmpty()) {
    throw new RuntimeException("At least one passenger required");
}
```

## Business Rule 3: "Payment cannot exceed booking amount"

```java
// Validated in PaymentServiceImpl
if (paymentAmount.compareTo(booking.getTotalAmount()) > 0) {
    throw new RuntimeException("Payment exceeds booking amount");
}
```

## Business Rule 4: "Cascade delete passengers when booking deleted"

```java
@OneToMany(mappedBy = "booking", cascade = CascadeType.ALL, orphanRemoval = true)
private List<Passenger> passengers;
```

---

# ⚠️ What Happens with Poor DB Design?

## Scenario 1: No Foreign Keys

```
Problem: Booking references customer_id = 999 (doesn't exist)
Result: Orphan bookings, data corruption
E-TOUR Solution: FK with ON DELETE CASCADE where appropriate
```

## Scenario 2: No Indexes

```
Problem: SELECT * FROM booking_header WHERE customer_id = 5
         Full table scan on 1 million records
Result: Slow queries, poor user experience
E-TOUR Solution: Indexes on frequently queried columns
```

## Scenario 3: Denormalization Without Strategy

```
Problem: Customer name stored in booking_header
         Customer updates name
Result: Old bookings show old name
E-TOUR Solution: Store customer_id, join to get name
```

---

> **Continue to Part 8: Security & Authentication Flow →**
