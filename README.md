# Detector Channel Key-Value Store

A Redis-style key–value store for mapping detector channels to human-readable names, with full version history.  
Supports **fast current lookups** and **time-based queries** for reproducibility (e.g., retrieving mappings as of a run timestamp).

---

## Data Model

### Current View
- **Channel metadata**
```

chan:{crate}:{slot}:{ch} → Hash
name        = human-friendly channel name
detector    = detector identifier
since\_ts    = validity start time (ISO8601 or epoch)
until\_ts    = validity end time (∞ for current)

```

- **Detector → channel index**
```

det:{detector}\:channels → Set of channel IDs

```

- **Name → channel convenience lookup (optional)**
```

name:{name} → chan\_id

```

- **Global channel list**
```

all\:channels → Set of all known channel IDs

```

### History
- **Per-channel history**
```

hist\:chan:{chan\_id} → Sorted Set
score   = since\_ts
member  = JSON blob or version\_id

```

- **Version details**
```

ver\:chan:{chan\_id}:{since\_ts} → Hash
name, detector, since\_ts, until\_ts

```

---

## Updates

When a channel changes (detector or name):

1. **Close old version**  
 - Set `until_ts = now` in `chan:*` and corresponding `ver:*`.

2. **Open new version**  
 - Create `ver:chan:{id}:{now}` with `since_ts=now`, `until_ts=+inf`.  
 - Update `chan:{crate}:{slot}:{ch}` to mirror current values.

3. **Update detector membership** *(atomically with MULTI/EXEC or Lua)*  
 - Remove chan_id from `det:{old_detector}:channels`  
 - Add chan_id to `det:{new_detector}:channels`

4. **Append to history**
```

ZADD hist\:chan:{id} now "{since\_ts}"

````

---

## Queries

### Current channels for a detector
```redis
SMEMBERS det:{detector}:channels
HMGET chan:{crate}:{slot}:{ch} name detector
````

### Mapping at a run timestamp

Requires run timestamp (direct or via `run:{run_id}:ts`).

For each channel ID:

```redis
ZRANGEBYSCORE hist:chan:{id} -inf run_ts
```

* Take the **last entry** (since\_ts ≤ run\_ts).
* Ensure `until_ts > run_ts`.

### Materialized run views (optional optimization)

At run start, snapshot into:

```
run:{run_id}:channels        → Set of all valid channel IDs
run:{run_id}:det:{detector}  → Set of channels per detector
```

Lookups during the run become O(1).

---

## Example

```redis
> HGETALL chan:1:3:12
name      "HMS_CALO_PMT5"
detector  "HMS_CALO"
since_ts  "2025-08-01T00:00:00Z"
until_ts  "+inf"

> SMEMBERS det:HMS_CALO:channels
["1/3/12", "1/3/13", "1/4/1", ...]
```

---

## Notes

* **Consistency**: Use atomic transactions or Lua scripts for updates.
* **Scalability**: Sets/Hashes keep lookups fast; ZSETs provide efficient time travel.
* **Flexibility**: Can store synthetic `chan_id`s instead of crate/slot/ch strings.
* **Convenience**: Optional `name:{name}` mapping for quick lookups.

---
