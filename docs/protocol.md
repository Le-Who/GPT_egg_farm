# MVP Realtime Protocol Specification

**Status:** Draft for MVP implementation  
**Transport:** Colyseus/WebSocket messages with JSON payloads  
**Authority:** Server-authoritative for all world/economy mutations

## 1) Envelope and common fields

All client requests MUST use this envelope:

```json
{
  "event": "string",
  "requestId": "uuid-v7",
  "sentAt": 1710000000000,
  "sessionId": "uuid",
  "protocolVersion": "0.1.0",
  "payload": {}
}
```

### Common validation constraints

- `event`: required string, MUST match one of the event names below.
- `requestId`: required UUID, unique per client mutation request.
- `sentAt`: required unix epoch ms; server accepts within ±120s skew.
- `sessionId`: required after auth/session init.
- `protocolVersion`: required semver-like string.
- `payload`: required object matching event schema.

All server replies MUST use this envelope:

```json
{
  "event": "string",
  "requestId": "uuid-v7",
  "ok": true,
  "serverTime": 1710000000000,
  "payload": {}
}
```

Error reply shape:

```json
{
  "event": "error",
  "requestId": "uuid-v7",
  "ok": false,
  "serverTime": 1710000000000,
  "error": {
    "code": "ERROR_CODE",
    "message": "short diagnostic",
    "retryable": true,
    "details": {}
  }
}
```

## 2) Client -> Server messages

---

### 2.1 `session.init`

Initial authentication/session bootstrap.

**Request payload schema**

```json
{
  "discordAuthToken": "string",
  "discordUserId": "string",
  "deviceId": "string",
  "resumeSessionId": "string|null"
}
```

**Validation constraints**

- `discordAuthToken`: required, non-empty.
- `discordUserId`: required, non-empty.
- `deviceId`: required, non-empty.
- `resumeSessionId`: optional; if provided must be UUID.

**Success response schema (`session.init.ok`)**

```json
{
  "sessionId": "uuid",
  "player": {
    "userId": "uuid",
    "coins": 1000,
    "gems": 10,
    "xp": 0
  },
  "room": {
    "roomId": "string",
    "ownerUserId": "uuid"
  },
  "worldSnapshot": {
    "items": [],
    "tiles": [],
    "pets": []
  },
  "protocolVersion": "0.1.0",
  "serverFeatures": ["hatch_speed_up", "friend_rooms"]
}
```

**Recoverable errors**

- `ERROR_AUTH_INVALID` -> Show login-expired UI and force re-auth.
- `ERROR_AUTH_EXPIRED` -> Retry once with fresh token, then return to bootstrap.
- `ERROR_PROTOCOL_VERSION_UNSUPPORTED` -> Prompt user to refresh/reload client.
- `ERROR_RATE_LIMITED` -> Backoff 1-5s and retry.

**Idempotency / anti-replay**

- `session.init` is idempotent for same valid `resumeSessionId` + `deviceId` within 5 minutes.
- Server binds `sessionId` to `deviceId` and rotates on suspicious reuse.

---

### 2.2 `furniture.place`

Place owned furniture into world grid.

**Request payload schema**

```json
{
  "itemInstanceId": "uuid",
  "itemId": "string",
  "gridX": 0,
  "gridY": 0,
  "rotation": 0,
  "clientRevision": 1
}
```

**Validation constraints**

- `itemInstanceId`: required UUID owned by session user and currently unplaced.
- `gridX`, `gridY`: required int; must be inside room bounds.
- `rotation`: enum `[0, 90, 180, 270]`.
- Tile footprint must not collide with blocked/occupied tiles.

**Success response schema (`furniture.place.ok`)**

```json
{
  "placement": {
    "itemInstanceId": "uuid",
    "itemId": "string",
    "gridX": 0,
    "gridY": 0,
    "rotation": 0
  },
  "newRevision": 2,
  "statePatch": {
    "placedItemsUpsert": []
  }
}
```

**Recoverable errors**

- `ERROR_ITEM_NOT_OWNED` -> Refresh inventory and display ownership warning.
- `ERROR_INVALID_TILE` -> Snap item back to inventory and show invalid placement toast.
- `ERROR_TILE_OCCUPIED` -> Keep drag mode active and highlight conflicting tile.
- `ERROR_REVISION_CONFLICT` -> Pull latest room state and re-apply locally.

