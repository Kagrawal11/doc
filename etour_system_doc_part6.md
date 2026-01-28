# 📘 E-TOUR System Documentation

## PART 6: CORE BUSINESS MODULES OVERVIEW

---

# 🧩 Module Interconnection Map

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     E-TOUR BUSINESS MODULES                                 │
└─────────────────────────────────────────────────────────────────────────────┘

                              ┌────────────────┐
                              │  CUSTOMER/AUTH │
                              │    MODULE      │
                              └───────┬────────┘
                                      │
                                      │ provides identity
                                      ▼
┌──────────────────┐           ┌────────────────┐           ┌────────────────┐
│  SEARCH MODULE   │──────────▶│ TOURS/CATEGORY │◀──────────│ DEPARTURE      │
│                  │  filters   │    MODULE      │  has dates│   MODULE       │
└──────────────────┘           └───────┬────────┘           └────────────────┘
                                       │
                                       │ user selects
                                       ▼
                              ┌────────────────┐
                              │   BOOKING      │
                              │    MODULE      │
                              └───────┬────────┘
                                      │
                         ┌────────────┴───────────┐
                         │                        │
                         ▼                        ▼
                ┌────────────────┐       ┌────────────────┐
                │  PASSENGER     │       │   PAYMENT      │
                │    MODULE      │       │    MODULE      │
                └────────────────┘       └────────────────┘
```

---

# 1️⃣ AUTHENTICATION & USER MANAGEMENT MODULE

## Purpose
Handles user identity: who is accessing the system and what they can do.

## Components
| Component | File |
|-----------|------|
| Controller | `AuthController.java` |
| Service | `AuthService.java` |
| Repository | `CustomerRepository.java` |
| Entity | `CustomerMaster.java` |
| Utility | `JwtUtil.java`, `JwtAuthFilter.java` |

## Supported Operations

| Operation | Input | Output | Description |
|-----------|-------|--------|-------------|
| **Register** | `CustomerDTO` (name, email, password) | `CustomerModel` | Create new account |
| **Login** | `LoginDTO` (email, password) | `JWT token` | Verify credentials |
| **OAuth2 Login** | Google OAuth callback | `JWT token` | Social login |
| **Get Profile** | JWT token | `CustomerDTO` | Fetch user data |
| **Update Profile** | `CustomerDTO` | Updated `CustomerDTO` | Modify user data |
| **Forgot Password** | Email | Email sent | Password reset flow |
| **Reset Password** | Token + new password | Success | Set new password |

## Connects To
- **All modules**: Every authenticated request passes through JWT filter
- **Booking module**: `customerId` required to create booking
- **Email module**: Sends password reset emails

---

# 2️⃣ TOURS & CATEGORIES MODULE

## Purpose
Manages the tour catalog — what tours are available and their details.

## Components
| Component | File |
|-----------|------|
| Controller | `TourController.java`, `CategoryController.java` |
| Service | `TourService.java`, `CategoryService.java` |
| Repository | `TourRepository.java`, `CategoryRepository.java` |
| Entity | `TourMaster.java`, `CategoryMaster.java` |

## Data Model Relationships

```
CategoryMaster (Tour Category)
    │
    │ one-to-many
    ▼
TourMaster (Specific Tour Package)
    │
    ├── CostMaster (Pricing for different passenger types)
    ├── DepartureMaster (Available departure dates)
    ├── ItineraryMaster (Day-wise itinerary)
    └── TourGuide (Assigned guides)
```

## Supported Operations

| Operation | Input | Output |
|-----------|-------|--------|
| **Get All Tours** | None | List of `CategoryMaster` with nested data |
| **Get Tour by ID** | `categoryId` | Full tour details with costs, departures |
| **Get Tour Details** | `categoryId` | Complete tour package info |
| **Get Tour ID** | `categoryId`, `departureId` | Specific `tourId` |

## Connects To
- **Search module**: Search returns filtered tour results
- **Booking module**: Booking references `tourId`
- **Frontend**: Tour browsing and selection UI

---

# 3️⃣ BOOKING SYSTEM MODULE

## Purpose
Core transaction module — creates and manages tour reservations.

## Components
| Component | File |
|-----------|------|
| Controller | `BookingController.java` |
| Service | `BookingService.java` |
| Repository | `BookingRepository.java` |
| Entity | `BookingHeader.java` |

## Booking Lifecycle

```
PENDING → CONFIRMED → (optional) CANCELLED
   │          │
   │          └── After successful payment
   │
   └── Initial state after creation
