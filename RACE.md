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

Start: no bookings. Threads A and B both book the same room for `[540, 600)`
at the same time.

| Step | Thread A | Thread B |
|---|---|---|
| t1 | `book()` calls `hasOverlap()` first. The underlying store checks under its lock, gets `false`, and releases the lock. `PausingStore` then pauses A before the result is returned to `book()`. | |
| t2 | | `book()` calls `hasOverlap()`. A has not inserted anything yet, so B's check also returns `false`. |
| t3 | | Inserts a CONFIRMED booking and finishes its request. |
| t4 | The test releases A. A receives the `false` computed earlier, does not check again, and inserts a second CONFIRMED booking. | |
| | **End state:** two CONFIRMED bookings for the same room and slot. This violates "at most one CONFIRMED booking covering any given minute", so the assertion expecting 1 fails with an actual count of 2. | |

Each store method is locked on its own, but nothing makes the check and the
insert atomic together.

If the run turned out different from your prediction, add one line on what
you missed. Leave the table as you first wrote it.

### Your race test

**How you aimed `PausingStore`, and the result.** Method name and call index, why
that pair pins the schedule you predicted, and the output of two separate
runs (assertion failure and log lines).

Test: `RaceTest.twoCustomersBookingTheSameSlotConfirmOnlyOne`.

Aim: `new PausingStore(real, "hasOverlap", 1)`. A starts alone and B starts
only after `awaitPaused()` returns, so call 1 of `hasOverlap` is A's. The gate
parks A after the real store has answered `false` and released its lock, but
before `book()` gets that answer, which is the gap between the check and the
insert (t1). B's check is call 2, which the one-shot gate lets through, so B
books start to finish (t2, t3). `release()` lets A insert on its old answer
(t4). The test asserts that the gate fired and that both threads finished,
then checks the end state in `real`: exactly one CONFIRMED booking for
`WEH-5409 [540, 600)`.

Result: red on both runs, failing on the end-state assertion with 2 instead
of 1, as predicted. `bk-1` is B's booking and `bk-2` is A's: `book()` takes
its id after the check returns, and A was still parked when B took its id.
The pair has the ticket's shape: two confirmations for one room and range,
consecutive ids, 2 ms apart here (1 ms in the ticket), nothing logged between
them. The full suite on the same code: 12 tests, 1 failure (this one); the 11
shipped tests stay green.

Run 1 (`mvn -B test -Dtest=RaceTest`):

```
2026-11-06T08:55:33.334Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
2026-11-06T08:55:33.336Z INFO booking.confirmed booking=bk-2 room=WEH-5409 start=540 end=600
[ERROR] Tests run: 1, Failures: 1, Errors: 0, Skipped: 0, Time elapsed: 0.043 s <<< FAILURE! -- in edu.cmu.cs214.lab10.harness.RaceTest
[ERROR] edu.cmu.cs214.lab10.harness.RaceTest.twoCustomersBookingTheSameSlotConfirmOnlyOne -- Time elapsed: 0.029 s <<< FAILURE!
org.opentest4j.AssertionFailedError: CONFIRMED bookings for WEH-5409 [540, 600) ==> expected: <1> but was: <2>
```

Run 2 (`mvn -B test -Dtest=RaceTest`):

```
2026-11-06T08:55:36.091Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
2026-11-06T08:55:36.093Z INFO booking.confirmed booking=bk-2 room=WEH-5409 start=540 end=600
[ERROR] Tests run: 1, Failures: 1, Errors: 0, Skipped: 0, Time elapsed: 0.042 s <<< FAILURE! -- in edu.cmu.cs214.lab10.harness.RaceTest
[ERROR] edu.cmu.cs214.lab10.harness.RaceTest.twoCustomersBookingTheSameSlotConfirmOnlyOne -- Time elapsed: 0.027 s <<< FAILURE!
org.opentest4j.AssertionFailedError: CONFIRMED bookings for WEH-5409 [540, 600) ==> expected: <1> but was: <2>
```

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
