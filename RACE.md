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

The patch adds a second `hasOverlap` call in `book()`, after `nextId()` and
right before `insert`. Applied and evaluated in milestone 2 with
`git apply proposed-fix.diff`; `git diff --stat` showed only
`BookingService.java` (7 insertions).

Green (`mvn -B test -Dtest='RaceTest#twoCustomersBookingTheSameSlotConfirmOnlyOne'`):

```
2026-11-06T14:26:37.762Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
2026-11-06T14:26:37.765Z INFO booking.rejected room=WEH-5409 start=540 end=600 reason=slot already held
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.042 s -- in edu.cmu.cs214.lab10.harness.RaceTest
[INFO] BUILD SUCCESS
```

The gate still parks A at call 1 of `hasOverlap`, which is now A's first
check. B makes calls 2 and 3 and inserts `bk-1`. Once A is released it makes
the patch's re-check (call 4), sees B's booking, and is rejected. The pause sat
before the re-check, so the re-check ran after B had finished and had nothing
to race with.

**The re-aimed run.** Method name, call index, the pasted result, and one line
on why that aim tests the patch rather than sidestepping it.

Test: `RaceTest.twoCustomersStillConfirmTwiceWhenPausedAfterTheRecheck`. Aim:
`new PausingStore(real, "hasOverlap", 2)`. A starts alone and makes two checks
before its insert, so calls 1 and 2 are both A's; call 2 is the re-check. The
gate parks A after the re-check has returned `false`. B books start to finish
(calls 3 and 4, then its insert), and A, once released, inserts on its old
answer. Same end-state assertion as milestone 1.

Red on both runs (`mvn -B test -Dtest=RaceTest`; the class runs both tests,
and the milestone-1 test passed in each run, so only the re-aimed test's lines
are shown):

```
2026-11-06T14:26:57.564Z INFO booking.confirmed booking=bk-2 room=WEH-5409 start=540 end=600
2026-11-06T14:26:57.566Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
[ERROR] edu.cmu.cs214.lab10.harness.RaceTest.twoCustomersStillConfirmTwiceWhenPausedAfterTheRecheck -- Time elapsed: 0.030 s <<< FAILURE!
org.opentest4j.AssertionFailedError: CONFIRMED bookings for WEH-5409 [540, 600) ==> expected: <1> but was: <2>
```

```
2026-11-06T14:26:59.285Z INFO booking.confirmed booking=bk-2 room=WEH-5409 start=540 end=600
2026-11-06T14:26:59.287Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
[ERROR] edu.cmu.cs214.lab10.harness.RaceTest.twoCustomersStillConfirmTwiceWhenPausedAfterTheRecheck -- Time elapsed: 0.028 s <<< FAILURE!
org.opentest4j.AssertionFailedError: CONFIRMED bookings for WEH-5409 [540, 600) ==> expected: <1> but was: <2>
```

`bk-1` is A's this time: on the patched path A takes its id before the
re-check, so it was parked holding `bk-1`, and B's booking (`bk-2`) is logged
first.

Why this aim tests the patch: call 2 is the last read before the write, so the
pause sits in the window the patch says it closes; call 1 parks A before the
re-check and lets the re-check run after B is done, which sidesteps it.

The patch is not committed. On the committed (unpatched) code this test fails
on its gate assertion instead (`A was never parked in its second overlap
check`, after the 2 s timeout), because unpatched `book()` makes only one
overlap check. The full suite there is 13 tests with 2 failures: this one and
the milestone-1 test (2 instead of 1).

Other runs on the patched code: the full suite is 13 tests with 1 failure
(the re-aimed test; the 11 shipped tests and the milestone-1 test pass). The
stress runner gave 1, 0, and 0 bad rounds of 100 and 0 of 2000 (before the
patch: 19, 1, 2 of 100 and 30 of 2000). Fewer failures were observed in these
runs, but that does not show the window got narrower, let alone that the race
is gone. The rejection rests on the controlled interleaving still reproducing
the error, not on the stress failure rate.

