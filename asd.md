# SafeRide — Transport Tracking & Safety Platform

A unified system for **passengers, drivers, and admins** covering private transport (school/office buses, carpool) and **public transport multi-bus tracking**, with safety, monitoring, and emergency response.

> Important scope note: Lovable builds **web apps** (responsive, installable as PWA). A true native **Android APK** is not built here — we'll ship a PWA that installs to the home screen and behaves like an app, and (optionally) wrap it with Capacitor to produce an `.apk`. The "software" (admin console) is a web dashboard. Same codebase, three experiences.

---

## 1. Feature List (everything you asked + 3 extras)

Core 10:
1. Transport management (routes, stops, vehicles, drivers)
2. Live GPS tracking (passenger sees bus on map)
3. Automatic alerts (push/SMS/email on events)
4. Estimated arrival time (ETA, live updating)
5. Driver verification (KYC, license, face match at login)
6. Carpool (offer/find rides, seat booking, split fare)
7. Emergency / SOS (harassment, accident, panic button → admin + nearest contacts + police info)
8. Speed monitoring (over-speed alerts)
9. Wrong-route detection (geofence deviation)
10. Stop-arrived alert (auto notify parent/passenger)
11. Multi-bus public transport tracking (city-wide live map)

3 added extras:
12. **Parent/Guardian linked accounts** (parent watches child's ride live)
13. **Trip history & incident log** (audit trail, downloadable reports)
14. **Offline cache + low-bandwidth mode** (last known position, queued SOS)

---

## 2. User Roles

- **Passenger** (student / commuter / parent)
- **Driver**
- **Admin / Operator** (school, company, transit authority)
- **Super Admin** (platform owner)

Roles stored in a separate `user_roles` table with a `has_role()` security-definer function (never on profiles).

---

## 3. Platforms & How Each Is Built

| Surface | Tech | Output |
|---|---|---|
| Passenger app | React PWA (installable) | Works in browser + "Add to Home Screen" on Android/iOS |
| Driver app | Same PWA, driver role | Uses device GPS in background |
| Admin dashboard ("software") | React web app | Desktop-first console |
| Android APK | Capacitor wrap of the PWA | Real `.apk` installable file |
| Backend | Lovable Cloud (Postgres + Auth + Storage + Server functions) | Managed |
| Realtime | Supabase Realtime channels | Live bus positions |
| Maps | Google Maps Platform connector | Map, geocoding, routes, ETA |
| Notifications | Web Push + email (Lovable Email); SMS via Twilio if added | Multi-channel |

---

## 4. How Each Feature Works (mechanics)

**Live tracking** — Driver app posts `{lat, lng, speed, heading, ts}` every 5s to a `vehicle_pings` table; Realtime broadcasts to subscribers; passenger map animates marker.

**ETA** — Server function calls Google Routes API with current bus position → next stop, returns seconds; cached 30s.

**Automatic alerts** — Postgres trigger on `vehicle_pings` evaluates rules (speed > limit, outside route polygon, within 200m of stop, stopped > N min) → inserts into `alerts` → server function fans out push/email/SMS.

**Driver verification** — On signup: upload license + selfie → admin approves. On each shift start: selfie + liveness check stored in `driver_sessions`. Cannot start trip until verified.

**Carpool** — `rides` table (origin, destination, time, seats, price). Riders request → driver accepts → in-app chat → shared live track during ride → rating after.

**SOS / harassment** — Big red button. One press: records audio 30s, captures location, sets `incidents.status='active'`, alerts admin + emergency contacts + (if configured) shows local police number. Silent mode option (no UI change, just sends).

**Speed monitoring** — Per-route `speed_limit_kmh`. Ping > limit for >10s → alert. Repeated → driver score drops.

**Wrong route** — Each route has a polyline + buffer (e.g., 100m). If 3 consecutive pings outside buffer → "Off route" alert + map flag for admin.

**Stop-arrived** — Geofence (radius 80m) around each stop. Enter geofence → notify subscribed parents/passengers for that stop.

**Multi-bus public transport** — Public map page (no login) shows all active buses for a transit operator, filter by route/line. Same `vehicle_pings`, just public read policy on selected vehicles.

**Parent link** — Parent account links a child passenger; sees child's assigned bus, stop ETAs, ride history, receives stop-arrived + SOS alerts.

**Trip history** — Every trip = row in `trips` with start/end, distance, max speed, alerts count. Exportable CSV/PDF.

**Offline / low-bandwidth** — Service worker caches last 5 positions + map tiles around route; SOS queued in IndexedDB and flushed when online.

---

## 5. Data Model (key tables)

```text
profiles(id, full_name, phone, avatar_url)
user_roles(user_id, role)                  -- passenger|driver|admin|parent|super_admin
organizations(id, name, type)              -- school|company|transit
vehicles(id, org_id, plate, capacity, type)
drivers(id, user_id, license_no, verified_at, status)
routes(id, org_id, name, polyline, speed_limit_kmh, is_public)
stops(id, route_id, name, lat, lng, order_idx, geofence_m)
trips(id, vehicle_id, driver_id, route_id, started_at, ended_at, stats jsonb)
vehicle_pings(id, vehicle_id, trip_id, lat, lng, speed, heading, ts)
alerts(id, type, severity, vehicle_id, trip_id, payload jsonb, created_at)
incidents(id, user_id, type, location, audio_url, status, created_at)
carpool_rides(id, driver_id, origin, dest, depart_at, seats, price)
carpool_bookings(id, ride_id, rider_id, status)
parent_links(parent_id, child_id)
device_subscriptions(user_id, endpoint, keys)   -- web push
notifications(id, user_id, channel, body, sent_at)
```

All tables: **RLS on**. Policies use `has_role(auth.uid(), 'admin')` etc.

---

## 6. Architecture

```text
[Driver PWA] --pings--> [Server Fn] --> [Postgres + Realtime]
[Passenger PWA] <--subscribe-- [Realtime]            |
[Admin Dashboard] <--queries/realtime---             |
                                                     v
                                              [Triggers] -> [alerts] -> [Notifier Fn]
                                                                          | push/email/sms
[Public Map]  <--read public routes/vehicles--
[Google Maps connector] used by server fns for ETA, geocoding, routing
```

---

## 7. Phased Build Plan (step by step)

Each phase is shippable. I will execute them after you approve, one phase per turn so you can review.

**Phase 0 — Foundations**
- Enable Lovable Cloud
- Design system (colors, typography, dark map theme)
- Auth (email + phone OTP), roles table, `has_role()`
- App shell + role-based routing

**Phase 1 — Org, Vehicles, Routes, Stops (Admin)**
- Admin dashboard CRUD for orgs, vehicles, drivers, routes (draw polyline on map), stops (drop pins, set geofence)
- Driver verification queue (approve/reject KYC)

**Phase 2 — Driver app (PWA)**
- Driver login + start-shift selfie check
- Start trip → background location streaming → `vehicle_pings`
- End trip summary

**Phase 3 — Passenger live tracking**
- Choose route/bus → live map with marker
- ETA to my stop, stop list with live status
- Stop-arrived push notification

**Phase 4 — Alerts engine**
- Triggers: over-speed, off-route, long-stop, stop-arrived
- Notifier server fn (Web Push + email; SMS optional via Twilio secret)

**Phase 5 — Safety**
- SOS button (audio + location)
- Incidents admin console (acknowledge, resolve, export)
- Emergency contacts per user

**Phase 6 — Parent linking + history**
- Parent ↔ child link flow
- Trip history page + CSV/PDF export

**Phase 7 — Carpool**
- Offer ride, search, book, in-app chat, shared live track, rating

**Phase 8 — Public multi-bus map**
- Public route at `/live` showing all active public-transit vehicles, filter by line

**Phase 9 — PWA polish + APK packaging**
- Service worker, install prompt, offline cache, queued SOS
- Capacitor wrap → `.apk` build instructions (run locally, not inside Lovable)

**Phase 10 — Hardening**
- Rate limits, abuse protection on SOS, audit logs, security scan, load testing notes

---

## 8. Integrations & Secrets

- **Google Maps Platform** — managed connector (maps, routing, geocoding, ETA)
- **Lovable Email** — transactional alerts
- **Web Push** — VAPID keys (we generate)
- **Twilio (optional)** — SMS for SOS; you'd provide `TWILIO_*` secrets
- **Capacitor** — for APK packaging (done on your machine, I'll provide exact commands)

---

## 9. What I Need From You Before Phase 0

1. **Primary use case first**: school buses, office transport, **or** public transit? (We build all eventually; pick what Phase 1–4 demos.)
2. **Country / region** (for map defaults, emergency numbers, language).
3. **Org name & branding** (logo, color preference) — or I pick a clean default and you change later.
4. **SMS via Twilio now or later?** (Push + email always included.)

Reply with answers (or "you choose") and I'll start Phase 0.

---

## 10. Realistic Expectations

- This is a large system — expect ~10 build turns to reach a complete v1, plus iteration.
- The Lovable preview will be a fully working web app at every phase.
- The Android `.apk` is produced by wrapping the final PWA with Capacitor on your computer — I'll give you the exact commands; it cannot be built inside Lovable's sandbox.
- iOS `.ipa` requires a Mac + Apple Developer account — same Capacitor codebase.
