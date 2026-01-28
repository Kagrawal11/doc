# 📘 E-TOUR System Documentation

## PART 3: FRONTEND ARCHITECTURE & FLOW

---

# ⚛️ Why React + Vite?

| Technology | Purpose | Why Chosen |
|------------|---------|------------|
| **React** | UI library | Component-based, reusable UI, large ecosystem |
| **Vite** | Build tool | Fast dev server, instant HMR, modern bundling |

## React Benefits for E-TOUR:

| Feature | How It Helps |
|---------|--------------|
| **Component reusability** | One TourCard component used everywhere |
| **Virtual DOM** | Fast UI updates without full page reload |
| **Hooks** | Clean state management (useState, useReducer) |
| **Context API** | Share auth/booking state across components |
| **Ecosystem** | react-router-dom, axios, react-toastify |

---

# 📁 Frontend Folder Architecture

```
Frontend_E_tour/src/
├── api/                    # API communication layer
│   └── index.js            # Axios instance + all API functions
├── assets/                 # Static files (images, icons)
├── components/             # Reusable UI components
│   ├── Bookings/           # Booking-specific components
│   ├── Forms/              # Form components (PassengerForm, PaymentForm)
│   ├── Layout/             # Navbar, Footer
│   └── UI/                 # Generic UI (Cards, Modals, Loaders)
├── context/                # Global state providers
│   ├── AuthContext.jsx     # Authentication state
│   └── BookingContext.jsx  # Booking wizard state
├── pages/                  # Full page components
│   ├── Home.jsx            # Landing page
│   ├── Tours.jsx           # Tour listing
│   ├── TourDetail.jsx      # Single tour details
│   ├── BookingStart.jsx    # Booking wizard
│   ├── CustomerProfile.jsx # User profile
│   ├── Login.jsx           # Login form
│   ├── Register.jsx        # Registration form
│   └── ...more pages
├── routes/                 # Routing configuration
│   └── AppRoutes.jsx       # All route definitions
├── App.jsx                 # Root component
├── main.jsx                # Entry point
└── index.css               # Global styles
```

## Why This Structure?

| Folder | Principle | Benefit |
|--------|-----------|---------|
| `api/` | Centralized API calls | Change base URL in one place |
| `components/` | Reusability | Build once, use everywhere |
| `context/` | Global state | Avoid prop drilling |
| `pages/` | Route targets | Each file = one URL |
| `routes/` | Navigation logic | Clear routing overview |

---

# 🧱 Component-Based Design

## Component Hierarchy:

```
App
├── AuthProvider (wraps everything)
│   ├── BookingProvider (wraps everything)
│   │   ├── Navbar
│   │   ├── AppRoutes
│   │   │   ├── Home
│   │   │   ├── Tours
│   │   │   ├── TourDetail
│   │   │   ├── BookingStart
│   │   │   │   ├── TourSelection
│   │   │   │   ├── DepartureSelection
│   │   │   │   ├── PassengerForm
│   │   │   │   └── PaymentForm
│   │   │   ├── CustomerProfile
│   │   │   └── ...
│   │   └── Footer
```

## Types of Components:

| Type | Example | Responsibility |
|------|---------|----------------|
| **Layout** | Navbar, Footer | Always visible, navigation |
| **Page** | Home, Tours | Corresponds to a route |
| **Feature** | BookingWizard | Complex business logic |
| **UI** | TourCard, Button | Reusable presentation |
| **Form** | PassengerForm | User input collection |

---

# 📄 Pages and Their Purpose

| Page | Route | Purpose |
|------|-------|---------|
| **Home** | `/` | Landing page, featured tours, search |
| **Tours** | `/tours` | Browse all tour categories |
| **TourDetail** | `/tours/details/:id` | Full tour info, itinerary, pricing |
| **BookingStart** | `/booking/start/:tourId` | Multi-step booking wizard |
| **Login** | `/login` | User authentication |
| **Register** | `/register` | New user signup |
| **ForgotPassword** | `/forgot-password` | Password recovery |
| **ResetPassword** | `/reset-password` | Set new password |
| **CustomerProfile** | `/customer/profile` | View/edit profile |
| **CustomerBookings** | `/customer/bookings` | Booking history |
| **OAuth2Success** | `/oauth2-success` | Google login callback |
| **Health** | `/health` | System health check |

---

# 🧭 Routing and Navigation Flow

## Route Definitions (AppRoutes.jsx):

```jsx
<Routes>
  <Route path="/" element={<Home />} />
  <Route path="/tours" element={<Tours />} />
  <Route path="/tours/details/:id" element={<TourDetail />} />
  <Route path="/booking/start/:tourId" element={<BookingStart />} />
  <Route path="/login" element={<Login />} />
  <Route path="/register" element={<Register />} />
  <Route path="/customer/profile" element={<CustomerProfile />} />
  <Route path="/customer/bookings" element={<CustomerBookings />} />
</Routes>
```

## Navigation Pattern:

```
Home → Tours → TourDetail → BookingStart → Payment → Success
  │                              │
  │                              └── Requires login
  │
  └── Search filters also lead to Tours
```

---

# 🔄 State Handling Strategy

## Three Types of State in E-TOUR:

| State Type | Where Stored | Example |
|------------|--------------|---------|
| **Local State** | `useState` in component | Form input values |
| **Route State** | URL parameters | `/tours/details/5` (id=5) |
| **Global State** | Context API | AuthContext, BookingContext |

## AuthContext (Authentication State):

```
┌─────────────────────────────────────────────────────────┐
│ AuthContext State:                                      │
│                                                         │
│ {                                                       │
│   user: null | { email, name },                         │
│   token: "eyJ..." | null,                               │
│   isAuthenticated: true | false,                        │
│   loading: boolean,                                     │
│   error: string | null                                  │
│ }                                                       │
│                                                         │
│ Methods:                                                │
│   login(credentials) → stores token in sessionStorage   │
│   register(data) → creates account                      │
│   loginWithOAuth2(token) → Google login                 │
│   logout() → clears token, resets state                 │
└─────────────────────────────────────────────────────────┘
```

## BookingContext (Booking Wizard State):

```
┌─────────────────────────────────────────────────────────┐
│ BookingContext State:                                   │
│                                                         │
│ {                                                       │
│   customerId: number,                                   │
│   selectedTour: { id, name, costs, ... },               │
│   selectedDeparture: { id, date, ... },                 │
│   passengers: [ { name, pax_type, ... }, ... ],         │
│   booking: { id, status, totalAmount },                 │
│   currentStep: 0 | 1 | 2 | 3 | 4,                       │
│   loading: boolean,                                     │
│   error: string | null                                  │
│ }                                                       │
│                                                         │
│ Methods:                                                │
│   setTour(tour) → moves to step 1                       │
│   setDeparture(departure) → moves to step 2             │
│   addPassenger(passenger) → adds to list                │
│   removePassenger(index) → removes from list            │
│   calculateTotal() → sums passenger costs               │
│   resetBooking() → clears all booking state             │
└─────────────────────────────────────────────────────────┘
```

---

# 📊 Page-to-Page Data Flow

## Booking Flow Data Movement:

```
┌────────────────┐     ┌────────────────┐     ┌────────────────┐
│   TourDetail   │     │  BookingStart  │     │  PaymentForm   │
│                │     │                │     │                │
│ Tour info from │────▶│ setTour()      │     │ Uses:          │
│ API response   │     │ saves to       │────▶│ - selectedTour │
│                │     │ BookingContext │     │ - passengers   │
└────────────────┘     │                │     │ - calculateTotal()
                       │ User adds      │     │                │
                       │ passengers     │     │ Creates booking│
                       │ via forms      │     │ via API        │
                       └────────────────┘     └────────────────┘
```

## Why This Works:

1. **BookingContext wraps entire app** → all pages can access booking state
2. **Reducer pattern** → predictable state updates via actions
3. **No prop drilling** → any component can read/write via useBooking()

---

# 🔄 What Happens on Page Refresh?

## Problem: React state is memory-only

```
User on BookingStart → selects tour, departure, passengers
User refreshes page
Result: ALL STATE IS LOST! 😱
```

## E-TOUR's Handling Strategy:

| State Type | Persistence | Refresh Behavior |
|------------|-------------|------------------|
| **Auth token** | sessionStorage | ✅ Survives refresh |
| **Booking state** | Memory only | ❌ Lost on refresh |
| **Route params** | URL | ✅ Survives refresh |

## AuthContext Refresh Recovery:

```javascript
// On app load:
const tokenFromStorage = sessionStorage.getItem('token');
const tokenValid = tokenFromStorage && !isTokenExpired(tokenFromStorage);

// If valid token exists, user stays logged in
initialState = {
  token: tokenValid ? tokenFromStorage : null,
  isAuthenticated: tokenValid  // ✅ Auto-login on refresh
}
```

## BookingContext Limitation:

```
Current behavior:
- User starts booking → adds passengers → refreshes
- LOST: selectedTour, passengers, currentStep
- User must restart booking

Possible improvement:
- Store booking state in localStorage
- Restore on page load
- Ask "Continue previous booking?"
```

---

# 🎯 Frontend vs Backend Responsibility

| Aspect | Frontend | Backend |
|--------|----------|---------|
| **UI rendering** | ✅ Full responsibility | ❌ None |
| **User input validation** | ✅ Quick feedback | ✅ Final validation |
| **Business logic** | ❌ Never | ✅ Full responsibility |
| **Data persistence** | ❌ Never | ✅ Full responsibility |
| **Authentication check** | ✅ UI-level (hide buttons) | ✅ Actual enforcement |
| **API calls** | ✅ Initiates | ✅ Processes |
| **Error display** | ✅ Shows to user | ✅ Returns error codes |

## Golden Rule:
> **Never trust the frontend.** Backend must validate everything because frontend code can be manipulated by users.

---

> **Continue to Part 4: Frontend → Backend Communication →**
