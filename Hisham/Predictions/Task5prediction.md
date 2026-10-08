# Task 5 prediction: unit testing (white-box + black-box)

> Every test class below (7 classes, 67 tests) passes on the correct implementation, and **39 deliberately broken
> versions** of the code were each caught by at least one test (table in section 4).

## 1. What Task 5 looks like (from the practice hackathon)

| Part | Marks | What earns it |
|---|---|---|
| White-box: **branch-complete coverage** of one method (`addReport`) **and the helpers your team wrote that it calls** | 40% | every `if` / `&&` / `||` / ternary / loop outcome executed at least once |
| Black-box: tests for another method (`getReportedMessages`) that tell a correct implementation from faulty ones | 40% | each likely bug makes some test fail; the correct version passes all |
| Quality | 20% | JUnit4 annotations, `@Before` reset, clear names and messages, no copy-paste |

Predicted pairs for tomorrow: white-box on the Task 1 "add" method (`addReaction`, `addReport`, `follow`, `vote`...),
black-box on the Task 4 query (`getTopMessages`, `getReportedMessages`...).

## 2. White-box: how to get branch-complete coverage

1. Open the method and **every helper it calls that your team wrote**. List each decision.
2. For `a && b`, you need: `a` false (short-circuit), `a` true + `b` false, both true.
3. Write one small test per outcome; name it after the branch.
4. Put the branch map in the class javadoc (markers can check it quickly).

Branch map used here (`addReport` + `messageAndUserExist` + `MessageIndex.get` + `UserDAO.getByUUID` + `ReportStore.add` + `MessageReports.add`):

| Decision | Outcome | Test |
|---|---|---|
| `MessageIndex.get`: `id == null` | true | `nullMessage` |
| not in index | true | `unknownMessage` |
| `message.thread() == null` | true | `messageWithoutThread` |
| post no longer stored | true | `postRemovedFromDAO` |
| message no longer in its post | true | `messageRemovedFromPost` |
| all of the above false (found) | | `validReport` |
| `&&` second operand: user missing | `UserDAO.getByUUID` returns null | `unknownUser`, `nullUser` (`id == null` branch) |
| `MessageReports.add`: user already reported | true / false | `duplicateReport` / `validReport` |
| `ReportStore.add`: first report on message vs existing group | new / existing | `validReport` / `secondUserCanReport` |

**`app/test/ModerationToolsAddReportTests.java`** (full file)

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
import org.junit.runner.RunWith;
import org.junit.runners.JUnit4;

import java.util.UUID;

import static org.junit.Assert.assertFalse;
import static org.junit.Assert.assertTrue;

/**
 * Task 5 (white-box): branch-complete coverage of ModerationTools.addReport and every
 * helper our team wrote that it calls. Each test names the branch it covers.
 *
 * Branch map (decision -> tests):
 *  addReport:                 exists? false -> unknownMessage..., true -> validReport
 *  messageAndUserExist (&&):  message missing (short-circuit) -> unknownMessage; user missing -> unknownUser
 *  MessageIndex.get:          id == null -> nullMessage; not indexed -> unknownMessage;
 *                             thread == null -> messageWithoutThread; post gone -> postRemovedFromDAO;
 *                             message gone from post -> messageRemovedFromPost; found -> validReport
 *  UserDAO.getByUUID:         id == null -> nullUser; missing -> unknownUser; found -> validReport
 *  MessageReports.add:        new reporter -> validReport; same reporter -> duplicateReport
 *  ReportStore.add:           first report on message -> validReport; existing group -> secondUserCanReport
 */
@RunWith(JUnit4.class)
public class ModerationToolsAddReportTests {
	private User reporter;
	private User otherReporter;
	private Post post;
	private Message message;

	@Before
	public void setUp() {
		// fresh state for every test (the DAOs and store are singletons)
		UserDAO.getInstance().clear();
		PostDAO.getInstance().clear();
		ReportStore.getInstance().clear();

		reporter = new User(UUID.randomUUID(), User.Role.Member, "reporter", "password");
		otherReporter = new User(UUID.randomUUID(), User.Role.Member, "otherReporter", "password");
		UserDAO.getInstance().add(reporter);
		UserDAO.getInstance().add(otherReporter);

		post = new Post(UUID.randomUUID(), reporter.id(), "topic");
		PostDAO.getInstance().add(post);
		message = new Message(UUID.randomUUID(), reporter.id(), post.id, 1000, "hello");
		post.messages.insert(message);
	}

	@Test(timeout = 1000)
	public void validReport() {
		assertTrue("A first report on an existing message must succeed",
				ModerationTools.addReport(message.id(), reporter.id(), 2000));
		assertTrue(ModerationTools.hasReported(message.id(), reporter.id()));
	}

	@Test(timeout = 1000)
	public void secondUserCanReport() {
		ModerationTools.addReport(message.id(), reporter.id(), 2000);
		assertTrue("A different user may report the same message",
				ModerationTools.addReport(message.id(), otherReporter.id(), 2001));
	}

