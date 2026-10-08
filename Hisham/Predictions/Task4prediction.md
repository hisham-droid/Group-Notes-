# Task 4 prediction: queries with design patterns (Iterator + Factory)

> All code here passed ordering, limit, exception and independence tests on the practice base.

## 1. What Task 4 looks like (from the practice hackathon)

`Iterator<Message> ModerationTools.getReportedMessages(String strategy, int amount)`

| Part | Marks | What earns it |
|---|---|---|
| Behaviour | 40% | right order, at most `amount`, no zero-count items, each message once, exceptions on bad input |
| **Iterator AND Factory are mandatory** | 40% | real classes implementing the patterns, used for the right job |
| Quality | 20% | |

Where each pattern goes (say this in javadoc, markers look for it):

| Pattern | Class | Job |
|---|---|---|
| **Factory** | `ReportOrderingFactory.create(strategy)` | turns `"OLDEST"`/`"MOST"` into an ordering object; the only place that knows the strategy names |
| **Strategy** (bonus) | the `Comparator<MessageReports>` it returns | interchangeable ordering rules |
| **Iterator** | `ReportedMessageIterator implements Iterator<Message>` | hands out results one at a time, enforces `amount`, skips deleted messages |
| Facade | `ModerationTools` | one entry point; validates `amount`, delegates |

## 2. Every Task 4 prediction (all coded and tested)

| Theme | Method | Strategies | Section |
|---|---|---|---|
| Reports (practice) | `ModerationTools.getReportedMessages` | `OLDEST`, `MOST` | 3 |
| Reports, extended | `ReportQueries.getReportedMessages` | `OLDEST`, `NEWEST`, `MOST`, `LEAST`, `MOST_THEN_OLDEST` (two-key tie-break) | 5 |
| Reactions | `ReactionTools.getTopMessages` | `MOST`, `NEWEST`, any type name (`LIKE`, ...) | 4 |
| Votes | `VoteTools.getTopMessages` | `TOP` (score), `CONTROVERSIAL` (both sides, then most votes), `NEWEST` | 6 |
| Posts | `TrendingTools.getTopPosts` | `BUSY` (most messages), `ACTIVE` (latest message) | 7 |
| Trending | `TrendingTools.getTrendingPosts(now, window, k)` | most messages in a time window (counted with `rank`) | 7 |
| Leaderboard | `LeaderboardTools.getTopUsers` | `MESSAGES`, `POSTS`, `KARMA`, `REACTIONS` | 8 |
| Tags | `TagTools.getPopularTags(k)` | most-used tags | Task1prediction.md 9 |
| Censor | `CensorFactory.create` | `STAR`, `FIRST`, `REMOVE`, `REPLACE` (Strategy + Factory + Decorator) | ExtraPredictions.md |

Two-key ordering when the spec defines tie-breaks: `comparingInt(G::count).reversed().thenComparingLong(G::oldestTimestamp)`.

## 3. Worked solution: `getReportedMessages`

**`app/src/moderation/ReportOrderingFactory.java`** (full file)

```java
package moderation;

import java.util.Comparator;

/**
 * HACKATHON (Task 4): FACTORY pattern.
 * Turns the strategy string into an ordering (a Comparator is a STRATEGY object).
 * Adding a new strategy later means adding one case here and nothing else.
 */
public final class ReportOrderingFactory {
	public static final String OLDEST = "OLDEST";
	public static final String MOST = "MOST";

	private ReportOrderingFactory() {} // static factory, no instances

	/**
	 * @param strategy "OLDEST" (oldest active report first) or "MOST" (most active reports first)
	 * @return the matching ordering
	 * @throws IllegalArgumentException for null or any other string (case-sensitive)
	 */
	public static Comparator<MessageReports> create(String strategy) {
		if (strategy == null) throw new IllegalArgumentException("strategy must not be null");
		return switch (strategy) {
			case OLDEST -> Comparator.comparingLong(MessageReports::oldestTimestamp);
			case MOST -> Comparator.comparingInt(MessageReports::activeCount).reversed();
			default -> throw new IllegalArgumentException("Unknown strategy: " + strategy);
		};
	}
}
```

**`app/src/moderation/ReportedMessageIterator.java`** (full file)

