# RM Private Pool — Project Process Map & Activity Diagrams

> Generated: 2026-06-16  
> Project: RM Private Pool (Lubao, Pampanga)  
> Stack: Bootstrap 5 HTML · PHP SOAP API (Frontend) · PHP REST API/Leaf (Admin)

---

## 1. Project Architecture Overview

```
public_html/
├── index.html, amenities_inclusions.html, rates_bookings.html   ← Public site pages
├── events.html, contact.html
├── login.html, signup.html, profile.html                        ← Member auth pages
├── assets/js/auth.js                                            ← SOAP client helper
├── api/                                                         ← SOAP API (Member auth)
│   ├── index.php                                                ← SOAP handler
│   ├── soap/SoapHandler.php, WsdlGenerator.php
│   ├── services/AuthService.php
│   └── config/database.php
└── admin/                                                       ← ADMIN PANEL (separated)
    ├── login.html                                               ← Admin login gate
    ├── dashboard.html, members.html, bookings.html ...
    ├── admin.js, admin.css, layout.js
    └── api/                                                     ← REST API (Admin)
        └── index.php                                            ← Leaf PHP router
```

### Two Separate Systems

| Layer | Public Site | Admin Panel |
|---|---|---|
| URL | `/` | `/admin/` |
| API type | SOAP (`/api/`) | REST JSON (`/admin/api/`) |
| Auth token | `localStorage.rm_token` | `localStorage.rm_admin_token` |
| Auth table | `user_tokens` | `admin_tokens` (24 hr expiry) |
| Users | `users` table | `admin_users` table |
| Roles | Member only | superadmin / admin / staff |

---

## 2. Admin Panel — Separation Details

The admin panel lives at `/admin/` and is already isolated from the public site:

- **Separate login page** — `admin/login.html` with its own session (no relation to member login)
- **Separate token storage** — `rm_admin_token` vs public `rm_token`
- **Separate API** — `/admin/api/index.php` (Leaf REST) vs `/api/index.php` (SOAP)
- **Separate DB tables** — `admin_users`, `admin_tokens`, `audit_trails`
- **Directory listing disabled** — `.htaccess` has `Options -Indexes`
- **Cache busting** — JS/CSS served with `no-cache` headers

### Admin Pages Map

| Page | Route | Function |
|---|---|---|
| Login | `/admin/login.html` | Admin authentication gate |
| Dashboard | `/admin/dashboard.html` | Stats, charts, mini calendar, quick actions |
| Members | `/admin/members.html` | View/Add/Edit/Deactivate members |
| Points | `/admin/points.html` | Points transactions ledger |
| Points Settings | `/admin/points-earning.html` | Earning & redemption rules |
| Reservations | `/admin/bookings.html` | Create/manage bookings |
| Calendar | `/admin/calendar.html` | Availability calendar view |
| Rates | `/admin/rates.html` | Manage pricing rates |
| Add-ons | `/admin/addons.html` | Manage add-on services |
| Maintenance | `/admin/maintenance.html` | Schedule/track maintenance |
| Admin Users | `/admin/admins.html` | Manage admin accounts |
| Reports | `/admin/reports.html` | Revenue, bookings, points reports |
| Audit Trail | `/admin/audit-trail.html` | Full admin action log |
| Settings | `/admin/settings.html` | System-wide configuration |

---

## 3. Activity Diagram — Member Registration & Login (Public Site)

```mermaid
flowchart TD
    A([Visitor]) --> B[Visit Website]
    B --> C{Has Account?}

    C -- No --> D[Go to signup.html]
    D --> E[Fill Registration Form\nusername / email / password\nfirst_name / last_name\ncontact / address / birthdate]
    E --> F[POST SOAP: UserSignup]
    F --> G{Valid?}
    G -- Duplicate username/email --> H[Show Error Message]
    H --> E
    G -- Success --> I[Auto-login\nStore rm_token + rm_profile\nin localStorage]
    I --> J[Redirect to profile.html]

    C -- Yes --> K[Go to login.html]
    K --> L[Enter username + password]
    L --> M[POST SOAP: UserLogin]
    M --> N{Credentials OK?}
    N -- No --> O[Show Error]
    O --> L
    N -- Yes --> P[Store rm_token in localStorage]
    P --> Q[auth.js calls GetProfile\non every page load]
    Q --> R[Nav shows: My Profile dropdown\nHides: Login / Sign Up]

    J --> S[View/Edit profile.html]
    S --> T[POST SOAP: UpdateProfile\nOptional: change password]
    S --> U[POST SOAP: DeleteAccount\nRequires password confirm]
    U --> V[Clear localStorage\nRedirect to home]
```

---

## 4. Activity Diagram — Admin Authentication

