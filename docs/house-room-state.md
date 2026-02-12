# HouseRoom State Specification

This document defines the canonical runtime and persistence behavior for the Colyseus `HouseRoom` in the MVP.

## 1. Canonical `HouseRoom` State Tree

The server is authoritative for all mutable game state. The room state is modeled as:

```ts
HouseRoomState {
  room: {
    room_id: string,
    owner_user_id: UUID,
    revision: number,
    server_time_ms: number,
    created_at_ms: number,
    last_persisted_revision: number,
  },

  players: {
    by_session_id: {
      [session_id: string]: {
        user_id: UUID,
        display_name: string,
        role: 'owner' | 'guest',
        online: boolean,
        connection_state: 'connected' | 'reconnecting' | 'disconnected',
        avatar_cosmetic_id?: string,

        // Presence/runtime
        pos: { x: number, y: number, dir: 'N' | 'S' | 'E' | 'W' },
        emote_state?: { emote_id: string, until_ms: number },
        joined_at_ms: number,
        last_input_seq: number,
      }
    }
  },

  tile_entities: {
    by_tile_id: {
      [tile_id: string]: {
        tile_id: string,
        entity_type: 'crop' | 'incubator' | 'decor_anchor' | 'empty',
        grid_x: number,
        grid_y: number,

        // Gameplay state
        state: {
          planted_item_id?: string,
          growth_stage?: number,
          planted_at_ms?: number,
          ready_at_ms?: number,

          hatch_egg_type?: string,
          hatch_start_ms?: number,
          hatch_duration_ms?: number,
        },

        updated_at_ms: number,
      }
    }
  },

  placed_items: {
    by_item_instance_id: {
      [item_instance_id: string]: {
        item_instance_id: string,
        owner_user_id: UUID,
        item_id: string,
        category: 'furniture' | 'decor' | 'utility',
        grid_x: number,
        grid_y: number,
        rotation: 0 | 90 | 180 | 270,
        collision_mask?: string,
        placed_at_ms: number,
        updated_at_ms: number,
      }
    }
  },

  transient_effects: {
    by_effect_id: {
      [effect_id: string]: {
        effect_id: string,
        effect_type: 'harvest_pop' | 'coin_burst' | 'sparkle' | 'emote',
        origin: { x: number, y: number },
        started_at_ms: number,
        ttl_ms: number,
      }
    }
  }
}
```

### Notes
- `revision` increments on every accepted mutating action.
- `server_time_ms` is updated on a fixed server tick and used by clients to correct timer UI.
- `transient_effects` is non-persistent and may be dropped on restart/rejoin.

## 2. Field Visibility and Authority

### 2.1 Authoritative server-only fields
The following fields are **never trusted from client input** and are set/derived server-side only:

- `room.owner_user_id`, `room.revision`, `room.last_persisted_revision`
- `players.*.role` (computed from `user_id` vs owner relationship)
- `players.*.last_input_seq` (used for stale/duplicate action protection)
- `tile_entities.*.state.ready_at_ms` (computed from config + planted time)
- `tile_entities.*.state.growth_stage` when progression is server-clock-derived
- Any anti-cheat check fields (cooldowns, occupancy/collision validation results)

### 2.2 Client-visible fields
Sent in room snapshot/patch stream:

- Stable room identity: `room_id`, `owner_user_id`
- Presence and avatars: user IDs, names, online status, player transforms
- Tile entity public state required for rendering (crop stage, incubator progress)
- Placed item coordinates/rotation
- Transient effect payloads for visual sync
- `revision` and `server_time_ms` for synchronization

### 2.3 Client-submitted fields accepted only as requests
Clients may send **intent**, not authoritative state:

- Movement intents (`target_x`, `target_y`, `input_seq`)
- Interaction intents (`tile_id`, `item_instance_id`, action type)
- Placement intents (`item_id`, desired grid coordinates/rotation)

Server validates and either applies canonical update or returns a rejection.

## 3. Join and Reconnect Flow

## 3.1 Initial join snapshot
1. Client joins `HouseRoom` by room ID.
2. Server loads canonical base from persistence (latest DB checkpoint) if room is cold, then applies any in-memory pending changes (if room already active).
3. Server emits full snapshot with current `revision` and `server_time_ms`.
4. Client stores `base_revision = snapshot.revision` and starts accepting patches.