```java
package moderation;

import dao.MessageIndex;
import dao.model.Message;

import java.util.Collection;
import java.util.Comparator;
import java.util.Iterator;
import java.util.NoSuchElementException;
import java.util.PriorityQueue;

/**
 * HACKATHON (Task 4): ITERATOR pattern.
 * Yields at most `amount` reported messages, best first, each at most once.
 * <p>
 * Efficiency: building the heap is O(m); each next() is O(log m). So asking for the
 * top k of m reported messages costs O(m + k log m), cheaper than sorting all m.
 * Messages that no longer exist are skipped silently.
 */
public class ReportedMessageIterator implements Iterator<Message> {
	private final PriorityQueue<MessageReports> queue;
	private int remaining;
	private Message next; // look-ahead: the message next() will return, or null when done

	public ReportedMessageIterator(Collection<MessageReports> reported, Comparator<MessageReports> order, int amount) {
		this.queue = new PriorityQueue<>(Math.max(1, reported.size()), order);
		for (MessageReports reports : reported) {
			if (!reports.isEmpty()) queue.add(reports); // never return zero-report messages
		}
		this.remaining = amount;
		advance();
	}

	// moves `next` to the next existing message, respecting the amount limit
	private void advance() {
		next = null;
		while (remaining > 0 && !queue.isEmpty()) {
			Message candidate = MessageIndex.getInstance().get(queue.poll().getMessageId());
			if (candidate != null) {
				next = candidate;
				remaining--;
				return;
			}
		}
	}

	@Override
	public boolean hasNext() {
		return next != null;
	}

	@Override
	public Message next() {
		if (next == null) throw new NoSuchElementException();
		Message result = next;
		advance();
		return result;
	}
}
```

The method in `ModerationTools` (full file in Task1prediction.md):

```java
public static Iterator<Message> getReportedMessages(String strategy, int amount) {
	if (amount <= 0) throw new IllegalArgumentException("amount must be positive, was " + amount);
	return new ReportedMessageIterator(
			ReportStore.getInstance().getAllMessageReports(),
			ReportOrderingFactory.create(strategy),   // throws IllegalArgumentException for bad names
			amount);
}
```

Why a heap (`PriorityQueue`) and not a full sort: building it is O(m) and each `next()` is O(log m),
so the top k of m costs O(m + k log m), while sorting is always O(m log m). Say this in a comment.

## 4. Reusable version for any theme

Write this once and every "top k by strategy" question becomes 5 lines:

**`app/src/util/TopKIterator.java`** (full file)

```java
package util;

import java.util.Collection;
import java.util.Comparator;
import java.util.Iterator;
import java.util.NoSuchElementException;
import java.util.PriorityQueue;
import java.util.function.Function;
import java.util.function.Predicate;

/**
 * HACKATHON (Task 4, reusable): ITERATOR over the best `amount` groups, best first.
 * Works for ANY "rank things by a strategy and return the top k" question:
 * reports, reactions, votes, leaderboards...
 * <p>
 * @param <G> the group being ranked (e.g. MessageReactions)
 * @param <T> what the iterator returns (e.g. Message)
 * Cost: O(m) to heapify + O(log m) per next().
 */
public class TopKIterator<G, T> implements Iterator<T> {
	private final PriorityQueue<G> queue;
	private final Function<G, T> toResult; // returning null means "skip this one" (e.g. deleted)
	private int remaining;
	private T next;

	public TopKIterator(Collection<G> groups, Comparator<G> order, Predicate<G> include, Function<G, T> toResult, int amount) {
		if (amount <= 0) throw new IllegalArgumentException("amount must be positive, was " + amount);
		this.queue = new PriorityQueue<>(Math.max(1, groups.size()), order);
		for (G group : groups) if (include.test(group)) queue.add(group);
		this.toResult = toResult;
		this.remaining = amount;
		advance();
	}

	private void advance() {
		next = null;
		while (remaining > 0 && !queue.isEmpty()) {
			T candidate = toResult.apply(queue.poll());
			if (candidate != null) {
				next = candidate;
				remaining--;
				return;
			}
		}
	}

	@Override
	public boolean hasNext() {
		return next != null;
	}

	@Override
	public T next() {
		if (next == null) throw new NoSuchElementException();
		T result = next;
		advance();
		return result;
	}
}
```

