# Loom Presentation: Property Revenue Dashboard Bug Investigation

**Target length:** 5–10 minutes  
**Audience:** Dev lead / hiring team

---

## Step 0

### 1. Start the app
```bash
cd New_devs_App
docker-compose up --build
# Flush cache right before you hit record on the Ocean tab                                   
docker exec $(docker ps --filter "name=redis" -q) redis-cli FLUSHALL
```
Wait until all 4 containers are healthy (backend, frontend, db, redis).

- Frontend: http://localhost:3000  
- Backend API docs: http://localhost:8000/docs

### 2. Open your browser tabs in advance
Have these ready before recording so you're not fumbling:
- Tab 1: http://localhost:3000 (login as Sunset Properties)
- Tab 2: http://localhost:3000 (login as Ocean Rentals — use incognito)
- Tab 3: Your code editor open to:
  - `backend/app/services/cache.py`
  - `backend/app/services/reservations.py`
  - `backend/app/api/v1/dashboard.py`
  - `database/seed.sql`

### 3. Credentials
| Client | Email | Password |
|--------|-------|----------|
| Sunset Properties (Client A) | sunset@propertyflow.com | client_a_2024 |
| Ocean Rentals (Client B) | ocean@propertyflow.com | client_b_2024 |

---

## Step 1: Introduction (30 sec)

> "I was asked to investigate three reported issues on the property revenue dashboard. I found four bugs — but before I could even confirm them with real data, I had to fix a first issue: the backend was never connecting to the database at all. I'll walk through everything."

---

## Step 2: Bug 0 — DB Never Connected (Silent Fallback to Mock Data)

**Mention this first — it's what allowed you to find and confirm everything else.**

### What was happening (1 min)

Open `backend/app/core/database_pool.py`. Three compounding errors meant the app **never hit the real database**:

**Error 1** — Line 18, wrong config fields:
```python
# BROKEN: these attributes don't exist in config.py
database_url = f"postgresql+asyncpg://{settings.supabase_db_user}:..."

# FIX: use the actual configured URL
database_url = settings.database_url.replace("postgresql://", "postgresql+asyncpg://")
```

**Error 2** — `poolclass=QueuePool` is incompatible with async SQLAlchemy engines. Removed it.

**Error 3** — `get_session()` was declared `async`, so calling it returned a coroutine, not a context manager. Changed to a regular `def`.

### Why this matters

Because the pool failed silently, every revenue request fell back to hardcoded mock data in `reservations.py:92`:
```python
mock_data = {
    'prop-001': {'total': '1000.00', 'count': 3},  # same for every tenant
    ...
}
```
Mock data is tenant-blind and always returns the same numbers — which is exactly why the cache leak and wrong totals weren't caught in initial testing. The system was never using real data.

> "I fixed this first, which then let me confirm all four reported bugs with live database values."

---

### Show the problem live (1 min)
1. Log in as **Sunset Properties** → navigate to property `prop-001` → note the revenue total
2. Log in as **Ocean Rentals** (incognito) → navigate to their `prop-001` → show it shows **Sunset's number**
3. Hit refresh a few times to show the inconsistency

### Explain the root cause in code (1.5 min)

Open `backend/app/services/cache.py`, line 13:

```python
# BROKEN: cache key only uses property_id — tenant_id is ignored
cache_key = f"revenue:{property_id}"
```

Open `database/seed.sql`, lines 8–9:

```sql
('prop-001', 'tenant-a', 'Beach House Alpha',    'Europe/Paris'),
('prop-001', 'tenant-b', 'Mountain Lodge Beta',  'America/New_York'),
```

> "Both tenants share the property ID `prop-001`. When Sunset's data gets cached under the key `revenue:prop-001`, Ocean Rentals' next request hits that same key and gets back Sunset's numbers. This is why it's inconsistent on refresh — it depends on who cached first."

### API proof

```
# Ocean requests first — their prop-001 has no reservations
GET /api/v1/dashboard/summary?property_id=prop-001  (as Ocean)
→ { "total_revenue": 0.0, "reservations_count": 0 }   ✓ correct

# Sunset requests after — should see $2250, gets Ocean's cached $0 instead
GET /api/v1/dashboard/summary?property_id=prop-001  (as Sunset)
→ { "total_revenue": 0.0, "reservations_count": 0 }   ✗ WRONG

# Redis key stored:
revenue:prop-001   ← single key, no tenant isolation
```

### Show the fix (30 sec)

```python
# FIX: include tenant_id in the cache key
cache_key = f"revenue:{tenant_id}:{property_id}"
```

One line. The `tenant_id` is already passed into this function — it just wasn't being used.

---

## Step 5: Bug 2a — No Date Filter (query returns all-time revenue, not monthly) (Ocean Rentals sees another company's numbers)

**This is the most critical bug. Lead with it.**

### Show the problem live (1 min)
1. Log in as **Sunset Properties** → navigate to property `prop-001` → note the revenue total
2. Log in as **Ocean Rentals** (incognito) → navigate to their `prop-001` → show it shows **Sunset's number**
3. Hit refresh a few times to show the inconsistency

### root cause in code 

Open `backend/app/services/cache.py`, line 13:

```python
# BROKEN: cache key only uses property_id — tenant_id is ignored
cache_key = f"revenue:{property_id}"
```

Open `database/seed.sql`, lines 8–9:

```sql
('prop-001', 'tenant-a', 'Beach House Alpha',    'Europe/Paris'),
('prop-001', 'tenant-b', 'Mountain Lodge Beta',  'America/New_York'),
```

> "Both tenants share the property ID `prop-001`. When Sunset's data gets cached under the key `revenue:prop-001`, Ocean Rentals' next request hits that same key and gets back Sunset's numbers. This is why it's inconsistent on refresh — it depends on who cached first."

