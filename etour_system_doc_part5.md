# 📘 E-TOUR System Documentation

## PART 5: BACKEND ARCHITECTURE (LAYERED DESIGN)

---

# ☕ Why Spring Boot?

| Feature | Benefit for E-TOUR |
|---------|-------------------|
| **Auto-configuration** | Minimal boilerplate, fast setup |
| **Embedded server** | No separate Tomcat installation |
| **Spring Security** | JWT authentication out of the box |
| **Spring Data JPA** | Database operations without SQL |
| **Dependency Injection** | Clean, testable code |
| **Ecosystem** | Razorpay SDK, Mail, OAuth2 integration |
| **Production-ready** | Actuator for health checks |

---

# 🏗️ Layered Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              BACKEND LAYERS                                 │
└─────────────────────────────────────────────────────────────────────────────┘

    HTTP Request
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  CONTROLLER LAYER                               @RestController             │
│  ───────────────                                                            │
│  • Receives HTTP requests                                                   │
│  • Maps URLs to methods                                                     │
│  • Validates input format                                                   │
│  • Returns HTTP responses                                                   │
│  Files: AuthController, BookingController, PaymentController, etc.          │
└─────────────────────────────────────────────────────────────────────────────┘
         │ Calls
         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  SERVICE LAYER                                  @Service                    │
│  ─────────────                                                              │
│  • Contains business logic                                                  │
│  • Validates business rules                                                 │
│  • Orchestrates multiple repositories                                       │
│  • Transaction management                                                   │
│  Files: AuthService, BookingService, PaymentService, SearchService          │
└─────────────────────────────────────────────────────────────────────────────┘
         │ Calls
         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  REPOSITORY LAYER                               @Repository                 │
│  ────────────────                                                           │
│  • Database operations                                                      │
│  • CRUD methods                                                             │
│  • Custom queries                                                           │
│  • No business logic!                                                       │
│  Files: CustomerRepository, BookingRepository, PaymentRepository            │
└─────────────────────────────────────────────────────────────────────────────┘
         │ Uses
         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  ENTITY LAYER                                   @Entity                     │
│  ────────────                                                               │
│  • Java classes mapped to DB tables                                         │
│  • Defines relationships (FK)                                               │
│  • JPA annotations                                                          │
│  Files: CustomerMaster, BookingHeader, PaymentMaster, TourMaster            │
└─────────────────────────────────────────────────────────────────────────────┘
         │ Maps to
         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  DATABASE                                                                   │
│  ────────                                                                   │
│  • MySQL tables                                                             │
│  • customer_master, booking_header, payment_master, etc.                    │
└─────────────────────────────────────────────────────────────────────────────┘

         ┌─────────────────────────────────────────────────────────────────┐
         │  DTO LAYER (Data Transfer Objects)                              │
         │  ─────────                                                      │
         │  • Carries data between layers                                  │
         │  • Shapes API request/response                                  │
         │  • Hides internal entity structure                              │
         │  Files: CustomerDTO, BookingDTO, PaymentDTO, SearchResultDTO    │
         └─────────────────────────────────────────────────────────────────┘
