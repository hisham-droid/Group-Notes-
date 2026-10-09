# Task 5: Unit testing

## What it is

Two test classes, using **JUnit 4**:

1. **`ModerationToolsAddReportTests` (40%)**: tests that reach **every branch** of `addReport`, plus every branch in the helper methods it calls that your team wrote.
2. **`ModerationToolsGetReportsTests` (40%)**: **black-box** tests for `getReportedMessages`. They must catch a buggy version and pass on a correct one, using only the spec.

Plus 20% for clean test code.

## The two kinds of testing

**Branch coverage (white-box).** A branch is every place the code can go two ways: each `if`, each loop condition, and each `true`/`false` result of a comparison. Branch-complete means your tests make **every one** go both ways at least once. You write these tests by **looking at the code** and aiming a test at each path.

**Black-box.** You pretend you **can't see the code**. You only read the spec and ask "what could a programmer get wrong?" Then you write tests that would catch each mistake. A good black-box test fails on a buggy implementation and passes on any correct one, including ones written differently from yours.

## First: one change to Task 1's `addReport`

The old `addReport` checked `hasReported` first, then later called `store.add(...)`, which checks for duplicates **again** inside `MessageReports.add` (`putIfAbsent(...) == null`). Because the first check already caught duplicates, the second check's "already there" path could **never** happen. That's a branch no test can reach, so 100% branch coverage was impossible. Dead code is also a code smell.

The fix is to remove the early check and let `MessageReports.add` be the **one** place duplicates are detected. In `moderation/ModerationTools.java`, replace `addReport` with:

```java
	/**
	 * Records that a user has reported a message.
	 * @return true if the report was recorded; false if the message or user
	 * does not exist, or the user has already reported this message
	 */
	public static boolean addReport(UUID message, UUID user, long timestamp) {
		if (UserDAO.getInstance().getByUUID(user) == null) return false;

		ReportStore store = ReportStore.getInstance();
		Message target = findMessage(store, message);
		if (target == null) return false;

		return store.add(target, user, timestamp);
	}
```

Same behaviour, simpler code, and every branch is now reachable. All the Task 1–4 checks still pass with this change.

## Part A: `ModerationToolsAddReportTests` (branch coverage)

### Which branches exist, and which test covers each

`addReport` calls these methods your team wrote: `findMessage`, `PostDAO.getMessageByUUID`, `ReportStore.add` and `MessageReports.add`. `UserDAO.getByUUID` and `PostDAO.getAllMessages` were already in the project, so they don't count.

| Method | Branch | Taken by test |
|---|---|---|
| `addReport` | user doesn't exist → `return false` | `reportFromUnknownUserIsRejected` |
| | user exists → continue | every other test |
| | message not found → `return false` | `reportOnUnknownMessageIsRejected` |
| | message found → continue | `firstReportOnMessageIsAccepted` |
| `findMessage` | already reported → reuse stored message | `secondUserCanReportSameMessage` |
| | not yet reported → search the posts | `firstReportOnMessageIsAccepted` |
| `getMessageByUUID` | loop: more messages → keep looping | `reportOnUnknownMessageIsRejected` |
| | loop: no more messages → stop | `reportOnUnknownMessageIsRejected` |
| | loop body never runs (no messages at all) | `reportIsRejectedWhenNoMessagesExist` |
| | ID matches → return it | `firstReportOnMessageIsAccepted` |
| | ID doesn't match → next message | `reportOnUnknownMessageIsRejected` |
| `MessageReports.add` | new report → `true` | `firstReportOnMessageIsAccepted` |
| | user already reported → `false` | `duplicateReportIsRejectedAndChangesNothing` |
| `ReportStore.add` | no `if`s, but runs on both a new message and an existing one | first-report and second-user tests |

### `app/test/ModerationToolsAddReportTests.java`

