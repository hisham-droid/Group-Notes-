Before writing this up I'll compile the code against the real project so what I give you actually builds.

I compiled the code below against your project and ran 15 checks on it (duplicates, unknown users and messages, removing, re-reporting). They all pass.

# Task 1: Reporting

## What it is

Any user can flag ("report") any message. You write three functions in `ModerationTools`:

| Function | Returns `true` when… | Returns `false` when… |
|---|---|---|
| `addReport(message, user, timestamp)` | the report was recorded | the message doesn't exist, the user doesn't exist, or this user already reported this message |
| `removeReport(message, user, timestamp)` | the user's report was deleted | the user wasn't reporting that message, or either UUID doesn't exist |
| `hasReported(message, user)` | this user currently has a report on this message | otherwise |

Extra rules:

- A user can report a message **only once**. After removing their report, they can report it again.
- You don't need to remember removed reports. Just delete them.
- Hiding a message (Task 2) must **not** delete its reports.
- 30% of the marks are for **performance**, so the choice of data structures matters.

## The design (the thinking part)

**1. What is a report?** One user flagging one message at a time. So a report is just *(user, timestamp)*, grouped under the message it's about:

```java
record Report(UUID user, long timestamp)
```

**2. How do we store them?** Think about the questions the code keeps asking:

- "Does message X have reports?"
- "Has user Y reported message X?"
- "Remove user Y's report on X."

Every question starts with a message, then a user. So the ideal structure is a **map inside a map**:

```
reportsByMessage:  messageUUID → MessageReports
                                   ├─ message (the actual Message object)
                                   └─ reportsByUser:  userUUID → Report
```

**3. Why `HashMap`?** A HashMap lookup, insert and delete are all **O(1)** (constant time, no matter how many reports exist). The alternatives are slower:

- A plain `ArrayList` of reports: you'd scan the whole list every time, O(n).
- The project's `SortedArrayList`: inserting shifts elements, O(n).
- An AVL tree: O(log n). Fine, but slower than a hash map, and you don't need sorting here (Task 4 sorts on its own).

This is the main argument for your 30% performance marks.

**4. Why keep the `Message` object, not just its UUID?** The project has no fast way to find a message by UUID. You have to search every post. If we store the `Message` the first time it's reported, later reports on it skip the search. Task 4 also needs the actual `Message` to return.

**5. Why a separate `ReportStore` class?** Task 3 (saving to disk) and Task 4 (viewing reports) both need the same report data. Putting it in one shared **singleton** (the same pattern as `PostDAO.getInstance()`) gives everyone one place to get it. It also keeps `ModerationTools` short.

**6. Tidy rule:** when a message's last report is removed, delete its entry entirely. Then every message in the store has at least one active report, and Task 4 can't accidentally return a message with 0 reports.

## The code

You add three new files in `app/src/moderation/`, a small addition to `PostDAO`, and you fill in `ModerationTools`.

### `moderation/Report.java` (new)

```java
package moderation;

import java.util.UUID;

/**
 * A single report: one user flagging one message at a point in time.
 * The message is not stored here because reports are always grouped
 * under their message (see MessageReports).
 */
public record Report(UUID user, long timestamp) {}
```

### `moderation/MessageReports.java` (new)

All the reports on **one** message.

```java
package moderation;

import dao.model.Message;

import java.util.Collection;
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.UUID;

/**
 * All active reports on one message, keyed by the UUID of the reporting user.
 * Keying by user gives O(1) duplicate checks, lookups and removals.
 */
public class MessageReports {
	private final Message message;
	private final Map<UUID, Report> reportsByUser = new HashMap<>();

	public MessageReports(Message message) {
		this.message = message;
	}

	public Message getMessage() {
		return message;
	}

	/**
	 * Adds a report from the given user.
	 * @return true if added, false if that user had already reported this message
	 */
	public boolean add(UUID user, long timestamp) {
		return reportsByUser.putIfAbsent(user, new Report(user, timestamp)) == null;
	}

	/**
	 * Removes the given user's report.
	 * @return true if a report was removed, false if that user had not reported this message
	 */
	public boolean remove(UUID user) {
		return reportsByUser.remove(user) != null;
	}

	public boolean hasReportFrom(UUID user) {
		return reportsByUser.containsKey(user);
	}

	/** @return the number of active reports on this message */
	public int count() {
		return reportsByUser.size();
	}

	public boolean isEmpty() {
		return reportsByUser.isEmpty();
	}

	/** @return the earliest timestamp among the active reports, or Long.MAX_VALUE if there are none */
	public long oldestTimestamp() {
		long oldest = Long.MAX_VALUE;
		for (Report report : reportsByUser.values())
			oldest = Math.min(oldest, report.timestamp());
		return oldest;
	}

	/** @return a read-only view of the active reports */
	public Collection<Report> getReports() {
		return Collections.unmodifiableCollection(reportsByUser.values());
	}
}
```