```

---

# 📂 E-TOUR Backend File Structure

```
src/main/java/com/example/
├── config/                  # Configuration classes
│   ├── RazorpayConfig.java
│   ├── SecurityConfig.java
│   ├── CorsConfig.java
│   └── OAuth2SuccessHandler.java
├── controller/              # REST endpoints
│   ├── AuthController.java
│   ├── BookingController.java
│   ├── PaymentController.java
│   ├── PaymentGatewayController.java
│   ├── SearchController.java
│   ├── TourController.java
│   └── CategoryController.java
├── dto/                     # Data transfer objects
│   ├── CustomerDTO.java
│   ├── LoginDTO.java
│   ├── BookingDTO.java
│   ├── PaymentDTO.java
│   └── SearchResultDTO.java
├── entities/                # JPA entities
│   ├── CustomerMaster.java
│   ├── BookingHeader.java
│   ├── PaymentMaster.java
│   ├── TourMaster.java
│   └── Passenger.java
├── enums/                   # Enumerations
│   ├── PaymentStatus.java
│   ├── AuthProvider.java
│   └── CustomerRole.java
├── filter/                  # Security filters
│   └── JwtAuthFilter.java
├── mapper/                  # Entity ↔ DTO mappers
│   └── CustomerMapper.java
├── repositories/            # Database access
│   ├── CustomerRepository.java
│   ├── BookingRepository.java
│   └── PaymentRepository.java
├── services/                # Business logic interfaces
│   ├── AuthService.java
│   ├── BookingService.java
│   ├── PaymentService.java
│   └── impl/               # Interface implementations
│       ├── BookingServiceImpl.java
│       ├── PaymentServiceImpl.java
│       └── RazorpayServiceImpl.java
└── util/                    # Utilities
    └── JwtUtil.java
```

---

# 🎯 Responsibility of Each Layer

## 1. Controller Layer

**What it DOES:**
- Receive HTTP requests
- Extract path variables, request params, request body
- Call appropriate service method
- Return HTTP response with status code

**What it MUST NOT do:**
- ❌ Database queries
- ❌ Business logic
- ❌ Direct entity manipulation

```java
@RestController
@RequestMapping("/api/bookings")
public class BookingController {
    
    private final BookingService bookingService;
    
    @PostMapping
    public ResponseEntity<BookingDTO> createBooking(
            @RequestBody CreateBookingDTO dto) {
        // ONLY: receive request, call service, return response
        return ResponseEntity.ok(bookingService.createBooking(dto));
    }
}
```

---

## 2. Service Layer

**What it DOES:**
- Implement business rules
- Validate data against business logic
- Orchestrate multiple repositories
- Handle transactions
- Convert between DTOs and Entities

**What it MUST NOT do:**
- ❌ Directly handle HTTP concerns
- ❌ Know about request/response objects

```java
@Service
public class AuthService {
    
    @Autowired
    private CustomerRepository repository;
    
    @Autowired
    private PasswordEncoder passwordEncoder;
    
    @Autowired
    private JwtUtil jwtUtil;
    
    public String login(LoginDTO dto) {
        // 1. Find user
        CustomerMaster user = repository.findByEmail(dto.getEmail())
            .orElseThrow(() -> new RuntimeException("Invalid credentials"));
        
        // 2. Business rule: verify password
        if (!passwordEncoder.matches(dto.getPassword(), user.getPassword())) {
            throw new RuntimeException("Invalid credentials");
        }
        
        // 3. Business rule: check auth provider
        if (user.getAuthProvider() != AuthProvider.LOCAL) {
            throw new RuntimeException("Use " + user.getAuthProvider());
        }
        
        // 4. Generate token
        return jwtUtil.generateToken(user.getEmail(), user.getCustomerRole().name());
    }
}
```

---

## 3. Repository Layer

**What it DOES:**
- Define database operations
- Execute queries
- Return entities or projections

**What it MUST NOT do:**
- ❌ Business logic
- ❌ Validation
- ❌ HTTP handling

```java
@Repository
public interface PaymentRepository extends JpaRepository<PaymentMaster, Integer> {
    
    // JPA generates SQL automatically
    Optional<PaymentMaster> findByTransactionRef(String transactionRef);
    
    boolean existsByBooking_IdAndPaymentStatus(Integer bookingId, String status);
    
    @Query("SELECT p FROM PaymentMaster p WHERE p.booking.id = :bookingId")
    List<PaymentMaster> findByBookingId(@Param("bookingId") Integer bookingId);
}
```

---

## 4. Entity Layer

**What it DOES:**
- Map Java objects to database tables
- Define relationships between tables
- Configure column properties

**What it MUST NOT do:**
- ❌ Business logic
- ❌ Service calls

```java
@Entity
@Table(name = "payment_master")
public class PaymentMaster {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "booking_id")
    private BookingHeader booking;
    
    @Column(name = "payment_amount")
    private BigDecimal paymentAmount;
    
    @Column(name = "payment_status")
    private String paymentStatus;
    
    // Getters and setters only
}
```

---

## 5. DTO Layer

**What it DOES:**
- Shape data for API input/output
- Hide internal entity structure
- Allow API evolution without entity changes

```java
public class PaymentDTO {
    private Integer paymentId;
    private Integer bookingId;      // Not the full BookingHeader object
    private BigDecimal amount;
    private String status;
    private String transactionRef;
    private Instant paymentDate;
    