	@Test(timeout = 1000)
	public void duplicateReport() {
		ModerationTools.addReport(message.id(), reporter.id(), 2000);
		assertFalse("The same user cannot report the same message twice",
				ModerationTools.addReport(message.id(), reporter.id(), 3000));
	}

	@Test(timeout = 1000)
	public void unknownMessage() {
		assertFalse(ModerationTools.addReport(UUID.randomUUID(), reporter.id(), 2000));
	}

	@Test(timeout = 1000)
	public void nullMessage() {
		assertFalse(ModerationTools.addReport(null, reporter.id(), 2000));
	}

	@Test(timeout = 1000)
	public void unknownUser() {
		assertFalse(ModerationTools.addReport(message.id(), UUID.randomUUID(), 2000));
		assertFalse("A failed report must not be stored",
				ModerationTools.hasReported(message.id(), reporter.id()));
	}

	@Test(timeout = 1000)
	public void nullUser() {
		assertFalse(ModerationTools.addReport(message.id(), null, 2000));
	}

	@Test(timeout = 1000)
	public void messageWithoutThread() {
		Message orphan = new Message(UUID.randomUUID(), reporter.id(), null, 5, "no thread");
		post.messages.insert(orphan); // indexed, but it does not belong to a stored post
		assertFalse(ModerationTools.addReport(orphan.id(), reporter.id(), 2000));
	}

	@Test(timeout = 1000)
	public void postRemovedFromDAO() {
		PostDAO.getInstance().clear(); // the message's post no longer exists
		assertFalse(ModerationTools.addReport(message.id(), reporter.id(), 2000));
	}