```mermaid
flowchart TD
    A([Admin User]) --> B[Visit /admin/]
    B --> C{rm_admin_token\nin localStorage?}
    C -- No --> D[Redirect to /admin/login.html]
    D --> E[Enter username or email\n+ password]
    E --> F[POST /admin/api/login]
    F --> G{Credentials Valid\nand status='active'?}
    G -- No --> H[Show 'Invalid credentials']
    H --> E
    G -- Yes --> I[Revoke old tokens\nIssue new 24-hour token]
    I --> J[Log audit: ADMIN LOGIN]
    J --> K[Return token + admin info]
    K --> L[Store rm_admin_token +\nrm_admin_user in localStorage]
    L --> M[Redirect to dashboard.html]

    C -- Yes --> N[GET /admin/api/me]
    N --> O{Token valid\nand not expired?}
    O -- No → 401 --> D
    O -- Yes --> M

    M --> P[Use Admin Panel]
    P --> Q[Every API call sends\nAuthorization: Bearer token]
    Q --> R{Token still valid?}
    R -- No → 401 --> D
    R -- Yes --> S[Action Performed\nAudit Log Written]

    P --> T[Click Logout]
    T --> U[POST /admin/api/logout\nDELETE token from admin_tokens]
    U --> V[Clear localStorage]
    V --> D
```

---

## 5. Activity Diagram — Booking Creation Process

```mermaid
flowchart TD
    A([Admin]) --> B[Click 'New Booking'\nbookings.html or dashboard]
    B --> C[Open Booking Modal]
    C --> D{Member or Walk-in?}

    D -- Member --> E[Search by name/ID/email]
    E --> F[GET /admin/api/members/search]
    F --> G[Select member from results]
    G --> H[Load member points balance]

    D -- Walk-in --> I[Enter guest name + contact]

    H --> J[Fill booking details]
    I --> J
    J --> K[Select overnight date\nfrom calendar]
    K --> L[Select rate\nGET /admin/api/rates]
    L --> M[Add optional add-ons\nGET /admin/api/addons]
    M --> N{Member with points?}
    N -- Yes --> O[Enter points to redeem\nSystem calculates discount]
    N -- No --> P[Skip redemption]
    O --> Q[Auto-compute totals:\nbase_rate + addons − redemption_value]
    P --> Q

    Q --> R[Set booking_status\npending / confirmed]
    R --> S[Set payment_status\nunpaid / partial / paid]
    S --> T[POST /admin/api/bookings]
    T --> U{Date already booked\nor under maintenance?}
    U -- Conflict --> V[Show Error: Date unavailable]
    V --> K
    U -- Clear --> W[Insert booking record\nInsert booking_addons]

    W --> X{Payment = paid OR\nStatus = completed?}
    X -- Yes --> Y[Compute earned points\nper earning settings]
    Y --> Z[Insert points_transaction: earned\nUpdate user points balance]
    X -- No --> AA[Save booking, points TBD later]
    Z --> AB[Log audit: CREATE booking]
    AA --> AB
    AB --> AC[Show success toast\nRefresh bookings table]
```

---

## 6. Activity Diagram — Points Lifecycle

```mermaid
flowchart TD
    A([Member Books Stay]) --> B[Booking marked\npaid or completed]
    B --> C[computeEarnedPoints total_amount]
    C --> D{Earning rule active?}
    D -- Yes --> E[Formula: floor amt ÷ php_amount × pts_value\nDefault: 1pt per ₱100]
    D -- No --> F[0 points earned]
    E --> G[INSERT points_transactions: earned\nUPDATE users.points balance]

    H([Admin Manual Adjust]) --> I[POST /admin/api/members/id/points/adjust]
    I --> J{type?}
    J -- credit --> K[Add points to balance]
    J -- debit --> L{Sufficient balance?}
    L -- No, neg disabled --> M[Error: Insufficient]
    L -- Yes or neg allowed --> N[Subtract from balance]
    K --> O[Insert: transaction_type = credited]
    N --> P[Insert: transaction_type = debited]

    Q([Member Redeems at Booking]) --> R[Admin enters redeemed_points\nin booking form]
    R --> S[Check user.points ≥ redeemed_points]
    S -- Insufficient --> T[Error: Insufficient points]
    S -- OK --> U[GET redemption settings\npoints_per_unit, php_value]
    U --> V[Compute discount:\nredeemed ÷ per_unit × php_value]
    V --> W[Deduct from total_amount]
    W --> X[INSERT points_transactions: redeemed\nnegative points_amount\nUPDATE user balance]

    O --> Y[Audit logged]
    P --> Y
    X --> Y
    G --> Y
```

---

## 7. Activity Diagram — Maintenance Scheduling