Used by the reactions prediction:

**`app/src/reactions/ReactionOrderingFactory.java`** (full file)

```java
package reactions;

import java.util.Comparator;

/**
 * FACTORY for reaction rankings (STRATEGY = the Comparator).
 *   "MOST"      -> most reactions of any type first
 *   "NEWEST"    -> most recent reaction first
 *   "LIKE", "LOVE", ... (any ReactionType name) -> most reactions of that type first
 */
public final class ReactionOrderingFactory {
	private ReactionOrderingFactory() {}

	public static Comparator<MessageReactions> create(String strategy) {
		if (strategy == null) throw new IllegalArgumentException("strategy must not be null");
		switch (strategy) {
			case "MOST":
				return Comparator.comparingInt(MessageReactions::total).reversed();
			case "NEWEST":
				return Comparator.comparingLong(MessageReactions::latestTimestamp).reversed();
			default:
				ReactionType type = parseType(strategy);
				return Comparator.comparingInt((MessageReactions r) -> r.count(type)).reversed();
		}
	}

	/** @return a predicate-friendly type, or null for MOST/NEWEST */
	public static ReactionType typeFilter(String strategy) {
		if (strategy == null) throw new IllegalArgumentException("strategy must not be null"); // valueOf(null) would throw NullPointerException
		return ("MOST".equals(strategy) || "NEWEST".equals(strategy)) ? null : parseType(strategy);
	}

	private static ReactionType parseType(String strategy) {
		if (strategy == null) throw new IllegalArgumentException("strategy must not be null");
		try {
			return ReactionType.valueOf(strategy);
		} catch (IllegalArgumentException e) {
			throw new IllegalArgumentException("Unknown strategy: " + strategy);
		}
	}
}
```

```java
// in ReactionTools (full file in Task1prediction.md)
public static Iterator<Message> getTopMessages(String strategy, int amount) {
	ReactionType type = ReactionOrderingFactory.typeFilter(strategy); // also validates the name
	return new TopKIterator<>(
			ReactionStore.getInstance().all(),
			ReactionOrderingFactory.create(strategy),
			group -> type == null ? !group.isEmpty() : group.count(type) > 0,  // never zero-count items
			group -> MessageIndex.getInstance().get(group.getMessageId()),       // null = deleted, skipped
			amount);
}
```

## 5. Reports with more strategies (separate class: the practice spec says NEWEST must throw)

**`app/src/moderation/ReportQueries.java`** (full file)

```java
package moderation;

import dao.MessageIndex;
import dao.model.Message;
import util.TopKIterator;

import java.util.Comparator;
import java.util.Iterator;

/**
 * PREDICTED Task 4 variants beyond the practice spec (kept separate so the practice
 * getReportedMessages still throws for "NEWEST", as its spec requires).
 *  "OLDEST"           oldest active report first
 *  "NEWEST"           newest active report first
 *  "MOST"             most active reports first
 *  "MOST_THEN_OLDEST" most reports first; ties broken by oldest report (two-key ordering)
 *  "LEAST"            fewest reports first
 */
public final class ReportQueries {
	private ReportQueries() {}

	/** FACTORY: strategy name -> ordering (STRATEGY object) */
	public static Comparator<MessageReports> ordering(String strategy) {
		if (strategy == null) throw new IllegalArgumentException("strategy must not be null");
		return switch (strategy) {
			case "OLDEST" -> Comparator.comparingLong(MessageReports::oldestTimestamp);
			case "NEWEST" -> Comparator.comparingLong(MessageReports::newestTimestamp).reversed();
			case "MOST" -> Comparator.comparingInt(MessageReports::activeCount).reversed();
			case "MOST_THEN_OLDEST" -> Comparator.comparingInt(MessageReports::activeCount).reversed()
					.thenComparingLong(MessageReports::oldestTimestamp);
			case "LEAST" -> Comparator.comparingInt(MessageReports::activeCount);
			default -> throw new IllegalArgumentException("Unknown strategy: " + strategy);
		};
	}

	/** Same rules as getReportedMessages (amount > 0, no zero-report messages, each once). */
	public static Iterator<Message> getReportedMessages(String strategy, int amount) {
		return new TopKIterator<>(ReportStore.getInstance().getAllMessageReports(), ordering(strategy),
				r -> !r.isEmpty(), r -> MessageIndex.getInstance().get(r.getMessageId()), amount);
	}
}
```