	@Test(timeout = 1000)
	public void messageRemovedFromPost() {
		post.messages.remove(message); // post exists, message does not
		assertFalse(ModerationTools.addReport(message.id(), reporter.id(), 2000));
	}
}
```

## 3. Black-box: how to kill faulty implementations

Test **only what the spec promises**. Anything the spec leaves open (tie order, what `next()` does after the end,
lowercase strategy names) must **not** be asserted, or the correct implementation fails your test and you lose marks.

Think "what would a buggy version get wrong?", then write a test that only a correct version passes:

| Likely bug | Test that catches it |
|---|---|
| ascending instead of descending (or the reverse) | distinct counts / timestamps, assert the full order |
| strategies swapped | same data, different order for OLDEST vs MOST |
| `amount` off by one or ignored | amount 1, amount 2 of 3, amount exactly available, amount larger |
| no exception on bad input | unknown, empty, `null` strategy; amount 0 and negative (`expected = IllegalArgumentException.class`) |
| zero-report messages returned | report then retract everything on one message |
| "oldest" not updated after a retraction | retract the oldest report, check the order changes |
| counting retracted reports | retract, check MOST order |
| same message returned several times | 4 reports on one message, expect one result |
| hidden messages dropped | hide a reported message, still expected |
| `hasNext()` consumes an element | call it twice before `next()` |

**`app/test/ModerationToolsGetReportsTests.java`** (full file)

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
import org.junit.runner.RunWith;
import org.junit.runners.JUnit4;

import java.util.ArrayList;
import java.util.HashSet;
import java.util.Iterator;
import java.util.List;
import java.util.Set;
import java.util.UUID;

import static org.junit.Assert.assertEquals;
import static org.junit.Assert.assertFalse;
import static org.junit.Assert.assertTrue;

/**
 * Task 5 (black-box): tests ONLY what the Task 4 spec promises for getReportedMessages.
 * Deliberately NOT tested (unspecified, a correct implementation may differ):
 *  - the order of tied messages
 *  - what next() does after the end (NoSuchElementException is not in the spec)
 *  - case-insensitive strategy names
 */
@RunWith(JUnit4.class)
public class ModerationToolsGetReportsTests {
	private final List<User> users = new ArrayList<>();
	private final List<Message> messages = new ArrayList<>();

	@Before
	public void setUp() {
		UserDAO.getInstance().clear();
		PostDAO.getInstance().clear();
		ReportStore.getInstance().clear();
		users.clear();
		messages.clear();

		for (int i = 0; i < 5; i++) {
			User user = new User(UUID.randomUUID(), i == 0 ? User.Role.Admin : User.Role.Member, "user" + i, "password");
			UserDAO.getInstance().add(user);
			users.add(user);
		}
		Post post = new Post(UUID.randomUUID(), users.get(0).id(), "topic");
		PostDAO.getInstance().add(post);
		for (int i = 0; i < 5; i++) {
			Message message = new Message(UUID.randomUUID(), users.get(0).id(), post.id, i, "message " + i);
			post.messages.insert(message);
			messages.add(message);
		}
	}

	// ---------------------------------------------------------------- helpers

	private void report(int message, int user, long timestamp) {
		assertTrue("test setup failed", ModerationTools.addReport(messages.get(message).id(), users.get(user).id(), timestamp));
	}

	private void unreport(int message, int user) {
		assertTrue("test setup failed", ModerationTools.removeReport(messages.get(message).id(), users.get(user).id(), 0));
	}

	private List<Message> collect(String strategy, int amount) {
		List<Message> result = new ArrayList<>();
		Iterator<Message> iterator = ModerationTools.getReportedMessages(strategy, amount);
		while (iterator.hasNext()) result.add(iterator.next());
		return result;
	}

	private List<Message> expected(int... indices) {
		List<Message> result = new ArrayList<>();
		for (int i : indices) result.add(messages.get(i));
		return result;
	}

	// ---------------------------------------------------------------- invalid input

	@Test(timeout = 1000, expected = IllegalArgumentException.class)
	public void unknownStrategyThrows() {
		ModerationTools.getReportedMessages("NEWEST", 3);
	}

	@Test(timeout = 1000, expected = IllegalArgumentException.class)
	public void nullStrategyThrows() {
		ModerationTools.getReportedMessages(null, 3);
	}

	@Test(timeout = 1000, expected = IllegalArgumentException.class)
	public void emptyStrategyThrows() {
		ModerationTools.getReportedMessages("", 3);
	}

	@Test(timeout = 1000, expected = IllegalArgumentException.class)
	public void zeroAmountThrows() {
		report(0, 1, 10);
		ModerationTools.getReportedMessages("MOST", 0);
	}

	@Test(timeout = 1000, expected = IllegalArgumentException.class)
	public void negativeAmountThrows() {
		report(0, 1, 10);
		ModerationTools.getReportedMessages("OLDEST", -1);
	}

	// ---------------------------------------------------------------- empty results

	@Test(timeout = 1000)
	public void noReportsGivesEmptyIterator() {
		assertTrue(collect("MOST", 3).isEmpty());
		assertTrue(collect("OLDEST", 3).isEmpty());
	}

	@Test(timeout = 1000)
	public void fullyRetractedMessageIsExcluded() {
		report(0, 1, 10);
		report(1, 1, 20);
		unreport(0, 1);
		assertEquals(expected(1), collect("MOST", 5));
		assertEquals(expected(1), collect("OLDEST", 5));
	}

	// ---------------------------------------------------------------- MOST

	@Test(timeout = 1000)
	public void mostOrdersByActiveReportCount() {
		report(2, 1, 10);                                    // 1 report
		report(0, 1, 10); report(0, 2, 10); report(0, 3, 10); // 3 reports
		report(4, 1, 10); report(4, 2, 10);                   // 2 reports
		assertEquals(expected(0, 4, 2), collect("MOST", 5));
	}

	@Test(timeout = 1000)
	public void mostIgnoresRetractedReports() {
		report(0, 1, 10); report(0, 2, 10); report(0, 3, 10); // 3, then 1 after retractions
		report(1, 1, 10); report(1, 2, 10);                   // 2
		unreport(0, 2);
		unreport(0, 3);
		assertEquals(expected(1, 0), collect("MOST", 5));
	}

	// ---------------------------------------------------------------- OLDEST

	@Test(timeout = 1000)
	public void oldestOrdersByOldestReport() {
		report(0, 1, 300);
		report(1, 1, 100);
		report(2, 1, 200);
		report(2, 2, 50); // message 2's oldest report is 50
		assertEquals(expected(2, 1, 0), collect("OLDEST", 5));
	}

	@Test(timeout = 1000)
	public void oldestUsesOldestActiveReport() {
		report(0, 1, 100);
		report(1, 1, 50);
		report(1, 2, 500);
		unreport(1, 1); // message 1's oldest ACTIVE report is now 500
		assertEquals(expected(0, 1), collect("OLDEST", 5));
	}

	@Test(timeout = 1000)
	public void strategiesGiveDifferentOrders() {
		report(0, 1, 10);                                      // oldest, fewest
		report(1, 1, 20); report(1, 2, 21); report(1, 3, 22);  // newest, most
		assertEquals(expected(0, 1), collect("OLDEST", 5));
		assertEquals(expected(1, 0), collect("MOST", 5));
	}

	// ---------------------------------------------------------------- amount

	@Test(timeout = 1000)
	public void amountLimitsResults() {
		report(0, 1, 10); report(0, 2, 11); report(0, 3, 12);
		report(1, 1, 20); report(1, 2, 21);
		report(2, 1, 30);
		assertEquals(expected(0), collect("MOST", 1));
		assertEquals(expected(0, 1), collect("MOST", 2));
		assertEquals(expected(0, 1), collect("OLDEST", 2));
	}

	@Test(timeout = 1000)
	public void amountLargerThanAvailableReturnsAll() {
		report(3, 1, 10);
		report(4, 1, 20);
		assertEquals(expected(3, 4), collect("OLDEST", 100));
	}

	@Test(timeout = 1000)
	public void amountExactlyAvailable() {
		report(3, 1, 10);
		report(4, 1, 20);
		assertEquals(expected(3, 4), collect("OLDEST", 2));
	}

	// ---------------------------------------------------------------- uniqueness, ties, hidden

	@Test(timeout = 1000)
	public void messageReportedManyTimesAppearsOnce() {
		for (int user = 1; user < 5; user++) report(0, user, 10 + user);
		assertEquals(expected(0), collect("MOST", 10));
		assertEquals(expected(0), collect("OLDEST", 10));
	}

	@Test(timeout = 1000)
	public void tiesMayBeInAnyOrder() {
		report(0, 1, 10); report(1, 1, 10); report(2, 1, 10); // same count AND same timestamp
		Set<Message> most = new HashSet<>(collect("MOST", 5));
		assertEquals(new HashSet<>(expected(0, 1, 2)), most);
		assertEquals(3, collect("OLDEST", 5).size());
	}

	@Test(timeout = 1000)
	public void hiddenMessagesAreStillReported() {
		report(0, 1, 10);
		assertTrue(ModerationTools.setHidden(messages.get(0).id(), users.get(0).id(), true));
		assertEquals(expected(0), collect("MOST", 5));
	}

	@Test(timeout = 1000)
	public void hasNextDoesNotConsume() {
		report(0, 1, 10);
		Iterator<Message> iterator = ModerationTools.getReportedMessages("MOST", 5);
		assertTrue(iterator.hasNext());
		assertTrue(iterator.hasNext());
		assertEquals(messages.get(0), iterator.next());
		assertFalse(iterator.hasNext());
	}
}
```