**Idempotency / anti-replay**

- Idempotent by `requestId` for 60s.
- Duplicate accepted request returns same `placement` and `newRevision`.

---

### 2.3 `furniture.move`

Move already placed furniture.

**Request payload schema**

```json
{
  "itemInstanceId": "uuid",
  "toGridX": 0,
  "toGridY": 0,
  "rotation": 90,
  "clientRevision": 2
}
```

**Validation constraints**

- `itemInstanceId`: required UUID of already placed item owned by room owner.
- Destination must be in bounds and non-colliding.
- `clientRevision` required for optimistic concurrency.

**Success response schema (`furniture.move.ok`)**

```json
{
  "placement": {
    "itemInstanceId": "uuid",
    "gridX": 1,
    "gridY": 2,
    "rotation": 90
  },
  "newRevision": 3,
  "statePatch": {
    "placedItemsUpsert": []
  }
}
```

**Recoverable errors**

- `ERROR_ITEM_NOT_FOUND` -> Force room sync and redraw item map.
- `ERROR_INVALID_TILE` -> Revert local drag ghost to server location.
- `ERROR_TILE_OCCUPIED` -> Keep item selected; prompt user to choose another tile.
- `ERROR_REVISION_CONFLICT` -> Re-sync and retry user action once.

**Idempotency / anti-replay**

- Idempotent by `requestId` for 60s.
- Latest-write-wins only when `clientRevision` matches authoritative revision.

---

### 2.4 `seed.plant`

Consume seed and start crop timer.

**Request payload schema**

```json
{
  "tileId": "string",
  "seedItemId": "seed_mint",
  "clientNonce": "uuid"
}
```

**Validation constraints**

- `tileId`: required, must reference plantable tile in current room.
- `seedItemId`: required, must be a valid seed SKU in server config.
- User must have seed quantity >= 1.
- Tile must be empty/not growing.

**Success response schema (`seed.plant.ok`)**

```json
{
  "tile": {
    "tileId": "string",
    "seedItemId": "seed_mint",
    "plantedAt": 1710000000000,
    "readyAt": 1710000120000,
    "growthDurationMs": 120000
  },
  "inventoryDelta": {
    "seed_mint": -1
  },
  "statePatch": {
    "tilesUpsert": []
  }
}
```

**Recoverable errors**

- `ERROR_INVALID_TILE` -> Show invalid tile hint, keep seed selected.
- `ERROR_TILE_OCCUPIED` -> Show "already planted" and clear optimistic seed spend.
- `ERROR_ITEM_NOT_OWNED` -> Refresh inventory and disable unavailable seed.
- `ERROR_INSUFFICIENT_FUNDS` -> If seed requires purchase, open shop panel fallback.

**Idempotency / anti-replay**

- Economically sensitive (consumes inventory):
  - Require unique `clientNonce` + `requestId` pair.
  - Server stores dedupe key for 10 minutes.
  - Replays return original success/failure without double-spend.

---

### 2.5 `crop.harvest`

Harvest ready crop from tile.

**Request payload schema**

```json
{
  "tileId": "string",
  "clientNonce": "uuid"
}
```

**Validation constraints**

- `tileId`: required and must contain active crop state.
- `now >= readyAt` on authoritative server clock.

**Success response schema (`crop.harvest.ok`)**

```json
{
  "tile": {
    "tileId": "string",
    "cleared": true
  },
  "rewards": {
    "coins": 25,
    "items": [
      { "itemId": "mint_leaf", "qty": 1 }
    ]
  },
  "wallet": {
    "coins": 1025,
    "gems": 10
  },
  "statePatch": {
    "tilesUpsert": []
  }
}
```

**Recoverable errors**

- `ERROR_NOT_READY` -> Keep crop visible and show remaining timer.
- `ERROR_INVALID_TILE` -> Request tile state refresh.
- `ERROR_ALREADY_PROCESSED` -> Accept as eventual success and sync wallet/tile.

**Idempotency / anti-replay**