## 6. Votes

The ordering factory and query are in `VoteTools` (full file in Task1prediction.md section 7):

```java
static Comparator<MessageVotes> ordering(String strategy) {
	if (strategy == null) throw new IllegalArgumentException("strategy must not be null");
	return switch (strategy) {
		case "TOP" -> Comparator.comparingInt(MessageVotes::score).reversed();
		case "CONTROVERSIAL" -> Comparator.comparingInt((MessageVotes v) -> Math.min(v.up(), v.down())).reversed()
				.thenComparing(Comparator.comparingInt(MessageVotes::total).reversed());
		case "NEWEST" -> Comparator.comparingLong(MessageVotes::latestTimestamp).reversed();
		default -> throw new IllegalArgumentException("Unknown strategy: " + strategy);
	};
}

public static Iterator<Message> getTopMessages(String strategy, int amount) {
	return new TopKIterator<>(votesByMessage.values(), ordering(strategy), v -> !v.isEmpty(),
			v -> MessageIndex.getInstance().get(v.getMessageId()), amount);
}
```

## 7. Posts: busiest, most recently active, trending

**`app/src/trending/TrendingTools.java`** (full file)

```java
package trending;

import dao.PostDAO;
import dao.model.Post;
import queries.MessageQueries;
import util.TopKIterator;

import java.util.ArrayList;
import java.util.Comparator;
import java.util.Iterator;
import java.util.List;

/**
 * PREDICTED theme: trending posts = most messages in the last `windowMillis`.
 * Per post the count is O(log n) (rank on the tree), so the whole query is O(P log n + k log P).
 */
public final class TrendingTools {
	private TrendingTools() {}

	private record Activity(Post post, int count) {}

	/** @return up to `amount` posts with recent activity, most active first */
	public static Iterator<Post> getTrendingPosts(long now, long windowMillis, int amount) {
		if (windowMillis <= 0) throw new IllegalArgumentException("window must be positive");
		List<Activity> activity = new ArrayList<>();
		for (Iterator<Post> it = PostDAO.getInstance().getAll(); it.hasNext(); ) {
			Post post = it.next();
			activity.add(new Activity(post, MessageQueries.countMessagesBetween(post.id, now - windowMillis, now, false)));
		}
		return new TopKIterator<>(activity, Comparator.comparingInt(Activity::count).reversed(),
				a -> a.count() > 0, Activity::post, amount);
	}

	/**
	 * Factory for post rankings: "BUSY" (most messages ever) or "ACTIVE" (latest message first).
	 */
	public static Iterator<Post> getTopPosts(String strategy, int amount) {
		Comparator<Post> order = switch (strategy == null ? "" : strategy) {
			case "BUSY" -> Comparator.comparingInt((Post p) -> p.messages.size()).reversed();
			case "ACTIVE" -> Comparator.comparingLong(TrendingTools::lastActivity).reversed();
			default -> throw new IllegalArgumentException("Unknown strategy: " + strategy);
		};
		List<Post> posts = new ArrayList<>();
		PostDAO.getInstance().getAll().forEachRemaining(posts::add);
		return new TopKIterator<>(posts, order, p -> p.messages.size() > 0, p -> p, amount);
	}

	private static long lastActivity(Post post) {
		Iterator<dao.model.Message> last = post.messages.getRange(null, 1, true); // newest message
		return last.hasNext() ? last.next().timestamp() : Long.MIN_VALUE;
	}
}
```

## 8. Users: leaderboards

**`app/src/trending/LeaderboardTools.java`** (full file)