## 4. Proof the tests work: mutation results (all suites in this file)

Each row is a deliberately broken copy of the implementation; "caught" = at least one test failed (good).
Generated from the actual run: **39 of 39 caught**.

| Broken version | Result | Caught by (first few tests) |
|---|---|---|
| MOST ascending | caught | mostOrdersByActiveReportCount, amountLimitsResults, strategiesGiveDifferentOrders, mostIgnoresRetractedReports |
| OLDEST descending | caught | oldestUsesOldestActiveReport, amountExactlyAvailable, amountLimitsResults, strategiesGiveDifferentOrders |
| strategies swapped | caught | oldestUsesOldestActiveReport, amountExactlyAvailable, strategiesGiveDifferentOrders, amountLargerThanAvailableReturnsAll |
| amount + 1 | caught | amountLimitsResults |
| amount ignored | caught | amountLimitsResults |
| amount - 1 | caught | amountExactlyAvailable, amountLimitsResults |
| zero-report messages kept | caught | fullyRetractedMessageIsExcluded |
| oldest not updated on retract | caught | oldestUsesOldestActiveReport |
| count includes retracted | caught | oldestUsesOldestActiveReport, mostIgnoresRetractedReports |
| no exception for amount <= 0 | caught | negativeAmountThrows, zeroAmountThrows |
| unknown strategy -> MOST | caught | emptyStrategyThrows, unknownStrategyThrows, nullStrategyThrows |
| one entry per report (duplicates) | caught | mostOrdersByActiveReportCount, amountLimitsResults, strategiesGiveDifferentOrders, messageReportedManyTimesAppearsOnce |
| hidden messages skipped | caught | hiddenMessagesAreStillReported |
| addReport accepts duplicates | caught | duplicateReport |
| addReport skips user check | caught | nullUser, unknownUser |
| addReport skips message check | caught | nullMessage, unknownMessage, messageRemovedFromPost, postRemovedFromDAO |
| votes TOP ascending | caught | switchedVotesAreCountedOnce, topByScoreNotByVoteCount, switchesReplaceTheOldVote |
| votes TOP by vote count | caught | switchedVotesAreCountedOnce, topByScoreNotByVoteCount, switchesReplaceTheOldVote |
| votes switch keeps old vote | caught | switchesReplaceTheOldVote |
| votes zero-vote messages kept | caught | retractedVotesExcluded |
| votes NEWEST ascending | caught | newestVoteFirst |
| votes CONTROVERSIAL = most votes | caught | controversialNeedsBothSides |
| votes no exception for amount | caught | negativeAmount |
| vote accepts direction 0 | caught | badDirectionZero, badDirectionTwo |
| vote accepts direction 2 | caught | badDirectionTwo |
| vote same twice returns true | caught | sameVoteTwice |
| vote skips user check | caught | missingUser |
| vote skips message check | caught | missingMessage |
| reactions MOST ascending | caught | mostOrdersByTotal |
| reactions type ranks by total | caught | perTypeCountsOnlyThatType |
| reactions NEWEST ascending | caught | newestFirst |
| reactions remove keeps user | caught | removedReactionsDoNotCount |
| reactions null strategy NPE | caught | nullStrategy |
| reactions duplicates per reaction | caught | eachMessageOnce, mostOrdersByTotal, removedReactionsDoNotCount |
| follow allows self | caught | selfFollow |
| follow skips followee check | caught | missingFollowee |
| follow skips follower check | caught | missingFollower |
| follow twice returns true | caught | followTwice |
| follow stored symmetric | caught | follow |

## 5. Reactions version of the white-box tests (top prediction)

Note: it uses `ReactionType.values()[0]` instead of a constant name, so it compiles with whatever constants the real enum has.

**`app/test/ReactionToolsAddReactionTests.java`** (full file)