```

## Supported Operations

| Operation | Input | Output |
|-----------|-------|--------|
| **Create Booking** | `customerId`, `tourId`, `departureId`, passengers | `BookingHeader` |
| **Get Booking** | `bookingId` | `BookingDTO` |
| **Get by Customer** | `customerId` | List of `BookingDTO` |
| **Update Status** | `bookingId`, `statusId` | Updated booking |

## Booking Data Structure

```
BookingHeader
├── booking_id (PK)
├── customer_id (FK → CustomerMaster)
├── tour_id (FK → TourMaster)
├── departure_id (FK → DepartureMaster)
├── number_of_pax (total passengers)
├── total_amount (calculated)
├── status_id (FK → BookingStatusMaster)
├── booking_date
└── Passengers[] (one-to-many)
```

## Connects To
- **Customer module**: Requires authenticated customer
- **Tour module**: References specific tour and departure
- **Passenger module**: Contains passenger list
- **Payment module**: Payment linked to booking

---

# 4️⃣ PASSENGER MANAGEMENT MODULE

## Purpose
Manages traveler information for each booking.

## Components
| Component | File |
|-----------|------|
| Controller | `PassengerController.java` |
| Service | `PassengerService.java` |
| Repository | `PassengerRepository.java` |
| Entity | `Passenger.java` |

## Passenger Types & Pricing

| Type | Description | Pricing Field |
|------|-------------|---------------|
| `adult` | 12+ years | `singlePersonCost` or `extraPersonCost` |
| `child_with_bed` | 2-12 years, needs bed | `childWithBedCost` |
| `child_without_bed` | 2-12 years, no bed | `childWithoutBedCost` |

## Passenger Data Structure

```
Passenger
├── passenger_id (PK)
├── booking_id (FK → BookingHeader)
├── first_name
├── last_name
├── age
├── pax_type (adult, child_with_bed, child_without_bed)
├── id_proof_type
└── id_proof_number
```

## Connects To
- **Booking module**: Passengers belong to a booking
- **Pricing calculation**: Total = Σ(passenger costs based on type)
- **Invoice generation**: Lists all passengers

---

# 5️⃣ SEARCH & FILTERING MODULE

## Purpose
Enables users to find tours matching their criteria.

## Components
| Component | File |
|-----------|------|
| Controller | `SearchController.java` |
| Service | `SearchService.java`, `SearchServiceImpl.java` |
| Repository | `SearchRepository.java` |
| DTO | `SearchResultDTO.java` |

## Search Criteria Supported

| Search Type | Input Parameters | Backend Query |
|-------------|------------------|---------------|
| **By Duration** | `minDays`, `maxDays` | Filters by `no_of_days` |
| **By Cost** | `minCost`, `maxCost` | Filters by `single_person_cost` |
| **By Location** | `keyword` | Matches tour name/destination |
| **By Date** | `fromDate`, `toDate` | Filters departure dates |

## Search Flow

```
User enters filters → Frontend calls searchAPI
                           │
                           ▼
SearchController receives request
                           │
                           ▼
SearchService.searchByXxx()
                           │
                           ▼
SearchRepository.searchByXxx() - Native SQL query
                           │
                           ▼
Returns List<SearchResultDTO>
                           │
                           ▼
Frontend displays filtered tours
```

## Connects To
- **Tour module**: Results are tour references
- **Frontend**: Search form on Home/Tours page
- **Category display**: Search results lead to tour details

---

# 6️⃣ PAYMENT PROCESSING MODULE

## Purpose
Handles financial transactions for tour bookings.

## Components
| Component | File |
|-----------|------|
| Controller | `PaymentController.java`, `PaymentGatewayController.java` |
| Service | `PaymentService.java`, `PaymentGatewayService.java` |
| Implementation | `PaymentServiceImpl.java`, `RazorpayServiceImpl.java` |
| Repository | `PaymentRepository.java` |
| Entity | `PaymentMaster.java` |
| Config | `RazorpayConfig.java` |
| DTO | `CreateOrderRequestDTO.java`, `CreateOrderResponseDTO.java`, `PaymentDTO.java` |

## Payment Lifecycle

```
No Payment → INITIATED → SUCCESS
                │
                └── Or FAILED if payment fails
```

## Payment Flow Summary

```
1. User clicks "Pay Now"
   │
2. Frontend calls POST /payment-gateway/create-order
   │
3. Backend creates Razorpay order, saves INITIATED payment
   │
4. Frontend opens Razorpay checkout popup
   │
5. User completes payment on Razorpay
   │
6. Frontend calls POST /payment-gateway/confirm-payment
   │
7. Backend updates payment to SUCCESS, confirms booking
```

## Key Operations

| Operation | Purpose |
|-----------|---------|
| **Create Order** | Generate Razorpay order, initiate payment record |
| **Confirm Payment** | Verify and mark payment SUCCESS, confirm booking |
| **Handle Webhook** | Process Razorpay server callbacks |
| **Get Payment** | Retrieve payment details |
| **Get Receipt** | Fetch payment for invoice/receipt |

## Connects To
- **Booking module**: Payment linked to booking, updates booking status
- **Razorpay**: External payment gateway
- **Invoice module**: Generates receipt after payment

---

# 🔗 Module Interaction Summary

| From Module | To Module | Interaction |
|-------------|-----------|-------------|
| **Auth** | All | JWT token validates every request |
| **Search** | Tours | Returns filtered tour list |
| **Tours** | Booking | Provides tour/departure for booking |
| **Booking** | Customer | Associates booking with customer |
| **Booking** | Passenger | Contains passenger records |
| **Booking** | Payment | Payment references booking |
| **Payment** | Booking | Confirms booking on success |
| **Payment** | Razorpay | External API calls |

---

# 🎯 Key Design Decisions Across Modules

| Decision | Rationale |
|----------|-----------|
| **Separate TourController & CategoryController** | Tours vs Categories are distinct resources |
| **PaymentController vs PaymentGatewayController** | Generic payment ops vs Razorpay-specific |
| **Interface + Impl pattern in services** | Enables swapping implementations (e.g., Stripe) |
| **DTOs at all boundaries** | Security, decoupling, API stability |
| **Status tables (BookingStatusMaster)** | Extensible status management |

---

> **Continue to Part 7: Data Flow & Database Design →**
