# REVIEW.md

List every defect you found in the starter `app/main.py` (helpers and endpoints). For each one:

| # | Where (function / line) | What is wrong | How you'd notice it (test, input, or symptom) | How you fixed it |
|---|---|---|---|---|
| 1 | `health()` | Always returns 200; never checks if Mongo is alive. Should 503 when DB is down. | Stop MongoDB container, call `GET /health`. Still returns 200. |Wrapped `db.command("ping")` in try/except, 503 on failure.|
| 2 | `create_employee()` |Classic check-then-insert race: two parallel creates with the same `emp_code` both pass `find_one` and both insert. | Fire two `POST /employees` with the same code at the same time — both come back 201 and the collection ends up with two rows. | Added a unique index on `emp_code`; `insert_one` now raises `DuplicateKeyError`, which I map to 409. |
| 3 | `create_employee()` | `created_at = datetime.now()` is naive (no timezone). | Create employee, read with `mongosh`. Stored date has no timezone. | `datetime.now(timezone.utc)`; convert to epoch ms at response boundary. |
| 4 | `create_employee()` | Does not validate `shift_start != shift_end`. Contract says 422. | `POST /employees` with `shift_start == shift_end`. Returns 201. | Pydantic validator; 422 when equal. |
| 5 | `list_employees()` | Off-by-one pagination: `skip = page * page_size` skips one extra page. | On a 30-employee collection, `GET /employees?page=1&page_size=20` returns only the last 10 rows instead of the first 20. | Changed to `skip = (page - 1) * page_size`.|
| 6 | `list_employees()` | `total = count_documents({})` ignores `department` filter. | `GET /employees?department=Sales` returns total = whole collection. | `count_documents(q)` with same filter. |
| 7 | `list_employees()` | No sort; contract requires `emp_code` ascending. | Insert employees out of order; response in insertion order. | `.sort("emp_code", 1)`. |
| 8 | `punch_in()` | Unknown `emp_code` → `emp` is `None` → `TypeError` → 500. Contract requires 404. | `POST /attendance/punch-in` with unknown `emp_code`. Returns 500. | `if emp is None: raise HTTPException(404, ...)`. |
| 9 | `punch_in()` | `datetime.fromtimestamp(ms/1000)` uses local timezone, not IST. | Set TZ=UTC, punch-in; stored time off by 5:30. | `datetime.fromtimestamp(ms/1000, tz=timezone.utc)`. |
| 10 | `punch_in()` | Race: `find_one` + `insert_one` not atomic. Two concurrent punch-ins both pass check. | Two parallel punch-ins same day. Both return 201. | Unique compound index `(emp_code, date)`; `DuplicateKeyError` → 409. |
| 11 | `punch_in()` | `status` is free string; `ABSENT`/`LEAVE` accepted, but contract limits to `PRESENT`/`WFH`/`ON_DUTY`. | `POST /attendance/punch-in` with `status: "LEAVE"`. Returns 201. | Enum of three presence statuses; 422 otherwise. |
| 12 | `punch_in()` | `punched_at` in seconds silently accepted (multiplied by 1000). | `punched_at: 1783312500` returns 201 with wrong date. | Pydantic int with `ge=100000000000, le=4102444800000`; 422. |
| 13 | `punch_in()` | Response includes `id` (stringified `_id`). Contract never exposes `_id`. | Response has `id` field not in contract. | Removed `id`; serialized via shared shape. |
| 14 | `compute_late_minutes()` | Uses `punch_in.replace(hour, minute)` on naive datetime. With aware datetime, raises TypeError. | Aware `punch_in` → `punch_in - start` raises. | Convert both to IST-aware; `start = punch_in.astimezone(IST).replace(...)`. |
| 15 | `compute_late_minutes()` | `minutes > 10` checks past shift_start, but rule is strictly > 10:00; also `int()` truncates wrong for negatives. | Punch 09:39:59 → returns 9, but rule says 0. | `if (punch_in - start) <= timedelta(minutes=10): return 0; return int((punch_in - start).total_seconds() // 60)`. |
| 16 | `compute_work_hours()` | Uses Python's `round()`, which is banker's rounding (`round(2.675, 2) == 2.67`). Contract requires half-up. | Any punch pair that produces exactly 2.675 hours returns 2.67 instead of 2.68. | Replaced with `Decimal(...).quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)`|
| 17 | `compute_overtime()` | Missing "at least 30 minutes" gate. Returns 5 for 5-min overrun. | Shift end 18:30, punch-out 18:35 → returns 5, contract says 0. | `mins = int(...); return mins if mins >= 30 else 0`. |
| 18 | `compute_overtime()` | `datetime.fromisoformat(date_str)` builds naive datetime; wrong day for overnight shifts. | Overnight EMP0005 → shift end on wrong day. | Compute shift end from `punch_in` + shift hours; add 1 day if `shift_end <= shift_start`. |
| 19 | `list_attendance()` | Loads all documents into memory, sorts in Python. Fails at 100k records. | Seed 100k records, call `GET /attendance`. Slow. | `find(q).sort([("date",-1),("emp_code",1)]).skip(...).limit(...)` + `count_documents(q)`. |
| 20 | `list_attendance()` | No validation of `date_from <= date_to`. Contract says 422. | `date_from=2026-07-10&date_to=2026-07-01`. Returns 200. | Explicit check; 422 if `date_from > date_to`. |
| 21 | `list_attendance()` | `status` query param is free string. Invalid status returns empty list. | `GET /attendance?status=FOO`. Returns 200 with `items: []`. | Typed as `Status` enum; FastAPI returns 422. |
| 22 | App startup | No indexes are created anywhere in the file. Contract requires idempotent index creation at startup. | Run `explain` on `GET /attendance` — the plan shows a `COLLSCAN` instead of `IXSCAN`. | Added `create_index(...)` calls in a `startup` handler; MongoDB ignores duplicates so it is idempotent.|

## Things I looked at and decided were **not** defects

- **IST offset** — a fixed +05:30 with no DST is correct for India, so `IST = timezone(timedelta(hours=5, minutes=30))` is fine.
- **`load_dotenv()` at import** — acceptable; `python-dotenv` does not override real environment variables by default, so the priority rule in the contract still holds.
- **Default `MONGO_URI`** — harmless fallback for local dev; the grader injects the real URI, so the default never wins in practice.
- **`doc.pop("_id", None)`** — right intent; `_id` must never appear in a response, so stripping it is correct.
- **Missing endpoints** — the `TODO` block already lists them; that is the assignment itself, not a defect in the starter.





Also note anything you looked at and decided was **not** a defect, and why.