```java
import dao.PostDAO;
import dao.UserDAO;
import dao.model.Message;
import dao.model.Post;
import dao.model.User;
import org.junit.Before;
import org.junit.Test;
import reactions.ReactionStore;
import reactions.ReactionTools;
import reactions.ReactionType;

import java.util.UUID;

import static org.junit.Assert.*;

/**
 * PREDICTION (reactions variant of Task 5): branch coverage of ReactionTools.addReaction.
 * Branches: type == null | message missing | user missing | already reacted | success
 * (+ MessageReactions.add: first reaction on message vs. existing group; counts.merge new vs existing key)
 */
public class ReactionToolsAddReactionTests {
	private User user, other;
	private Message message;

	@Before
	public void setUp() {
		UserDAO.getInstance().clear();
		PostDAO.getInstance().clear();
		ReactionStore.getInstance().clear();
		user = new User(UUID.randomUUID(), User.Role.Member, "reactor", "password");
		other = new User(UUID.randomUUID(), User.Role.Member, "reactor2", "password");
		UserDAO.getInstance().add(user);
		UserDAO.getInstance().add(other);
		Post post = new Post(UUID.randomUUID(), user.id(), "topic");
		PostDAO.getInstance().add(post);
		message = new Message(UUID.randomUUID(), user.id(), post.id, 1, "hi");
		post.messages.insert(message);
	}

	@Test public void success() {
		assertTrue(ReactionTools.addReaction(message.id(), user.id(), ReactionType.values()[0], 5));
		assertEquals(ReactionType.values()[0], ReactionTools.getReaction(message.id(), user.id()));
		assertEquals(1, ReactionTools.countReactions(message.id(), ReactionType.values()[0]));
	}
	@Test public void sameTypeFromSecondUserIncrementsCount() {
		ReactionType type = ReactionType.values()[0];
		ReactionTools.addReaction(message.id(), user.id(), type, 5);
		assertTrue(ReactionTools.addReaction(message.id(), other.id(), type, 6));
		assertEquals(2, ReactionTools.countReactions(message.id(), type));
	}
	@Test public void alreadyReacted() {
		ReactionTools.addReaction(message.id(), user.id(), ReactionType.values()[0], 5);
		assertFalse(ReactionTools.addReaction(message.id(), user.id(), ReactionType.values()[1], 6));
		assertEquals("count must not change", 0, ReactionTools.countReactions(message.id(), ReactionType.values()[1]));
	}
	@Test public void nullType() {
		assertFalse(ReactionTools.addReaction(message.id(), user.id(), null, 5));
	}
	@Test public void missingMessage() {
		assertFalse(ReactionTools.addReaction(UUID.randomUUID(), user.id(), ReactionType.values()[0], 5));
	}
	@Test public void missingUser() {
		assertFalse(ReactionTools.addReaction(message.id(), UUID.randomUUID(), ReactionType.values()[0], 5));
		assertFalse(ReactionTools.hasReacted(message.id(), user.id()));
	}
}
```

## 6. Votes version: white-box for `vote`

**`app/test/VoteToolsVoteTests.java`** (full file)

```java
import dao.PostDAO;
import dao.UserDAO;
import dao.model.Message;
import dao.model.Post;
import dao.model.User;
import org.junit.Before;
import org.junit.Test;
import votes.VoteTools;

import java.util.UUID;

import static org.junit.Assert.*;

/**
 * PREDICTION (votes variant of Task 5, white-box): branch coverage of VoteTools.vote
 * and the helpers it calls.
 * Branch map:
 *  vote:  direction != UP && != DOWN -> badDirection*; !exists -> missingMessage / missingUser; else -> MessageVotes.set
 *  ||  :  direction == UP (first operand false) -> upvote; == DOWN -> downvote; neither -> badDirectionZero / Two
 *  exists (&&): message missing -> missingMessage (short-circuit); user missing -> missingUser
 *  computeIfAbsent: first vote on message -> upvote; existing group -> secondUserVotes
 *  MessageVotes.set: previous == null -> upvote; previous == same -> sameVoteTwice; previous == other -> switchVote
 *  count(): direction > 0 -> upvote; else -> downvote
 */
public class VoteToolsVoteTests {
	private User voter, other;
	private Message message;

	@Before
	public void setUp() {
		UserDAO.getInstance().clear();
		PostDAO.getInstance().clear();
		VoteTools.clear();
		voter = new User(UUID.randomUUID(), User.Role.Member, "voter", "password");
		other = new User(UUID.randomUUID(), User.Role.Member, "otherVoter", "password");
		UserDAO.getInstance().add(voter);
		UserDAO.getInstance().add(other);
		Post post = new Post(UUID.randomUUID(), voter.id(), "topic");
		PostDAO.getInstance().add(post);
		message = new Message(UUID.randomUUID(), voter.id(), post.id, 1, "hi");
		post.messages.insert(message);
	}

	@Test(timeout = 1000) public void upvote() {
		assertTrue(VoteTools.vote(message.id(), voter.id(), VoteTools.UP, 5));
		assertEquals(1, VoteTools.getScore(message.id()));
	}

	@Test(timeout = 1000) public void downvote() {
		assertTrue(VoteTools.vote(message.id(), voter.id(), VoteTools.DOWN, 5));
		assertEquals(-1, VoteTools.getScore(message.id()));
	}

	@Test(timeout = 1000) public void secondUserVotes() {
		VoteTools.vote(message.id(), voter.id(), VoteTools.UP, 5);
		assertTrue(VoteTools.vote(message.id(), other.id(), VoteTools.UP, 6));
		assertEquals(2, VoteTools.getScore(message.id()));
	}

	@Test(timeout = 1000) public void sameVoteTwice() {
		VoteTools.vote(message.id(), voter.id(), VoteTools.UP, 5);
		assertFalse(VoteTools.vote(message.id(), voter.id(), VoteTools.UP, 6));
		assertEquals("score must not change", 1, VoteTools.getScore(message.id()));
	}

	@Test(timeout = 1000) public void switchVote() {
		VoteTools.vote(message.id(), voter.id(), VoteTools.UP, 5);
		assertTrue(VoteTools.vote(message.id(), voter.id(), VoteTools.DOWN, 6));
		assertEquals(-1, VoteTools.getScore(message.id()));
		assertEquals(0, VoteTools.getUpvotes(message.id()));
	}

	@Test(timeout = 1000) public void badDirectionZero() {
		assertFalse(VoteTools.vote(message.id(), voter.id(), 0, 5));
		assertEquals(0, VoteTools.getVote(message.id(), voter.id()));
	}

	@Test(timeout = 1000) public void badDirectionTwo() {
		assertFalse(VoteTools.vote(message.id(), voter.id(), 2, 5));
	}

	@Test(timeout = 1000) public void missingMessage() {
		assertFalse(VoteTools.vote(UUID.randomUUID(), voter.id(), VoteTools.UP, 5));
	}

	@Test(timeout = 1000) public void missingUser() {
		assertFalse(VoteTools.vote(message.id(), UUID.randomUUID(), VoteTools.UP, 5));
		assertEquals(0, VoteTools.getScore(message.id()));
	}
}
```

