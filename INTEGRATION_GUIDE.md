# Buffalo Hotel — Full Project Integration Guide

> This document is written for the **backend developer (Gemini)** connecting the Buffalo Hotel frontend projects to the StayFlow RMS backend API.
> Read this fully before touching any code.

---

## 1. Project Overview

The Buffalo Hotel client system consists of **three separate projects** that work together:

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT ECOSYSTEM                         │
│                                                                  │
│  ┌──────────────────┐    ┌──────────────────┐                   │
│  │  buffalo-hotel/  │    │  buffalo-guest/  │                   │
│  │  (Marketing      │    │  (Guest          │                   │
│  │   Website)       │    │   Dashboard)     │                   │
│  │                  │    │                  │                   │
│  │ buffalohotel.    │    │ guest.buffalo    │                   │
│  │ co.tz            │    │ hotel.co.tz      │                   │
│  └────────┬─────────┘    └────────┬─────────┘                   │
│           │                       │                             │
│           └───────────┬───────────┘                             │
│                       │ REST API (JSON)                         │
│                       ▼                                         │
│            ┌──────────────────────┐                             │
│            │     StayFlow RMS     │                             │
│            │  (Backend — Gemini)  │                             │
│            │  localhost:5000      │                             │
│            └──────────────────────┘                             │
└─────────────────────────────────────────────────────────────────┘
```

### Project 1 — `buffalo-hotel/` (Marketing Website)
- **URL:** `buffalohotel.co.tz`
- **Type:** Static HTML/CSS/JS — no framework, no build step
- **Purpose:** Public-facing marketing site. Handles initial room bookings.
- **Already connected:** `POST /api/bookings/website` and `POST /api/payments/initiate` — done in previous session.
- **Remaining:** Nothing new needed here unless noted below.

### Project 2 — `buffalo-guest/` (Guest Dashboard)
- **URL:** `guest.buffalohotel.co.tz`
- **Type:** Static HTML/CSS/JS — Single Page App (SPA)
- **Purpose:** Private dashboard for guests who have an active booking. Lets them order food, request services, and extend their stay.
- **Status:** Frontend 100% complete. **All API calls are stubbed with demo fallbacks.** Your job is to build the backend endpoints so the real data flows in.

### Project 3 — `StayFlow/` (RMS Backend)
- **Your codebase.** The API server running on port 5000.
- All new endpoints for the guest dashboard go here.

---

## 2. File Structure

### `buffalo-hotel/` — Marketing Website

```
buffalo-hotel/
├── index.html              ← Homepage (booking bar + featured rooms)
├── rooms.html              ← All 4 room types with filter
├── services.html           ← Hotel services + FAQ
├── about.html              ← About page + testimonials
├── contact.html            ← BOOKING FORM (main API entry point)
├── README.md               ← Website-specific readme (read this too)
└── assets/
    ├── css/style.css       ← All styles (1 file, ~1050 lines)
    ├── js/main.js          ← All JS (1 file, ~120 lines)
    └── images/             ← 22 image files
```

**API calls in `assets/js/main.js`:**

| What | Where in code | Status |
|------|--------------|--------|
| `POST /api/bookings/website` | `contactForm` submit handler | ✅ Connected (done) |
| `POST /api/payments/initiate` | After booking success | ✅ Connected (done) |

The `main.js` currently has the mock submit handler (fake success). This was replaced by Gemini in the previous session with real API calls. Confirm this is in place.

---

### `buffalo-guest/` — Guest Dashboard

```
buffalo-guest/
├── login.html              ← Email + password login, OTP option
├── activate.html           ← First-time password setup (token from email/SMS)
├── dashboard.html          ← MAIN SPA — all 4 dashboard pages
├── payment-success.html    ← Post-payment landing page
└── assets/
    ├── css/guest.css       ← All dashboard styles (~900 lines)
    └── images/
        ├── hotel-reservation-logo.svg
        ├── hotel-reservation-cta-2.jpg
        └── hotel-reservation-hero-2-1.jpg
