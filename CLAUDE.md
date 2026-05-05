# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

OpenCam is a self-hosted home security camera platform for ESP32-S3 devices. Three independent components live in this monorepo:

| Directory | Language | Purpose |
|---|---|---|
| `frontend-iot/` | TypeScript / Next.js 16 | Web dashboard |
| `backend-iot/` | JavaScript / Node.js | Camera stream server |
| `esp32-firmware/ESP32/` | C (ESP-IDF) | Device firmware |
| `old-frontend/` | TypeScript / Next.js | **Deprecated** — ignore |

---

## frontend-iot (Next.js dashboard)

### Commands

```bash
cd frontend-iot
npm install
docker compose up -d              # start PostgreSQL on port 5433
npx drizzle-kit push              # apply schema to database
npm run dev                       # dev server at http://localhost:3000
npm run build
npm run typecheck
npm run lint
npm run format                    # Prettier
npx drizzle-kit studio            # visual DB browser
```

### Environment

Copy `.env` (or `env.example`) and set `DATABASE_URL`. The Docker Compose PostgreSQL runs on port 5433 (not 5432) to avoid conflicts.

### Architecture

- **App Router** under `app/` — no `src/` directory.
- **Route hierarchy** mirrors the data model: `/dashboard/[groupId]/[houseId]/[roomId]`.
- **Database layer** lives entirely in `drizzle/`:
  - `drizzle/schema.ts` — all table definitions (`User`, `Session`, `Group`, `GroupMember`, `House`, `Room`, `Camera`).
  - `drizzle/actions/` — one file per entity, containing all Drizzle queries. Import these from server components and Server Actions — never query the DB directly in components.
- **Auth** is fully custom (not NextAuth, despite the package being present):
  - Sessions use a `sessionId.sessionSecret` cookie format.
  - The secret is SHA-256 hashed for storage; comparison uses `timingSafeEqual`.
  - `lib/session.ts` — `getCurrentSession()` for server components (React `cache`-deduped).
  - `proxy.ts` — **this is the Next.js middleware file** (named `proxy.ts`, re-exported from `middleware.ts`). It guards `/dashboard` routes and redirects `/auth/*` when already authenticated.
- **UI** uses shadcn/ui components in `components/` and Tailwind CSS v4.
- `ADMIN_EMAIL` in `.env` controls which account gets admin-level access.

---

## backend-iot (Stream server)

### Commands

```bash
cd backend-iot
npm install
node server.js       # or: npm start
docker compose up --build
```

### Architecture

The server runs three listeners simultaneously:

| Port | Protocol | Clients |
|---|---|---|
| 7890 | HTTP + WebSocket | Browsers (live JPEG stream via WS, motion alerts via SSE) |
| 7891 | Plain TCP | ESP32 devices (unencrypted) |
| 7893 | TLS TCP (mTLS) | ESP32 devices (mutual TLS) |

**ESP32 ↔ server protocol** (TCP/TLS, port 7891 or 7893):

1. On connect, server sends `I` → ESP32 replies with its MAC address + `\n`.
2. After identification, server drives the ESP32 with single-byte commands:
   - `G` — grab one JPEG frame (~2 fps while clients watch)
   - `P` — keepalive ping (every 5 s when no clients)
   - `S` / `M` / `N` — stop feed / motion on / motion off
3. Each frame arrives as `[4-byte little-endian size][JPEG bytes]`.

**Browser ↔ server protocol:**
- WebSocket: `ws://<host>:7890?device=<deviceId>` → receives raw JPEG buffers.
- SSE: `GET /motion?device=<deviceId>` → `{ deviceId, changedPercent, ts }` events (triggered when >35% of pixels change by >25 brightness levels).
- HTTP: `GET /devices` → `{ devices: [...] }`.

Motion detection is done server-side using `sharp` (frame diffing, no disk writes).

**TLS certs** expected at `backend-iot/certs/server.key`, `server.crt`, `ca.crt` (override via `TLS_KEY_PATH`, `TLS_CERT_PATH`, `TLS_CA_PATH` env vars). If missing, the TLS server is silently skipped and only plain TCP runs.

---

## esp32-firmware (ESP-IDF, C)

**Target hardware:** ESP32-S3-WROOM N16R8.

### Build & flash

Use the ESP-IDF toolchain (`idf.py`). Compiled binaries are in `ESP32/build/`.

```bash
cd esp32-firmware/ESP32
idf.py build
idf.py flash monitor
```

### Architecture

`main/main.c` contains `app_main()`, which runs the full lifecycle:

1. **NVS + SPIFFS init** — persistent storage for Wi-Fi credentials (`/spiffs/wf`).
2. **Wi-Fi setup** — reads cached credentials from SPIFFS; if absent or failed, enters SoftAP mode and serves a provisioning web page (via `soft_ap_sub.c` / `http_website.c`) until the user submits credentials.
3. **SNTP sync** — waits until year ≥ 2026 before proceeding.
4. **TLS connect** — connects to the backend at `SERVER_IP:7893` (hardcoded in `main.c`) using mTLS. Client cert/key live in `main/certs_com/`.
5. **Main loop** — captures JPEG via `esp_camera_fb_get()`, reads a command byte from the server, then sends `[4-byte LE size][JPEG]`.

Key source files:
- `main/camera.c/h` — camera init and frame capture.
- `main/tls.c/h` — TLS connection setup using `esp-tls`.
- `main/station_wifi.c/h` — STA mode Wi-Fi.
- `main/soft_ap_sub.c/h` + `http_website.c/h` — SoftAP provisioning web page.
- `main/led.c/h` — RGB LED status indicator.
- `main/utilities/factory_reset.c/h` — button-triggered NVS erase.
- `main/certs_com/` — `ca_cert.crt`, `esp32_cert.crt`, `esp32_private.key` (mTLS identity).

### Device provisioning

Wi-Fi credentials are stored in SPIFFS at `/spiffs/wf` as raw bytes: `[ssid_len][pass_len][ssid][password]`. On first boot (or after factory reset), the device broadcasts a SoftAP (`myssid` / `Mypassword` by default during development) and serves an HTTP provisioning page.

---

## Planned / future work

`possible-tech-flow.md` describes a planned device onboarding spec (per-device ECDSA keypairs, claim tokens, X.509 certificate issuance). This is **not yet implemented** in any of the three components — treat it as design documentation only.