## 7. Follow version: white-box for `follow`

**`app/test/SocialToolsFollowTests.java`** (full file)

```java
import dao.PostDAO;
import dao.UserDAO;
import dao.model.User;
import org.junit.Before;
import org.junit.Test;
import relations.SocialTools;

import java.util.Set;
import java.util.UUID;

import static org.junit.Assert.*;

/**
 * PREDICTION (follow variant of Task 5, white-box): branch coverage of SocialTools.follow,
 * usersExist and RelationStore.add.
 * Branch map:
 *  follow: !usersExist -> missingFollower / missingFollowee; self -> selfFollow; else -> RelationStore.add
 *  usersExist (&&): follower missing (short-circuit) -> missingFollower; followee missing -> missingFollowee
 *  RelationStore.add: forward set new/existing (computeIfAbsent) -> follow / secondFollowBySameUser;
 *                     pair already there -> followTwice
 */
public class SocialToolsFollowTests {
	private User a, b, c;

	@Before
	public void setUp() {
		UserDAO.getInstance().clear();
		PostDAO.getInstance().clear();
		SocialTools.clearAll();
		a = new User(UUID.randomUUID(), User.Role.Member, "aaaa", "password");
		b = new User(UUID.randomUUID(), User.Role.Member, "bbbb", "password");
		c = new User(UUID.randomUUID(), User.Role.Member, "cccc", "password");
		UserDAO.getInstance().add(a);
		UserDAO.getInstance().add(b);
		UserDAO.getInstance().add(c);
	}

	@Test(timeout = 1000) public void follow() {
		assertTrue(SocialTools.follow(a.id(), b.id()));
		assertTrue(SocialTools.isFollowing(a.id(), b.id()));
		assertFalse("not symmetric", SocialTools.isFollowing(b.id(), a.id()));
		assertEquals(Set.of(a.id()), SocialTools.getFollowers(b.id()));
	}

	@Test(timeout = 1000) public void secondFollowBySameUser() {
		SocialTools.follow(a.id(), b.id());
		assertTrue(SocialTools.follow(a.id(), c.id()));
		assertEquals(1, SocialTools.followerCount(c.id()));
	}

	@Test(timeout = 1000) public void followTwice() {
		SocialTools.follow(a.id(), b.id());
		assertFalse(SocialTools.follow(a.id(), b.id()));
		assertEquals(1, SocialTools.followerCount(b.id()));
	}

	@Test(timeout = 1000) public void selfFollow() {
		assertFalse(SocialTools.follow(a.id(), a.id()));
	}

	@Test(timeout = 1000) public void missingFollower() {
		assertFalse(SocialTools.follow(UUID.randomUUID(), b.id()));
		assertEquals(0, SocialTools.followerCount(b.id()));
	}

	@Test(timeout = 1000) public void missingFollowee() {
		assertFalse(SocialTools.follow(a.id(), UUID.randomUUID()));
	}
}
```

## 8. Reactions version: black-box for `getTopMessages`

**`app/test/ReactionToolsGetTopMessagesTests.java`** (full file)

