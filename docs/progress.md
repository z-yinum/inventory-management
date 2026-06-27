# Project Progress & Next Steps

## Session Summary — Branch: `new_features`

---

## Completed

### Infrastructure
- [x] Started frontend (Vite, port 3000) and backend (FastAPI, port 8001)
- [x] Confirmed `uv` at `/c/Users/Yineth/.local/bin/uv` and Node.js at `C:\Program Files\nodejs`

### Architecture Reference
- [x] Created `docs/architecture.html` — visual system architecture page (open in browser via `cmd.exe /c start docs\architecture.html`)
  - Three-layer diagram: Frontend → Backend → Data
  - Tech stack with versions, all 19 API endpoints, filter system, data flow, file structure

### Restocking Tab (full feature)
- [x] **`server/main.py`** — 3 new endpoints + Pydantic models + in-memory store:
  - `GET /api/restocking/recommendations` — cross-references demand gaps with inventory unit costs
  - `POST /api/restocking/orders` — creates order with 14-day lead time, stores in memory
  - `GET /api/restocking/orders` — returns submitted orders
- [x] **`client/src/api.js`** — 3 new API functions: `getRestockingRecommendations`, `createRestockingOrder`, `getRestockingOrders`
- [x] **`client/src/views/Restocking.vue`** — new view:
  - Budget slider (auto-initialized to full potential cost)
  - Greedy auto-selection: items checked in priority order while within budget
  - Manual checkbox override per row
  - Progress bar showing budget consumed
  - Place Order button with success state showing order number
- [x] **`client/src/App.vue`** — "Restocking" nav tab added between Demand and Reports
- [x] **`client/src/main.js`** — `/restocking` route registered
- [x] **`client/src/views/Orders.vue`** — "Submitted Restocking Orders" section added below main table (purple badge, bold expected delivery, shown only when orders exist)

---

## Pending / Blocked

### Backend Restart Required
The backend server (PID 42708) was started before the new endpoints were added and cannot be killed from the Claude Code shell. **The new `/api/restocking/*` endpoints will return 404 until the server is restarted.**

**Action needed — run in PowerShell:**
```powershell
Stop-Process -Id (Get-NetTCPConnection -LocalPort 8001).OwningProcess -Force
cd c:\Projects\Anthropic\inventory-management\server
uv run python main.py
```

### End-to-End Verification (after restart)
- [ ] Visit `http://localhost:3000/restocking` — table loads with priced recommendations
- [ ] Move budget slider left — items auto-deselect, running total updates
- [ ] Manually check/uncheck individual rows
- [ ] Click "Place Order" — success message shows order number (e.g. `RST-2025-0623-0001`)
- [ ] Navigate to `/orders` — "Submitted Restocking Orders" section appears with 14-day delivery date
- [ ] Submit a second order — both appear in the submitted section
- [ ] Verify `GET http://localhost:8001/api/restocking/recommendations` returns items with `unit_cost` and `total_cost`

### Open Items / Potential Next Steps
- [ ] **Commit & PR** — changes are on branch `new_features`, not yet committed
- [ ] **i18n** — Restocking view uses hardcoded English strings; Japanese translations not added
- [ ] **Filter integration** — Restocking currently ignores global filters (warehouse/category); could scope recommendations to selected warehouse
- [ ] **Quantity editing** — users can only accept the auto-calculated gap quantity; no way to adjust per-item quantity before ordering
- [ ] **Persistence** — submitted orders reset on server restart; a real implementation would write to a database or JSON file
- [ ] **Tests** — no backend tests written for the 3 new endpoints (`tests/backend/`)

---

## Key Files Changed This Session

| File | Change |
|------|--------|
| `server/main.py` | +3 endpoints, +2 Pydantic models, +in-memory store |
| `client/src/api.js` | +3 API functions |
| `client/src/views/Restocking.vue` | New file (created) |
| `client/src/views/Orders.vue` | Added Submitted Orders section |
| `client/src/App.vue` | Added Restocking nav tab |
| `client/src/main.js` | Added /restocking route |
| `docs/architecture.html` | New file (created) |
| `docs/progress.md` | This file |
| `CLAUDE.md` | Updated with new endpoints and file locations |