```java
import dao.PostDAO;
import dao.UserDAO;
import dao.model.Message;
import dao.model.Post;
import dao.model.User;
import moderation.ModerationTools;
import moderation.ReportStore;
import org.junit.Before;
import org.junit.Test;

import java.util.UUID;

import static org.junit.Assert.assertEquals;
import static org.junit.Assert.assertFalse;
import static org.junit.Assert.assertNull;
import static org.junit.Assert.assertTrue;

/**
 * Branch-complete tests for ModerationTools.addReport, including the methods it calls
 * that were written for the hackathon: ModerationTools.findMessage,
 * PostDAO.getMessageByUUID, ReportStore.add and MessageReports.add.
 */
public class ModerationToolsAddReportTests {
	private User alice;
	private User bob;
	private Message firstMessage;
	private Message secondMessage;

	@Before
	public void setUp() {
		UserDAO.getInstance().clear();
		PostDAO.getInstance().clear();
		ReportStore.getInstance().clear();

		alice = addUser("alice");
		bob = addUser("bobby");

		Post post = new Post(UUID.randomUUID(), alice.id(), "A topic");
		PostDAO.getInstance().add(post);
		firstMessage = addMessage(post, 100);
		secondMessage = addMessage(post, 200);
	}

	private static User addUser(String username) {
		User user = new User(UUID.randomUUID(), User.Role.Member, username, "password");
		UserDAO.getInstance().add(user);
		return user;
	}

	private static Message addMessage(Post post, long timestamp) {
		Message message = new Message(UUID.randomUUID(), post.poster, post.id, timestamp, "Hello");
		post.messages.insert(message);
		return message;
	}

	// addReport: the user does not exist
	@Test
	public void reportFromUnknownUserIsRejected() {
		assertFalse(ModerationTools.addReport(firstMessage.id(), UUID.randomUUID(), 10));
		assertNull("no report should be stored", ReportStore.getInstance().get(firstMessage.id()));
	}

	// addReport: the message does not exist
	// getMessageByUUID: checks every message without finding a match, then the loop ends
	@Test
	public void reportOnUnknownMessageIsRejected() {
		UUID missing = UUID.randomUUID();
		assertFalse(ModerationTools.addReport(missing, alice.id(), 10));
		assertNull("no report should be stored", ReportStore.getInstance().get(missing));
	}

	// getMessageByUUID: there are no messages at all, so the loop body never runs
	@Test
	public void reportIsRejectedWhenNoMessagesExist() {
		PostDAO.getInstance().clear();
		assertFalse(ModerationTools.addReport(firstMessage.id(), alice.id(), 10));
	}

	// findMessage: the message has no reports yet, so it is found by searching the posts
	// getMessageByUUID: finds a match
	// MessageReports.add: the user has not reported this message before
	@Test
	public void firstReportOnMessageIsAccepted() {
		assertTrue(ModerationTools.addReport(firstMessage.id(), alice.id(), 10));
		assertTrue(ModerationTools.hasReported(firstMessage.id(), alice.id()));
	}

	// findMessage: the message already has a report, so the stored copy is reused
	@Test
	public void secondUserCanReportSameMessage() {
		assertTrue(ModerationTools.addReport(firstMessage.id(), alice.id(), 10));
		assertTrue(ModerationTools.addReport(firstMessage.id(), bob.id(), 20));
		assertTrue(ModerationTools.hasReported(firstMessage.id(), bob.id()));
		assertEquals(2, ReportStore.getInstance().get(firstMessage.id()).count());
	}

	// MessageReports.add: the user has already reported this message
	@Test
	public void duplicateReportIsRejectedAndChangesNothing() {
		assertTrue(ModerationTools.addReport(firstMessage.id(), alice.id(), 10));
		assertFalse(ModerationTools.addReport(firstMessage.id(), alice.id(), 5));
		assertEquals(1, ReportStore.getInstance().get(firstMessage.id()).count());
		assertEquals("the original report should be kept", 10, ReportStore.getInstance().get(firstMessage.id()).oldestTimestamp());
	}

	@Test
	public void userCanReportDifferentMessages() {
		assertTrue(ModerationTools.addReport(firstMessage.id(), alice.id(), 10));
		assertTrue(ModerationTools.addReport(secondMessage.id(), alice.id(), 20));
		assertTrue(ModerationTools.hasReported(secondMessage.id(), alice.id()));
	}

	@Test
	public void userCanReportAgainAfterRemovingTheirReport() {
		assertTrue(ModerationTools.addReport(firstMessage.id(), alice.id(), 10));
		assertTrue(ModerationTools.removeReport(firstMessage.id(), alice.id(), 15));
		assertTrue(ModerationTools.addReport(firstMessage.id(), alice.id(), 20));
	}
}
```

How it's put together:

- **`@Before setUp()` runs before every test.** `UserDAO`, `PostDAO` and `ReportStore` are **singletons**, so data would otherwise leak from one test into the next. Clearing them first makes every test start from the same clean state.
- **Comments above each test say which branch it covers.** This helps the marker see you went for branch coverage on purpose.
- **"Do nothing" is checked too, not just the `false` return.** For example, the duplicate test checks the count is still 1 and the original timestamp (10) wasn't replaced by 5. The spec says "do nothing **and** return false."