### Show the fix (30 sec)

```python
# FIX: include tenant_id in the cache key
cache_key = f"revenue:{tenant_id}:{property_id}"
```

One line. The `tenant_id` is already passed into this function — it just wasn't being used.

---

## Step 3: Bug 2a — No Date Filter (query returns all-time revenue, not monthly)

### Explain the root cause in code (1 min)

Open `backend/app/services/reservations.py`, lines 51–59:

```python
query = text("""
    SELECT 
        property_id,
        SUM(total_amount) as total_revenue,
        COUNT(*) as reservation_count
    FROM reservations 
    WHERE property_id = :property_id AND tenant_id = :tenant_id
    GROUP BY property_id
""")
```

> "There's no date filter at all. This query sums every reservation ever. When the client looks at their March report, they're seeing all-time revenue combined."

Point out that `calculate_monthly_revenue` already exists at lines 5–32 of the same file with correct date logic — but its DB call is commented out and returns `Decimal('0')`. The fix was already written, just never wired up.

### Show the fix (20 sec)

Add `check_in_date` range parameters to the active query:

```python
query = text("""
    SELECT 
        property_id,
        SUM(total_amount) as total_revenue,
        COUNT(*) as reservation_count
    FROM reservations 
    WHERE property_id = :property_id 
      AND tenant_id = :tenant_id
      AND check_in_date >= :start_date
      AND check_in_date < :end_date
    GROUP BY property_id
""")
```

---

## Step 6: Bug 2b — Timezone Not Applied to Date Filtering (the hidden depth of Bug 2)

**This is the subtlest bug — worth calling out separately to show depth of investigation.**

### The trap in the data (1 min)

Open `database/seed.sql`, line 19:

```sql
('res-tz-1', 'prop-001', 'tenant-a', '2024-02-29 23:30:00+00', '2024-03-05 10:00:00+00', 1250.000)
```

> "Notice the reservation is named `res-tz-1` — it's a timezone test case. The check-in is stored as `2024-02-29 23:30:00 UTC`. In UTC, that looks like February 29."

Now open `seed.sql` line 8:

```sql
('prop-001', 'tenant-a', 'Beach House Alpha', 'Europe/Paris'),
```

> "But this property is in `Europe/Paris` — UTC+1 in winter. So `Feb 29 at 23:30 UTC` is actually `March 1 at 00:30` in Paris local time. For Client A, this is a **March** booking. Without timezone conversion, a naive date filter would put it in February — wrong month, wrong revenue total."

### Why it matters

Even after fixing Bug 2a (adding the date filter), if you filter on raw UTC timestamps without converting to the property's timezone:
- This $1,250 reservation stays in February in the query
- Client A's March total is still $1,000 short

### Show the fix (30 sec)

The query needs to convert `check_in_date` into the property's local timezone before comparing. This requires joining to the `properties` table to get the `timezone` column:

```sql
SELECT 
    r.property_id,
    SUM(r.total_amount) as total_revenue,
    COUNT(*) as reservation_count
FROM reservations r
JOIN properties p ON p.id = r.property_id AND p.tenant_id = r.tenant_id
WHERE r.property_id = :property_id 
  AND r.tenant_id = :tenant_id
  AND (r.check_in_date AT TIME ZONE p.timezone) >= :start_date
  AND (r.check_in_date AT TIME ZONE p.timezone) < :end_date
GROUP BY r.property_id
```

> "Postgres's `AT TIME ZONE` operator converts the stored UTC timestamp into the property's local time before the date comparison. Now `Feb 29 23:30 UTC` correctly becomes March in the Paris timezone."

---

## Step 7: Bug 3 — Revenue Off by a Few Cents (Finance team's precision issue)

### Explain the root cause in code (1 min)

Open `backend/app/api/v1/dashboard.py`, line 18:

```python
total_revenue_float = float(revenue_data['total'])
```

> "Financial amounts come out of Postgres as `NUMERIC(10,3)` — exact decimal. The reservation service correctly wraps these in Python's `Decimal` type. But right here, on the way out, we convert to a float."

> "Floats can't represent most decimal fractions exactly. For example: `333.333 + 333.333 + 333.334` should be exactly `999.999`. As floats, you get `999.9990000000001`. Multiply that across dozens of properties and you start seeing cents disappear or appear out of nowhere."

### Show the fix (20 sec)

```python
# FIX: return the string directly — Decimal precision preserved, frontend parses it
return {
    "property_id": revenue_data['property_id'],
    "total_revenue": revenue_data['total'],   # string, not float
    "currency": revenue_data['currency'],
    "reservations_count": revenue_data['count']
}
```

Remove the `total_revenue_float = float(...)` line entirely.

---

## Step 8: Summary (30 sec)

| # | Client | Bug | File | Line | Fix |
|---|--------|-----|------|------|-----|
| 0 | — | DB never connected, fell back to mock data | `database_pool.py` | 18, 22, 50 | Use `settings.database_url`, fix pool class and session method |
| 1 | Ocean Rentals | Cross-tenant cache leak | `cache.py` | 13 | Add `tenant_id` to cache key |
| 2a | Sunset Properties | No date filter — all-time revenue returned | `reservations.py` | 51–59 | Add `check_in_date` range filter |
| 2b | Sunset Properties | Date filter ignores property timezone | `reservations.py` | 51–59 | Apply `AT TIME ZONE` from properties table |
| 3 | Finance team | Float precision loss on money | `dashboard.py` | 18 | Return string, not float |

> "Five bugs total. The first one — the DB never connecting — was the hidden root cause that masked everything else. Once fixed, all four reported issues showed up with real data. Bug 0 explains why testing didn't catch any of this."

---