Two tricks in here:

- `putIfAbsent` adds the entry only if the key isn't there yet. It returns `null` if it added and the old value if not. So `== null` means "added successfully" in one line.
- `remove` returns the removed value, or `null` if nothing was there. So `!= null` means "something was removed."

`count()` and `oldestTimestamp()` aren't needed by Task 1 itself. They're there for Task 4 (`"MOST"` and `"OLDEST"`).

### `moderation/ReportStore.java` (new)

The shared storage for **every** report in the app.

```java
package moderation;

import dao.model.Message;

import java.util.Collection;
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.UUID;

/**
 * Stores every active report in the application, grouped by message.
 * Singleton, so that ModerationTools, persistence (Task 3) and
 * getReportedMessages (Task 4) all share the same data.
 */
public class ReportStore {
	private static ReportStore instance;

	private final Map<UUID, MessageReports> reportsByMessage = new HashMap<>();

	private ReportStore() {}

	public static ReportStore getInstance() {
		if (instance == null) instance = new ReportStore();
		return instance;
	}

	/** @return the reports on the given message, or null if it has no active reports */
	public MessageReports get(UUID message) {
		return reportsByMessage.get(message);
	}

	public boolean hasReported(UUID message, UUID user) {
		MessageReports reports = reportsByMessage.get(message);
		return reports != null && reports.hasReportFrom(user);
	}

	/**
	 * Records a report. The caller is responsible for checking that the user exists.
	 * @return true if added, false if that user had already reported this message
	 */
	public boolean add(Message message, UUID user, long timestamp) {
		return reportsByMessage
				.computeIfAbsent(message.id(), id -> new MessageReports(message))
				.add(user, timestamp);
	}

	/**
	 * Removes a report. Messages left with no reports are dropped entirely,
	 * so every entry in the store always has at least one active report.
	 * @return true if a report was removed, false if there was none to remove
	 */
	public boolean remove(UUID message, UUID user) {
		MessageReports reports = reportsByMessage.get(message);
		if (reports == null || !reports.remove(user)) return false;
		if (reports.isEmpty()) reportsByMessage.remove(message);
		return true;
	}

	/** @return a read-only view of every message that has at least one active report */
	public Collection<MessageReports> getAll() {
		return Collections.unmodifiableCollection(reportsByMessage.values());
	}

	/** Deletes all reports. Used by tests and before reloading from persistent storage. */
	public void clear() {
		reportsByMessage.clear();
	}
}
```

`computeIfAbsent(key, makeNew)` means: "get the entry for this message; if there isn't one, create it with `makeNew` and store it first." It saves you writing an `if (get(...) == null) put(...)` block.

### `dao/PostDAO.java` (add this method and one import)

At the top, next to the other imports:

```java
import java.util.UUID;
```

Inside the class, below `getAllMessages()`:

```java
	/**
	 * Finds a message by its UUID by searching every post.
	 * @param id the UUID of the message
	 * @return the message if it exists, null otherwise
	 */
	public Message getMessageByUUID(UUID id) {
		for (Iterator<Message> it = getAllMessages(); it.hasNext(); ) {
			Message message = it.next();
			if (message.id().equals(id)) return message;
		}
		return null;
	}
```

This mirrors the existing `UserDAO.getByUUID`, so it fits the codebase's style.

### `moderation/ModerationTools.java` (fill in the three methods)

Imports at the top:

```java
import dao.PostDAO;
import dao.UserDAO;
import dao.model.Message;
import java.util.Iterator;
import java.util.UUID;
```

The methods (leave `setHidden` and `getReportedMessages` for Tasks 2 and 4):