**Your verdict.** Accept or reject, and what your milestone-1 result does and
does not show about the patch.

**Reject.** The milestone-1 test passing on the patch only shows that when B
finishes its insert before A's second check, the re-check catches the conflict
and rejects A. It does not show that the check and the insert have become one
atomic operation. The re-aimed test pauses A after its second check has
returned `false` but before it inserts. B can still complete its booking in
that gap, and A then inserts on its earlier answer, leaving two CONFIRMED
bookings. Two deterministic failures show that the patch still violates the
invariant that a room has at most one CONFIRMED booking covering any given
minute, so the patch should be rejected.

---

## Milestone 3: the fix you would ship

**Which option, and what you changed.** Name the lock or the atomic operation
and the two operations it now covers. Paste the diff of your change and the
passing run of your milestone-1 test and the full suite.

Option 2, an atomic check-and-insert in the store. `BookingStore` gets a plain
(not `default`) method, `boolean insertIfFree(Booking)`.
`InMemoryBookingStore.insertIfFree` is `synchronized` on the store, the monitor
every other store method already takes, and runs the two operations inside
that one lock: the overlap check (`overlapsConfirmed`) and the write
(`bookings.add`). `book()` now builds the booking and makes that one call
instead of `hasOverlap` and then `insert`. Adding the interface method broke
the build as expected (`PausingStore is not abstract and does not override
abstract method insertIfFree`, and the same for `InMemoryBookingStore`);
`PausingStore` now forwards it through its gate in three lines, so it can be
aimed at. The `BookingService` class comment no longer claims that operations
built on a thread-safe store are thread safe.

