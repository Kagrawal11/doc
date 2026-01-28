# 📘 E-TOUR System Documentation

## PART 8: SECURITY & AUTHENTICATION FLOW

---

# 🔐 Why Authentication is Required

| Without Auth | With Auth |
|--------------|-----------|
| Anyone can book as any customer | Only logged-in user can book |
| Customer data exposed | Profile protected |
| No booking ownership | "My Bookings" works |
| Fake payments possible | Payment linked to verified user |

---

# 🎫 Why JWT (JSON Web Token)?

## JWT vs Session-Based Auth:

| Aspect | Session-Based | JWT (E-TOUR) |
|--------|--------------|--------------|
| **Storage** | Server-side session store | Token stored in client |
| **Scalability** | Sticky sessions needed | Any server can validate |
| **Stateless** | ❌ No | ✅ Yes |
| **Mobile-friendly** | Complex | Simple |
| **Microservices** | Session sharing complex | Token passed everywhere |

## JWT Structure:

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyQGV4YW1wbGUuY29tIiwicm9sZSI6IkNVU1RPTUVSIiwiZXhwIjoxNzA0MDY3MjAwfQ.abc123signature

HEADER.PAYLOAD.SIGNATURE
   │       │        │
   │       │        └─ Verified by server's secret key
   │       └─ Contains: email, role, expiry
   └─ Algorithm: HS256
```

---

# 🔄 Complete Login Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           LOGIN FLOW                                        │
└─────────────────────────────────────────────────────────────────────────────┘

STEP 1: User enters credentials
─────────────────────────────────────────────────────────────────────────────
Frontend Login.jsx:
  email: "user@example.com"
  password: "secret123"

STEP 2: Frontend calls login API
─────────────────────────────────────────────────────────────────────────────
customerAPI.login({ email, password })
→ POST http://localhost:8080/auth/login

STEP 3: AuthController receives request
─────────────────────────────────────────────────────────────────────────────
@PostMapping("/login")
public ResponseEntity<?> login(@RequestBody LoginDTO dto) {
    String token = authService.login(dto);
    return ResponseEntity.ok(Map.of("token", token));
}

STEP 4: AuthService validates credentials
─────────────────────────────────────────────────────────────────────────────
1. Find user by email
   CustomerMaster user = repository.findByEmail(email);

2. Check auth provider (must be LOCAL, not GOOGLE)
   if (user.getAuthProvider() != AuthProvider.LOCAL) {
       throw new RuntimeException("Use Google login");
   }

3. Verify password
   if (!passwordEncoder.matches(dto.getPassword(), user.getPassword())) {
       throw new RuntimeException("Invalid credentials");
   }

4. Generate JWT token
   return jwtUtil.generateToken(email, role);

STEP 5: JWT token created
─────────────────────────────────────────────────────────────────────────────
JwtUtil.generateToken():
  Claims:
    sub: "user@example.com"
    role: "CUSTOMER"
    exp: current_time + 24_hours
  
  Signed with: SECRET_KEY (from application.properties)
  
  Result: "eyJhbGciOiJIUzI1NiJ9.eyJzdWI..."

STEP 6: Token returned to frontend
─────────────────────────────────────────────────────────────────────────────
Response: { "token": "eyJhbGciOiJIUzI1NiJ9..." }

STEP 7: Frontend stores token
─────────────────────────────────────────────────────────────────────────────
AuthContext.login():
  sessionStorage.setItem('token', token);
  dispatch({ type: 'LOGIN_SUCCESS', payload: { token } });

STEP 8: User is now authenticated
─────────────────────────────────────────────────────────────────────────────
isAuthenticated: true
All future requests include: Authorization: Bearer <token>
```

---

# 🔒 Stateless Authentication Concept

