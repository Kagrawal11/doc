# 📘 E-TOUR System Documentation

## PART 2: OVERALL SYSTEM ARCHITECTURE

---

# 🏗️ Client-Server Architecture

E-TOUR follows a **3-Tier Client-Server Architecture**:

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   PRESENTATION  │     │    BUSINESS     │     │      DATA       │
│      TIER       │────▶│      TIER       │────▶│      TIER       │
│   (Frontend)    │     │   (Backend)     │     │   (Database)    │
└─────────────────┘     └─────────────────┘     └─────────────────┘
     React/Vite           Spring Boot              MySQL
     Browser              Server                   Persistence
```

## Why Three Tiers?

| Tier | Responsibility | Benefit of Separation |
|------|----------------|----------------------|
| **Presentation** | User interface, user interaction | Can change UI without touching business logic |
| **Business** | Rules, validation, processing | Can update rules without changing UI or database |
| **Data** | Storage, retrieval, persistence | Can switch databases without changing code |

---

# 🔗 Why Frontend and Backend Are Separated

## Monolithic vs Separated Architecture

| Approach | Description | E-TOUR Choice |
|----------|-------------|---------------|
| **Monolithic** | UI + Logic + DB in one application | ❌ Not used |
| **Separated** | Frontend app + Backend API + Database | ✅ Used |

## Benefits of Separation in E-TOUR:

| Benefit | Explanation |
|---------|-------------|
| **Independent scaling** | If 1000 users browse, scale frontend servers only |
| **Technology freedom** | React team doesn't need to know Java |
| **Parallel development** | Frontend and backend teams work simultaneously |
| **Mobile-ready** | Same backend can serve mobile apps later |
| **Testing isolation** | Test UI and API independently |
| **Deployment flexibility** | Deploy frontend to CDN, backend to cloud |

---

# 🔄 Role of REST APIs

REST (Representational State Transfer) is the communication protocol between frontend and backend.

## How Frontend Talks to Backend:

```
Frontend                                           Backend
   │                                                  │
   │  GET /api/tours                                  │
   │ ─────────────────────────────────────────────▶   │
   │                                                  │
   │  { "tours": [...] }                              │
   │ ◀─────────────────────────────────────────────   │
   │                                                  │
   │  POST /api/bookings                              │
   │  { "tourId": 5, "passengers": [...] }            │
   │ ─────────────────────────────────────────────▶   │
   │                                                  │
   │  { "bookingId": 123, "status": "PENDING" }       │
   │ ◀─────────────────────────────────────────────   │