```diff
diff --git a/src/main/java/edu/cmu/cs214/lab10/harness/PausingStore.java b/src/main/java/edu/cmu/cs214/lab10/harness/PausingStore.java
index 5e6b639..9c65544 100644
--- a/src/main/java/edu/cmu/cs214/lab10/harness/PausingStore.java
+++ b/src/main/java/edu/cmu/cs214/lab10/harness/PausingStore.java
@@ -53,8 +53,8 @@ public final class PausingStore implements BookingStore {
     /**
      * @param delegate     the real store, which does all the work
      * @param targetMethod the {@link BookingStore} method to park inside, by
-     *     name: {@code hasOverlap}, {@code insert}, {@code bookingsForRoom}, or
-     *     {@code cancelAndPromote}
+     *     name: {@code hasOverlap}, {@code insert}, {@code insertIfFree},
+     *     {@code bookingsForRoom}, or {@code cancelAndPromote}
      * @param targetCall   which invocation of that method to park, counting
      *     from 1 across all threads
      */
@@ -109,6 +109,11 @@ public final class PausingStore implements BookingStore {
         });
     }
 
+    @Override
+    public boolean insertIfFree(Booking booking) {
+        return gate("insertIfFree", () -> delegate.insertIfFree(booking));
+    }
+
     @Override
     public List<Booking> bookingsForRoom(String roomId) {
         return gate("bookingsForRoom", () -> delegate.bookingsForRoom(roomId));
diff --git a/src/main/java/edu/cmu/cs214/lab10/service/BookingService.java b/src/main/java/edu/cmu/cs214/lab10/service/BookingService.java
index 514c3e1..3297dd8 100644
--- a/src/main/java/edu/cmu/cs214/lab10/service/BookingService.java
+++ b/src/main/java/edu/cmu/cs214/lab10/service/BookingService.java
@@ -22,10 +22,12 @@ import java.util.concurrent.atomic.AtomicLong;
  * <p>Thread safety: safe for concurrent use. The service owns no mutable
  * booking state of its own. The room catalog is built once in the constructor
  * and never written again, and every read and write of booking data goes
- * through {@link BookingStore}, whose methods are each synchronized. Since the
- * store is thread safe, the operations built on top of it are too. The only
- * other field that changes after construction is {@link BookingStats}, and its
- * numbers are documented as approximate.
+ * through {@link BookingStore}, whose methods are each atomic on their own. Two
+ * store calls in a row are not atomic together, so an operation that must
+ * check and then write makes it one store call ({@link #book} uses
+ * {@link BookingStore#insertIfFree}). The only other field that changes after
+ * construction is {@link BookingStats}, and its numbers are documented as
+ * approximate.
  */
 public final class BookingService {
 
@@ -67,11 +69,12 @@ public final class BookingService {
         if (startMinute < 0 || endMinute <= startMinute) {
             return reject(roomId, startMinute, endMinute, "empty or reversed slot");
         }
-        if (store.hasOverlap(roomId, startMinute, endMinute)) {
+        // One store call checks and writes, so no other booking can land between
+        // the two. The id is taken first, so a rejected attempt uses one up.
+        Booking booking = new Booking(nextId(), roomId, startMinute, endMinute, BookingStatus.CONFIRMED);
+        if (!store.insertIfFree(booking)) {
             return reject(roomId, startMinute, endMinute, "slot already held");
         }
-        Booking booking = new Booking(nextId(), roomId, startMinute, endMinute, BookingStatus.CONFIRMED);
-        store.insert(booking);
         stats.recordConfirmed();
         BookingLog.confirmed(booking);
         return BookingResult.confirmed(booking);
diff --git a/src/main/java/edu/cmu/cs214/lab10/store/BookingStore.java b/src/main/java/edu/cmu/cs214/lab10/store/BookingStore.java
index 3ae3544..6ff0c77 100644
--- a/src/main/java/edu/cmu/cs214/lab10/store/BookingStore.java
+++ b/src/main/java/edu/cmu/cs214/lab10/store/BookingStore.java
@@ -28,6 +28,18 @@ public interface BookingStore {
     /** Stores the booking. Ids are assumed unique; the store does not check. */
     void insert(Booking booking);
 
+    /**
+     * Stores the booking only if no CONFIRMED booking for the same room touches
+     * its half-open range, as one indivisible step: no other call on the store
+     * can run between the check and the write.
+     *
+     * @param booking the booking to store; ids are assumed unique, as for
+     *     {@link #insert}
+     * @return true when the booking was stored, false when the range was taken
+     *     and nothing was stored
+     */
+    boolean insertIfFree(Booking booking);
+
     /**
      * Every booking held for the room, in insertion order, in every status.
      *
diff --git a/src/main/java/edu/cmu/cs214/lab10/store/InMemoryBookingStore.java b/src/main/java/edu/cmu/cs214/lab10/store/InMemoryBookingStore.java
index f7fd85f..4ab6b91 100644
--- a/src/main/java/edu/cmu/cs214/lab10/store/InMemoryBookingStore.java
+++ b/src/main/java/edu/cmu/cs214/lab10/store/InMemoryBookingStore.java
@@ -36,6 +36,17 @@ public final class InMemoryBookingStore implements BookingStore {
         bookings.add(booking);
     }
 
+    @Override
+    public synchronized boolean insertIfFree(Booking booking) {
+        // Check and write under the one lock every other method takes, so no
+        // insert or promotion can land between them.
+        if (overlapsConfirmed(booking)) {
+            return false;
+        }
+        bookings.add(booking);
+        return true;
+    }
+
     @Override
     public synchronized List<Booking> bookingsForRoom(String roomId) {
         List<Booking> forRoom = new ArrayList<>();
```

Milestone-1 test, two runs (`mvn -B test -Dtest='RaceTest#twoCustomersBookingTheSameSlotConfirmOnlyOne'`):

```
2026-11-06T14:58:21.459Z INFO booking.rejected room=WEH-5409 start=540 end=600 reason=slot already held
2026-11-06T14:58:21.463Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.040 s -- in edu.cmu.cs214.lab10.harness.RaceTest
[INFO] BUILD SUCCESS
```

