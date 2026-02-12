# MVP Acceptance Criteria by Sprint

This document defines objective acceptance criteria for each MVP sprint and traces every criterion to an explicit feature/task in `PROJECT.txt`.

## Traceability Map (PROJECT.txt -> Acceptance IDs)

- **P-S1-A**: Sprint 1 goal: empty room, furniture placement, persistence.
- **P-S1-T1**: Setup Colyseus + Node.js.
- **P-S1-T2**: Setup Phaser Grid System.
- **P-S1-T3**: Drag & Drop мебели.
- **P-S1-T4**: Сохранение координат в Postgres.
- **P-S2-A**: Sprint 2 goal: planting/harvest loop.
- **P-S2-T1**: Server-side growth timer logic.
- **P-S2-T2**: Seed and harvest inventory.
- **P-S2-T3**: Coin shop (soft currency).
- **P-S3-A**: Sprint 3 goal: pets and visits.
- **P-S3-T1**: Incubator (gacha mechanic).
- **P-S3-T2**: Neighbors panel from Discord Voice API.
- **P-S3-T3**: Join friend room.
- **P-S4-A**: Sprint 4 goal: release readiness.
- **P-S4-T1**: Discord IAP integration.
- **P-S4-T2**: Sounds and UI animations.
- **P-S4-T3**: WebSocket load testing.
- **P-3.1**: Overripe harvest algorithm (`HARVEST_REQUEST`, authoritative timestamp check).
- **P-3.3**: Colyseus room model (`HouseRoom`, join by room id, sync to all clients).
- **P-4.2**: Optimistic UI with rollback on server error.
- **P-5.1**: Hardcoded server-side pricing/timing configs.
- **P-5.2**: Three MVP SKUs.

---

## Sprint 1 — The Empty Room

### Functional Acceptance Scenarios (Given/When/Then)

1. **S1-F1: Owner can enter their room and load an empty grid** *(maps: P-S1-A, P-S1-T1, P-S1-T2, P-3.3)*
   - **Given** an authenticated player with no placed `House_Items`
   - **When** the player joins their `HouseRoom`
   - **Then** the client renders the base room grid, and room state contains zero placed furniture objects.

2. **S1-F2: Player can drag and place furniture on valid tiles** *(maps: P-S1-T2, P-S1-T3)*
   - **Given** a player has at least one furniture item in inventory
   - **When** the item is dragged and dropped onto a valid grid coordinate
   - **Then** the item snaps to the target tile, is visible in Phaser, and local HUD reflects placement success.

3. **S1-F3: Placement is rejected on invalid or occupied tile** *(maps: P-S1-T1, P-S1-T2, P-S1-T3)*
   - **Given** a target tile is out of bounds or occupied
   - **When** the player attempts drop
   - **Then** server rejects the action, client reverts preview state, and item remains unplaced.

4. **S1-F4: Furniture coordinates persist across reconnect** *(maps: P-S1-T4)*
   - **Given** a player has successfully placed furniture
   - **When** the session disconnects and reconnects
   - **Then** furniture is loaded from PostgreSQL with the same `grid_x`/`grid_y`.

### Non-Functional Targets

- **Action latency (place/move acknowledgement):** p95 <= **250 ms** from client send to authoritative server response in normal regional test env (maps: P-S1-T1).
- **Initial room load latency:** p95 <= **1200 ms** to first interactive frame after join (maps: P-S1-A, P-S1-T1).
- **Basic room concurrency goal:** support **1 owner + 4 guests** in one `HouseRoom` with state sync tick success >= 99% (maps: P-3.3).

### Minimal Automated Test Categories Required

- **Unit**
  - Grid coordinate validity and tile occupancy checks.
  - Cartesian-to-isometric conversion helpers.
- **Integration**
  - Colyseus join + place furniture + persistence roundtrip to PostgreSQL.
  - Reconnect flow restores placed item positions.
- **Protocol contract**
  - Client->server message schema for place/move actions.
  - Server->client rejection payload contract for invalid placement.

### Release Gate Checklist (Objective Pass Conditions)

- [ ] All S1 functional scenarios S1-F1..S1-F4 pass in CI and in one manual end-to-end run.
- [ ] Automated tests exist in all three categories (unit/integration/protocol contract) with no failing tests.
- [ ] Measured p95 action latency <= 250 ms on >= 500 placement actions.
- [ ] Measured room-load p95 <= 1200 ms across >= 100 joins.
- [ ] Concurrency run with 5 simultaneous clients in one room shows no desync causing divergent furniture coordinates.