- Economically sensitive (grants loot):
  - Unique `clientNonce` and `requestId` required.
  - Server keeps dedupe record for 24h.
  - Duplicate requests return same `rewards` snapshot; never mint twice.

---

### 2.6 `hatch.start`

Put egg into incubator and start hatch timer.

**Request payload schema**

```json
{
  "incubatorId": "string",
  "eggItemInstanceId": "uuid",
  "clientNonce": "uuid"
}
```

**Validation constraints**

- Incubator exists and is currently idle.
- Egg instance exists in user inventory and is hatchable.

**Success response schema (`hatch.start.ok`)**

```json
{
  "hatch": {
    "incubatorId": "string",
    "eggItemInstanceId": "uuid",
    "hatchStart": 1710000000000,
    "hatchDurationMs": 120000,
    "hatchReadyAt": 1710000120000
  },
  "inventoryDelta": {
    "egg_basic": -1
  },
  "statePatch": {
    "incubatorsUpsert": []
  }
}
```

**Recoverable errors**

- `ERROR_NOT_READY` -> Incubator busy; display remaining hatch time.
- `ERROR_ITEM_NOT_OWNED` -> Refresh inventory and clear selected egg.
- `ERROR_INVALID_TILE` -> If incubator placement invalidated, force room resync.

**Idempotency / anti-replay**

- Economically sensitive (consumes egg): dedupe by `clientNonce` for 10 minutes.
- Duplicate request returns original hatch record.

---

### 2.7 `hatch.speed_up`

Spend gems to instantly complete active hatch.

**Request payload schema**

```json
{
  "incubatorId": "string",
  "speedUpType": "instant_finish",
  "clientNonce": "uuid"
}
```

**Validation constraints**

- Incubator has active hatch and not already complete.
- `speedUpType` currently only `instant_finish`.
- User gems balance >= configured speed-up cost.

**Success response schema (`hatch.speed_up.ok`)**

```json
{
  "hatch": {
    "incubatorId": "string",
    "completed": true,
    "hatchReadyAt": 1710000000000
  },
  "wallet": {
    "gems": 7
  },
  "cost": {
    "gems": 3
  },
  "statePatch": {
    "incubatorsUpsert": []
  }
}
```

**Recoverable errors**

- `ERROR_NOT_READY` -> If no active hatch, disable speed-up CTA.
- `ERROR_INSUFFICIENT_FUNDS` -> Open gem purchase flow / store panel.
- `ERROR_ALREADY_PROCESSED` -> Treat as success and sync incubator/wallet.

**Idempotency / anti-replay**

- High-risk economic action:
  - Mandatory `clientNonce` + `requestId`.
  - Server verifies monotonic wallet transaction index.
  - Dedupe window 24h.
  - Replays MUST NOT apply gem deduction twice.

---

### 2.8 `room.join_friend`

Join another player's room for visit/multiplayer presence.

**Request payload schema**

```json
{
  "friendRoomId": "string",
  "friendUserId": "uuid"
}
```

**Validation constraints**

- `friendRoomId` required.
- Caller must have permission to visit (friendship/privacy checks).

**Success response schema (`room.join_friend.ok`)**

```json
{
  "room": {
    "roomId": "string",
    "ownerUserId": "uuid",
    "joinedAsGuest": true
  },
  "worldSnapshot": {
    "items": [],
    "tiles": [],
    "pets": []
  },
  "presence": {
    "occupants": []
  }
}
```

**Recoverable errors**

- `ERROR_ROOM_NOT_FOUND` -> Show "friend offline" and return to own room.
- `ERROR_FORBIDDEN` -> Show privacy message; disable join button.
- `ERROR_RATE_LIMITED` -> Backoff and retry.

**Idempotency / anti-replay**

- Safe, non-economic action; standard requestId dedupe for 30s.

## 3) Server -> Client push messages

These are unsolicited or fan-out updates delivered after mutations or periodic sync.

### 3.1 `state.patch`

Incremental state updates.

**Payload schema**

```json
{
  "roomId": "string",
  "revision": 4,
  "patch": {
    "placedItemsUpsert": [],
    "tilesUpsert": [],
    "incubatorsUpsert": [],
    "inventoryDelta": {},
    "wallet": { "coins": 0, "gems": 0 }
  },
  "reason": "furniture.move|seed.plant|crop.harvest|hatch.*"
}
```