```

**API_BASE constant** — defined at the top of the `<script>` block in each file:
```javascript
const API_BASE = 'http://localhost:5000'; // ← Change to production URL when deploying
```

This is in:
- `login.html` — line 139
- `activate.html` — line 104
- `dashboard.html` — line 602

---

## 3. Authentication Flow (Guest Portal)

This is the most important thing to understand before building.

```
BOOKING CONFIRMED (via buffalo-hotel website)
         │
         ▼
StayFlow creates guest account automatically
  - email from booking form
  - generates activation token
  - status: PENDING_ACTIVATION
         │
         ▼
Send SMS + Email to guest:
  "Welcome! Activate your account: guest.buffalohotel.co.tz/activate.html?token=TOKEN"
         │
         ▼
Guest opens activate.html?token=TOKEN
  → Frontend calls: GET /api/guest/activate/verify?token=TOKEN
  → If valid: show "Set password" form
  → Guest sets password
  → Frontend calls: POST /api/guest/activate  { token, password }
  → Backend: saves password, returns JWT token + guest object
  → Frontend: saves to localStorage, redirects to dashboard.html
         │
         ▼
Next time guest visits login.html:
  → Email + Password → POST /api/guest/login
  → Returns JWT token + guest object
  → Saved to localStorage as:
      localStorage.setItem('guest_token', token)
      localStorage.setItem('guest_data', JSON.stringify(guest))
         │
         ▼
dashboard.html loads:
  → Reads token from localStorage
  → Sends as: Authorization: Bearer <token>
  → All API calls use this header
  → If no token → redirect to login.html