```
2026-11-06T14:58:23.206Z INFO booking.rejected room=WEH-5409 start=540 end=600 reason=slot already held
2026-11-06T14:58:23.208Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.038 s -- in edu.cmu.cs214.lab10.harness.RaceTest
[INFO] BUILD SUCCESS
```

Full suite (`mvn -B clean test`):

```
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.036 s -- in edu.cmu.cs214.lab10.harness.RaceTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.012 s -- in edu.cmu.cs214.lab10.service.BookingServiceTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.003 s -- in edu.cmu.cs214.lab10.domain.BookingTest
[INFO] Tests run: 12, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

The 12 are the 11 shipped tests and the milestone-1 test. The milestone-2 test
is deleted: it was aimed at the second `hasOverlap` call that only the
rejected patch makes, so on the shipped code it can only fail on its gate.
Its runs are recorded in milestone 2 and in commit `0e04719`.

**Is your milestone-1 test still testing anything?** After your fix, describe
what that test now exercises. Did `PausingStore` pause a call during the green run? How do
you know?

The fix changed what the test can observe, so I adjusted it. With its
milestone-1 aim (`hasOverlap`, call 1) it failed on the fixed code with `A was
never parked in its overlap check`, since `book()` no longer calls
`hasOverlap`. It now aims at `insertIfFree`, call 1; the two customers, the
slot, and the end-state assertion are unchanged.

Yes, it paused A. The test asserts that `awaitPaused()` returned true, which
is the assertion that failed under the old aim. The log shows it too: B's
rejection comes before A's confirmation, although A's store call ran first,
because A logs only after it is released.

What it exercises now: A parked at the only point left between its store work
and the rest of `book()`, after its check and its write have both happened,
and a second customer for the same slot arriving during that pause and being
rejected. What it cannot show is B waiting on the lock. The gate parks A after
`insertIfFree` returns, when the store's lock is already released, so B never
blocks. That the check and the write cannot be split rests on `synchronized`
inside the store, where no aim of `PausingStore` reaches.

**What runs one at a time now that ran in parallel before.** Name concretely which callers
queue behind which. "Nothing, it is just correct now" does not count as an
answer.

Two bookers' check and write. Before, B's `hasOverlap` could run between A's
`hasOverlap` and A's `insert`. Now B's `insertIfFree` waits on the store's
monitor until A's check and write have both finished. The same goes for every
other store call, from any thread and for any room: a `listBookings` reader
(`bookingsForRoom`), a `cancelBooking` (`cancelAndPromote`), a `joinWaitlist`
insert, and a booker for GHC-4301 all queue behind a WEH-5409 booking's check
and write as one block, where before they could run between its two halves.
Each store call already ran one at a time; what is new is that the lock is
held across the check and the write together. Validation, id allocation,
stats, and the log line in `book()` stay outside the lock and still run in
parallel.

**What you left unfixed.** For anything your review flagged that you chose not
to fix, one line each on why.

- `BookingStats` lost updates (suspect 2): documented as approximate, nothing
  in the booking logic reads them, and they are not the ticket's invariant.
- Per-instance `sequence` (suspect 3): two service instances over one store
  can still hand out the same id, and the store does not check ids. The
  ticket's ids were distinct; fixing it means moving id allocation into the
  shared store, which is a separate change.
- Ids used up by rejected attempts (new with this fix): `book()` takes an id
  before `insertIfFree`, so a rejection leaves a gap. Ids stay unique, and
  nothing depends on them being consecutive.
- `hasOverlap` stays on `BookingStore`, though `book()` no longer calls it.
  Removing it changes the interface beyond this fix; a future caller that
  pairs it with `insert` would reopen the same check-then-act gap.
- Not unfixed, but untested: the cancel path. `cancelAndPromote` takes the
  same store lock as `insertIfFree`, so a promotion can no longer land between
  a booking's check and its write. That comes from reading the code; no test
  drives it.
