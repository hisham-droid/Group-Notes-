Task 4 is done and plugs straight into your Task 1–3 code. It passes 17 checks, including the A/B/C/D example from earlier, bad inputs, ties, and removed reports. All the Task 1–3 checks still pass too.

# Task 4: Viewing reports

## What it is (quick recap)

```java
Iterator<Message> getReportedMessages(String strategy, int amount)
```

- `"OLDEST"`: the message whose **oldest active** report is earliest comes first.
- `"MOST"`: the message with the **most active** reports comes first.
- Return at most `amount` messages, never a message with 0 reports, and each message at most once.
- Throw an exception for an unknown strategy or `amount <= 0`.
- You **must** use both the **Factory** and **Iterator** patterns (40% of the marks).

## The design (the thinking part)

**Most of the hard work is already done by Task 1.** `ReportStore` keeps one `MessageReports` per reported message. That gives us three requirements for free:

- **"Each message at most once"**: there's exactly one `MessageReports` per message.
- **"Never 0 reports"**: Task 1 deletes a message's entry when its last report is removed.
- **"Only active reports count"**: removed reports are actually deleted, so `count()` and `oldestTimestamp()` only see active ones.

So Task 4 only has to: **check the input → pick an ordering → sort → hand out the first `amount`.**

**Where the Factory goes: choosing the ordering.** The string `"OLDEST"` or `"MOST"` decides *how to compare* messages. In Java, "how to compare" is a `Comparator`. So `ReportOrderingFactory.create(strategy)` turns the string into the right `Comparator`. It's the one place that knows which strategies exist, and it throws if the string isn't one of them. Adding a new strategy later (say `"NEWEST"`) means adding one line there and changing nothing else.

**Where the Iterator goes: handing out results.** `ReportedMessageIterator` is our own class that implements `Iterator<Message>`. It sorts the reported messages once when created, then `next()` hands them out one at a time and stops after `amount`. Writing our own class (instead of returning `someList.iterator()`) is what shows the pattern, and it neatly handles the "at most `amount`" rule inside `hasNext()`.

**Why sort a copy?** The iterator copies the reported messages into its own list before sorting. If someone reports or un-reports something while a moderator is mid-way through the list, the iterator isn't corrupted. It keeps the order from when it was created. That matches how the project's own `SortedData` iterators behave (their docs say "the Iterator will not reflect changes").

**Why `ArrayList.sort` and not the project's `SortedData`?** `SortedData` treats two items the comparator calls "equal" as duplicates and drops one. With `"MOST"`, two messages with 3 reports each would compare equal, and one would silently vanish. `List.sort` keeps ties, and the spec allows ties in any order.

## The code

Two new files in `app/src/moderation/`, plus the method in `ModerationTools`.

### New: `moderation/ReportOrderingFactory.java` (the Factory)

```java
package moderation;

import java.util.Comparator;

/**
 * Creates the ordering used by ModerationTools.getReportedMessages for each supported strategy.
 */
public final class ReportOrderingFactory {
	public static final String OLDEST = "OLDEST";
	public static final String MOST = "MOST";

	private ReportOrderingFactory() {}

	/**
	 * @param strategy "OLDEST" to put the message with the earliest active report first,
	 *                 or "MOST" to put the message with the most active reports first
	 * @return a comparator that sorts reported messages into that order
	 * @throws IllegalArgumentException if the strategy is not recognised
	 */
	public static Comparator<MessageReports> create(String strategy) {
		if (strategy == null) throw new IllegalArgumentException("Strategy must not be null");
		return switch (strategy) {
			case OLDEST -> Comparator.comparingLong(MessageReports::oldestTimestamp);
			case MOST -> Comparator.comparingInt(MessageReports::count).reversed();
			default -> throw new IllegalArgumentException("Unknown strategy: " + strategy);
		};
	}
}
```

How to read it:

- **`Comparator.comparingLong(MessageReports::oldestTimestamp)`** means "compare two `MessageReports` by their `oldestTimestamp()`, smallest first." So the oldest report comes first, which is what `"OLDEST"` wants.
- **`Comparator.comparingInt(MessageReports::count).reversed()`** means "compare by `count()`," then flip it so the **biggest** count comes first, which is what `"MOST"` wants.
- **The `null` check comes first** because `switch` on a `null` string crashes with a `NullPointerException`. We'd rather throw the same clear `IllegalArgumentException` as for any other bad input.
- **`OLDEST`/`MOST` constants**: no "magic strings" scattered around, and other code (like tests) can use `ReportOrderingFactory.MOST`.
- **`final` class + private constructor**: it only has a static method, so nobody should create or subclass one.

### New: `moderation/ReportedMessageIterator.java` (the Iterator)

