# Smart College Bus Attendance & Live Tracking System

A working full-stack prototype: Node.js/Express + Socket.IO + SQLite backend,
React (Vite) frontend with four role-based dashboards (Student, Driver, Parent, Admin),
live GPS tracking, attendance with late-entry detection, and proximity-based
"come fast" alerts.

## Stack
- **Backend:** Express, Socket.IO, better-sqlite3 (zero-config file DB), JWT auth, bcrypt
- **Frontend:** React + Vite, react-router-dom, react-leaflet (maps), socket.io-client, axios

## Project Structure
```
smart-bus-system/
├── backend/       # Express API + Socket.IO server + SQLite DB
└── frontend/      # React dashboards (Student / Driver / Parent / Admin)
```

## 1. Backend Setup

```bash
cd backend
npm install
cp .env.example .env      # edit JWT_SECRET etc. if you want
npm run dev                # or: npm start
```

The server starts on **http://localhost:5000** and auto-creates+seeds
`backend/smartbus.sqlite` on first run with demo data:

| Role    | Email                | Password    |
|---------|-----------------------|-------------|
| Admin   | admin@college.edu     | admin123    |
| Driver  | driver@college.edu    | driver123   |
| Parent  | parent@college.edu    | parent123   |
| Student | student@college.edu   | student123  |

It also seeds one bus (`AP-BUS-01`), one route with 3 stops, and links the
demo student/parent to that route.

## 2. Frontend Setup

Open a **second terminal**:

```bash
cd frontend
npm install
cp .env.example .env       # points VITE_BACKEND_URL at http://localhost:5000
npm run dev
```

Visit **http://localhost:5173** — you'll land on the login page. Click any
demo account in the list to autofill credentials.

## 3. Try the live tracking flow
1. Log in as **driver@college.edu** in one browser tab.
2. Log in as **student@college.edu** (or **parent@college.edu**) in another tab
   (or another browser / incognito window).
3. On the Driver dashboard, pick the seeded bus + route, choose trip type, click
   **Start Trip**. Allow location access if prompted (falls back to a simulated
   moving marker if GPS isn't available, e.g. on desktop).
4. Watch the bus marker move live on the Student/Parent map (pushed via
   Socket.IO), and mark a student "Present" from the Driver dashboard to see it
   reflected instantly.
5. Log in as **admin@college.edu** to see fleet management, live monitoring of
   all active trips, and the late-entry report.

## How the pieces connect
- **REST API** (`/api/...`) handles auth, CRUD (buses/routes/stops/students),
  starting/ending trips, marking attendance, and fetching reports — protected
  by JWT (`middleware/auth.js`) with role checks.
- **Socket.IO** (`backend/socket/index.js`) handles the real-time layer:
  - Driver's browser streams `driver:location` events (via `navigator.geolocation.watchPosition`,
    or a simulated fallback) every few seconds.
  - Server stores each ping in the `locations` table and broadcasts `bus:location`
    to all connected clients (student/parent/admin dashboards listening via `react-leaflet`).
  - Server computes straight-line distance (Haversine) from the bus to each
    upcoming stop, estimates ETA using an assumed average speed, and fires a
    `come_fast_alert` event + persists an `alerts` row when ETA crosses configured
    thresholds (default 5 min / 2 min, set via `COME_FAST_ALERT_MINUTES` in `.env`).
- **Late-entry logic** (`routes/attendance.js`): each stop has an `expected_time`;
  attendance marked after that time is flagged `late`. Once a student crosses
  `LATE_ENTRIES_WARNING_COUNT` late entries in the current month, a warning alert
  is created for both the student and their linked parent.

## Extending this prototype
This is a functional MVP covering the core loop end-to-end. Natural next steps
for a production build:
- Replace SQLite with PostgreSQL/MySQL for concurrent multi-instance deployments.
- Real QR/RFID hardware integration for boarding scans (currently a manual "Mark Present" button).
- Push notifications via Firebase Cloud Messaging + SMS gateway (Twilio/MSG91) instead of in-app alerts only.
- OSRM/Google Directions API for real route-distance ETA instead of straight-line Haversine.
- Refresh tokens / token rotation, rate limiting, and input validation (e.g. zod) on all endpoints.
- Dedicated mobile apps (Flutter/React Native) for driver and student/parent for reliable background GPS.