```java
import dao.PostDAO;
import dao.UserDAO;
import dao.model.Message;
import dao.model.Post;
import dao.model.User;
import org.junit.Before;
import org.junit.Test;
import reactions.ReactionStore;
import reactions.ReactionTools;
import reactions.ReactionType;

import java.util.*;

import static org.junit.Assert.*;

/**
 * PREDICTION (reactions variant of Task 5, black-box) for ReactionTools.getTopMessages.
 * Uses ReactionType.values()[i] so it compiles with any real enum constants (needs >= 2).
 */
public class ReactionToolsGetTopMessagesTests {
	private final List<User> users = new ArrayList<>();
	private final List<Message> messages = new ArrayList<>();
	private ReactionType first, second;

	@Before
	public void setUp() {
		UserDAO.getInstance().clear();
		PostDAO.getInstance().clear();
		ReactionStore.getInstance().clear();
		users.clear();
		messages.clear();
		first = ReactionType.values()[0];
		second = ReactionType.values()[1];
		for (int i = 0; i < 5; i++) {
			User u = new User(UUID.randomUUID(), User.Role.Member, "user" + i, "password");
			UserDAO.getInstance().add(u);
			users.add(u);
		}
		Post post = new Post(UUID.randomUUID(), users.get(0).id(), "t");
		PostDAO.getInstance().add(post);
		for (int i = 0; i < 4; i++) {
			Message m = new Message(UUID.randomUUID(), users.get(0).id(), post.id, i, "m" + i);
			post.messages.insert(m);
			messages.add(m);
		}
	}

	private void react(int message, int user, ReactionType type, long time) {
		assertTrue("setup", ReactionTools.addReaction(messages.get(message).id(), users.get(user).id(), type, time));
	}

	private List<Message> top(String strategy, int amount) {
		List<Message> l = new ArrayList<>();
		ReactionTools.getTopMessages(strategy, amount).forEachRemaining(l::add);
		return l;
	}

	private List<Message> expected(int... i) {
		List<Message> l = new ArrayList<>();
		for (int x : i) l.add(messages.get(x));
		return l;
	}

	@Test(timeout = 1000, expected = IllegalArgumentException.class) public void unknownStrategy() { top("BEST", 1); }
	@Test(timeout = 1000, expected = IllegalArgumentException.class) public void nullStrategy() { top(null, 1); }
	@Test(timeout = 1000, expected = IllegalArgumentException.class) public void zeroAmount() { top("MOST", 0); }

	@Test(timeout = 1000) public void emptyWhenNoReactions() {
		assertTrue(top("MOST", 3).isEmpty());
	}

	@Test(timeout = 1000) public void mostOrdersByTotal() {
		react(2, 1, first, 1);
		react(0, 1, first, 1); react(0, 2, second, 1); react(0, 3, first, 1);
		react(3, 1, second, 1); react(3, 2, second, 1);
		assertEquals(expected(0, 3, 2), top("MOST", 5));
		assertEquals(expected(0, 3), top("MOST", 2));
	}

	@Test(timeout = 1000) public void perTypeCountsOnlyThatType() {
		react(0, 1, first, 1); react(0, 2, first, 1);
		react(1, 1, second, 1); react(1, 2, second, 1); react(1, 3, second, 1);
		react(2, 1, first, 1);
		assertEquals(expected(0, 2), top(first.name(), 5));
		assertEquals(expected(1), top(second.name(), 5));
	}

	@Test(timeout = 1000) public void newestFirst() {
		react(0, 1, first, 300); react(1, 1, first, 100); react(2, 1, first, 200);
		assertEquals(expected(0, 2, 1), top("NEWEST", 5));
	}

	@Test(timeout = 1000) public void removedReactionsDoNotCount() {
		react(0, 1, first, 1); react(0, 2, first, 1);
		react(1, 1, first, 1);
		ReactionTools.removeReaction(messages.get(0).id(), users.get(1).id());
		ReactionTools.removeReaction(messages.get(0).id(), users.get(2).id());
		assertEquals(expected(1), top("MOST", 5));
	}

	@Test(timeout = 1000) public void eachMessageOnce() {
		for (int u = 1; u < 5; u++) react(0, u, first, u);
		assertEquals(expected(0), top("MOST", 5));
	}
}
```

## 9. Votes version: black-box for `getTopMessages`

Note `switchesReplaceTheOldVote`: the first version of this suite let one broken implementation through (switching a
vote without undoing the old one only moved the score by 1, which did not change the order). Making two users switch
moves it by 2 and changes the order, so the bug is caught. That is the kind of thinking black-box marks reward.

**`app/test/VoteToolsGetTopMessagesTests.java`** (full file)