## Part B: `ModerationToolsGetReportsTests` (black-box)

### How the tests were planned

I listed every rule in the Task 4 spec and asked "how could someone get this wrong?"

| Spec rule | Possible mistake | Test(s) |
|---|---|---|
| Throw for strategy other than "OLDEST"/"MOST" | accepts unknown strings, lowercase, or `null` | `unknownStrategyThrows`, `strategyIsCaseSensitive`, `nullStrategyThrows` |
| Throw if amount not positive | allows 0 (`<` instead of `<=`), or negatives | `zeroAmountThrows`, `negativeAmountThrows` |
| MOST: most reports first | sorted backwards | `mostPutsMostReportedFirst` |
| OLDEST: earliest report first | sorted backwards, or by the **message's** time | `oldestPutsEarliestReportFirst` |
| OLDEST uses each message's **oldest** report | uses the newest, or the first one added | `oldestUsesEachMessagesEarliestReport` |
| Strategies differ | both strategies give the same order | `strategiesProduceDifferentOrders` |
| Return `amount` messages | ignores `amount`, off by one | `amountLimitsTheNumberOfResults`, `amountOfOneReturnsOnlyTheTopMessage`, `amountMatchingTheNumberReportedReturnsAll` |
| Fewer if not enough | crashes or pads with nulls | `fewerReportedMessagesThanAmountReturnsAllOfThem`, `noReportsGivesNoResults` |
| Each message at most once | one entry per report | `messageReportedManyTimesAppearsOnce` |
| No 0-report messages | removed reports leave the message listed | `messageWhoseReportsWereAllRemovedIsExcluded` |
| Only **active** reports count | removed reports still counted / still "oldest" | `removedReportsDoNotCountTowardsMost`, `removedReportIsIgnoredWhenFindingTheOldest` |
| Ties in any order | a tied message is lost | `tiedCountsAreAllReturned`, `tiedOldestTimestampsAreAllReturned` |

Three deliberate choices:

1. **`assertThrows(Exception.class, ...)` rather than `IllegalArgumentException.class`.** The spec only says "throw an exception," not which kind. A black-box test must accept *any* correct implementation, so it accepts any exception.
2. **The messages' own timestamps run backwards** (a = 3000, b = 2000, c = 1000) compared to the report times used in the tests. A buggy version that sorts by when the *message* was posted instead of when it was *reported* gets the wrong order and fails.
3. **Ties are checked with a `Set`.** The spec says tied messages can come in any order, so the test checks "the first two are a and b, in some order," not a fixed order. Otherwise a correct implementation could fail just for breaking the tie differently.

### `app/test/ModerationToolsGetReportsTests.java`