```mermaid
flowchart TD
    A([Admin]) --> B[Go to maintenance.html]
    B --> C[Click 'Schedule Maintenance']
    C --> D[Fill maintenance form:\n- Date\n- Type from maintenance_types\n- Status: scheduled/ongoing\n- Blocks bookings? yes/no\n- Cost, notes, recommendation]
    D --> E[POST /admin/api/maintenance-records]
    E --> F{blocks_booking = 1?}
    F -- Yes --> G[Date marked 'maintenance'\non calendar — no new bookings allowed]
    F -- No --> H[Date still bookable\nbut maintenance note shown]
    G --> I[Audit log: CREATE maintenance]
    H --> I

    I --> J[Admin Updates Record]
    J --> K{New status?}
    K -- ongoing --> L[PUT /admin/api/maintenance-records/id\nstatus = ongoing]
    K -- completed --> M[PUT: status = completed\nRecord actual cost]
    K -- cancelled --> N[DELETE endpoint\nSets status = cancelled]
    L --> O[Calendar updates automatically]
    M --> O
    N --> O

    P([New Booking Attempt\nfor Maintenance Date]) --> Q[POST /admin/api/bookings]
    Q --> R{maintenance record exists\nblocks_booking=1\nstatus ∈ scheduled/ongoing?}
    R -- Yes --> S[Error: Date blocked for maintenance]
    R -- No --> T[Booking allowed]
```

---

## 8. Activity Diagram — Audit Trail Flow

```mermaid
flowchart TD
    A([Any Admin Action]) --> B{Action type}
    B --> C[CREATE / UPDATE / DEACTIVATE\nCANCEL / LOGIN / LOGOUT\nCREDIT / DEBIT / REDEEMED]
    C --> D[logAudit helper called\nautomatically after every write]
    D --> E[INSERT audit_trails:\n- admin_id + admin_name\n- action_type + module\n- record_id\n- old_value JSON\n- new_value JSON\n- description\n- ip_address + user_agent]
    E --> F[Stored permanently\nNo delete endpoint]

    G([Admin views audit-trail.html]) --> H[GET /admin/api/audit-trails\nFilters: admin / action / module / date range / search]
    H --> I[Display paginated table\nwith old/new value diff]
    H --> J[GET /admin/api/reports/audit-trails\nDate-range export view]
```

---

## 9. Database Schema — Entity Relationships

```
users ──────────────────────────────────────────────────────┐
│ id, membership_id, username, email, password               │
│ first_name, last_name, contact_number, address, birthdate  │
│ points, status, remarks, created_at                        │
└──┬──────────────────────────────────────────────────────── │
   │ 1:N                                                      │
   ▼                                                          │
bookings ──────────────────────────────────────────────────  │
│ id, user_id (FK→users), guest_name, guest_contact          │
│ overnight_date (UNIQUE), checkin_datetime, checkout_datetime│
│ rate_id (FK→rates), base_rate, addons_total, subtotal      │
│ redeemed_points, redeemed_points_value, total_amount       │
│ earned_points                                              │
│ booking_status: pending|confirmed|completed|cancelled       │
│ payment_status: unpaid|partial|paid|refunded               │
│ created_by (FK→admin_users)                                │
└──┬──────────────┬───────────────────────────────────────── │
   │ 1:N           │ 1:N                                      │
   ▼               ▼                                          │
booking_addons    points_transactions ─────────────────────── │
│ booking_id      │ user_id (FK→users)                       │
│ addon_id        │ booking_id (FK→bookings, nullable)        │
│ addon_name      │ transaction_type:                         │
│ quantity        │   earned|redeemed|credited|debited        │
│ unit_price      │ points_amount (negative = debit)          │
│ total_price     │ previous_balance, new_balance             │
└─────────────    │ performed_by (FK→admin_users)            │
                  └───────────────────────────────────────── │

rates                        addons
│ id, rate_name, amount      │ id, addon_name, price
│ rate_type:                 │ status, created_by
│   weekday|weekend|holiday  └──────────────────────
│   promo|custom
│ effective_date, status
└──────────────────────────

admin_users ──────────────────────────────────────────────────
│ id, username, email, full_name, password                    │
│ role: superadmin|admin|staff                                │
│ status: active|inactive                                     │
└──┬───────────────────────────────────────────────────────── │
   │ 1:N                                                       │
   ▼                                                           │
admin_tokens                  audit_trails                     │
│ admin_id (FK)               │ admin_id (FK→admin_users)     │
│ token (64-char hex)         │ action_type, module           │
│ expires_at (+24h)           │ record_id, old_value,new_value│
└──────────────              │ ip_address, user_agent         │
                             └───────────────────────────────  │

maintenance_types ──────────────────────────────────────────── │
│ id, type_name, description, status                           │
└──┬───────────────────────────────────────────────────────── │
   │ 1:N                                                       │
   ▼                                                           │
maintenance_records                                            │
│ maintenance_date, maintenance_type_id                        │
│ status: scheduled|ongoing|completed|cancelled                │
│ blocks_booking (TINYINT)                                     │
│ cost, notes, recommendation                                  │
└──────────────────────────────────────────────────────────── │

points_earning_settings      points_redemption_settings
│ php_amount, points_value   │ points_per_unit, php_value
│ Default: ₱100 → 1pt        │ Default: 1pt → ₱1.00
└──────────────────          └──────────────────────────

system_settings              membership_sequence
│ setting_key / value        │ Auto-increment counter
│ setting_group              │ for membership_id (e.g. RM-0001)
└──────────────              └───────────────────────────
```