```java
import dao.PostDAO;
import dao.UserDAO;
import dao.model.Message;
import dao.model.Post;
import dao.model.User;
import org.junit.Before;
import org.junit.Test;
import votes.VoteTools;

import java.util.*;

import static org.junit.Assert.*;

/** PREDICTION (votes variant of Task 5, black-box) for VoteTools.getTopMessages. */
public class VoteToolsGetTopMessagesTests {
	private final List<User> users = new ArrayList<>();
	private final List<Message> messages = new ArrayList<>();

	@Before
	public void setUp() {
		UserDAO.getInstance().clear();
		PostDAO.getInstance().clear();
		VoteTools.clear();
		users.clear();
		messages.clear();
		for (int i = 0; i < 5; i++) {
			User u = new User(UUID.randomUUID(), User.Role.Member, "user" + i, "password");
			UserDAO.getInstance().add(u);
			users.add(u);
		}
		Post post = new Post(UUID.randomUUID(), users.get(0).id(), "t");
		PostDAO.getInstance().add(post);
		for (int i = 0; i < 4; i++) {
			Message m = new Message(UUID.randomUUID(), users.get(0).id(), post.id, i, "m" + i);
			post.messages.insert(m);
			messages.add(m);
		}
	}

	private void vote(int message, int user, int direction, long time) {
		assertTrue("setup", VoteTools.vote(messages.get(message).id(), users.get(user).id(), direction, time));
	}

	private List<Message> top(String strategy, int amount) {
		List<Message> l = new ArrayList<>();
		VoteTools.getTopMessages(strategy, amount).forEachRemaining(l::add);
		return l;
	}

	private List<Message> expected(int... i) {
		List<Message> l = new ArrayList<>();
		for (int x : i) l.add(messages.get(x));
		return l;
	}

	@Test(timeout = 1000, expected = IllegalArgumentException.class) public void unknownStrategy() { top("HOT", 1); }
	@Test(timeout = 1000, expected = IllegalArgumentException.class) public void negativeAmount() { top("TOP", -2); }

	@Test(timeout = 1000) public void topByScoreNotByVoteCount() {
		vote(0, 1, 1, 1); vote(0, 2, 1, 1); vote(0, 3, -1, 1);          // score 1, 3 votes
		vote(1, 1, 1, 1); vote(1, 2, 1, 1);                             // score 2, 2 votes
		vote(2, 1, -1, 1);                                              // score -1
		assertEquals(expected(1, 0, 2), top("TOP", 5));
		assertEquals(expected(1), top("TOP", 1));
	}

	@Test(timeout = 1000) public void switchedVotesAreCountedOnce() {
		vote(0, 1, 1, 1);
		vote(0, 1, -1, 2);                     // switch: score -1, not 0
		vote(1, 1, 1, 3);
		assertEquals(expected(1, 0), top("TOP", 5));
	}

	@Test(timeout = 1000) public void switchesReplaceTheOldVote() {
		vote(0, 1, 1, 1); vote(0, 1, -1, 2);   // user 1 switches up -> down
		vote(0, 2, 1, 3); vote(0, 2, -1, 4);   // user 2 switches up -> down: score must be -2 (not 0)
		vote(1, 3, -1, 5);                     // score -1
		assertEquals(expected(1, 0), top("TOP", 5));
	}

	@Test(timeout = 1000) public void controversialNeedsBothSides() {
		vote(0, 1, 1, 1); vote(0, 2, 1, 1); vote(0, 3, 1, 1);            // all up
		vote(1, 1, 1, 1); vote(1, 2, -1, 1);                            // 1 vs 1
		vote(2, 1, 1, 1); vote(2, 2, 1, 1); vote(2, 3, -1, 1); vote(2, 4, -1, 1); // 2 vs 2
		assertEquals(expected(2, 1, 0), top("CONTROVERSIAL", 5));
	}

	@Test(timeout = 1000) public void newestVoteFirst() {
		vote(0, 1, 1, 50); vote(1, 1, 1, 70); vote(2, 1, -1, 60);
		assertEquals(expected(1, 2, 0), top("NEWEST", 5));
	}

	@Test(timeout = 1000) public void retractedVotesExcluded() {
		vote(0, 1, 1, 1);
		VoteTools.removeVote(messages.get(0).id(), users.get(1).id());
		vote(1, 1, 1, 1);
		assertEquals(expected(1), top("TOP", 5));
	}
}
```

## 10. JUnit4 cheat sheet

```java
@RunWith(JUnit4.class)                       // optional for plain JUnit4 classes
@Before public void setUp() { ... }           // runs before EVERY test: reset singletons here
@Test(timeout = 1000)                         // fail if slower than 1 s (stops infinite loops)
@Test(expected = IllegalArgumentException.class)  // pass only if this exception is thrown
assertEquals("message shown on failure", expected, actual);   // expected FIRST
assertTrue / assertFalse / assertNull / assertNotNull / assertSame / assertArrayEquals
// order-free comparison (ties): compare as sets
assertEquals(new HashSet<>(expectedList), new HashSet<>(actualList));
```

## 11. Trap checklist (Task 5)

- [ ] `@Before` clears **every** singleton the test touches (`UserDAO`, `PostDAO`, your stores), or tests depend on run order.
- [ ] Usernames are unique per test (UserDAO compares them **case-insensitively**) and 4 to 20 alphanumeric characters if you use `register`.
- [ ] Setup steps assert they worked (`report(...)` helper asserts `addReport` returned true) so a broken setup cannot pass silently.
- [ ] Black-box: never assert tie order or unspecified behaviour.
- [ ] Black-box: design data so a bug changes the ORDER, not just a number (see section 9).
- [ ] Use `ReactionType.values()[i]` instead of constant names if the enum is given and you are unsure of its constants.
- [ ] White-box: include the helpers your team wrote; list the branch map in javadoc.
- [ ] Test class names must match the provided files exactly (`ModerationToolsAddReportTests`, ...) and stay in the default package.
