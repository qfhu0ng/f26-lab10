# RACE.md

One section per milestone. Fill each one in order as you go.

The prediction in milestone 1 is written BEFORE you run your race test. A TA
may ask about it. Commit it on its own so the order is visible.

Keep it short and specific. Point at methods, call sites, and the invariant.

---

## Milestone 1: from the ticket to a test that fails every time

### Suspect list

Everything your review turned up that touches shared state, with a one-line
verdict for each. Add or remove rows as needed.

| # | What you found (method, field, or class) | Verdict, in one line |
|---|---|---|
| 1 | `BookingService.book()`: `store.hasOverlap(...)` (line 70), then `store.insert(...)` (line 74) | Check-then-act race. Each store call is synchronized on its own, but the store's lock is released between the check and the insert, so two `book()` calls for the same slot can both pass the check and both insert a CONFIRMED booking. The class comment's claim that operations built on a thread-safe store are thread safe is where it goes wrong. **Explains the ticket.** |
| 2 | `BookingStats` counters (`confirmed++` and the other three) | Lost updates and stale reads from unsynchronized read-modify-write. Real, but documented as approximate, nothing in the service reads them, and the ticket's bad state is in the booking data, not the dashboard. Not the ticket. |
| 3 | `BookingService.sequence` / `nextId()` | `incrementAndGet()` is one atomic call, so ids are unique within one service. The counter is per instance and the store does not check ids, so two service instances over one store could hand out the same id. Not the ticket: its ids (`bk-8412`, `bk-8413`) are distinct and consecutive. |
| 4 | `InMemoryBookingStore`: every method `synchronized` on the store; `bookingsForRoom` copies inside the lock | Safe per call, as `BookingStore` promises. That does not make a sequence of two calls atomic, which is the gap row 1 falls into. |
| 5 | Waitlist paths: `BookingService.joinWaitlist()` and `InMemoryBookingStore.cancelAndPromote()` | Safe for the invariant. `joinWaitlist` only inserts WAITLISTED entries, which block nothing; `cancelAndPromote` cancels, re-checks the slot (`overlapsConfirmed`), and promotes under one lock. The ticket also rules this path out: neither booking was cancelled or waitlisted. |
| 6 | `BookingService.catalog` (a `HashMap`), the `Room` and `Booking` records, `BookingLog` | Safe. The map is filled in the constructor, held in a `final` field, and only read afterwards; the records are immutable; the log's formatter is immutable and each `println` writes a whole line. |

**Which one explains the ticket, and how you know.** Tie it to the invariant
the ticket names.

Row 1. The invariant is that a room has at most one CONFIRMED booking covering
any minute. Only two places create a CONFIRMED booking: the insert in `book()`
and the promotion in `cancelAndPromote()`. The promotion re-checks the slot
under the store's lock, and the ticket says neither booking was cancelled or
waitlisted, so both bookings came from `book()`. The only guard in `book()` is
`hasOverlap`, a separate store call that releases the lock before `insert`
runs. The log fits that gap: two `booking.confirmed` lines for the same room
and range 1 ms apart, consecutive ids from one sequence, and nothing logged
between them (no rejection, cancellation, or promotion). All three incidents
came in the first minute after a slot opened, when many people book the same
slot at once. The suite calls `book()` one caller at a time, and two people on
laptops are unlikely to land inside a gap that short. The stress runner, which
releases two threads together from a barrier, ended with 2 CONFIRMED bookings
in 19, 1, and 2 of 100 rounds and in 30 of 2000 rounds across four runs on
2026-11-06.

### Predicted interleaving (write this before you run your race test)

Two threads, one row per step, in the form from lecture. Describe what each
thread does at that step and leave the other column empty. Add or remove rows
as your prediction needs.

| Step | Thread A | Thread B |
|---|---|---|
| t1 | | |
| t2 | | |
| t3 | | |
| | **End state:** | |

If the run turned out different from your prediction, add one line on what
you missed. Leave the table as you first wrote it.

### Your race test

**How you aimed `PausingStore`, and the result.** Method name and call index, why
that pair pins the schedule you predicted, and the output of two separate
runs (assertion failure and log lines).

---

## Milestone 2: the patch a teammate sent

**Your milestone-1 test on the patched code.** Green or red. Paste the result.

**The re-aimed run.** Method name, call index, the pasted result, and one line
on why that aim tests the patch rather than sidestepping it.

**Your verdict.** Accept or reject, and what your milestone-1 result does and
does not show about the patch.

---

## Milestone 3: the fix you would ship

**Which option, and what you changed.** Name the lock or the atomic operation
and the two operations it now covers. Paste the diff of your change and the
passing run of your milestone-1 test and the full suite.

**Is your milestone-1 test still testing anything?** After your fix, describe
what that test now exercises. Did `PausingStore` pause a call during the green run? How do
you know?

**What runs one at a time now that ran in parallel before.** Name concretely which callers
queue behind which. "Nothing, it is just correct now" does not count as an
answer.

**What you left unfixed.** For anything your review flagged that you chose not
to fix, one line each on why.
