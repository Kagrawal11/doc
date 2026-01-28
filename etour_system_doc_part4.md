# 📘 E-TOUR System Documentation

## PART 4: FRONTEND → BACKEND COMMUNICATION

---

# 📡 API Calling Strategy

E-TOUR uses **Axios** as the HTTP client for all API calls.

## Why Axios Over Fetch?

| Feature | Axios | Native Fetch |
|---------|-------|--------------|
| **Interceptors** | ✅ Built-in | ❌ Manual |
| **Auto JSON parsing** | ✅ Automatic | ❌ Manual `.json()` |
| **Timeout support** | ✅ Built-in | ❌ AbortController |
| **Error handling** | ✅ Cleaner | ❌ Verbose |
| **Request cancellation** | ✅ Easy | ❌ Complex |

---

# 🔧 Centralized API Configuration

All API calls go through a single configured Axios instance:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ src/api/index.js                                                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  const API_BASE_URL = import.meta.env.VITE_API_URL || 'http://localhost:8080'│
│                                                                             │
│  const api = axios.create({                                                 │
│    baseURL: API_BASE_URL,                                                   │
│    timeout: 10000,  // 10 seconds                                           │
│    headers: { 'Content-Type': 'application/json' }                          │
│  });                                                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Benefits of Centralization:

| Benefit | Explanation |
|---------|-------------|
| **Single config point** | Change base URL once, affects all calls |
| **Consistent headers** | All requests have same content type |
| **Unified timeout** | No request hangs forever |
| **One place for interceptors** | Auth logic in one file |

---

# 🔐 Request Interceptor (Auto Auth)

Every outgoing request automatically includes the JWT token:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ Request Interceptor                                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  api.interceptors.request.use((config) => {                                 │
│    const token = sessionStorage.getItem('token');                           │
│    if (token) {                                                             │
│      config.headers.Authorization = `Bearer ${token}`;                      │
│    }                                                                        │
│    return config;                                                           │
│  });                                                                        │
│                                                                             │
│  RESULT: Every API call automatically has:                                  │
│  Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Why This Matters:

Components don't need to manually add tokens:
```javascript
// WITHOUT interceptor (bad):
axios.get('/api/profile', {
  headers: { Authorization: `Bearer ${token}` }
});

// WITH interceptor (good):
api.get('/api/profile');  // Token auto-added!
```

---

# 🔄 Response Interceptor (Auto Error Handling)

All responses pass through error handling:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ Response Interceptor                                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  api.interceptors.response.use(                                             │
│    (response) => response,  // Success: pass through                        │
│    (error) => {                                                             │
│      if (error.response?.status === 401) {                                  │
│        sessionStorage.removeItem('token');                                  │
│        window.location.href = '/login';  // ← Force re-login                │
│      }                                                                      │
│      return Promise.reject(error);                                          │
│    }                                                                        │
│  );                                                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Automatic 401 Handling:

1. Any API returns 401 (Unauthorized)
2. Token is cleared from sessionStorage
3. User is redirected to login page
4. No component needs to handle this manually

---

# 📦 API Function Organization

APIs are organized by domain:

```javascript
// TOUR APIs
export const tourAPI = {
  getTours: () => api.get('/api/tours'),
  getTour: (id) => api.get(`/api/tours/${id}`),
  getTourDetails: (catId) => api.get(`/api/tours/details/${catId}`)
};

// BOOKING APIs
export const bookingAPI = {
  createBooking: (data) => api.post('/api/bookings', data),
  getBooking: (id) => api.get(`/api/bookings/${id}`),
  getBookingsByCustomer: (customerId) => 
    api.get(`/api/bookings/customer/${customerId}`),
  createOrder: (data) => api.post('/payment-gateway/create-order', data),
  verifyPayment: (params) => api.post('/payment-gateway/confirm-payment', null, { params })
};

// CUSTOMER APIs
export const customerAPI = {
  register: (data) => api.post('/auth/register', data),
  login: (credentials) => api.post('/auth/login', credentials),
  getProfile: () => api.get('/api/customer/profile'),
  updateProfile: (data) => api.put('/api/customer/profile', data)
};

// SEARCH APIs
export const searchAPI = {
  searchByDuration: (minDays, maxDays) => 
    api.get('/search/by-duration', { params: { minDays, maxDays } }),
  searchByCost: (minCost, maxCost) => 
    api.get('/search/by-cost', { params: { minCost, maxCost } }),
  searchByLocation: (keyword) => 
    api.get('/search/by-location', { params: { keyword } })
};
```

## Why This Organization?

| Benefit | Explanation |
|---------|-------------|
| **Discoverability** | `tourAPI.getTours()` is self-documenting |
| **Refactoring safety** | Change endpoint in one place |
| **Type hints** | IDE shows available methods |
| **Testing** | Easy to mock entire API groups |

---

# 🔄 Request-Response Lifecycle

## Complete Flow for `customerAPI.login()`:

```
STEP 1: Component calls API
─────────────────────────────────────────────────────
const result = await customerAPI.login({ 
  email: 'user@example.com', 
  password: 'secret123' 
});

STEP 2: Axios creates request
─────────────────────────────────────────────────────
Method: POST
URL: http://localhost:8080/auth/login
Headers:
  Content-Type: application/json
Body: { "email": "user@example.com", "password": "secret123" }

STEP 3: Request interceptor runs
─────────────────────────────────────────────────────
Checks sessionStorage for token
No token (user not logged in yet) → no Authorization header

STEP 4: Request sent over network
─────────────────────────────────────────────────────
HTTP POST to http://localhost:8080/auth/login

STEP 5: Backend processes (see Part 5)
─────────────────────────────────────────────────────
AuthController → AuthService → CustomerRepository
Returns: { token: "eyJ...", user: { email, name } }

STEP 6: Response received
─────────────────────────────────────────────────────
Status: 200 OK
Body: { "token": "eyJhbGc...", "user": { ... } }

STEP 7: Response interceptor runs
─────────────────────────────────────────────────────
Status is 200 (not 401) → passes through unchanged

STEP 8: Component receives response
─────────────────────────────────────────────────────
const { token, user } = result.data;
sessionStorage.setItem('token', token);
dispatch({ type: 'LOGIN_SUCCESS', payload: { user, token } });
```

---

# 🎯 How Frontend Decides Which API to Call

## Decision Tree:

```
User Action                    API Called
───────────                    ──────────
Browse tours                → tourAPI.getTours()
Click on tour               → tourAPI.getTourDetails(id)
Search by duration          → searchAPI.searchByDuration(min, max)
Search by location          → searchAPI.searchByLocation(keyword)
Login                       → customerAPI.login(credentials)
Register                    → customerAPI.register(data)
Get profile                 → customerAPI.getProfile()
Start booking               → bookingAPI.getTourId(categoryId, departureId)
Create booking              → bookingAPI.createBooking(data)
Initiate payment            → bookingAPI.createOrder({ bookingId })
Confirm payment             → bookingAPI.verifyPayment(orderId, paymentId, amount)
View my bookings            → bookingAPI.getBookingsByCustomer(customerId)
```

---

# ⚠️ Error Handling Strategy

## Error Types and Handling:

| Error Type | HTTP Code | Frontend Handling |
|------------|-----------|-------------------|
| **Validation error** | 400 | Show field-specific errors |
| **Unauthorized** | 401 | Redirect to login (interceptor) |
| **Forbidden** | 403 | Show "Access denied" message |
| **Not found** | 404 | Show "Not found" page |
| **Server error** | 500 | Show "Something went wrong" |
| **Network error** | N/A | Show "Check connection" |
| **Timeout** | N/A | Show "Request timed out" |

## Example Error Handling in Component:

```javascript
const handleLogin = async () => {
  try {
    const result = await customerAPI.login(credentials);
    // Success path
    sessionStorage.setItem('token', result.data.token);
    navigate('/');
  } catch (error) {
    if (error.response) {
      // Server responded with error
      if (error.response.status === 400) {
        setError('Invalid email or password');
      } else if (error.response.status === 500) {
        setError('Server error. Please try again later.');
      }
    } else if (error.request) {
      // No response received
      setError('Network error. Check your connection.');
    } else {
      // Request setup error
      setError('Something went wrong.');
    }
  }
};
```

---

# ⏱️ Timeout and Server Failure Scenarios

## Timeout (10 seconds):

```
User Action: Click "Book Now"
↓
API call starts → 10 seconds pass → no response
↓
Axios throws timeout error
↓
Frontend shows: "Request timed out. Please try again."
↓
User can retry
```

## Server Down:

```
User Action: Click "View Tours"
↓
API call fails → ECONNREFUSED (connection refused)
↓
error.request exists, but no error.response
↓
Frontend shows: "Unable to connect to server."
```

## Invalid Response:

```
Server returns: HTML error page instead of JSON
↓
Axios JSON parsing fails
↓
Frontend catches error
↓
Shows: "Unexpected response from server."
```

---

# 🔄 Summary: Communication Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        FRONTEND TO BACKEND FLOW                             │
└─────────────────────────────────────────────────────────────────────────────┘

  Component        API Layer       Network         Backend         Database
     │                │               │               │               │
     │ userAction()   │               │               │               │
     ├───────────────▶│               │               │               │
     │                │ axios.post()  │               │               │
     │                ├──────────────▶│               │               │
     │                │ interceptor   │               │               │
     │                │ adds token    │               │               │
     │                │               │ HTTP Request  │               │
     │                │               ├──────────────▶│               │
     │                │               │               │ Controller    │
     │                │               │               ├───────────────┤
     │                │               │               │ Service       │
     │                │               │               ├───────────────┤
     │                │               │               │ Repository    │
     │                │               │               ├──────────────▶│
     │                │               │               │◀──────────────┤
     │                │               │◀──────────────┤               │
     │                │               │ HTTP Response │               │
     │                │◀──────────────┤               │               │
     │                │ interceptor   │               │               │
     │                │ checks 401    │               │               │
     │◀───────────────┤               │               │               │
     │ update state   │               │               │               │
     │ update UI      │               │               │               │
```

---

> **Continue to Part 5: Backend Architecture (Layered Design) →**