```java
import dao.PostDAO;
import dao.UserDAO;
import dao.model.Message;
import dao.model.Post;
import dao.model.User;
import moderation.ModerationTools;
import moderation.ReportStore;
import org.junit.Before;
import org.junit.Test;

import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;
import java.util.Set;
import java.util.UUID;

import static org.junit.Assert.assertEquals;
import static org.junit.Assert.assertThrows;
import static org.junit.Assert.assertTrue;

/**
 * Black-box tests for ModerationTools.getReportedMessages, based only on its specification.
 * Reports are set up with addReport and removeReport, which are assumed to be correct.
 */
public class ModerationToolsGetReportsTests {
	private final User[] users = new User[4];
	private Message a;
	private Message b;
	private Message c;

	@Before
	public void setUp() {
		UserDAO.getInstance().clear();
		PostDAO.getInstance().clear();
		ReportStore.getInstance().clear();

		for (int i = 0; i < users.length; i++) {
			users[i] = new User(UUID.randomUUID(), User.Role.Member, "user" + i, "password");
			UserDAO.getInstance().add(users[i]);
		}

		Post post = new Post(UUID.randomUUID(), users[0].id(), "A topic");
		PostDAO.getInstance().add(post);
		// Message timestamps run opposite to most expected orders below, so an
		// implementation that sorts by message time instead of report data will fail.
		a = addMessage(post, 3000);
		b = addMessage(post, 2000);
		c = addMessage(post, 1000);
		addMessage(post, 500); // never reported, so it must never be returned
	}

	private static Message addMessage(Post post, long timestamp) {
		Message message = new Message(UUID.randomUUID(), post.poster, post.id, timestamp, "Hello");
		post.messages.insert(message);
		return message;
	}

	private void report(Message message, int user, long timestamp) {
		assertTrue("test setup failed", ModerationTools.addReport(message.id(), users[user].id(), timestamp));
	}

	private void unreport(Message message, int user) {
		assertTrue("test setup failed", ModerationTools.removeReport(message.id(), users[user].id(), 0));
	}

	private static List<Message> get(String strategy, int amount) {
		List<Message> result = new ArrayList<>();
		Iterator<Message> it = ModerationTools.getReportedMessages(strategy, amount);
		while (it.hasNext()) result.add(it.next());
		return result;
	}

	// ---- Invalid input ----

	@Test
	public void unknownStrategyThrows() {
		assertThrows(Exception.class, () -> ModerationTools.getReportedMessages("NEWEST", 5));
	}

	@Test
	public void strategyIsCaseSensitive() {
		assertThrows(Exception.class, () -> ModerationTools.getReportedMessages("most", 5));
		assertThrows(Exception.class, () -> ModerationTools.getReportedMessages("Oldest", 5));
	}

	@Test
	public void nullStrategyThrows() {
		assertThrows(Exception.class, () -> ModerationTools.getReportedMessages(null, 5));
	}

	@Test
	public void zeroAmountThrows() {
		assertThrows(Exception.class, () -> ModerationTools.getReportedMessages("MOST", 0));
		assertThrows(Exception.class, () -> ModerationTools.getReportedMessages("OLDEST", 0));
	}

	@Test
	public void negativeAmountThrows() {
		assertThrows(Exception.class, () -> ModerationTools.getReportedMessages("MOST", -1));
		assertThrows(Exception.class, () -> ModerationTools.getReportedMessages("OLDEST", -1));
	}

	// ---- Ordering ----

	@Test
	public void mostPutsMostReportedFirst() {
		report(a, 0, 10);
		report(b, 0, 20);
		report(b, 1, 30);
		report(b, 2, 40);
		report(c, 0, 50);
		report(c, 1, 60);
		assertEquals(List.of(b, c, a), get("MOST", 10));
	}

	@Test
	public void oldestPutsEarliestReportFirst() {
		report(a, 0, 30);
		report(b, 0, 10);
		report(c, 0, 20);
		assertEquals(List.of(b, c, a), get("OLDEST", 10));
	}

	@Test
	public void oldestUsesEachMessagesEarliestReport() {
		// a's latest report is the newest overall, and its earliest is added second
		report(a, 0, 100);
		report(a, 1, 5);
		report(b, 0, 50);
		assertEquals(List.of(a, b), get("OLDEST", 10));
	}

	@Test
	public void strategiesProduceDifferentOrders() {
		report(a, 0, 10);
		report(b, 0, 20);
		report(b, 1, 30);
		report(b, 2, 40);
		assertEquals(List.of(a, b), get("OLDEST", 10));
		assertEquals(List.of(b, a), get("MOST", 10));
	}

	// ---- Amount ----

	@Test
	public void amountLimitsTheNumberOfResults() {
		report(a, 0, 10);
		report(b, 0, 20);
		report(b, 1, 30);
		report(c, 0, 40);
		report(c, 1, 50);
		report(c, 2, 60);
		assertEquals(List.of(c, b), get("MOST", 2));
		assertEquals(List.of(a, b), get("OLDEST", 2));
	}

	@Test
	public void amountOfOneReturnsOnlyTheTopMessage() {
		report(a, 0, 10);
		report(b, 0, 20);
		report(b, 1, 30);
		assertEquals(List.of(b), get("MOST", 1));
		assertEquals(List.of(a), get("OLDEST", 1));
	}

	@Test
	public void amountMatchingTheNumberReportedReturnsAll() {
		report(a, 0, 10);
		report(b, 0, 20);
		assertEquals(List.of(a, b), get("OLDEST", 2));
	}

	@Test
	public void fewerReportedMessagesThanAmountReturnsAllOfThem() {
		report(a, 0, 10);
		report(b, 0, 20);
		assertEquals(List.of(a, b), get("OLDEST", 10));
	}

	@Test
	public void noReportsGivesNoResults() {
		assertEquals(List.of(), get("MOST", 5));
		assertEquals(List.of(), get("OLDEST", 5));
	}

	// ---- Which messages are included ----

	@Test
	public void messageReportedManyTimesAppearsOnce() {
		report(a, 0, 10);
		report(a, 1, 20);
		report(a, 2, 30);
		assertEquals(List.of(a), get("MOST", 10));
		assertEquals(List.of(a), get("OLDEST", 10));
	}

	@Test
	public void messageWhoseReportsWereAllRemovedIsExcluded() {
		report(a, 0, 10);
		report(a, 1, 20);
		report(b, 0, 30);
		unreport(a, 0);
		unreport(a, 1);
		assertEquals(List.of(b), get("MOST", 10));
		assertEquals(List.of(b), get("OLDEST", 10));
	}

	@Test
	public void removedReportsDoNotCountTowardsMost() {
		report(a, 0, 10);
		report(a, 1, 20);
		report(a, 2, 30);
		report(b, 0, 40);
		report(b, 1, 50);
		unreport(a, 0);
		unreport(a, 1);
		assertEquals(List.of(b, a), get("MOST", 10));
	}

	@Test
	public void removedReportIsIgnoredWhenFindingTheOldest() {
		report(a, 0, 10);
		report(a, 1, 90);
		report(b, 0, 50);
		unreport(a, 0);
		assertEquals(List.of(b, a), get("OLDEST", 10));
	}

	// ---- Ties (any order is allowed, but nothing may be lost) ----

	@Test
	public void tiedCountsAreAllReturned() {
		report(a, 0, 10);
		report(a, 1, 20);
		report(b, 0, 30);
		report(b, 1, 40);
		report(c, 0, 50);
		List<Message> result = get("MOST", 10);
		assertEquals(3, result.size());
		assertEquals(Set.of(a, b), Set.copyOf(result.subList(0, 2)));
		assertEquals(c, result.get(2));
	}

	@Test
	public void tiedOldestTimestampsAreAllReturned() {
		report(a, 0, 10);
		report(b, 0, 10);
		report(c, 0, 20);
		List<Message> result = get("OLDEST", 10);
		assertEquals(3, result.size());
		assertEquals(Set.of(a, b), Set.copyOf(result.subList(0, 2)));
		assertEquals(c, result.get(2));
	}
}
```