Snapshot source of truth:
- **Active room in memory** if already running.
- **PostgreSQL checkpoint** on cold start.

## 3.2 Patch cadence
- State patch stream is emitted by Colyseus on mutation (event-driven), with a max flush interval target of `50-100 ms` under burst conditions.
- Heartbeat/no-op time sync (`server_time_ms`) is pushed at least every `1s` even with no mutations.

## 3.3 Reconnect behavior
1. Reconnecting client presents prior session token and last applied revision (`client_revision`).
2. If server has patch history window covering `client_revision+1..head`, server sends incremental catch-up patches.
3. Otherwise server sends a fresh full snapshot (authoritative reset).

## 3.4 Conflict resolution for stale client actions
When a client action references stale assumptions (e.g., tile already harvested):

- Server compares action preconditions against current canonical state.
- If invalid/stale, server rejects with deterministic error code:
  - `ERR_STALE_REVISION`
  - `ERR_PRECONDITION_FAILED`
  - `ERR_PERMISSION_DENIED`
- Client must rollback optimistic UI and reconcile to latest server patch/snapshot.
- Server never merges client-sent state diffs; only validated intents mutate state.

## 4. Persistence Checkpoints (PostgreSQL)

## 4.1 Write triggers
Room state is persisted to PostgreSQL at these checkpoints:

1. **Mutation debounce checkpoint**: after mutating actions, flush at most once per `2s` while dirty.
2. **Critical action checkpoint**: immediately after high-value economic events (purchase, harvest grant, hatch completion, item place/remove that changes inventory).
3. **Room lifecycle checkpoint**: on room shutdown/owner disconnect timeout.
4. **Periodic safety checkpoint**: every `30s` while room active and dirty.

## 4.2 Persistence payload
- Upsert changed `House_Items` rows (placed items + tile entity state JSONB as applicable).
- Persist user economy deltas and inventory changes in same DB transaction when caused by same action.
- Record `last_persisted_revision` in room metadata table (or equivalent checkpoint table).

## 4.3 Retry strategy on DB failure
- On failure, mark room as `dirty_persist_failed` and keep authoritative state in memory.
- Retry with exponential backoff + jitter (e.g., `500ms`, `1s`, `2s`, `4s`, cap `15s`).
- Coalesce retries to latest dirty revision (drop superseded intermediate snapshots).
- During persistent failure:
  - Continue gameplay for non-destructive actions.
  - Gate hard-currency/IAP-finalization actions to avoid unrecoverable economic divergence.
- Emit operational metrics/logs for failure count, oldest unpersisted revision, and retry latency.
- On successful retry, clear failure flag and update `last_persisted_revision`.

## 5. Guest Permissions and Ownership Rules

## 5.1 Ownership check
- `room.owner_user_id` is the sole authority for room ownership.
- Every mutating action must resolve `actor.user_id` from authenticated session and compare against `owner_user_id` when owner-only.
- Client-provided `user_id` in payload is ignored.

## 5.2 Guest role model
Guests are assigned one of two permission profiles:

1. **Read-only guest**
   - Allowed: join room, receive state, chat/emote/presence movement (non-persistent).
   - Denied: place/remove/move items, plant/harvest, incubator operations, purchases, inventory mutation.

2. **Interactive guest** (explicitly enabled by owner policy)
   - Allowed: non-economic interactions configured by owner (e.g., sit/trigger decor, emotes, optional watering/help actions).
   - Denied by default: any inventory/economy mutation, ownership transfer, destructive layout edits.

## 5.3 Enforcement rules
- Permission checks happen server-side per action opcode before validation logic.
- Suggested policy table:
  - `MOVE_AVATAR`: owner + all guests
  - `EMOTE`: owner + all guests
  - `PLACE_ITEM`: owner only
  - `MOVE_ITEM`: owner only
  - `REMOVE_ITEM`: owner only
  - `PLANT_SEED`: owner only (or interactive guest if explicit feature flag)
  - `HARVEST`: owner only (or interactive guest with reward routed to owner, if enabled)
  - `START_HATCH`: owner only
  - `SPEEDUP_HATCH`: owner only
- Failed checks return `ERR_PERMISSION_DENIED` and do not mutate room revision.

## 5.4 Auditability
For each rejected action, log:
- `room_id`, `actor_user_id`, `action`, `reason_code`, `room_revision`.

This enables moderation and debugging of guest access behavior.