    // Getters and setters
}
```

---

# 🔄 Dependency Flow

```
Controller → Service → Repository → Entity → Database
    │           │           │
    │           │           └── Returns Entity
    │           └── Returns Entity (converts to DTO)
    └── Returns DTO (HTTP response)
```

## Rules:
1. **Controllers depend on Services** (never repositories directly)
2. **Services depend on Repositories** (and other services)
3. **Repositories depend on Entities** (JPA mapping)
4. **DTOs are used at boundaries** (API in/out)

---

# ⚠️ Why Business Logic Must NOT Be in Controllers

## Bad Example (Don't Do This):

```java
@PostMapping("/login")
public String login(@RequestBody LoginDTO dto) {
    // ❌ BAD: Business logic in controller
    CustomerMaster user = repository.findByEmail(dto.getEmail())
        .orElseThrow();
    
    if (!passwordEncoder.matches(dto.getPassword(), user.getPassword())) {
        throw new RuntimeException("Bad password");
    }
    
    return jwtUtil.generateToken(user.getEmail());
}
```

## Problems:
| Issue | Impact |
|-------|--------|
| **Untestable** | Must test via HTTP, can't unit test |
| **Not reusable** | Another endpoint needs same logic? Copy-paste |
| **Hard to read** | Controller mixed with business rules |
| **No transactions** | Multiple operations not atomic |

---

# ⚠️ Why Repositories Must NOT Contain Logic

## Bad Example (Don't Do This):

```java
@Repository
public interface PaymentRepository extends JpaRepository<PaymentMaster, Integer> {
    
    // ❌ BAD: This is a default method with business logic
    default PaymentMaster createPaymentIfNotExists(Integer bookingId) {
        if (!existsByBookingId(bookingId)) {
            PaymentMaster p = new PaymentMaster();
            p.setBookingId(bookingId);
            p.setStatus("INITIATED");
            return save(p);
        }
        return findByBookingId(bookingId).get(0);
    }
}
```

## Problems:
| Issue | Impact |
|-------|--------|
| **Wrong responsibility** | Repository = data access, not logic |
| **Hidden behavior** | Developers expect CRUD only |
| **Testing difficulty** | Can't mock partial behavior |

---

# 💥 What Breaks If Layers Are Mixed?

## Scenario: Logic in Controller

```
Developer A: Adds login logic in AuthController
Developer B: Needs same logic for OAuth callback
Result: Logic duplicated in OAuth2SuccessHandler
Later: Bug found in password validation
Problem: Must fix in multiple places
```

## Scenario: Repository Does Business Logic

```
Repository method handles "create if not exists"
Service A calls: expects creation
Service B calls: didn't know creation happens
Result: Unexpected side effects, data integrity issues
```

## Scenario: Entity Returned to Frontend

```
API returns CustomerMaster entity directly
Entity has: password hash, auth provider, internal IDs
Result: Security breach - sensitive data exposed
Later: Entity structure changes
Result: Frontend breaks (coupled to internal structure)
```

---

# ✅ Clean Architecture Benefits

| Benefit | How Layers Enable It |
|---------|---------------------|
| **Testability** | Mock services to test controllers |
| **Maintainability** | Change logic in one place |
| **Scalability** | Add more services without touching controllers |
| **Security** | DTOs filter what's exposed |
| **Flexibility** | Swap MySQL for PostgreSQL without changing services |

---

> **Continue to Part 6: Core Business Modules Overview →**