---

## 10. REST API Endpoint Reference (Admin)

| Method | Endpoint | Action |
|---|---|---|
| POST | `/login` | Admin login → token |
| POST | `/logout` | Revoke token |
| GET | `/me` | Get current admin info |
| GET | `/admins` | List admin users |
| POST | `/admins` | Create admin user |
| GET/PUT/DELETE | `/admins/{id}` | Get / Update / Deactivate admin |
| GET | `/users` | List members |
| POST | `/users` | Create member |
| GET/PUT/DELETE | `/users/{id}` | Get / Update / Deactivate member |
| GET | `/members/search` | Search members (typeahead) |
| GET | `/users/{id}/points` | Get member points balance |
| POST | `/members/{id}/points/adjust` | Credit or debit points |
| POST | `/points/credit` | Manual credit points |
| POST | `/points/debit` | Manual debit points |
| GET | `/points/transactions` | List points ledger |
| GET/POST | `/points/settings` | Get / Update earning+redemption rules |
| PUT | `/points/settings/{id}` | Update specific rule |
| GET | `/rates` | List rates |
| POST | `/rates` | Create rate |
| GET/PUT/DELETE | `/rates/{id}` | Get / Update / Deactivate rate |
| GET | `/addons` | List add-ons |
| POST | `/addons` | Create add-on |
| GET/PUT/DELETE | `/addons/{id}` | Get / Update / Deactivate add-on |
| GET | `/bookings` | List bookings (filterable) |
| POST | `/bookings` | Create booking |
| GET/PUT/DELETE | `/bookings/{id}` | Get / Update / Cancel booking |
| GET | `/calendar/availability` | Month calendar with booking+maintenance status |
| GET | `/maintenance-types` | List maintenance categories |
| POST | `/maintenance-types` | Create category |
| GET/PUT/DELETE | `/maintenance-types/{id}` | Get / Update / Deactivate |
| GET | `/maintenance-records` | List maintenance records |
| POST | `/maintenance-records` | Schedule maintenance |
| GET/PUT/DELETE | `/maintenance-records/{id}` | Get / Update / Cancel record |
| GET | `/audit-trails` | Query audit log |
| GET | `/dashboard/stats` | Dashboard aggregate stats |
| GET | `/reports/bookings` | Booking report (date range) |
| GET | `/reports/revenue` | Monthly revenue summary |
| GET | `/reports/maintenance` | Maintenance cost report |
| GET | `/reports/points` | Points transactions report |
| GET | `/reports/points-earned` | Earned points report |
| GET | `/reports/points-redeemed` | Redeemed points report |
| GET | `/reports/audit-trails` | Audit log export |
| GET/POST | `/settings` | Get / Save system settings |

---

## 11. SOAP API Reference (Public / Member)

| Operation | Auth Required | Action |
|---|---|---|
| `UserLogin` | No | username + password → token |
| `UserSignup` | No | Register new member → auto-login |
| `GetProfile` | Bearer token in SOAP header | Fetch logged-in member data |
| `UpdateProfile` | Bearer token | Update fields (optional: new_password + current_password) |
| `DeleteAccount` | Bearer token | Delete own account (requires password) |

---

## 12. Key Business Rules

1. **One booking per overnight date** — `UNIQUE KEY uq_overnight_date` on `bookings.overnight_date`
2. **Maintenance blocks bookings** — when `blocks_booking=1` and status is `scheduled` or `ongoing`, new bookings on that date are rejected
3. **Points earning** — triggered automatically when booking becomes `completed` or `paid`; formula: `floor(total ÷ php_amount) × points_value`
4. **Points redemption** — deducted at booking creation; formula: `(pts ÷ points_per_unit) × php_value` = peso discount
5. **Soft deletes** — no hard deletes anywhere; records are set to `inactive` or `cancelled`
6. **Audit everything** — every admin write operation logs to `audit_trails` with old/new JSON values, IP, and user agent
7. **Token expiry** — admin tokens expire after 24 hours; frontend member tokens use their own expiry mechanism
8. **Membership IDs** — auto-assigned sequential IDs via `membership_sequence` table