```java
	/**
	 * Records that a user has reported a message.
	 * @return true if the report was recorded; false if the message or user
	 * does not exist, or the user has already reported this message
	 */
	public static boolean addReport(UUID message, UUID user, long timestamp) {
		ReportStore store = ReportStore.getInstance();
		if (store.hasReported(message, user)) return false;
		if (UserDAO.getInstance().getByUUID(user) == null) return false;

		Message target = findMessage(store, message);
		if (target == null) return false;

		return store.add(target, user, timestamp);
	}

	/**
	 * Retracts a user's report on a message.
	 * @return true if a report was removed; false if the user had not reported
	 * this message (which includes the case where either UUID does not exist)
	 */
	public static boolean removeReport(UUID message, UUID user, long timestamp) {
		return ReportStore.getInstance().remove(message, user);
	}

	/** @return true if the user currently has an active report on the message */
	public static boolean hasReported(UUID message, UUID user) {
		return ReportStore.getInstance().hasReported(message, user);
	}

	/**
	 * Finds a message, reusing the copy already held by the report store when
	 * the message has been reported before, to avoid searching every post.
	 */
	private static Message findMessage(ReportStore store, UUID message) {
		MessageReports existing = store.get(message);
		if (existing != null) return existing.getMessage();
		return PostDAO.getInstance().getMessageByUUID(message);
	}
```

## How `addReport` works, step by step

The checks are ordered **cheapest first**:

1. **Already reported?** That's two HashMap lookups, O(1). If yes, return `false` straight away. If they've reported it, the user and message must exist, so there's no point checking further.
2. **Does the user exist?** `UserDAO.getByUUID` (existing code) scans the users.
3. **Does the message exist?** If someone has reported it before, the store already holds the `Message`, so it's O(1). Only a never-reported message triggers the search through all posts.
4. **Record it.** `store.add(...)` creates the message's entry if needed and adds the report.

**Why `removeReport` and `hasReported` don't check existence at all:** if a message or user doesn't exist, there can't be a report linking them. So "is there a report to remove?" already returns `false` for those cases. This keeps both functions O(1) while still matching the spec exactly.

`null` inputs are handled too: `HashMap` accepts `null` keys, and the lookups just find nothing, so you get `false` (I tested this).

## Performance summary (good for explaining your choices)

| Operation | Cost |
|---|---|
| `hasReported` | O(1) |
| `removeReport` | O(1) |
| `addReport`, duplicate check | O(1) |
| `addReport`, message already reported before | O(1) message lookup |
| `addReport`, first report on a message | O(total messages) search, using existing project data |
| `addReport`, user check | O(users), from the existing `UserDAO.getByUUID` |

The spec says the performance mark applies only to **your** code, not pre-existing code. The slow parts are the existing DAO lookups. Everything you wrote is O(1).

## How this connects to the other tasks

- **Task 2 (hiding):** nothing to do. Reports live in `ReportStore`, separate from the message's visibility, so hiding never touches them.
- **Task 3 (saving to disk):** they loop over `ReportStore.getInstance().getAll()`, and for each `MessageReports`, over `getReports()`. Each line saved is `messageId, userId, timestamp`, which is easy to put in CSV. When loading, call `ReportStore.getInstance().clear()`, then call `ModerationTools.addReport(...)` for each saved line *after* messages and users are loaded. That reuses all your validation for free.
- **Task 4 (viewing reports):** they use `ReportStore.getInstance().getAll()`, plus `count()`, `oldestTimestamp()` and `getMessage()`. It's exactly the shape I described last time.
- **Task 5 (tests for `addReport`):** branch-complete coverage needs a test for each `return false` line plus the success case: already reported, unknown user, unknown message, new message, and a message that's already been reported by someone else (covers both sides of `findMessage`). Tests should call `ReportStore.getInstance().clear()` and `PostDAO`/`UserDAO` `.clear()` in an `@Before` method. These are singletons, so data from one test would otherwise leak into the next.

## Two things to confirm with your group

- **`removeReport`'s `timestamp` parameter** isn't mentioned anywhere in the spec, so this code ignores it. If your group decides it should mean something, like "only remove if the timestamps match," it's a one-line change in `ReportStore.remove`.
- **`addReport` silently relies on `UserDAO.getByUUID`.** If someone refactors `UserDAO` (it has a TODO about the singleton pattern), make sure that method stays.