## How Server "Remembers" User Without Sessions:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ EVERY AUTHENTICATED REQUEST:                                                │
│                                                                             │
│ Frontend:                                                                   │
│   axios.get('/api/customer/profile', {                                      │
│     headers: { Authorization: 'Bearer eyJhbGc...' }                         │
│   })                                                                        │
│                                                                             │
│ Server:                                                                     │
│   1. Extracts token from header                                             │
│   2. Verifies signature (proves token wasn't tampered)                      │
│   3. Checks expiry timestamp                                                │
│   4. Extracts email and role from payload                                   │
│   5. Sets SecurityContext with this user                                    │
│   6. Proceeds with request                                                  │
│                                                                             │
│ NO DATABASE LOOKUP for session!                                             │
│ Token itself proves identity.                                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

# 🛡️ JwtAuthFilter: The Security Gatekeeper

## How Every Request is Validated:

```java
@Component
public class JwtAuthFilter extends OncePerRequestFilter {

    @Autowired
    private JwtUtil jwtUtil;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) {
        
        // 1. Get Authorization header
        String header = request.getHeader("Authorization");
        
        // 2. Check if Bearer token exists
        if (header != null && header.startsWith("Bearer ")) {
            String token = header.substring(7);
            
            // 3. Extract claims from token
            String username = jwtUtil.extractUsername(token);
            String role = jwtUtil.extractRole(token);
            
            // 4. Create authentication object
            SimpleGrantedAuthority authority = 
                new SimpleGrantedAuthority("ROLE_" + role);
            
            UsernamePasswordAuthenticationToken auth = 
                new UsernamePasswordAuthenticationToken(
                    username, null, List.of(authority)
                );
            
            // 5. Set in security context
            SecurityContextHolder.getContext().setAuthentication(auth);
        }
        
        // 6. Continue filter chain
        chain.doFilter(request, response);
    }
}
```

## Filter Chain Position:

```
HTTP Request
     │
     ▼
┌─────────────────┐
│  CORS Filter    │  ← Handles cross-origin
└────────┬────────┘
         ▼
┌─────────────────┐
│  JwtAuthFilter  │  ← Validates token, sets auth
└────────┬────────┘
         ▼
┌─────────────────┐
│ SecurityFilter  │  ← Checks if route requires auth
└────────┬────────┘
         ▼
┌─────────────────┐
│   Controller    │  ← Request reaches endpoint
└─────────────────┘
```

---

# ⚙️ SecurityConfig: Route Protection

```java
@Configuration
public class SecurityConfig {

    @Autowired
    private JwtAuthFilter jwtFilter;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) {
        
        http.cors(Customizer.withDefaults())
            .csrf(csrf -> csrf.disable())
            .formLogin(form -> form.disable())
            .httpBasic(basic -> basic.disable())
            
            .authorizeHttpRequests(auth -> auth
                // Public endpoints - no auth required
                .requestMatchers("/auth/**").permitAll()
                
                // All other endpoints (currently permitAll for dev)
                .anyRequest().permitAll()
                // Production: .anyRequest().authenticated()
            )
            
            // OAuth2 login support
            .oauth2Login(oauth -> oauth
                .successHandler(oAuth2SuccessHandler)
            )
            
            // Add JWT filter before standard auth filter
            .addFilterBefore(jwtFilter, 
                UsernamePasswordAuthenticationFilter.class)
            
            // Stateless - no sessions
            .sessionManagement(sess -> 
                sess.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            );
        
        return http.build();
    }
}
```

---

# 💻 Frontend Token Handling

## Token Storage:

```javascript
// On login success:
sessionStorage.setItem('token', token);

// Why sessionStorage not localStorage?
// - sessionStorage: cleared when browser tab closes
// - localStorage: persists indefinitely
// E-TOUR uses sessionStorage for security (auto-logout on tab close)
```

## Auto Token Injection (Interceptor):

```javascript
api.interceptors.request.use((config) => {
    const token = sessionStorage.getItem('token');
    if (token) {
        config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
});
```

## Token Expiry Check (On App Load):

```javascript
const isTokenExpired = (token) => {
    const decoded = decodeJWT(token);
    if (!decoded || !decoded.exp) return true;
    return decoded.exp * 1000 < Date.now();
};

// Clear expired tokens
if (tokenFromStorage && !isTokenExpired(tokenFromStorage)) {
    // Token valid, user stays logged in
} else {
    sessionStorage.removeItem('token');
}
```

## Auto-Logout Timer:

```javascript
useEffect(() => {
    if (!state.token) return;
    
    const decoded = decodeJWT(state.token);
    const expiresInMs = decoded.exp * 1000 - Date.now();
    
    // Auto logout when token expires
    const timerId = setTimeout(() => {
        logout();
    }, expiresInMs);
    
    return () => clearTimeout(timerId);
}, [state.token]);
```

---

# ⏰ Token Expiry Scenarios

## Scenario 1: Token Expires During Session

```
User logged in at 10:00 AM
Token expires at 10:00 PM (12 hours)
User still browsing at 10:30 PM

RESULT:
1. Next API call fails with 401
2. Response interceptor catches 401
3. Token removed from sessionStorage
4. User redirected to /login
5. User sees "Session expired" message
```

## Scenario 2: User Returns After Token Expired

```
User logged in yesterday
Token expired overnight
User opens app today

RESULT:
1. App loads, checks sessionStorage
2. Token exists but isTokenExpired() returns true
3. Token cleared immediately
4. User sees login form
```

## Scenario 3: Token Tampered

```
Attacker modifies JWT payload
Changes role: "CUSTOMER" → "ADMIN"

RESULT:
1. JwtAuthFilter tries to validate
2. Signature verification FAILS
3. Token rejected
4. Request returns 401/403
```

---

> **Continue to Part 9: Payment & Transaction Handling →**