---

## Sprint 2 — The Gardener

### Functional Acceptance Scenarios (Given/When/Then)

1. **S2-F1: Planting consumes seed and creates timed tile state** *(maps: P-S2-A, P-S2-T1, P-S2-T2, P-3.1)*
   - **Given** player inventory contains `seed_mint`
   - **When** player plants on an empty farm tile
   - **Then** server stores planted state with timestamp and decrements seed inventory by 1.

2. **S2-F2: Early harvest is rejected by authoritative server check** *(maps: P-S2-T1, P-3.1)*
   - **Given** a planted tile where `now < readyTime`
   - **When** player sends `HARVEST_REQUEST`
   - **Then** server returns `ERROR_NOT_READY`, no loot is granted, tile remains planted.

3. **S2-F3: Ready harvest grants loot and resets tile** *(maps: P-S2-A, P-S2-T1, P-S2-T2, P-3.1)*
   - **Given** a planted tile where `now >= readyTime`
   - **When** player sends `HARVEST_REQUEST`
   - **Then** server grants configured loot, resets tile, and updates inventory/currency.

4. **S2-F4: Coin shop purchase uses server-owned prices** *(maps: P-S2-T3, P-5.1)*
   - **Given** player has sufficient coins
   - **When** player buys a seed from shop
   - **Then** server validates against hardcoded config, deducts correct coin amount, and adds item.

5. **S2-F5: Optimistic harvest rollback on server rejection** *(maps: P-4.2, P-S2-T1)*
   - **Given** client starts optimistic harvest animation
   - **When** server returns `ERROR_NOT_READY`
   - **Then** client rolls back visual state and crop remains visible.

### Non-Functional Targets

- **Action latency (plant/harvest/shop ack):** p95 <= **300 ms** (maps: P-S2-T1, P-S2-T3).
- **Timer correctness tolerance:** max absolute error <= **1 second** between expected and actual readiness in tests (maps: P-3.1).
- **Basic room concurrency goal:** **1 owner + 6 guests** can observe harvest state updates within **500 ms** p95 fan-out delay (maps: P-3.3).

### Minimal Automated Test Categories Required

- **Unit**
  - Ready-time calculation and branch behavior (`>= readyTime` vs `< readyTime`).
  - Inventory debit/credit primitives.
- **Integration**
  - Plant -> wait/simulate time -> harvest lifecycle persisted in DB.
  - Shop purchase uses server config and rejects tampered client price.
- **Protocol contract**
  - `HARVEST_REQUEST` request/response codes (`SUCCESS`, `ERROR_NOT_READY`).
  - Shop purchase message schema and deterministic error contract.

### Release Gate Checklist (Objective Pass Conditions)

- [ ] S2 scenarios S2-F1..S2-F5 pass in CI and one scripted E2E flow.
- [ ] Test suites for unit/integration/protocol contract all present and green.
- [ ] Timer correctness tests demonstrate <= 1 second tolerance across >= 50 samples.
- [ ] p95 action latency <= 300 ms across >= 1000 combined plant/harvest/shop actions.
- [ ] Concurrency check with 7 clients verifies all observers converge on same tile state after each harvest.

---

## Sprint 3 — Life & Friends

### Functional Acceptance Scenarios (Given/When/Then)

1. **S3-F1: Incubator starts hatch timer** *(maps: P-S3-A, P-S3-T1, P-3.2)*
   - **Given** player owns an egg item
   - **When** player places egg into incubator tile
   - **Then** server stores `hatch_start` and `hatch_duration` for that egg.

2. **S3-F2: Hatch completion creates a pet and updates active roster** *(maps: P-S3-T1)*
   - **Given** an incubating egg where elapsed time >= `hatch_duration`
   - **When** player claims hatch result
   - **Then** server creates pet record and egg is removed from incubator state.

3. **S3-F3: Neighbors panel reflects Discord voice participants** *(maps: P-S3-T2, P-1.2)*
   - **Given** Discord voice state includes N participants in channel
   - **When** app receives voice state update
   - **Then** Neighbors panel shows the same participant list (excluding current user if configured).

4. **S3-F4: Visitor can join friend room and observe synchronized state** *(maps: P-S3-T3, P-3.3)*
   - **Given** friend room id is known and owner room is active
   - **When** visitor joins by room id
   - **Then** visitor sees owner world state and movement/tile updates broadcast consistently.