```java
package moderation;

import dao.model.Message;

import java.util.ArrayList;
import java.util.Collection;
import java.util.Comparator;
import java.util.Iterator;
import java.util.List;
import java.util.NoSuchElementException;

/**
 * Iterates over reported messages in a given order, stopping after a set number.
 * The order is fixed when the iterator is created: reports added or removed
 * afterwards do not change what this iterator returns.
 */
public class ReportedMessageIterator implements Iterator<Message> {
	private final List<MessageReports> ranked;
	private final int limit;
	private int index = 0;

	/**
	 * @param reported every message with at least one active report
	 * @param order the order in which to return them
	 * @param amount the maximum number of messages to return
	 */
	public ReportedMessageIterator(Collection<MessageReports> reported, Comparator<MessageReports> order, int amount) {
		this.ranked = new ArrayList<>(reported);
		this.ranked.sort(order);
		this.limit = Math.min(amount, ranked.size());
	}

	@Override
	public boolean hasNext() {
		return index < limit;
	}

	@Override
	public Message next() {
		if (!hasNext()) throw new NoSuchElementException();
		return ranked.get(index++).getMessage();
	}
}
```

How to read it:

- **Constructor**: copy the reported messages into a new list, sort it with the comparator from the factory, and work out the stopping point. `Math.min(amount, ranked.size())` covers "fewer if there aren't enough."
- **`hasNext()`**: "have we handed out fewer than `limit` yet?"
- **`next()`**: hand out the message at the current position and move along. Calling `next()` when there's nothing left throws `NoSuchElementException`, which is the official `Iterator` rule.
- It hands out the `Message` (what the moderator wants), not the `MessageReports` wrapper.

### `moderation/ModerationTools.java`: fill in `getReportedMessages`

Add `import java.util.Comparator;` with the other imports, then replace the stub:

```java
	/**
	 * Lists reported messages for moderators to review.
	 * @param strategy "OLDEST" or "MOST" (see ReportOrderingFactory)
	 * @param amount the maximum number of messages to return; must be positive
	 * @return the messages with at least one active report, in the requested order, each at most once
	 * @throws IllegalArgumentException if the strategy is unknown or amount is not positive
	 */
	public static Iterator<Message> getReportedMessages(String strategy, int amount) {
		if (amount <= 0) throw new IllegalArgumentException("Amount must be positive, but was " + amount);
		Comparator<MessageReports> order = ReportOrderingFactory.create(strategy);
		return new ReportedMessageIterator(ReportStore.getInstance().getAll(), order, amount);
	}
```

The whole function reads like the spec: check `amount`, get the ordering (the factory also checks `strategy`), return an iterator over the reported messages. Both inputs are checked **before** any work is done.

## Walking through the example

Using the example from earlier:

| Message | Active reports | `count()` | `oldestTimestamp()` |
|---|---|---|---|
| A | 50, 90 | 2 | 50 |
| B | 10 | 1 | 10 |
| C | 70, 80, 95 | 3 | 70 |
| D | *(removed, so not in the store at all)* | n/a | n/a |

`getReportedMessages("MOST", 2)`:

1. `amount = 2`, which is fine.
2. The factory returns "sort by count, biggest first."
3. The iterator copies [A, B, C] (D isn't there; Task 1 removed it), sorts to [C(3), A(2), B(1)], and sets `limit = min(2, 3) = 2`.
4. The caller gets **C, A**.

With `"OLDEST"`, it sorts by oldest timestamp to [B(10), A(50), C(70)], so `getReportedMessages("OLDEST", 2)` returns **B, A**.

## What I tested

| Test | Result |
|---|---|
| Nothing reported → empty iterator | ✅ |
| `OLDEST, 2` → B, A · `OLDEST, 10` → B, A, C | ✅ |
| `MOST, 2` → C, A · `MOST, 10` → C, A, B (D skipped, fewer than asked) | ✅ |
| `"NEWEST"`, lowercase `"oldest"`, `null`, `0`, `-3` → `IllegalArgumentException` | ✅ |
| After removing B's only report, B disappears | ✅ |
| After removing A's report at 50, A's oldest becomes 90, so it moves behind C | ✅ |
| Two messages tied on count → both returned, neither dropped | ✅ |
| Hidden messages still appear (moderators need to see them) | ✅ |
| `next()` past the end throws `NoSuchElementException` | ✅ |
| Adding reports after creating the iterator doesn't change its results | ✅ |
| All Task 1, 2 and 3 checks still pass | ✅ |

## Performance (if asked)

Sorting n reported messages costs O(n log n) comparisons. For `"OLDEST"`, each comparison calls `oldestTimestamp()`, which scans that message's reports. Messages rarely have many reports, so this is fine.

If you want to optimise, there are two options:

- Use a `PriorityQueue` of size `amount` instead of sorting everything. That's O(n log k), which is good when moderators only ask for the top 10.
- Have `MessageReports` remember its oldest timestamp.

Neither is needed for the spec.

## How this connects to the other tasks

- **Task 5 (black-box tests for this method):** a black-box test only uses `addReport`/`removeReport` to set things up and `getReportedMessages` to check results. The tests above are a good list of cases to turn into JUnit: each strategy, `amount` smaller and larger than the number reported, removed reports, the oldest active report changing after a removal, ties, and every exception case. Use `assertThrows(IllegalArgumentException.class, () -> ...)` for the exceptions. Clear `ReportStore`, `UserDAO` and `PostDAO` in `@Before`.
- **UML (group task):** `ReportOrderingFactory` --creates--> `Comparator<MessageReports>`, and `ReportedMessageIterator` --implements--> `Iterator<Message>`. Both are used by `ModerationTools`. These are good examples to show the two patterns in your diagram.