```

**localStorage keys used by frontend:**

| Key | Value | Set by |
|-----|-------|--------|
| `guest_token` | JWT string | login.html, activate.html |
| `guest_data` | JSON string `{firstName, lastName, email}` | login.html, activate.html |

Both are cleared on logout (`dashboard.html` → logout button).

---

## 4. All API Endpoints Required

### 4.1 Auth Endpoints

#### `POST /api/guest/login`
Called from: `login.html`

Request:
```json
{
  "email": "john@example.com",
  "password": "their_password"
}
```

Response (success):
```json
{
  "success": true,
  "token": "eyJhbGciOiJIUzI1NiJ9...",
  "guest": {
    "firstName": "John",
    "lastName": "Doe",
    "email": "john@example.com"
  }
}
```

Response (failure):
```json
{
  "success": false,
  "message": "Invalid email or password."
}
```

---

#### `POST /api/guest/otp`
Called from: `login.html` (OTP tab)

Request:
```json
{ "email": "john@example.com" }
```

Response:
```json
{ "success": true, "message": "OTP sent via SMS" }
```

- Generate 6-digit OTP
- Store against guest record (expires 10 min)
- Send via SMS to guest's registered phone number

---

#### `POST /api/guest/otp/verify`
Called from: `login.html` (OTP tab)

Request:
```json
{
  "email": "john@example.com",
  "code": "483920"
}
```

Response (success): Same shape as `/api/guest/login` success — return token + guest object.

---

#### `GET /api/guest/activate/verify?token=TOKEN`
Called from: `activate.html` on page load

Response (valid token):
```json
{
  "success": true,
  "firstName": "John"
}
```

Response (invalid/expired):
```json
{
  "success": false,
  "message": "Token expired or already used."
}
```

---

#### `POST /api/guest/activate`
Called from: `activate.html` after guest sets password

Request:
```json
{
  "token": "activation_token_string",
  "password": "their_new_password"
}
```

Response (success): Same shape as `/api/guest/login` — return token + guest object. Frontend immediately logs them in and redirects to dashboard.

---

### 4.2 Dashboard Endpoints

**All endpoints below require:** `Authorization: Bearer <token>` header.
If token is missing or invalid → return `401`.

---

#### `GET /api/guest/booking`
Called from: `dashboard.html` on load

Returns the **current active booking** for the authenticated guest.

Response:
```json
{
  "success": true,
  "booking": {
    "bookingId": "BUF-2026-002",
    "roomNumber": "205",
    "roomType": "Standard",
    "checkIn": "2026-06-17",
    "checkOut": "2026-06-21",
    "guests": "2 Adults",
    "ratePerNight": 30,
    "currency": "USD",
    "totalAmount": 120,
    "nights": 4
  }
}
```

Notes:
- Find the booking where `guestEmail = authenticated guest email` AND `status = CONFIRMED or CHECKED_IN`
- If multiple, return the most recent/current one
- If none found → return `{ "success": false, "message": "No active booking found." }`

---

#### `POST /api/guest/room-service`
Called from: `dashboard.html` → Room Service page

Request:
```json
{
  "items": [
    { "itemId": 1, "name": "Full English Breakfast", "quantity": 2, "unitPrice": 12000 },
    { "itemId": 13, "name": "Chai ya Tangawizi", "quantity": 1, "unitPrice": 2500 }
  ],
  "notes": "No sugar in the tea please",
  "totalAmount": 26500
}
```

Response:
```json
{
  "success": true,
  "orderId": "ORD-2026-0014",
  "estimatedDelivery": "20-30 minutes",
  "message": "Order received! Estimated delivery: 20-30 minutes"
}
```

Notes:
- Link order to guest's active booking via their token
- Save to database so front desk sees it in StayFlow RMS
- `totalAmount` is in TZS

---

#### `POST /api/guest/request`
Called from: `dashboard.html` → Requests page

Request body varies by `type` field:

**Laundry:**
```json
{
  "type": "laundry",
  "items": "3 shirts, 2 trousers",
  "collectionTime": "Morning (7AM – 10AM)",
  "notes": "Handle with care"
}
```

**Taxi:**
```json
{
  "type": "taxi",
  "destination": "Kilimanjaro International Airport (JRO)",
  "customDestination": "",
  "pickupTime": "2026-06-20T06:00",
  "passengers": "2"
}
```

**Tour:**
```json
{
  "type": "tour",
  "tourType": "Serengeti Safari (1 day)",
  "date": "2026-06-19",
  "people": "2",
  "notes": "First time safari"
}
```

**Housekeeping:**
```json
{
  "type": "housekeeping",
  "requestType": "Fresh towels",
  "preferredTime": "Morning (7AM – 12PM)",
  "notes": ""
}
```

**Other:**
```json
{
  "type": "other",
  "message": "Can we get a baby cot in the room?"
}
```

Response (all types):
```json
{
  "success": true,
  "requestId": "REQ-2026-0031",
  "message": "Request received. Our team will respond shortly."
}
```

Notes:
- Save all requests to database linked to booking/guest
- Front desk should see them in StayFlow RMS dashboard

---

#### `POST /api/guest/extend`
Called from: `dashboard.html` → Extend Stay page

Request:
```json
{
  "bookingId": "BUF-2026-002",
  "extraNights": 2,
  "reason": "Delayed flight"
}
```

Response:
```json
{
  "success": true,
  "message": "Extension request received. We will confirm within 30 minutes via SMS.",
  "requestedNewCheckout": "2026-06-23"
}
```

Notes:
- Do **NOT** automatically extend the booking
- Create an extension request record with status `PENDING`
- Front desk confirms it manually in StayFlow RMS → status becomes `APPROVED` → booking checkout date updates
- Send SMS to guest on confirmation

---

#### `GET /api/guest/orders`
Called from: `dashboard.html` → loads recent activity feed

Response:
```json
{
  "success": true,
  "orders": [
    {
      "type": "room_service",
      "title": "Breakfast × 2 + Chai × 1",
      "time": "Today, 8:32 AM",
      "amount": 26500,
      "status": "confirmed"
    },
    {
      "type": "request",
      "title": "Laundry Request",
      "time": "Yesterday, 3:15 PM",
      "amount": null,
      "status": "pending"
    }
  ]
}
```

Notes:
- Return combined list: room service orders + service requests for this guest
- Sorted by most recent first
- Max 10 items
- `status` values used by frontend: `confirmed`, `pending`, `processing`, `cancelled`
- `amount` is TZS (null for non-food requests)

---

### 4.3 Trigger: Auto-create Guest Account After Booking

This is the **bridge between the two frontends.** When a booking is confirmed via the website:

```
POST /api/bookings/website → booking CONFIRMED → payment paid
```

At this point, StayFlow should automatically:

1. Check if a guest account exists for that email
2. If not → create one with status `PENDING_ACTIVATION`
3. Generate a secure activation token (random 32-char string, expires 48 hours)
4. Send **SMS** to guest phone: `"Welcome to Buffalo Hotel! Access your guest dashboard: guest.buffalohotel.co.tz/activate.html?token=TOKEN"`
5. Send **Email** to guest: Same message with a clickable link

This means the guest never manually "signs up" — the account is created silently when they book.

---

## 5. CORS Configuration

Both frontends run on different origins. Backend must allow all of them:

```typescript
// In app.ts / main.ts
app.use(cors({
  origin: [
    'https://buffalohotel.co.tz',           // main website (production)
    'https://guest.buffalohotel.co.tz',     // guest dashboard (production)
    'http://localhost:3000',
    'http://localhost:5500',
    'http://127.0.0.1:5500',               // VS Code Live Server
    'null'                                  // local file:// access
  ],
  methods: ['GET', 'POST', 'PUT', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true
}));
```

---

## 6. JWT Token Structure

Frontend expects to decode nothing from the token — it just sends it as a Bearer header. But the backend should encode at minimum:

```json
{
  "guestId": "uuid-here",
  "email": "john@example.com",
  "bookingId": "BUF-2026-002",
  "iat": 1718000000,
  "exp": 1718604800
}
```

Token expiry: **7 days** (guest stays are short, convenience matters more than rotation here).

---

## 7. Suggested File Locations in StayFlow

Based on how existing routes are structured (`apps/api/src/routes/website.routes.ts`):

```
StayFlow/
└── apps/
    └── api/
        └── src/
            ├── routes/
            │   ├── website.routes.ts         ← already exists (booking + payment)
            │   └── guest.routes.ts           ← NEW — all /api/guest/* routes
            │
            ├── controllers/
            │   ├── website.controller.ts     ← already exists
            │   └── guest.controller.ts       ← NEW
            │
            ├── services/
            │   └── guest.service.ts          ← NEW (or extend existing services)
            │
            └── middleware/
                └── guest.auth.ts             ← NEW — JWT verify middleware for guest routes
```

---

## 8. Database — New Records Needed

The following new data needs to be stored (add to Prisma schema if not already present):

```
GuestAccount
  - id
  - email (unique)
  - passwordHash
  - firstName
  - lastName
  - phone
  - activationToken (nullable)
  - tokenExpiresAt (nullable)
  - status: PENDING_ACTIVATION | ACTIVE
  - createdAt
  - linkedBookingId → Booking

RoomServiceOrder
  - id
  - orderId (BUF-ORD-XXXX)
  - guestAccountId → GuestAccount
  - bookingId → Booking
  - items (JSON)
  - totalAmount (TZS)
  - notes
  - status: PENDING | PREPARING | DELIVERED | CANCELLED
  - createdAt

ServiceRequest
  - id
  - requestId (REQ-XXXX)
  - guestAccountId → GuestAccount
  - bookingId → Booking
  - type: laundry | taxi | tour | housekeeping | other
  - payload (JSON — varies by type)
  - status: PENDING | IN_PROGRESS | COMPLETED | CANCELLED
  - createdAt

ExtensionRequest
  - id
  - bookingId → Booking
  - extraNights
  - requestedNewCheckout
  - reason
  - status: PENDING | APPROVED | REJECTED
  - createdAt
```

---

## 9. Testing Checklist

After building, verify each flow:

### Auth Flow
```
1. Make a booking on buffalo-hotel/contact.html
2. Confirm guest account was auto-created in DB
3. Confirm SMS/email sent with activation link
4. Open activate.html?token=TOKEN
5. Set password → redirects to dashboard.html ✓
6. Logout → back to login.html
7. Login with email + password → back to dashboard ✓
8. Login with OTP → request code → verify → dashboard ✓
```

### Dashboard Flow
```
1. Dashboard loads → booking data shows correctly ✓
2. Room service → add items → place order → appears in StayFlow RMS ✓
3. Requests → submit laundry → appears in StayFlow RMS ✓
4. Extend stay → submit 2 nights → extension request in StayFlow RMS ✓
5. Recent activity shows orders & requests ✓
```

### PowerShell test commands (Windows):

```powershell
# Test login
Invoke-RestMethod -Uri "http://localhost:5000/api/guest/login" -Method POST -ContentType "application/json" -Body '{"email":"test@guest.com","password":"test1234"}'

# Test get booking (replace TOKEN)
Invoke-RestMethod -Uri "http://localhost:5000/api/guest/booking" -Method GET -Headers @{ Authorization = "Bearer TOKEN_HERE" }

# Test room service order
Invoke-RestMethod -Uri "http://localhost:5000/api/guest/room-service" -Method POST -ContentType "application/json" -Headers @{ Authorization = "Bearer TOKEN_HERE" } -Body '{"items":[{"itemId":1,"name":"Ugali & Nyama Choma","quantity":1,"unitPrice":18000}],"totalAmount":18000}'
```

---

## 10. Environment Variables to Set

In StayFlow `.env`:

```env
# Existing
DATABASE_URL=...
PORT=5000

# New — add these
JWT_GUEST_SECRET=buffalo_guest_secret_change_this_in_production
JWT_GUEST_EXPIRES_IN=7d
ACTIVATION_TOKEN_EXPIRES_HOURS=48

# SMS (for OTP and activation links)
SMS_PROVIDER=africas_talking        # or twilio, nexmo — whichever is integrated
SMS_API_KEY=your_key_here
SMS_SENDER_ID=BuffaloHotel

# Email (for activation + confirmation)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=noreply@buffalohotel.co.tz
SMTP_PASS=your_app_password

# Frontend URLs (used in email/SMS links)
WEBSITE_URL=https://buffalohotel.co.tz
GUEST_PORTAL_URL=https://guest.buffalohotel.co.tz
```

---

## 11. Summary — What Gemini Needs to Build

| # | What | File(s) |
|---|------|---------|
| 1 | `guest.routes.ts` — all routes | `src/routes/guest.routes.ts` |
| 2 | `guest.controller.ts` — all handlers | `src/controllers/guest.controller.ts` |
| 3 | `guest.auth.ts` — JWT middleware | `src/middleware/guest.auth.ts` |
| 4 | Prisma schema additions | `schema.prisma` |
| 5 | Auto-create guest account on booking confirm | Inside `website.controller.ts` → booking confirmed block |
| 6 | Send SMS activation link | Trigger in step 5 |
| 7 | CORS update | `app.ts` |

Everything else (UI, forms, cart, page navigation, styling) is **already done** in the frontend. Gemini does not need to touch any HTML/CSS/JS files.

---

*Last updated: June 2026 — Buffalo Hotel Client Project*
*Frontend by: Claude (Anthropic)*
*Backend by: Gemini CLI*
*Project managed by: Davy @ Terra Consultant Limited*