### Non-Functional Targets

- **Action latency (hatch claim / room join):** p95 <= **350 ms** server acknowledgement (maps: P-S3-T1, P-S3-T3).
- **Neighbors panel freshness:** voice participant list refresh <= **2 seconds** after Discord state update in test harness (maps: P-S3-T2).
- **Basic room concurrency goal:** **1 owner + 10 guests** can remain connected for 15 minutes with disconnect error rate < **1%** (maps: P-3.3).

### Minimal Automated Test Categories Required

- **Unit**
  - Hatch timer completion logic and gem speed-up timestamp rewrite behavior.
  - Neighbors list mapping from Discord payload to UI model.
- **Integration**
  - End-to-end egg incubation to pet record persistence.
  - Visitor join and state sync with owner in same Colyseus room.
- **Protocol contract**
  - Friend-room join request/response schema.
  - Incubator claim/hatch event payload contract.

### Release Gate Checklist (Objective Pass Conditions)

- [ ] S3 scenarios S3-F1..S3-F4 pass in CI and manual multiplayer smoke test.
- [ ] Unit/integration/protocol contract suites are present and green.
- [ ] Neighbors panel freshness <= 2 seconds in >= 95% of sampled updates.
- [ ] Concurrency soak with 11 clients for 15 minutes meets disconnect error rate < 1%.
- [ ] No state divergence detected between owner and visitors for movement and tile changes.

---

## Sprint 4 — Monetization & Polish

### Functional Acceptance Scenarios (Given/When/Then)

1. **S4-F1: IAP flow opens Discord premium purchase path for configured SKU** *(maps: P-S4-A, P-S4-T1, P-1.2, P-5.2)*
   - **Given** player selects one of MVP SKUs
   - **When** player taps purchase
   - **Then** app calls Discord premium purchase command and receives success/cancel callback.

2. **S4-F2: Successful purchase credits correct rewards exactly once** *(maps: P-S4-T1, P-5.2, P-5.1)*
   - **Given** purchase callback indicates success for a valid SKU
   - **When** backend validates transaction
   - **Then** configured rewards (e.g., gems/pet) are granted once and duplicate callback does not double-credit.

3. **S4-F3: Core interactions include sound and UI animation feedback** *(maps: P-S4-T2, P-4.2)*
   - **Given** player performs place/harvest/purchase actions
   - **When** action is accepted (or optimistically started)
   - **Then** expected sound and UI animation cues are triggered.

4. **S4-F4: System sustains websocket load target without gameplay corruption** *(maps: P-S4-T3, P-3.3)*
   - **Given** load test reaches target concurrent room sessions
   - **When** synthetic gameplay traffic runs for fixed duration
   - **Then** server remains within error and latency budgets, and no critical data corruption occurs.

### Non-Functional Targets

- **Action latency (purchase ack / reward application):** p95 <= **400 ms** excluding external store confirmation time (maps: P-S4-T1).
- **Audio/UI feedback trigger delay:** <= **100 ms** from local action to cue start (maps: P-S4-T2).
- **Basic room concurrency goal:** at least **100 concurrent active rooms** (mixed owner/guest traffic) during load test with server error rate < **1%** (maps: P-S4-T3, P-3.3).

### Minimal Automated Test Categories Required

- **Unit**
  - SKU-to-reward mapping and idempotency checks for transaction handling.
  - UI event-to-feedback trigger wiring logic.
- **Integration**
  - Simulated successful/cancelled IAP callback handling.
  - Reward credit persistence and duplicate callback protection.
- **Protocol contract**
  - Purchase intent and callback payload schema validation.
  - Load-test websocket message envelope compatibility checks.

### Release Gate Checklist (Objective Pass Conditions)

- [ ] S4 scenarios S4-F1..S4-F4 pass in CI and release candidate smoke test.
- [ ] All required test categories exist and pass with zero known critical defects.
- [ ] SKU reward checks pass for all 3 configured MVP SKUs.
- [ ] Load test proves >= 100 active rooms with error rate < 1% and no Sev-1 crashes.
- [ ] Final regression confirms no double-credit on duplicate purchase callbacks.

---

## Cross-Sprint Exit Rule for MVP Release

MVP is releasable only if every sprint gate is fully checked and every criterion above has explicit evidence attached (test report, metric snapshot, or run log) with traceability back to `PROJECT.txt` IDs.