```java
package trending;

import dao.PostDAO;
import dao.UserDAO;
import dao.model.Message;
import dao.model.Post;
import dao.model.User;
import reactions.MessageReactions;
import reactions.ReactionStore;
import util.TopKIterator;
import votes.VoteTools;

import java.util.Comparator;
import java.util.HashMap;
import java.util.Iterator;
import java.util.Map;
import java.util.UUID;

/**
 * PREDICTED theme: leaderboards / karma. Factory picks what to count per user:
 *  "MESSAGES"  messages written
 *  "POSTS"     posts started
 *  "KARMA"     sum of vote scores of their messages
 *  "REACTIONS" reactions received on their messages
 * One pass over the data (O(n)) builds the totals, then TopKIterator picks the best.
 * Deleted users are skipped.
 */
public final class LeaderboardTools {
	private LeaderboardTools() {}

	private record Entry(User user, int value) {}

	public static Iterator<User> getTopUsers(String strategy, int amount) {
		Map<UUID, Integer> totals = new HashMap<>();
		switch (strategy == null ? "" : strategy) {
			case "MESSAGES" -> forEachMessage(m -> add(totals, m.poster(), 1));
			case "POSTS" -> PostDAO.getInstance().getAll().forEachRemaining(p -> add(totals, p.poster, 1));
			case "KARMA" -> forEachMessage(m -> add(totals, m.poster(), VoteTools.getScore(m.id())));
			case "REACTIONS" -> forEachMessage(m -> {
				MessageReactions r = ReactionStore.getInstance().find(m.id());
				add(totals, m.poster(), r == null ? 0 : r.total());
			});
			default -> throw new IllegalArgumentException("Unknown strategy: " + strategy);
		}
		java.util.List<Entry> entries = new java.util.ArrayList<>();
		totals.forEach((id, value) -> {
			User u = UserDAO.getInstance().getByUUID(id);
			if (u != null) entries.add(new Entry(u, value));
		});
		return new TopKIterator<>(entries, Comparator.comparingInt(Entry::value).reversed(),
				e -> e.value() > 0, Entry::user, amount);
	}

	private static void add(Map<UUID, Integer> totals, UUID user, int amount) {
		if (user != null) totals.merge(user, amount, Integer::sum);
	}

	private static void forEachMessage(java.util.function.Consumer<Message> action) {
		for (Iterator<Post> it = PostDAO.getInstance().getAll(); it.hasNext(); ) it.next().messages.getAll().forEachRemaining(action);
	}
}
```

## 9. Spec rules to code (checklist)

- [ ] Strategy names compared **exactly** (`"most"` is not `"MOST"`), `null` throws `IllegalArgumentException`, not `NullPointerException`.
- [ ] `amount <= 0` throws. `amount` larger than available returns everything without error.
- [ ] Items with **zero active** reports/reactions are never returned (remove empty groups, and filter anyway).
- [ ] A message appears **at most once** (rank groups per message, not individual reports).
- [ ] "Oldest" means oldest **active** report: it must update after a removal (that is why `MessageReports` keeps a `TreeSet`).
- [ ] Hidden messages are still returned (moderators need them) unless the spec says otherwise.
- [ ] Ties: any order. Do not spend time on tie-breaking unless asked.
- [ ] `hasNext()` must not consume anything (look-ahead field `next`).
- [ ] Return an empty iterator, never `null`.
- [ ] Iterators must be independent: build a fresh queue per call.

## 10. How this was verified

| Check | Result |
|---|---|
| MOST order; MOST after retractions | pass |
| OLDEST order; OLDEST after the oldest report is retracted | pass |
| amount 1, 2, exactly available, more than available | pass |
| `"NEWEST"`, `"most"`, `""`, `null`, amount 0, amount -1 all throw `IllegalArgumentException` | pass |
| zero-report messages excluded, message reported 4 times appears once, hidden message still listed | pass |
| two iterators from two calls are independent; `hasNext` twice does not skip | pass |
| reactions: MOST, per-type (`LIKE`, `ANGRY`), NEWEST, unknown/lowercase type throws | pass |
| votes: TOP by score (not count), switching, CONTROVERSIAL, NEWEST, retracted votes | pass |
| extended reports: NEWEST (updates after retraction), LEAST, MOST_THEN_OLDEST tie-break | pass |
| trending window, BUSY, ACTIVE; leaderboards MESSAGES / POSTS / KARMA / REACTIONS | pass |
| 39 deliberately broken versions of these queries and add methods: every one is caught by the Task 5 tests | pass (see Task5prediction.md) |