**Validation constraints**

- `revision` MUST be strictly increasing per room.
- Unknown fields ignored for forward compatibility.

### 3.2 `action.success`

Canonical success event for generic handlers when event-specific `*.ok` is not emitted.

**Payload schema**

```json
{
  "action": "string",
  "result": {},
  "statePatch": {}
}
```

### 3.3 `error` (recoverable)

Canonical error for all recoverable failures.

**Payload schema**

```json
{
  "code": "ERROR_CODE",
  "message": "string",
  "retryable": true,
  "userAction": "string",
  "details": {}
}
```

`userAction` recommended values:
- `RETRY_SILENT`
- `RETRY_WITH_BACKOFF`
- `REFRESH_STATE`
- `OPEN_SHOP`
- `REAUTH`
- `RETURN_HOME`

## 4) Canonical recoverable error codes

| Code | Typical triggers | Fallback behavior |
|---|---|---|
| `ERROR_NOT_READY` | Harvest/hatch called before ready time, incubator busy | Show timer/remaining time and keep current object state visible. |
| `ERROR_INVALID_TILE` | Out-of-bounds tile, not plantable, invalid footprint | Revert optimistic local action and highlight valid tiles. |
| `ERROR_TILE_OCCUPIED` | Placement/planting collision | Keep placement mode active and prompt reposition. |
| `ERROR_INSUFFICIENT_FUNDS` | Not enough coins/gems for action | Open shop/purchase UI with required amount context. |
| `ERROR_ITEM_NOT_OWNED` | Missing inventory instance/seed/egg | Refresh inventory, disable invalid action entry points. |
| `ERROR_ITEM_NOT_FOUND` | Stale local item reference | Pull latest room snapshot and retry once. |
| `ERROR_ROOM_NOT_FOUND` | Friend room unavailable/offline | Show offline state and navigate back to own room. |
| `ERROR_FORBIDDEN` | Privacy/friendship restriction | Show permission message and disable join path. |
| `ERROR_REVISION_CONFLICT` | Stale optimistic revision | Request full state sync and re-apply pending local intent. |
| `ERROR_ALREADY_PROCESSED` | Duplicate economic request | Treat as idempotent success, sync from authoritative payload. |
| `ERROR_RATE_LIMITED` | Too many requests in window | Exponential backoff and temporary UI throttle. |
| `ERROR_AUTH_INVALID` | Invalid token/signature | Force re-auth flow. |
| `ERROR_AUTH_EXPIRED` | Expired token/session | Attempt token refresh, then re-auth. |
| `ERROR_PROTOCOL_VERSION_UNSUPPORTED` | Client/server protocol mismatch | Force hard refresh/update before allowing play. |

## 5) Idempotency and anti-replay policy (MVP)

### Economically sensitive events

- `seed.plant` (inventory decrement)
- `crop.harvest` (reward minting)
- `hatch.start` (egg consumption)
- `hatch.speed_up` (gem deduction)

### Requirements

1. Client MUST send unique `requestId` + `clientNonce` per economic attempt.
2. Server MUST persist dedupe key and final result:
   - `seed.plant`, `hatch.start`: 10-minute retention.
   - `crop.harvest`, `hatch.speed_up`: 24-hour retention.
3. Server SHOULD reject stale `sentAt` outside skew window with rate-limit style recoverable error.
4. Duplicate request MUST return the same semantic result and MUST NOT mutate economy twice.
5. Optional hardening during MVP: include `walletTxnId` in responses and require monotonic sequence on server.

## 6) Protocol versioning

- `protocolVersion` is required in `session.init` and attached to every request.
- **Non-breaking changes** (new optional fields/events):
  - Bump minor version (`0.1.x -> 0.2.0` while pre-1.0 semantics are still unstable).
  - Server ignores unknown fields; client ignores unknown push fields.
- **Breaking changes** (rename/remove fields, changed meaning, incompatible validation):
  - Bump major development line (`0.x -> 1.0` at first stable contract, then semver major bumps).
  - Server responds with `ERROR_PROTOCOL_VERSION_UNSUPPORTED` during handshake.
  - Client must show update/reload UX and stop mutation requests until reconnected with supported version.