How it's put together:

- **Helpers keep each test short and readable.** `report(a, 0, 10)` reads as "user 0 reported a at time 10." `get("MOST", 10)` returns the results as a list, so a whole expected order can be checked in one `assertEquals`.
- **The helpers check their own setup.** If `addReport` ever fails during setup, the test stops with "test setup failed." Then a broken setup isn't mistaken for a bug in `getReportedMessages`.
- **Only `getReportedMessages` is checked.** `addReport`/`removeReport` are used only to set up the situation, as the spec allows ("assuming that all other functionality is correct").
- **An extra message that's never reported** sits in the post. Every test that checks an exact list would fail if it wrongly showed up.

## Proof the black-box tests work

I planted each of these bugs in a copy of Task 4, one at a time, and ran the tests. **All 15 were caught:**

| Planted bug | Tests that caught it |
|---|---|
| MOST sorted fewest-first (forgot `.reversed()`) | 6 |
| OLDEST sorted newest-first | 9 |
| OLDEST uses the message's posting time | 8 |
| OLDEST uses the latest report instead of the earliest | 1 (`oldestUsesEachMessagesEarliestReport`) |
| `amount` ignored | 2 |
| Off by one (returns `amount - 1`) | 3 |
| `amount = 0` allowed (`<` instead of `<=`) | 1 |
| No amount check at all | 2 |
| Strategy case-insensitive | 1 |
| Unknown strategy silently treated as MOST | 2 |
| `null` strategy treated as MOST | 1 |
| Ties dropped (the `TreeSet`/`SortedData` trap) | 2 |
| Messages with 0 reports still listed | 1 |
| A message returned more than once | 11 |
| Results not sorted at all | 9 |

Several bugs are caught by **only one** test, which shows each test earns its place.

## Running them in IntelliJ

- The `.iml` already marks `app/test` as the test folder and adds `junit-4.13.jar` and `hamcrest-core-1.3.jar` from `../miniproject/`. Make sure those jars are actually at that path on your machine, like in the miniproject setup.
- Right-click the `test` folder → **Run 'All Tests'**.
- To confirm Part A's branch coverage, use **Run with Coverage** on `ModerationToolsAddReportTests`, with branch coverage turned on in the coverage settings (IntelliJ shows only line coverage by default). The methods in the table should all show full branch coverage.