```

## REST Principles Used in E-TOUR:

| Principle | Implementation |
|-----------|----------------|
| **Stateless** | Each request carries all info (JWT token) |
| **Uniform Interface** | Consistent URL patterns (`/api/bookings/{id}`) |
| **Resource-based** | URLs represent things (tours, bookings, customers) |
| **HTTP Methods** | GET=read, POST=create, PUT=update, DELETE=remove |
| **JSON Format** | All data exchange in JSON |

---

# 🔐 Stateless vs Stateful: Why Stateless?

## Stateful (Traditional - NOT used):
```
Server remembers: "User #5 is logged in, their cart has 3 items"
Problem: Server restart = all sessions lost
Problem: Multiple servers = session sync nightmare
```

## Stateless (E-TOUR approach - USED):
```
Server remembers: NOTHING
Every request says: "I am User #5, here's my JWT proof"
Benefit: Any server can handle any request
Benefit: Horizontal scaling is trivial
```

## How Statelessness Works:

```
┌────────────────────────────────────────────────────────────────┐
│ REQUEST (Every single one):                                    │
│                                                                │
│ Headers:                                                       │
│   Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2Vy │
│                                                                │
│ The JWT contains: { email, role, expiry }                      │
│ Server validates JWT → knows who the user is                   │
│ No session storage needed on server                            │
└────────────────────────────────────────────────────────────────┘
```

---

# 🧩 Where Each Component Fits

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         COMPLETE SYSTEM MAP                                 │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                      USER'S BROWSER                          │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ React Frontend:                                        │  │
│  │  • UI Components (what user sees)                      │  │
│  │  • AuthContext (stores JWT locally)                    │  │
│  │  • BookingContext (tracks booking wizard state)        │  │
│  │  • API Layer (axios calls to backend)                  │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                              │
                              │ HTTP + JWT Token
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                    SPRING BOOT BACKEND                       │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ Security Layer:                                        │  │
│  │  • JwtAuthFilter (validates token on every request)    │  │
│  │  • SecurityConfig (defines what's public/protected)    │  │
│  │  • CORS config (allows frontend origin)                │  │
│  └────────────────────────────────────────────────────────┘  │
│                              │                               │
│                              ▼                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ Controller Layer:                                      │  │
│  │  • AuthController (/auth/*)                            │  │
│  │  • TourController (/api/tours/*)                       │  │
│  │  • BookingController (/api/bookings/*)                 │  │
│  │  • PaymentGatewayController (/payment-gateway/*)       │  │
│  │  • SearchController (/search/*)                        │  │
│  └────────────────────────────────────────────────────────┘  │
│                              │                               │
│                              ▼                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ Service Layer:                                         │  │
│  │  • AuthService (login, register, JWT generation)       │  │
│  │  • BookingService (create booking, add passengers)     │  │
│  │  • PaymentService (record payments)                    │  │
│  │  • SearchService (filter tours)                        │  │
│  │  • RazorpayServiceImpl (gateway integration)           │  │
│  └────────────────────────────────────────────────────────┘  │
│                              │                               │
│                              ▼                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ Repository Layer:                                      │  │
│  │  • CustomerRepository                                  │  │
│  │  • BookingRepository                                   │  │
│  │  • PaymentRepository                                   │  │
│  │  • TourRepository, CategoryRepository, etc.            │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                              │
                              │ JDBC/JPA
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                         MySQL DATABASE                       │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ Tables:                                                │  │
│  │  • customer_master (users)                             │  │
│  │  • category_master (tour categories)                   │  │
│  │  • tour_master (specific tours)                        │  │
│  │  • departure_master (tour dates)                       │  │
│  │  • booking_header (bookings)                           │  │
│  │  • passenger (travelers)                               │  │
│  │  • payment_master (transactions)                       │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                              │
                              │ (also connects to)
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                      EXTERNAL SERVICES                       │
│  ┌─────────────────────┐  ┌─────────────────────────────┐   │
│  │ Razorpay            │  │ Gmail SMTP                  │   │
│  │ • Create orders     │  │ • Send emails               │   │
│  │ • Process payments  │  │ • Password reset            │   │
│  │ • Webhooks          │  │ • Booking confirmations     │   │
│  └─────────────────────┘  └─────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

---

# 🔄 End-to-End Request Lifecycle

## Example: User Books a Tour

```
STEP 1: User clicks "Book Now" on Tour Detail page
─────────────────────────────────────────────────────
Browser → React component triggers API call

STEP 2: Frontend sends POST request
─────────────────────────────────────────────────────
axios.post('/api/bookings', {
    tourId: 5,
    customerId: 12,
    passengers: [...]
}, {
    headers: { Authorization: 'Bearer eyJ...' }
})

STEP 3: Request hits Spring Security filter
─────────────────────────────────────────────────────
JwtAuthFilter:
  - Extracts token from Authorization header
  - Validates signature and expiry
  - Sets authenticated user in SecurityContext
  - Allows request to proceed

STEP 4: Request reaches BookingController
─────────────────────────────────────────────────────
@PostMapping("/api/bookings")
public BookingDTO createBooking(@RequestBody CreateBookingDTO dto) {
    return bookingService.createBooking(dto);
}

STEP 5: Service layer processes business logic
─────────────────────────────────────────────────────
BookingServiceImpl:
  - Validates tour exists
  - Validates customer exists
  - Calculates total amount
  - Creates BookingHeader entity
  - Creates Passenger entities
  - Sets status = PENDING

STEP 6: Repository saves to database
─────────────────────────────────────────────────────
bookingRepository.save(bookingEntity);
passengerRepository.saveAll(passengers);

→ JPA generates INSERT SQL
→ MySQL stores records

STEP 7: Response travels back
─────────────────────────────────────────────────────
BookingServiceImpl → BookingController → HTTP Response
{ "bookingId": 123, "status": "PENDING", "totalAmount": 25000 }

STEP 8: Frontend receives and updates UI
─────────────────────────────────────────────────────
React:
  - Updates BookingContext with new booking
  - Navigates to payment step
  - Shows "Proceed to Payment" button
```

---

# ⏱️ Request Timeline

```
TIME      COMPONENT              ACTION
────      ─────────              ──────
0ms       Browser                User clicks button
5ms       React                  Creates HTTP request
10ms      Network                Request in transit
50ms      Spring Security        JWT validation
55ms      Controller             Receives request
60ms      Service                Business logic
100ms     Repository             Database query
150ms     MySQL                  Data saved
155ms     Repository             Returns entity
160ms     Service                Converts to DTO
165ms     Controller             Returns response
170ms     Network                Response in transit
200ms     React                  Updates state
210ms     Browser                UI refreshes
```

**Total round-trip: ~200ms** (varies with network/server load)

---

> **Continue to Part 3: Frontend Architecture & Flow →**
