# REVIEW.md

| # | Where (function / line) | What is wrong | How you'd notice it (test, input, or symptom) | How you'd fix it |
|---|---|---|---|---|
| 1 | `health` | Always reports ready without contacting MongoDB. | Stop MongoDB and call `/health`; it still says `ok`. | Ping MongoDB and return 503 on failure. |
| 2 | `compute_late_minutes` | Uses an unqualified datetime and a floored-minute grace check; it misses the exact `10:00:01` boundary. | Shift 09:30, punch at 09:40:01 should report 10 late minutes. | Use IST shift start on the attendance day and compare elapsed seconds to 600. |
| 3 | `compute_late_minutes` / punch-in | Does not model overnight attendance dates. | Punch into a 22:00–06:00 shift at 05:30 IST. | Determine the attendance day using R1 before computing lateness. |
| 4 | `compute_work_hours` | Built-in `round` uses ties-to-even, not R4's half-up rounding. | Test a duration whose hour value lies exactly on a second-decimal half. | Use `Decimal` with `ROUND_HALF_UP`. |
| 5 | `compute_overtime` | Computes shift end on the same calendar day even for overnight shifts. | Punch out at 06:40 after a 22:00–06:00 shift. | Move overnight shift end to the following IST day. |
| 6 | `compute_overtime` | Applies no 30-minute minimum overtime threshold. | Punch out 20 minutes after shift end; expected overtime is zero. | Floor to whole minutes and only report 30 or more. |
| 7 | `EmployeeIn` | Missing contract validation for employee code, email, dates, shift format and unequal shift times. | Send malformed values or equal shift times. | Add typed fields, patterns, calendar-date validation and a model validator. |
| 8 | `create_employee` | Check-then-insert alone cannot prevent simultaneous duplicate codes; `created_at` is naive. | Send concurrent creates for one `emp_code`. | Add a unique index, handle `DuplicateKeyError` as 409, and store UTC time. |
| 9 | `list_employees` | `skip = page * page_size` skips page one, and `total` ignores the department filter. | Request page 1 or filter to one department. | Use `(page - 1) * page_size` and count the filtered query. |
| 10 | `list_employees` | Does not explicitly sort by `emp_code`. | Insert codes in a different order and list them. | Apply ascending `emp_code` sort and support it with an index. |
| 11 | `punch_in` | Does not handle unknown employees before dereferencing the employee document. | Punch in with an unknown code; the starter can produce a 500. | Return 404 before reading shift fields. |
| 12 | `punch_in` | Interprets epoch milliseconds in the host timezone, doesn't truncate to whole seconds, and treats falsy timestamps as omitted. | Run with a non-IST host timezone or pass invalid timestamp types. | Validate integer epoch milliseconds, convert from UTC, truncate, then convert to IST for the attendance date. |
| 13 | `punch_in` | Check-then-insert races and there is no unique `(emp_code, date)` guarantee. | Send two punch-ins concurrently for one attendance day. | Enforce a compound unique index and translate the losing insert into 409. |
| 14 | `punch_in` | Accepts arbitrary status values and returns an `id` not present in the contract. | Send `status: "HOLIDAY"` or inspect the response. | Restrict status to presence values and omit MongoDB identifiers from API responses. |
| 15 | `list_attendance` | Reads and sorts the whole collection in Python. | Run against 100,000 records; memory and latency grow with collection size. | Filter, sort, count and paginate in MongoDB with indexes. |
| 16 | `list_attendance` | Sort is only by date, ties are not ordered by `emp_code`, and `_id` is exposed as an `id`. | Create multiple records on one date or inspect response keys. | Sort by date descending then code ascending; serialize only contract fields. |
| 17 | `list_attendance` | Does not validate reversed date bounds or page/page-size limits. | Pass `date_from > date_to`, page 0, or page size 101. | Add query validation and return 422. |
| 18 | All starter timestamp responses | BSON datetimes are returned as ISO strings instead of epoch milliseconds; nested audit values also need conversion. | Inspect `created_at`, `punch_in`, or history timestamps. | Convert all API instants, including `history[].changes`, to epoch milliseconds. |
| 19 | Whole starter | It has no startup index creation, so query performance and uniqueness rely on no guarantees. | Inspect database indexes after startup or test duplicates. | Create required indexes idempotently at startup. |
| 20 | Contract TODO section | Punch-out, regularization, four analytics endpoints, and explain are absent. | Call any missing path. | Implement the contract endpoints and use shared aggregation builders for endpoint and explain behavior. |

## Looked at and not treated as defects

- `load_dotenv()` is acceptable: by default, existing environment variables are not overwritten.
- The default `page_size` of 20 matches the contract; it was the pagination math and validation that were defective.
- The internal employee code is correctly named `emp_code`; it must remain separate from MongoDB `_id`.
