# Task 1 prediction: the core feature (software design + data structures)

> **Every prediction in this file has complete code below**, compiled with JDK 21 and tested with JUnit 4.13.2
> on top of the practice-hackathon base (201 tests in total across the pack, all passing). Section 21 lists what was tested.

## 0. How to use this file tomorrow

1. Read the real Task 1 spec. Find its row in **section 2** and jump to that section.
2. Do the **base fixes in section 3** first (about 10 minutes). Every theme needs them for the efficiency marks.
3. Copy the closest section, rename things to match the spec, and check the **trap list in section 20**.

---------------------------------------------------------------------

## 1. What Task 1 looks like (from the practice hackathon)

| Part | Marks | What earns it |
|---|---|---|
| Behaviour | 50% | `add`, `remove`, `has` work exactly as specified, including every "return false" case |
| Efficiency | 30% | no scanning all messages/users/posts inside add/remove/has: use HashMaps and indexes |
| Quality | 20% | small classes, javadoc, no duplicated code, sensible names |

| Spec sentence | Handling |
|---|---|
| "If the message or user does not exist ... return false" | `MessageIndex.get(...) != null && UserDAO.getByUUID(...) != null` (both O(1)) |
| "if that user has already reported that message, do nothing and return false" | `map.putIfAbsent(user, x) != null` returns false without changing anything |
| "removeReport returns false if the user was not reporting" | `map.remove(user) == null` means false |
| "even if a message is hidden, the reports should still remain" | each feature has its own store; hiding never touches it |
| "performance is important ... applies only to your code" | new methods must be O(1) / O(log n); old slow code may stay |

---------------------------------------------------------------------

## 2. Every theme I predict for Task 1 (all coded)

| # | Theme | Methods | Section |
|---|---|---|---|
| 1 | Reports / moderation (the practice itself) | `addReport`, `removeReport`, `hasReported` | 4 |
| 2 | Reactions (`ReactionType` is named in the practice spec) | `addReaction`, `changeReaction`, `removeReaction`, `getReaction`, `countReactions` | 5 |
| 3 | Up/down votes and score | `vote(msg,user,+1/-1)`, `removeVote`, `getVote`, `getScore` | 7 |
| 4 | Likes on posts | `likePost`, `unlikePost`, `hasLikedPost`, `getPostLikeCount` | 6 |
| 5 | Follow users | `follow`, `unfollow`, `isFollowing`, `getFollowers` | 6 |
| 6 | Block users (and hide their messages from you) | `block`, `isBlocked`, `getMessagesFor(post, viewer)` | 6 |
| 7 | Bookmarks | `bookmark`, `isBookmarked`, `getBookmarks` | 6 |
| 8 | Subscribe to a post + notifications | `subscribe`, `getNotifications`, `getUnreadCount`, `markAllRead` | 8 |
| 9 | Pin messages (admin) | `setPinned`, `getMessagesPinnedFirst` | Task2prediction.md section 5 |
| 10 | Tags / hashtags | `addTag`, `removeTag`, `getTags`, `getPostsWithTag`, `getPopularTags` | 9 |
| 11 | Polls (vote once, change, retract) | `createPoll`, `vote`, `changeVote`, `removeVote`, `getResults`, `getWinners` | 10 |
| 12 | Read / unread tracking | `markRead`, `getLastRead`, `getUnreadCount` | 11 |
| 13 | Bans with expiry | `ban`, `unban`, `isBanned` (and banned users cannot reply) | 12 |
| 14 | Friend requests | `sendRequest`, `accept`, `decline`, `areFriends`, `getFriends` | 13 |
| 15 | @mentions | `extractUsernames`, `getMentions` | 14 |
| 16 | Edit history (+ undo) | `editMessage`, `getHistory`, `undoLastEdit` | 15 |
| 17 | Account deletion / password / username / role changes | `deleteAccount`, `displayName`, `changePassword`, `changeUsername`, `setRole` | 16 |
| 18 | Moderator role | `setModerator`, `canModerate` | 16 |
| 19 | Rate limiting / spam | `tryAcquire(user, now)`, `remaining` | 17 |
| 20 | Search, DMs, scheduled posts, boards, censor, audit log, stats... | | ExtraPredictions.md |

---------------------------------------------------------------------

## 3. Base fixes every theme needs (do these first)

### 3.1 O(1) user lookup by UUID (+ shared username/password rules, + remove)

The provided `UserDAO.getByUUID` loops over every user (O(n)). **Trap:** `DAO`'s constructor calls `clear()`
*before* the subclass's fields are initialised, so the index map cannot be `final` and is created inside `clear()`.
The constructor is `private` (the practice base had it `public`, which fails the Singleton style test).

**`app/src/dao/UserDAO.java`** (full file)

```java
package dao;

import dao.model.User;

import java.util.HashMap;
import java.util.Map;
import java.util.UUID;

public class UserDAO extends DAO<User> {
	// TODO: apply the Singleton design pattern to this class.
	// You may modify the existing constructor, add new constructors,
	// and add new helper method and private fields.
	/**
	 * Generates a UserDAO. We enforce uniqueness in usernames (but not in passwords),
	 * and further two usernames are considered identical if they are equal, ignoring case
	 */
	private UserDAO() { // HACKATHON: private, so it is a real Singleton
		super((o1, o2) -> o1.username().compareToIgnoreCase(o2.username()));
	}
	
	private static UserDAO instance;
	public static UserDAO getInstance() {
		if (instance == null) instance = new UserDAO();
		return instance;
	}

	/**
	 * Attempts to authenticate as a particular user. If the user exists
	 * and their passwords match, the login is considered successful.
	 * @param username the username
	 * @param password the password
	 * @return the User if successful, null otherwise
	 */
	public User login(String username, String password) {
		User user = data.get(new User(username));
		return (user != null && user.password().equals(password)) ? user : null;
	}

	/**
	 * Attempts to register a new user. Users must have unique usernames,
	 * and their usernames must contain only alphanumeric characters.
	 * Usernames can be between 4 and 20 characters long.
	 * Passwords must be at least four characters long, and can include
	 * any codepoints.
	 * @param username the desired username
	 * @param password the desired password
	 * @return the newly-created User if successful, null otherwise
	 */
	public User register(String username, String password) {
		if (!isValidUsername(username) || !isValidPassword(password)) return null;
		if (data.get(new User(username)) != null) return null; // taken (case-insensitive)

		User newUser = new User(UUID.randomUUID(), User.Role.Member, username, password);
		return add(newUser) ? newUser : null; // HACKATHON: go through add() so the UUID index is updated
	}

	/** HACKATHON: shared by register and AccountTools (no duplicated rules). 4-20 letters/digits. */
	public static boolean isValidUsername(String username) {
		if (username == null || username.length() < 4 || username.length() > 20) return false;
		for (char c : username.toCharArray()) {
			if (!Character.isLetterOrDigit(c)) return false;
		}
		return true;
	}

	/** HACKATHON: at least 4 characters, any code points. */
	public static boolean isValidPassword(String password) {
		return password != null && password.length() >= 4;
	}

	// ---------------------------------------------------------------------
	// HACKATHON: O(1) lookup by UUID (the original getByUUID scanned every user)
	// ---------------------------------------------------------------------

	// NOT final and NOT initialised here on purpose: DAO's constructor calls clear()
	// BEFORE this class's field initialisers run, so clear() creates the map instead.
	private Map<UUID, User> usersById;

	@Override
	public boolean add(User user) {
		if (!super.add(user)) return false;      // duplicate username -> nothing to index
		if (user.id() != null) usersById.put(user.id(), user);
		return true;
	}

	@Override
	public boolean remove(User user) {
		User stored = get(user);                 // the comparator matches by username
		if (stored == null || !super.remove(stored)) return false;
		if (stored.id() != null) usersById.remove(stored.id());
		return true;
	}

	@Override
	public void clear() {
		super.clear();
		usersById = new HashMap<>();
	}

	/**
	 * Fetches a User by just a UUID, in O(1).
	 * @param id the UUID to search for
	 * @return the user if they exist, else null
	 */
	public User getByUUID(UUID id) {
		if (id == null) return null;
		return usersById.get(id);
	}
}
```

`DAO` gains a `remove` (used by account deletion and username changes):

> How to read a diff block: lines starting with `+` are new, lines starting with `-` are deleted, other lines are unchanged context so you can find the spot. `@@` lines just mark where the next chunk starts.

**`app/src/dao/DAO.java`** (changes only: `+` = add this line, `-` = remove this line)

```diff
@@ -38,6 +38,15 @@
 	}
 
 	/**
+	 * HACKATHON: removes an element (matched with the comparator).
+	 * @param element the element (or a key object that compares equal to it)
+	 * @return true if something was removed
+	 */
+	public boolean remove(T element) {
+		return data.remove(element);
+	}
+
+	/**
 	 * Resets the DAO into its initial state, where it stores no elements
 	 */
 	public void clear() {
```

### 3.2 O(1) message lookup by UUID (`MessageIndex`) + "a message was added" events

Messages live inside each Post's tree, sorted by (timestamp, thread, poster, id), so you **cannot** look one up by id.
Fix: `SortedData` gets an Observer list, every Post registers `MessageIndex` on its tree, and the index keeps
`HashMap<UUID, Message>`. The index also **forwards** events: features that care about every new message
(notifications, mentions, search) subscribe once with `addMessageListener` / `addChangeListener`.

**`app/src/sorteddata/SortedData.java`** (changes only: `+` = add this line, `-` = remove this line)

```diff
@@ -1,6 +1,8 @@
 package sorteddata;
 
+import java.util.ArrayList;
 import java.util.Iterator;
+import java.util.List;
 
 /**
  * Maintains a set of data, with no duplicates, in sorted order.
@@ -59,4 +61,72 @@
 	 * @implNote This function uses PRNG and is thus not suitable for cryptographic purposes.
 	 */
 	public abstract T getRandom();
+
+	// ---------------------------------------------------------------------
+	// HACKATHON ADDITIONS
+	// ---------------------------------------------------------------------
+
+	// Observer pattern: anyone who wants to know when an element is added
+	// (e.g. a UUID index, or a "visible messages" tree) registers here.
+	// Kept in the abstract parent so EVERY implementation gets it for free.
+	private final List<SortedDataSubject<T>> listeners = new ArrayList<>();
+
+	/**
+	 * Registers a listener that is told about every successful insert.
+	 * @param listener the listener to notify
+	 */
+	public void registerListener(SortedDataSubject<T> listener) {
+		listeners.add(listener);
+	}
+
+	/**
+	 * Stops notifying a listener.
+	 * @param listener the listener to remove
+	 */
+	public void deregisterListener(SortedDataSubject<T> listener) {
+		listeners.remove(listener);
+	}
+
+	/**
+	 * Implementations call this after a SUCCESSFUL insert (not for duplicates).
+	 * @param element the element that was just added
+	 */
+	protected void notifyAdded(T element) {
+		for (SortedDataSubject<T> listener : listeners) {
+			listener.onAdd(element);
+		}
+	}
+
+	/**
+	 * Removes a value from the data structure. Not every implementation
+	 * supports this, so the default throws (same idea as Iterator.remove()).
+	 * @param value the value to remove (matched with the comparator)
+	 * @return true if something was removed, false if it was not present
+	 */
+	public boolean remove(T value) {
+		throw new UnsupportedOperationException(getClass().getSimpleName() + " does not support remove");
+	}
+
+	/**
+	 * @return the number of elements currently stored
+	 */
+	public abstract int size();
+
+	/**
+	 * HACKATHON (order statistics): how many stored elements are STRICTLY smaller than value.
+	 * Works even if value itself is not stored. Equals the index value would have.
+	 */
+	public int rank(T value) {
+		throw new UnsupportedOperationException(getClass().getSimpleName() + " does not support rank");
+	}
+
+	/** @return the largest stored element <= value, or null */
+	public T floor(T value) {
+		throw new UnsupportedOperationException(getClass().getSimpleName() + " does not support floor");
+	}
+
+	/** @return the smallest stored element >= value, or null */
+	public T ceiling(T value) {
+		throw new UnsupportedOperationException(getClass().getSimpleName() + " does not support ceiling");
+	}
 }
```

**`app/src/sorteddata/avltree/AVLTree.java`** (changes only: `+` = add this line, `-` = remove this line)

```diff
@@ -26,6 +26,7 @@
 	public boolean insert(T element) {
 		if (root.contains(element)) return false;
 		root = root.insert(element);
+		notifyAdded(element); // HACKATHON: tell listeners (Observer)
 		return true;
 	}
 
@@ -50,4 +51,33 @@
 	public Iterator<T> getRange(T start, int count, boolean backwards) {
 		return new AVLIterator<>(start, root, comparator, count, backwards);
 	}
+
+	// HACKATHON: O(log n) removal using the immutable delete in AVLNodeFilled
+	@Override
+	public boolean remove(T element) {
+		if (!root.contains(element)) return false;
+		root = root.delete(element);
+		return true;
+	}
+
+	@Override
+	public int size() {
+		return root.size();
+	}
+
+	// HACKATHON: O(log n) order statistics
+	@Override
+	public int rank(T value) {
+		return root.rank(value);
+	}
+
+	@Override
+	public T floor(T value) {
+		return root.floor(value);
+	}
+
+	@Override
+	public T ceiling(T value) {
+		return root.ceiling(value);
+	}
 }
```

**`app/src/dao/MessageIndex.java`** (full file)

```java
package dao;

import dao.model.Message;
import dao.model.Post;
import sorteddata.SortedDataSubject;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;

/**
 * HACKATHON: finds a Message from just its UUID in O(1) (expected).
 * <p>
 * Why it is needed: messages live inside each Post's SortedData, which is ordered by
 * (timestamp, thread, poster, id). You cannot search it by id alone, so without an
 * index "does this message exist?" means scanning every message of every post: O(n).
 * <p>
 * How it stays up to date: every Post registers this index as a listener on its
 * messages (Observer pattern), so any insert anywhere, including test code calling
 * post.messages.insert(...) directly, is recorded automatically.
 * <p>
 * Singleton, like the other DAOs.
 */
public class MessageIndex implements SortedDataSubject<Message> {
	private static MessageIndex instance;

	public static MessageIndex getInstance() {
		if (instance == null) instance = new MessageIndex();
		return instance;
	}

	private final Map<UUID, Message> messagesById = new HashMap<>();

	private MessageIndex() {}

	// Other features (mentions, notifications, ...) that want to hear about EVERY new
	// message in EVERY post subscribe here once, instead of registering on each Post.
	private final List<SortedDataSubject<Message>> messageListeners = new ArrayList<>();

	/** Subscribe to every message added anywhere (Observer). */
	public void addMessageListener(SortedDataSubject<Message> listener) {
		messageListeners.add(listener);
	}

	public void removeMessageListener(SortedDataSubject<Message> listener) {
		messageListeners.remove(listener);
	}

	// Listeners that also want EDITS (a re-inserted message with a known id),
	// e.g. a search index that must learn the new words.
	private final List<SortedDataSubject<Message>> changeListeners = new ArrayList<>();

	/** Subscribe to every new OR edited message. */
	public void addChangeListener(SortedDataSubject<Message> listener) {
		changeListeners.add(listener);
	}

	@Override
	public void onAdd(Message message) {
		if (message.id() == null) return;
		boolean isNew = messagesById.put(message.id(), message) == null;
		// an edit re-inserts a message with a known id: update the index, but do not
		// tell listeners about a "new" message (no duplicate notifications)
		if (isNew) {
			for (SortedDataSubject<Message> listener : messageListeners) listener.onAdd(message);
		}
		for (SortedDataSubject<Message> listener : changeListeners) listener.onAdd(message);
	}

	/** Forget a message that was permanently removed (hard delete / edit replacement). */
	public void forget(UUID id) {
		messagesById.remove(id);
	}

	/**
	 * Finds a message by UUID.
	 * The index can contain "stale" messages from posts that were later removed
	 * (e.g. PostDAO.clear() between tests), so we double-check the message's post
	 * is still stored. That check is an O(log n) PostDAO lookup.
	 * @param id the message UUID
	 * @return the Message, or null if it does not exist in the application
	 */
	public Message get(UUID id) {
		if (id == null) return null;
		Message message = messagesById.get(id);
		if (message == null || message.thread() == null) return null;
		Post post = PostDAO.getInstance().get(new Post(message.thread()));
		if (post == null || post.messages.get(message) == null) return null;
		return message;
	}

	/**
	 * @param message a message that exists
	 * @return the Post the message belongs to, or null
	 */
	public Post getPostOf(Message message) {
		if (message == null || message.thread() == null) return null;
		return PostDAO.getInstance().get(new Post(message.thread()));
	}

	/** Forget everything (useful in tests). */
	public void clear() {
		messagesById.clear();
	}
}
```

### 3.3 Null-safe `MessageComparator`

Test code builds `new Message(id, null, post.id, 0, "Hi")`; two such messages with the same timestamp crash the original.
Being null-safe also lets Task 2 build "probe" messages for time-range queries.

**`app/src/dao/MessageComparator.java`** (changes only: `+` = add this line, `-` = remove this line)

```diff
@@ -30,13 +30,22 @@
 		int delta = Long.compare(o1.timestamp(), o2.timestamp());
 		if (delta != 0) return delta;
 
-		delta = o1.thread().compareTo(o2.thread());
+		delta = compareNullable(o1.thread(), o2.thread());
 		if (delta != 0) return delta;
 
-		delta = o1.poster().compareTo(o2.poster());
+		delta = compareNullable(o1.poster(), o2.poster());
 		if (delta != 0) return delta;
 
-		delta = o1.id().compareTo(o2.id());
+		delta = compareNullable(o1.id(), o2.id());
 		return delta;
 	}
+
+	// HACKATHON: test code often builds Messages with a null poster/thread.
+	// Without this, two such messages with the same timestamp throw NullPointerException.
+	private static <C extends Comparable<C>> int compareNullable(C a, C b) {
+		if (a == b) return 0;
+		if (a == null) return -1;
+		if (b == null) return 1;
+		return a.compareTo(b);
+	}
 }
```

---------------------------------------------------------------------

## 4. Reports (the practice Task 1)

Design: one `MessageReports` per reported message with `HashMap<user, Report>` (O(1) add/remove/has) and a `TreeSet`
by time (oldest/newest active report in O(log k), still right after removals). `ReportStore` maps message to group.
`ModerationTools` only validates and delegates (Facade).

**`app/src/moderation/Report.java`** (full file)

```java
package moderation;

import java.util.UUID;

/**
 * HACKATHON (Task 1): one user's report on one message.
 * A record = immutable value class with equals/hashCode/toString for free.
 * @param message   the reported message's UUID
 * @param user      the reporting user's UUID
 * @param timestamp when the report was made (UNIX ms)
 */
public record Report(UUID message, UUID user, long timestamp) {}
```

**`app/src/moderation/MessageReports.java`** (full file)

```java
package moderation;

import java.util.Collection;
import java.util.Collections;
import java.util.Comparator;
import java.util.HashMap;
import java.util.Map;
import java.util.TreeSet;
import java.util.UUID;

/**
 * HACKATHON (Task 1): all ACTIVE reports on ONE message.
 * <p>
 * Two structures, each picked for one job:
 * <ul>
 *   <li>HashMap user -> Report: add / remove / hasReported in O(1)</li>
 *   <li>TreeSet ordered by time: "oldest active report" in O(log k), and it stays
 *       correct after a removal (a plain "first timestamp" field would go stale)</li>
 * </ul>
 * activeCount() is O(1) (the map's size), which Task 4's "MOST" strategy needs.
 */
public class MessageReports {
	// ties on timestamp are broken by user so two different reports are never "equal"
	private static final Comparator<Report> BY_TIME =
			Comparator.comparingLong(Report::timestamp).thenComparing(Report::user);

	private final UUID messageId;
	private final Map<UUID, Report> reportsByUser = new HashMap<>();
	private final TreeSet<Report> reportsByTime = new TreeSet<>(BY_TIME);

	public MessageReports(UUID messageId) {
		this.messageId = messageId;
	}

	/** @return false if this user already reported the message */
	public boolean add(Report report) {
		if (reportsByUser.putIfAbsent(report.user(), report) != null) return false;
		reportsByTime.add(report);
		return true;
	}

	/** @return false if this user had no active report */
	public boolean remove(UUID user) {
		Report removed = reportsByUser.remove(user);
		if (removed == null) return false;
		reportsByTime.remove(removed);
		return true;
	}

	public boolean hasReported(UUID user) {
		return reportsByUser.containsKey(user);
	}

	public int activeCount() {
		return reportsByUser.size();
	}

	public boolean isEmpty() {
		return reportsByUser.isEmpty();
	}

	/** @return timestamp of the oldest ACTIVE report; only call when not empty */
	public long oldestTimestamp() {
		return reportsByTime.first().timestamp();
	}

	/** @return timestamp of the newest ACTIVE report; only call when not empty */
	public long newestTimestamp() {
		return reportsByTime.last().timestamp();
	}

	public UUID getMessageId() {
		return messageId;
	}

	/** @return every active report (read-only), used for persistence */
	public Collection<Report> getReports() {
		return Collections.unmodifiableCollection(reportsByUser.values());
	}
}
```

**`app/src/moderation/ReportStore.java`** (full file)

```java
package moderation;

import java.util.ArrayList;
import java.util.Collection;
import java.util.Collections;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.UUID;

/**
 * HACKATHON (Task 1): every report in the application, grouped by message.
 * HashMap message -> MessageReports, so finding a message's reports is O(1).
 * Singleton, like the DAOs (one shared store for the whole app).
 * <p>
 * Reports stay here even if the message is hidden (spec requirement).
 */
public class ReportStore {
	private static ReportStore instance;

	public static ReportStore getInstance() {
		if (instance == null) instance = new ReportStore();
		return instance;
	}

	private final Map<UUID, MessageReports> reportsByMessage = new HashMap<>();

	private ReportStore() {}

	/** @return false if this user already reported this message */
	public boolean add(Report report) {
		return reportsByMessage
				.computeIfAbsent(report.message(), MessageReports::new)
				.add(report);
	}

	/** @return false if there was no such active report */
	public boolean remove(UUID message, UUID user) {
		MessageReports reports = reportsByMessage.get(message);
		if (reports == null || !reports.remove(user)) return false;
		// drop empty entries so Task 4 never sees messages with zero active reports
		if (reports.isEmpty()) reportsByMessage.remove(message);
		return true;
	}

	public boolean hasReported(UUID message, UUID user) {
		MessageReports reports = reportsByMessage.get(message);
		return reports != null && reports.hasReported(user);
	}

	/** @return the per-message report groups (read-only), used by Task 4 */
	public Collection<MessageReports> getAllMessageReports() {
		return Collections.unmodifiableCollection(reportsByMessage.values());
	}

	/** @return every individual report, used for persistence (Task 3) */
	public Iterator<Report> getAllReports() {
		List<Report> all = new ArrayList<>();
		for (MessageReports reports : reportsByMessage.values()) all.addAll(reports.getReports());
		return all.iterator();
	}

	/** Account deletion: drops every report made by this user. O(number of reported messages). */
	public void removeAllByUser(UUID user) {
		reportsByMessage.values().removeIf(reports -> reports.remove(user) && reports.isEmpty());
	}

	/** Hard delete: drops every report on this message. */
	public void removeAllForMessage(UUID message) {
		reportsByMessage.remove(message);
	}

	public void clear() {
		reportsByMessage.clear();
	}
}
```

**`app/src/moderation/ModerationTools.java`** (full file)

```java
package moderation;

import dao.MessageIndex;
import dao.UserDAO;
import dao.model.Message;
import dao.model.Post;
import dao.model.User;

import java.util.Iterator;
import java.util.UUID;

/**
 * HACKATHON: entry point for the moderation module (a FACADE over the classes
 * in this package). Every method validates its input, then delegates.
 */
public class ModerationTools {
	private ModerationTools() {} // static-only

	// ---------------------------------------------------------------- Task 1

	/**
	 * @return true if recorded; false if message/user does not exist or the user already reported it
	 */
	public static boolean addReport(UUID message, UUID user, long timestamp) {
		if (!messageAndUserExist(message, user)) return false;
		return ReportStore.getInstance().add(new Report(message, user, timestamp));
	}

	/**
	 * The timestamp parameter is required by the given signature but not needed:
	 * a user has at most one active report per message.
	 * @return true if removed; false if no such report or either UUID does not exist
	 */
	public static boolean removeReport(UUID message, UUID user, long timestamp) {
		if (!messageAndUserExist(message, user)) return false;
		return ReportStore.getInstance().remove(message, user);
	}

	/** @return true if this user currently has an active report on this message */
	public static boolean hasReported(UUID message, UUID user) {
		return ReportStore.getInstance().hasReported(message, user);
	}

	// ---------------------------------------------------------------- Task 2

	/**
	 * @return false (and change nothing) if either UUID does not exist or the user is not an Admin
	 */
	public static boolean setHidden(UUID message, UUID user, boolean hidden) {
		Message target = MessageIndex.getInstance().get(message);
		User moderator = UserDAO.getInstance().getByUUID(user);
		if (target == null || moderator == null || moderator.role() != User.Role.Admin) return false;

		Post post = MessageIndex.getInstance().getPostOf(target);
		post.setHidden(target, hidden);
		return true;
	}

	// ---------------------------------------------------------------- Task 4

	/**
	 * @param strategy "OLDEST" or "MOST"
	 * @param amount   maximum number of messages to return (must be positive)
	 * @throws IllegalArgumentException for an unknown strategy or a non-positive amount
	 */
	public static Iterator<Message> getReportedMessages(String strategy, int amount) {
		if (amount <= 0) throw new IllegalArgumentException("amount must be positive, was " + amount);
		return new ReportedMessageIterator(
				ReportStore.getInstance().getAllMessageReports(),
				ReportOrderingFactory.create(strategy),
				amount);
	}

	// ---------------------------------------------------------------- helpers

	// both lookups are O(1) thanks to MessageIndex and UserDAO's UUID index
	private static boolean messageAndUserExist(UUID message, UUID user) {
		return MessageIndex.getInstance().get(message) != null
				&& UserDAO.getInstance().getByUUID(user) != null;
	}
}
```

---------------------------------------------------------------------

## 5. Reactions (top prediction)

`MessageReactions` per message: `HashMap<user, Reaction>` + `EnumMap<ReactionType, Integer>` counts updated on every
change. `EnumMap` works with whatever constants the real enum has. **Check the spec:** here a second reaction returns
false and `changeReaction` switches; if the spec says "a new reaction replaces the old", make `addReaction` call `change`.

**`app/src/reactions/ReactionType.java`** (full file)

```java
package reactions;

/**
 * PREDICTION: the practice spec says "Do not modify the ReactionType enum", so a real
 * version ships this enum. The constants below are a guess; your code must work for ANY
 * constants, so never hard-code them (use EnumMap / values()).
 */
public enum ReactionType { LIKE, LOVE, LAUGH, WOW, SAD, ANGRY }
```

**`app/src/reactions/Reaction.java`** (full file)

```java
package reactions;

import java.util.UUID;

/** One user's reaction to one message. */
public record Reaction(UUID message, UUID user, ReactionType type, long timestamp) {}
```

**`app/src/reactions/MessageReactions.java`** (full file)

```java
package reactions;

import java.util.Collection;
import java.util.Collections;
import java.util.EnumMap;
import java.util.HashMap;
import java.util.Map;
import java.util.UUID;

/**
 * All reactions on ONE message.
 *  - HashMap user -> Reaction: add / remove / getReaction in O(1)
 *  - EnumMap type -> count: countReactions in O(1) (kept up to date, never recounted)
 * EnumMap is an array indexed by the enum's ordinal: fast and memory-light.
 */
public class MessageReactions {
	private final UUID messageId;
	private final Map<UUID, Reaction> byUser = new HashMap<>();
	private final EnumMap<ReactionType, Integer> counts = new EnumMap<>(ReactionType.class);
	private long latestTimestamp = Long.MIN_VALUE;

	public MessageReactions(UUID messageId) {
		this.messageId = messageId;
	}

	/** @return false if this user already reacted (use change() to switch type) */
	public boolean add(Reaction reaction) {
		if (byUser.putIfAbsent(reaction.user(), reaction) != null) return false;
		counts.merge(reaction.type(), 1, Integer::sum);
		latestTimestamp = Math.max(latestTimestamp, reaction.timestamp());
		return true;
	}

	/** @return false if the user had no reaction */
	public boolean remove(UUID user) {
		Reaction removed = byUser.remove(user);
		if (removed == null) return false;
		// decrement; drop the key at zero so counts never holds zeros
		counts.computeIfPresent(removed.type(), (type, count) -> count == 1 ? null : count - 1);
		return true;
	}

	/** Replace a user's reaction with a different type. @return false if they had none */
	public boolean change(Reaction replacement) {
		if (!remove(replacement.user())) return false;
		return add(replacement);
	}

	public ReactionType getReaction(UUID user) {
		Reaction reaction = byUser.get(user);
		return reaction == null ? null : reaction.type();
	}

	public int count(ReactionType type) {
		return counts.getOrDefault(type, 0);
	}

	public int total() {
		return byUser.size();
	}

	public boolean isEmpty() {
		return byUser.isEmpty();
	}

	/** Note: not lowered by remove() (would need a TreeSet like MessageReports). */
	public long latestTimestamp() {
		return latestTimestamp;
	}

	public UUID getMessageId() {
		return messageId;
	}

	public Collection<Reaction> getReactions() {
		return Collections.unmodifiableCollection(byUser.values());
	}
}
```

**`app/src/reactions/ReactionStore.java`** (full file)

```java
package reactions;

import java.util.Collection;
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.UUID;

/** Singleton: message UUID -> its reactions. */
public class ReactionStore {
	private static ReactionStore instance;

	public static ReactionStore getInstance() {
		if (instance == null) instance = new ReactionStore();
		return instance;
	}

	private final Map<UUID, MessageReactions> byMessage = new HashMap<>();

	private ReactionStore() {}

	public MessageReactions forMessage(UUID message) {
		return byMessage.computeIfAbsent(message, MessageReactions::new);
	}

	public MessageReactions find(UUID message) {
		return byMessage.get(message);
	}

	/** call after a removal so empty groups disappear */
	public void dropIfEmpty(UUID message) {
		MessageReactions reactions = byMessage.get(message);
		if (reactions != null && reactions.isEmpty()) byMessage.remove(message);
	}

	public Collection<MessageReactions> all() {
		return Collections.unmodifiableCollection(byMessage.values());
	}

	/** Account deletion: drops every reaction by this user. */
	public void removeAllByUser(java.util.UUID user) {
		byMessage.values().removeIf(reactions -> reactions.remove(user) && reactions.isEmpty());
	}

	public void clear() {
		byMessage.clear();
	}
}
```

**`app/src/reactions/ReactionTools.java`** (full file)

```java
package reactions;

import dao.MessageIndex;
import dao.UserDAO;
import dao.model.Message;
import util.TopKIterator;

import java.util.Iterator;
import java.util.UUID;

/**
 * PREDICTED hackathon facade: reactions on messages.
 * Same shape as ModerationTools: validate, then delegate to O(1) structures.
 */
public class ReactionTools {
	private ReactionTools() {}

	/** @return false if message/user missing, type null, or user already reacted */
	public static boolean addReaction(UUID message, UUID user, ReactionType type, long timestamp) {
		if (type == null || !exists(message, user)) return false;
		return ReactionStore.getInstance().forMessage(message).add(new Reaction(message, user, type, timestamp));
	}

	/** Switch an existing reaction to another type. @return false if none existed or nothing changed */
	public static boolean changeReaction(UUID message, UUID user, ReactionType type, long timestamp) {
		if (type == null || !exists(message, user)) return false;
		MessageReactions reactions = ReactionStore.getInstance().find(message);
		if (reactions == null || reactions.getReaction(user) == null || reactions.getReaction(user) == type) return false;
		return reactions.change(new Reaction(message, user, type, timestamp));
	}

	/** @return false if message/user missing or the user had no reaction */
	public static boolean removeReaction(UUID message, UUID user) {
		if (!exists(message, user)) return false;
		MessageReactions reactions = ReactionStore.getInstance().find(message);
		if (reactions == null || !reactions.remove(user)) return false;
		ReactionStore.getInstance().dropIfEmpty(message);
		return true;
	}

	/** @return the user's reaction, or null if none */
	public static ReactionType getReaction(UUID message, UUID user) {
		MessageReactions reactions = ReactionStore.getInstance().find(message);
		return reactions == null ? null : reactions.getReaction(user);
	}

	public static boolean hasReacted(UUID message, UUID user) {
		return getReaction(message, user) != null;
	}

	/** @return how many reactions of this type the message has (0 if none / unknown) */
	public static int countReactions(UUID message, ReactionType type) {
		MessageReactions reactions = ReactionStore.getInstance().find(message);
		return reactions == null ? 0 : reactions.count(type);
	}

	/**
	 * Top messages by a strategy, using the Factory + the reusable TopKIterator.
	 * Messages with zero relevant reactions are never returned.
	 */
	public static Iterator<Message> getTopMessages(String strategy, int amount) {
		ReactionType type = ReactionOrderingFactory.typeFilter(strategy); // validates too
		return new TopKIterator<>(
				ReactionStore.getInstance().all(),
				ReactionOrderingFactory.create(strategy),
				group -> type == null ? !group.isEmpty() : group.count(type) > 0,
				group -> MessageIndex.getInstance().get(group.getMessageId()),
				amount);
	}

	private static boolean exists(UUID message, UUID user) {
		return MessageIndex.getInstance().get(message) != null && UserDAO.getInstance().getByUUID(user) != null;
	}
}
```

---------------------------------------------------------------------

## 6. Follow, block, bookmark, likes on posts: one `RelationStore`

All are a many-to-many relation stored in **both directions**, so "who do I follow" and "who follows me" are both O(1).

**`app/src/relations/RelationStore.java`** (full file)

```java
package relations;

import java.util.Collections;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Map;
import java.util.Set;

/**
 * HACKATHON (generic Task 1 building block): a many-to-many relation "A -> B",
 * stored in BOTH directions so every question is O(1):
 *   follows (user -> user), blocks (user -> user), bookmarks (user -> post),
 *   subscriptions (user -> post), mutes, friend requests, tags (post -> tag) ...
 * @param <A> source type (usually a UUID)
 * @param <B> target type
 */
public class RelationStore<A, B> {
	private final Map<A, Set<B>> forward = new HashMap<>();
	private final Map<B, Set<A>> backward = new HashMap<>();

	/** @return false if the pair already existed */
	public boolean add(A source, B target) {
		if (!forward.computeIfAbsent(source, k -> new HashSet<>()).add(target)) return false;
		backward.computeIfAbsent(target, k -> new HashSet<>()).add(source);
		return true;
	}

	/** @return false if the pair did not exist */
	public boolean remove(A source, B target) {
		Set<B> targets = forward.get(source);
		if (targets == null || !targets.remove(target)) return false;
		if (targets.isEmpty()) forward.remove(source);
		Set<A> sources = backward.get(target);
		sources.remove(source);
		if (sources.isEmpty()) backward.remove(target);
		return true;
	}

	public boolean contains(A source, B target) {
		Set<B> targets = forward.get(source);
		return targets != null && targets.contains(target);
	}

	/** e.g. "who does this user follow" */
	public Set<B> targetsOf(A source) {
		return Collections.unmodifiableSet(forward.getOrDefault(source, Collections.emptySet()));
	}

	/** e.g. "who follows this user" */
	public Set<A> sourcesOf(B target) {
		return Collections.unmodifiableSet(backward.getOrDefault(target, Collections.emptySet()));
	}

	public int countTargets(A source) {
		return forward.getOrDefault(source, Collections.emptySet()).size();
	}

	public int countSources(B target) {
		return backward.getOrDefault(target, Collections.emptySet()).size();
	}

	/** Removes every pair whose SOURCE is this value. @return how many were removed */
	public int removeSource(A source) {
		Set<B> targets = forward.get(source);
		if (targets == null) return 0;
		int removed = 0;
		for (B target : Set.copyOf(targets)) if (remove(source, target)) removed++;
		return removed;
	}

	/** Removes every pair whose TARGET is this value. @return how many were removed */
	public int removeTarget(B target) {
		Set<A> sources = backward.get(target);
		if (sources == null) return 0;
		int removed = 0;
		for (A source : Set.copyOf(sources)) if (remove(source, target)) removed++;
		return removed;
	}

	/** @return every source that has at least one target (read-only), for persistence */
	public Set<A> sources() {
		return Collections.unmodifiableSet(forward.keySet());
	}

	public void clear() {
		forward.clear();
		backward.clear();
	}
}
```

**`app/src/relations/SocialTools.java`** (full file)

```java
package relations;

import dao.MessageComparator;
import dao.PostDAO;
import dao.UserDAO;
import dao.model.Message;
import dao.model.Post;
import sorteddata.SortedData;
import sorteddata.SortedDataViews;

import java.util.Set;
import java.util.UUID;

/**
 * PREDICTED hackathon facades built on RelationStore: follow, block, bookmark.
 * Every check is O(1); the "visible to viewer" view reuses SortedDataViews.filter.
 */
public class SocialTools {
	private static final RelationStore<UUID, UUID> follows = new RelationStore<>();   // user -> user
	private static final RelationStore<UUID, UUID> blocks = new RelationStore<>();    // user -> user
	private static final RelationStore<UUID, UUID> bookmarks = new RelationStore<>(); // user -> post
	private static final RelationStore<UUID, UUID> postLikes = new RelationStore<>(); // user -> post

	private SocialTools() {}

	// ---------------- follow ----------------
	/** @return false if either user is missing, self-follow, or already following */
	public static boolean follow(UUID follower, UUID followee) {
		if (!usersExist(follower, followee) || follower.equals(followee)) return false;
		return follows.add(follower, followee);
	}
	public static boolean unfollow(UUID follower, UUID followee) { return follows.remove(follower, followee); }
	public static boolean isFollowing(UUID follower, UUID followee) { return follows.contains(follower, followee); }
	public static Set<UUID> getFollowers(UUID user) { return follows.sourcesOf(user); }
	public static int followerCount(UUID user) { return follows.countSources(user); }

	// ---------------- block ----------------
	public static boolean block(UUID blocker, UUID blocked) {
		if (!usersExist(blocker, blocked) || blocker.equals(blocked)) return false;
		return blocks.add(blocker, blocked);
	}
	public static boolean unblock(UUID blocker, UUID blocked) { return blocks.remove(blocker, blocked); }
	public static boolean isBlocked(UUID blocker, UUID blocked) { return blocks.contains(blocker, blocked); }

	/** Messages on a post that `viewer` should see (authors they blocked are filtered out). */
	public static SortedData<Message> getMessagesFor(Post post, UUID viewer) {
		return SortedDataViews.filter(post.messages, MessageComparator.getInstance(),
				m -> m.poster() == null || !blocks.contains(viewer, m.poster()));
	}

	// ---------------- bookmark ----------------
	public static boolean bookmark(UUID user, UUID post) {
		if (UserDAO.getInstance().getByUUID(user) == null || PostDAO.getInstance().get(new Post(post)) == null) return false;
		return bookmarks.add(user, post);
	}
	public static boolean unbookmark(UUID user, UUID post) { return bookmarks.remove(user, post); }
	public static boolean isBookmarked(UUID user, UUID post) { return bookmarks.contains(user, post); }
	public static Set<UUID> getBookmarks(UUID user) { return bookmarks.targetsOf(user); }

	// ---------------- likes on posts ----------------
	/** @return false if user/post missing or already liked */
	public static boolean likePost(UUID user, UUID post) {
		if (UserDAO.getInstance().getByUUID(user) == null || PostDAO.getInstance().get(new Post(post)) == null) return false;
		return postLikes.add(user, post);
	}
	public static boolean unlikePost(UUID user, UUID post) { return postLikes.remove(user, post); }
	public static boolean hasLikedPost(UUID user, UUID post) { return postLikes.contains(user, post); }
	public static int getPostLikeCount(UUID post) { return postLikes.countSources(post); }

	// ---------------- account deletion + persistence access ----------------
	/** Removes the user from every relation, on both sides. */
	public static void forgetUser(UUID user) {
		follows.removeSource(user); follows.removeTarget(user);
		blocks.removeSource(user); blocks.removeTarget(user);
		bookmarks.removeSource(user);
		postLikes.removeSource(user);
	}

	/** The raw stores, so the persistence layer can save/load them. */
	public static RelationStore<UUID, UUID> follows() { return follows; }
	public static RelationStore<UUID, UUID> blocks() { return blocks; }
	public static RelationStore<UUID, UUID> bookmarks() { return bookmarks; }
	public static RelationStore<UUID, UUID> postLikes() { return postLikes; }

	public static void clearAll() { follows.clear(); blocks.clear(); bookmarks.clear(); postLikes.clear(); }

	private static boolean usersExist(UUID a, UUID b) {
		return UserDAO.getInstance().getByUUID(a) != null && UserDAO.getInstance().getByUUID(b) != null;
	}
}
```

---------------------------------------------------------------------

## 7. Up/down votes

Same shape as reactions: per-message `HashMap<user, +1/-1>` plus up/down counters, so `getScore` is O(1).
Voting the other way **switches** (undoes the old vote first); voting the same way twice returns false.

**`app/src/votes/MessageVotes.java`** (full file)

```java
package votes;

import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.UUID;

/**
 * Up/down votes on ONE message. HashMap user -> +1/-1 for O(1) checks,
 * plus up/down counters kept in sync so score() is O(1).
 */
public class MessageVotes {
	private final UUID messageId;
	private final Map<UUID, Integer> votesByUser = new HashMap<>();
	private int up = 0;
	private int down = 0;
	private long latestTimestamp = Long.MIN_VALUE;

	public MessageVotes(UUID messageId) {
		this.messageId = messageId;
	}

	/** Records or switches a vote. @return false if the user already has this exact vote */
	public boolean set(UUID user, int direction, long timestamp) {
		Integer previous = votesByUser.put(user, direction);
		if (previous != null && previous == direction) return false;
		if (previous != null) count(previous, -1); // switching: undo the old vote
		count(direction, +1);
		latestTimestamp = Math.max(latestTimestamp, timestamp);
		return true;
	}

	/** @return false if the user had not voted */
	public boolean remove(UUID user) {
		Integer previous = votesByUser.remove(user);
		if (previous == null) return false;
		count(previous, -1);
		return true;
	}

	private void count(int direction, int delta) {
		if (direction > 0) up += delta;
		else down += delta;
	}

	/** @return +1, -1, or 0 if the user has not voted */
	public int getVote(UUID user) {
		return votesByUser.getOrDefault(user, 0);
	}

	public int score() { return up - down; }
	public int up() { return up; }
	public int down() { return down; }
	public int total() { return up + down; }
	public boolean isEmpty() { return votesByUser.isEmpty(); }
	public long latestTimestamp() { return latestTimestamp; }
	public UUID getMessageId() { return messageId; }

	/** read-only user -> direction, for persistence */
	public Map<UUID, Integer> getVotes() {
		return Collections.unmodifiableMap(votesByUser);
	}
}
```

**`app/src/votes/VoteTools.java`** (full file)

```java
package votes;

import dao.MessageIndex;
import dao.UserDAO;
import dao.model.Message;
import util.TopKIterator;

import java.util.Collection;
import java.util.Collections;
import java.util.Comparator;
import java.util.HashMap;
import java.util.Iterator;
import java.util.Map;
import java.util.UUID;

/**
 * PREDICTED Task 1 theme: up/down votes on messages (Reddit-style score).
 *  vote(msg, user, +1 or -1): new vote or switch direction; same vote twice -> false.
 */
public final class VoteTools {
	public static final int UP = 1;
	public static final int DOWN = -1;

	private static final Map<UUID, MessageVotes> votesByMessage = new HashMap<>();

	private VoteTools() {}

	/** @return false if ids missing, direction not +1/-1, or the user already voted this way */
	public static boolean vote(UUID message, UUID user, int direction, long timestamp) {
		if ((direction != UP && direction != DOWN) || !exists(message, user)) return false;
		return votesByMessage.computeIfAbsent(message, MessageVotes::new).set(user, direction, timestamp);
	}

	/** @return false if the user had no vote on that message */
	public static boolean removeVote(UUID message, UUID user) {
		MessageVotes votes = votesByMessage.get(message);
		if (votes == null || !votes.remove(user)) return false;
		if (votes.isEmpty()) votesByMessage.remove(message);
		return true;
	}

	/** @return +1, -1 or 0 */
	public static int getVote(UUID message, UUID user) {
		MessageVotes votes = votesByMessage.get(message);
		return votes == null ? 0 : votes.getVote(user);
	}

	public static int getScore(UUID message) {
		MessageVotes votes = votesByMessage.get(message);
		return votes == null ? 0 : votes.score();
	}

	public static int getUpvotes(UUID message) {
		MessageVotes votes = votesByMessage.get(message);
		return votes == null ? 0 : votes.up();
	}

	public static int getDownvotes(UUID message) {
		MessageVotes votes = votesByMessage.get(message);
		return votes == null ? 0 : votes.down();
	}

	/**
	 * Factory for vote rankings (Strategy = Comparator):
	 *  "TOP"           highest score first
	 *  "CONTROVERSIAL" most votes on BOTH sides first (min(up, down)), then most votes
	 *  "NEWEST"        most recently voted first
	 */
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

	/** Top messages (each once, only messages with at least one vote). */
	public static Iterator<Message> getTopMessages(String strategy, int amount) {
		return new TopKIterator<>(votesByMessage.values(), ordering(strategy), v -> !v.isEmpty(),
				v -> MessageIndex.getInstance().get(v.getMessageId()), amount);
	}

	/** Account deletion. */
	public static void removeAllByUser(UUID user) {
		votesByMessage.values().removeIf(votes -> votes.remove(user) && votes.isEmpty());
	}

	/** read-only, for persistence */
	public static Collection<MessageVotes> all() {
		return Collections.unmodifiableCollection(votesByMessage.values());
	}

	/** restore one saved vote without existence checks (persistence) */
	public static void restore(UUID message, UUID user, int direction, long timestamp) {
		votesByMessage.computeIfAbsent(message, MessageVotes::new).set(user, direction, timestamp);
	}

	public static void clear() {
		votesByMessage.clear();
	}

	private static boolean exists(UUID message, UUID user) {
		return MessageIndex.getInstance().get(message) != null && UserDAO.getInstance().getByUUID(user) != null;
	}
}
```

---------------------------------------------------------------------

## 8. Subscriptions + notifications (Observer)

**`app/src/notifications/Notification.java`** (full file)

```java
package notifications;

import java.util.UUID;

/**
 * "Something happened that you subscribed to."
 * @param user      who receives it
 * @param message   the new message
 * @param post      where it happened
 * @param timestamp when the message was posted
 */
public record Notification(UUID user, UUID message, UUID post, long timestamp) {}
```

**`app/src/notifications/NotificationTools.java`** (full file)

```java
package notifications;

import dao.MessageIndex;
import dao.PostDAO;
import dao.UserDAO;
import dao.model.Message;
import dao.model.Post;
import relations.RelationStore;
import sorteddata.SortedDataSubject;

import java.util.ArrayDeque;
import java.util.ArrayList;
import java.util.Deque;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;

/**
 * PREDICTED Task 1 theme: subscribe to a post, get notified about new replies.
 * OBSERVER pattern twice:
 *  - every Post notifies MessageIndex about new messages
 *  - MessageIndex forwards them to us (addMessageListener), we fan out to subscribers
 * Notifications are kept newest-first in a Deque (addFirst is O(1)).
 */
public final class NotificationTools {
	private static final RelationStore<UUID, UUID> subscriptions = new RelationStore<>(); // user -> post
	private static final Map<UUID, Deque<Notification>> inbox = new HashMap<>();
	private static final Map<UUID, Integer> unread = new HashMap<>();

	// the listener is registered once, the first time this class is used
	private static final SortedDataSubject<Message> LISTENER = NotificationTools::onMessage;
	static {
		MessageIndex.getInstance().addMessageListener(LISTENER);
	}

	private NotificationTools() {}

	/** @return false if user/post missing or already subscribed */
	public static boolean subscribe(UUID user, UUID post) {
		if (UserDAO.getInstance().getByUUID(user) == null || PostDAO.getInstance().get(new Post(post)) == null) return false;
		return subscriptions.add(user, post);
	}

	public static boolean unsubscribe(UUID user, UUID post) {
		return subscriptions.remove(user, post);
	}

	public static boolean isSubscribed(UUID user, UUID post) {
		return subscriptions.contains(user, post);
	}

	public static Set<UUID> getSubscribers(UUID post) {
		return subscriptions.sourcesOf(post);
	}

	// called for every new message anywhere: notify that post's subscribers (not the author)
	private static void onMessage(Message message) {
		if (message.thread() == null) return;
		for (UUID user : subscriptions.sourcesOf(message.thread())) {
			if (user.equals(message.poster())) continue; // do not notify people about their own message
			inbox.computeIfAbsent(user, k -> new ArrayDeque<>())
					.addFirst(new Notification(user, message.id(), message.thread(), message.timestamp()));
			unread.merge(user, 1, Integer::sum);
		}
	}

	/** @return newest first (a copy) */
	public static List<Notification> getNotifications(UUID user) {
		return new ArrayList<>(inbox.getOrDefault(user, new ArrayDeque<>()));
	}

	public static int getUnreadCount(UUID user) {
		return unread.getOrDefault(user, 0);
	}

	public static void markAllRead(UUID user) {
		unread.remove(user);
	}

	/** Account deletion. */
	public static void forgetUser(UUID user) {
		subscriptions.removeSource(user);
		inbox.remove(user);
		unread.remove(user);
	}

	/** raw store, for persistence (subscriptions are saved; notifications are temporary) */
	public static RelationStore<UUID, UUID> subscriptions() {
		return subscriptions;
	}

	public static void clear() {
		subscriptions.clear();
		inbox.clear();
		unread.clear();
	}
}
```

---------------------------------------------------------------------

## 9. Tags

Tags are normalised (`"#Exam"` and `"exam"` are the same) and stored in a `RelationStore<post, tag>`.

**`app/src/tags/TagTools.java`** (full file)

```java
package tags;

import dao.PostDAO;
import dao.UserDAO;
import dao.model.Post;
import dao.model.User;
import relations.RelationStore;

import java.util.ArrayList;
import java.util.Comparator;
import java.util.List;
import java.util.Set;
import java.util.TreeSet;
import java.util.UUID;

/**
 * PREDICTED Task 1 theme: tags / hashtags on posts.
 * Stored as a RelationStore post -> tag, so both "tags of a post" and "posts with a tag" are O(1).
 * Tags are normalised (trimmed, lower-case, leading # removed) so "#Exam" and "exam" are the same.
 */
public final class TagTools {
	private static final int MAX_TAG_LENGTH = 30;
	private static final RelationStore<UUID, String> tags = new RelationStore<>();

	private TagTools() {}

	/** @return the normalised tag, or null if it is not valid (1-30 letters/digits) */
	public static String normalise(String tag) {
		if (tag == null) return null;
		String t = tag.trim().toLowerCase();
		if (t.startsWith("#")) t = t.substring(1);
		if (t.isEmpty() || t.length() > MAX_TAG_LENGTH) return null;
		for (char c : t.toCharArray()) if (!Character.isLetterOrDigit(c)) return null;
		return t;
	}

	/** Post author or Admin only. @return false if invalid, not allowed, or already tagged */
	public static boolean addTag(UUID post, UUID user, String tag) {
		String t = normalise(tag);
		if (t == null || !canEdit(post, user)) return false;
		return tags.add(post, t);
	}

	public static boolean removeTag(UUID post, UUID user, String tag) {
		String t = normalise(tag);
		if (t == null || !canEdit(post, user)) return false;
		return tags.remove(post, t);
	}

	/** @return tags of a post, alphabetically */
	public static Set<String> getTags(UUID post) {
		return new TreeSet<>(tags.targetsOf(post));
	}

	/** @return posts with this tag (any order) */
	public static Set<UUID> getPostsWithTag(String tag) {
		String t = normalise(tag);
		return t == null ? Set.of() : tags.sourcesOf(t);
	}

	/** @return the most used tags, most posts first (ties alphabetical) */
	public static List<String> getPopularTags(int amount) {
		if (amount <= 0) throw new IllegalArgumentException("amount must be positive");
		List<String> all = new ArrayList<>();
		for (UUID post : tags.sources()) all.addAll(tags.targetsOf(post));
		List<String> unique = new ArrayList<>(new TreeSet<>(all));
		unique.sort(Comparator.comparingInt((String t) -> tags.countSources(t)).reversed().thenComparing(Comparator.naturalOrder()));
		return unique.subList(0, Math.min(amount, unique.size()));
	}

	/** raw store, for persistence */
	public static RelationStore<UUID, String> store() {
		return tags;
	}

	public static void clear() {
		tags.clear();
	}

	private static boolean canEdit(UUID post, UUID user) {
		Post p = post == null ? null : PostDAO.getInstance().get(new Post(post));
		User u = UserDAO.getInstance().getByUUID(user);
		return p != null && u != null && (u.role() == User.Role.Admin || u.id().equals(p.poster));
	}
}
```

---------------------------------------------------------------------

## 10. Polls

A poll has a variable number of options; `counts[]` is kept in sync so results never recount voters.

**`app/src/polls/Poll.java`** (full file)

```java
package polls;

import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;

/**
 * A poll attached to a post. Each user has at most ONE vote (an option index).
 * counts[i] is kept in sync so results are O(options), not O(voters).
 */
public class Poll {
	private final UUID id;
	private final UUID post;
	private final UUID creator;
	private final List<String> options;
	private final int[] counts;
	private final Map<UUID, Integer> voteByUser = new HashMap<>();

	public Poll(UUID id, UUID post, UUID creator, List<String> options) {
		this.id = id;
		this.post = post;
		this.creator = creator;
		this.options = List.copyOf(options);
		this.counts = new int[options.size()];
	}

	/** @return false if the option index is invalid or the user already voted */
	boolean vote(UUID user, int option) {
		if (!validOption(option) || voteByUser.containsKey(user)) return false;
		voteByUser.put(user, option);
		counts[option]++;
		return true;
	}

	/** @return false if the user had not voted, the option is invalid, or it is the same option */
	boolean changeVote(UUID user, int option) {
		Integer previous = voteByUser.get(user);
		if (previous == null || !validOption(option) || previous == option) return false;
		counts[previous]--;
		counts[option]++;
		voteByUser.put(user, option);
		return true;
	}

	boolean removeVote(UUID user) {
		Integer previous = voteByUser.remove(user);
		if (previous == null) return false;
		counts[previous]--;
		return true;
	}

	/** @return option index, or -1 if the user has not voted */
	public int getVote(UUID user) {
		return voteByUser.getOrDefault(user, -1);
	}

	public int[] getResults() {
		return counts.clone(); // a copy: callers cannot change our counts
	}

	private boolean validOption(int option) {
		return option >= 0 && option < options.size();
	}

	public UUID getId() { return id; }
	public UUID getPost() { return post; }
	public UUID getCreator() { return creator; }
	public List<String> getOptions() { return options; }
	public Map<UUID, Integer> getVotes() { return Collections.unmodifiableMap(voteByUser); }
}
```

**`app/src/polls/PollTools.java`** (full file)

```java
package polls;

import dao.PostDAO;
import dao.UserDAO;
import dao.model.Post;

import java.util.ArrayList;
import java.util.Collection;
import java.util.Collections;
import java.util.HashMap;
import java.util.HashSet;
import java.util.List;
import java.util.Map;
import java.util.UUID;

/**
 * PREDICTED Task 1 theme: polls (vote once, may change or retract the vote).
 */
public final class PollTools {
	private static final Map<UUID, Poll> polls = new HashMap<>();

	private PollTools() {}

	/**
	 * @param options at least 2 non-blank, distinct (case-insensitive) options
	 * @return the new poll's id, or null if the post/user is missing or the options are invalid
	 */
	public static UUID createPoll(UUID post, UUID creator, List<String> options) {
		if (post == null || PostDAO.getInstance().get(new Post(post)) == null) return null;
		if (UserDAO.getInstance().getByUUID(creator) == null || !validOptions(options)) return null;
		Poll poll = new Poll(UUID.randomUUID(), post, creator, options);
		polls.put(poll.getId(), poll);
		return poll.getId();
	}

	private static boolean validOptions(List<String> options) {
		if (options == null || options.size() < 2) return false;
		HashSet<String> seen = new HashSet<>();
		for (String option : options) {
			if (option == null || option.isBlank() || !seen.add(option.trim().toLowerCase())) return false;
		}
		return true;
	}

	/** @return false if poll/user missing, option invalid, or already voted (use changeVote) */
	public static boolean vote(UUID poll, UUID user, int option) {
		Poll p = polls.get(poll);
		return p != null && UserDAO.getInstance().getByUUID(user) != null && p.vote(user, option);
	}

	public static boolean changeVote(UUID poll, UUID user, int option) {
		Poll p = polls.get(poll);
		return p != null && p.changeVote(user, option);
	}

	public static boolean removeVote(UUID poll, UUID user) {
		Poll p = polls.get(poll);
		return p != null && p.removeVote(user);
	}

	/** @return option index, -1 if no vote or unknown poll */
	public static int getVote(UUID poll, UUID user) {
		Poll p = polls.get(poll);
		return p == null ? -1 : p.getVote(user);
	}

	/** @return vote count per option (copy), or null for an unknown poll */
	public static int[] getResults(UUID poll) {
		Poll p = polls.get(poll);
		return p == null ? null : p.getResults();
	}

	/** @return indices of the leading option(s) (several on a tie; empty if no votes) */
	public static List<Integer> getWinners(UUID poll) {
		int[] results = getResults(poll);
		List<Integer> winners = new ArrayList<>();
		if (results == null) return winners;
		int best = 0;
		for (int count : results) best = Math.max(best, count);
		if (best == 0) return winners;
		for (int i = 0; i < results.length; i++) if (results[i] == best) winners.add(i);
		return winners;
	}

	public static Poll getPoll(UUID poll) {
		return polls.get(poll);
	}

	/** Account deletion: their votes disappear from every poll. */
	public static void removeVotesBy(UUID user) {
		for (Poll poll : polls.values()) poll.removeVote(user);
	}

	/** persistence */
	public static Collection<Poll> all() {
		return Collections.unmodifiableCollection(polls.values());
	}

	/** persistence: put back a saved poll (votes are restored with restoreVote) */
	public static void restore(Poll poll) {
		polls.put(poll.getId(), poll);
	}

	public static void restoreVote(UUID poll, UUID user, int option) {
		Poll p = polls.get(poll);
		if (p != null) p.vote(user, option);
	}

	public static void clear() {
		polls.clear();
	}
}
```

---------------------------------------------------------------------

## 11. Read / unread tracking

One timestamp per (user, post). Unread count uses the tree's `rank` (Task 2), so it is O(log n), never a scan.

**`app/src/readtracking/ReadTracker.java`** (full file)

```java
package readtracking;

import dao.PostDAO;
import dao.UserDAO;
import dao.model.Post;
import dao.model.User;
import queries.MessageQueries;

import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.UUID;

/**
 * PREDICTED Task 1 theme: read / unread tracking.
 * We store ONE number per (user, post): the timestamp up to which they have read.
 * Unread = messages with a later timestamp, counted in O(log n) with the tree's rank
 * (MessageQueries.countMessagesBetween), never by scanning.
 */
public final class ReadTracker {
	private static final Map<UUID, Map<UUID, Long>> lastRead = new HashMap<>(); // user -> (post -> timestamp)

	private ReadTracker() {}

	/** Marks everything up to `timestamp` as read. Never moves backwards. */
	public static boolean markRead(UUID user, UUID post, long timestamp) {
		if (UserDAO.getInstance().getByUUID(user) == null || PostDAO.getInstance().get(new Post(post)) == null) return false;
		lastRead.computeIfAbsent(user, k -> new HashMap<>()).merge(post, timestamp, Math::max);
		return true;
	}

	/** @return the read-up-to timestamp, or Long.MIN_VALUE if never read */
	public static long getLastRead(UUID user, UUID post) {
		return lastRead.getOrDefault(user, Map.of()).getOrDefault(post, Long.MIN_VALUE);
	}

	/** @return number of messages the user can see that are newer than what they have read */
	public static int getUnreadCount(UUID user, UUID post) {
		User u = UserDAO.getInstance().getByUUID(user);
		if (u == null) return 0;
		long from = getLastRead(user, post);
		if (from == Long.MAX_VALUE) return 0;
		long start = from == Long.MIN_VALUE ? Long.MIN_VALUE : from + 1;
		return MessageQueries.countMessagesBetween(post, start, Long.MAX_VALUE, u.role() == User.Role.Admin);
	}

	public static void forgetUser(UUID user) {
		lastRead.remove(user);
	}

	/** persistence: user -> post -> timestamp (read-only) */
	public static Map<UUID, Map<UUID, Long>> all() {
		return Collections.unmodifiableMap(lastRead);
	}

	/** persistence: restore without existence checks */
	public static void restore(UUID user, UUID post, long timestamp) {
		lastRead.computeIfAbsent(user, k -> new HashMap<>()).merge(post, timestamp, Math::max);
	}

	public static void clear() {
		lastRead.clear();
	}
}
```

---------------------------------------------------------------------

## 12. Bans with expiry

**`app/src/bans/BanTools.java`** (full file)

```java
package bans;

import dao.UserDAO;
import dao.model.User;

import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.UUID;

/**
 * PREDICTED Task 1 theme: temporary bans (suspensions).
 * One number per banned user: the time the ban ends. isBanned is O(1).
 */
public final class BanTools {
	private static final Map<UUID, Long> bannedUntil = new HashMap<>();

	private BanTools() {}

	/**
	 * Admin only; admins cannot ban themselves or other admins.
	 * Banning an already-banned user replaces the end time.
	 */
	public static boolean ban(UUID admin, UUID user, long untilTimestamp) {
		User a = UserDAO.getInstance().getByUUID(admin);
		User target = UserDAO.getInstance().getByUUID(user);
		if (a == null || target == null || a.role() != User.Role.Admin || target.role() == User.Role.Admin) return false;
		bannedUntil.put(user, untilTimestamp);
		return true;
	}

	/** @return false if not an admin or the user was not banned */
	public static boolean unban(UUID admin, UUID user) {
		User a = UserDAO.getInstance().getByUUID(admin);
		if (a == null || a.role() != User.Role.Admin) return false;
		return bannedUntil.remove(user) != null;
	}

	/** @return true while now < end of ban */
	public static boolean isBanned(UUID user, long now) {
		Long until = bannedUntil.get(user);
		return until != null && now < until;
	}

	public static void forgetUser(UUID user) {
		bannedUntil.remove(user);
	}

	/** persistence */
	public static Map<UUID, Long> all() {
		return Collections.unmodifiableMap(bannedUntil);
	}

	public static void restore(UUID user, long until) {
		bannedUntil.put(user, until);
	}

	public static void clear() {
		bannedUntil.clear();
	}
}
```

`ContentTools.reply` (Task2prediction.md) refuses banned users: `if (bans.BanTools.isBanned(user, timestamp)) return null;`

---------------------------------------------------------------------

## 13. Friend requests

**`app/src/friends/FriendTools.java`** (full file)

```java
package friends;

import dao.UserDAO;
import relations.RelationStore;

import java.util.Set;
import java.util.UUID;

/**
 * PREDICTED Task 1 theme: friend requests (pending -> accepted).
 * Two RelationStores:
 *  - requests: from -> to (pending, one direction)
 *  - friends:  stored BOTH ways (a->b and b->a) so "friends of x" is one lookup
 */
public final class FriendTools {
	private static final RelationStore<UUID, UUID> requests = new RelationStore<>();
	private static final RelationStore<UUID, UUID> friends = new RelationStore<>();

	private FriendTools() {}

	/**
	 * @return false if a user is missing, self-request, already friends, or already requested.
	 * If `to` had already asked `from`, this accepts that request instead (both wanted it).
	 */
	public static boolean sendRequest(UUID from, UUID to) {
		if (!exist(from, to) || from.equals(to) || areFriends(from, to)) return false;
		if (requests.contains(to, from)) return accept(from, to);
		return requests.add(from, to);
	}

	/** `user` accepts the request sent by `from`. */
	public static boolean accept(UUID user, UUID from) {
		if (!requests.remove(from, user)) return false;
		friends.add(user, from);
		friends.add(from, user);
		return true;
	}

	public static boolean decline(UUID user, UUID from) {
		return requests.remove(from, user);
	}

	public static boolean removeFriend(UUID user, UUID other) {
		if (!friends.remove(user, other)) return false;
		friends.remove(other, user);
		return true;
	}

	public static boolean areFriends(UUID a, UUID b) {
		return friends.contains(a, b);
	}

	public static Set<UUID> getFriends(UUID user) {
		return friends.targetsOf(user);
	}

	/** requests waiting for this user to answer */
	public static Set<UUID> getIncomingRequests(UUID user) {
		return requests.sourcesOf(user);
	}

	public static Set<UUID> getOutgoingRequests(UUID user) {
		return requests.targetsOf(user);
	}

	public static void forgetUser(UUID user) {
		requests.removeSource(user);
		requests.removeTarget(user);
		friends.removeSource(user);
		friends.removeTarget(user);
	}

	/** persistence */
	public static RelationStore<UUID, UUID> requests() { return requests; }
	public static RelationStore<UUID, UUID> friends() { return friends; }

	public static void clear() {
		requests.clear();
		friends.clear();
	}

	private static boolean exist(UUID a, UUID b) {
		return UserDAO.getInstance().getByUUID(a) != null && UserDAO.getInstance().getByUUID(b) != null;
	}
}
```

---------------------------------------------------------------------

## 14. @mentions (derived data, rebuilt when messages load)

**`app/src/mentions/MentionTools.java`** (full file)

```java
package mentions;

import dao.MessageComparator;
import dao.MessageIndex;
import dao.UserDAO;
import dao.model.Message;
import dao.model.User;
import relations.RelationStore;
import sorteddata.SortedDataSubject;

import java.util.ArrayList;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.Set;
import java.util.UUID;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

/**
 * PREDICTED Task 1 theme: @mentions.
 * Every new message is scanned ONCE when it is added (Observer via MessageIndex), and
 * mentioned users are stored in a RelationStore user -> message. Asking "where was I
 * mentioned" is then a lookup, not a scan of every message.
 * This is DERIVED data: rebuilt automatically when messages are loaded, so it needs no file.
 */
public final class MentionTools {
	// same rules as usernames: 4-20 letters/digits
	private static final Pattern MENTION = Pattern.compile("@([A-Za-z0-9]{4,20})(?![A-Za-z0-9])");
	private static final RelationStore<UUID, UUID> mentions = new RelationStore<>(); // user -> message
	private static final SortedDataSubject<Message> LISTENER = MentionTools::onMessage;
	static {
		MessageIndex.getInstance().addChangeListener(LISTENER); // new AND edited messages
	}

	private MentionTools() {}

	/** Call once at start-up (e.g. in main / a test @Before) so no early message is missed. */
	public static void init() {
		// running the static block above is all that is needed
	}

	/** @return usernames mentioned in the text, in order, without duplicates (not checked against UserDAO) */
	public static List<String> extractUsernames(String text) {
		Set<String> names = new LinkedHashSet<>();
		if (text == null) return new ArrayList<>();
		Matcher m = MENTION.matcher(text);
		while (m.find()) names.add(m.group(1));
		return new ArrayList<>(names);
	}

	private static void onMessage(Message message) {
		for (String name : extractUsernames(message.message())) {
			User user = UserDAO.getInstance().get(new User(name)); // comparator ignores case
			if (user != null && user.id() != null) mentions.add(user.id(), message.id());
		}
	}

	/** @return messages that mention this user and still exist, newest first */
	public static List<Message> getMentions(UUID user) {
		List<Message> result = new ArrayList<>();
		for (UUID id : mentions.targetsOf(user)) {
			Message m = MessageIndex.getInstance().get(id);
			if (m != null) result.add(m);
		}
		result.sort(MessageComparator.getInstance().reversed());
		return result;
	}

	public static void forgetUser(UUID user) {
		mentions.removeSource(user);
	}

	public static void clear() {
		mentions.clear();
	}
}
```

---------------------------------------------------------------------

## 15. Edit history + undo (Memento)

`Message` is immutable, so an edit is a new record with the same id/timestamp/thread/poster swapped in by
`Post.replaceMessage` (Task2prediction.md section 4). The old text is saved as an `Edit`.

**`app/src/edits/Edit.java`** (full file)

```java
package edits;

/**
 * One previous version of a message.
 * @param text     the text BEFORE the edit
 * @param editedAt when it was replaced
 */
public record Edit(String text, long editedAt) {}
```

**`app/src/edits/EditTools.java`** (full file)

```java
package edits;

import dao.MessageIndex;
import dao.UserDAO;
import dao.model.Message;
import dao.model.Post;

import java.util.ArrayList;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;

/**
 * PREDICTED Task 1 theme: editing messages with history (MEMENTO pattern: each Edit is a
 * saved snapshot of the old state).
 * Message is an immutable record, so an edit builds a NEW record with the same id, poster,
 * thread and timestamp (so it keeps its place in the tree) and swaps it in via Post.replaceMessage.
 */
public final class EditTools {
	private static final Map<UUID, List<Edit>> history = new HashMap<>();

	private EditTools() {}

	/**
	 * Only the author may edit.
	 * @return false if ids missing, not the author, new text null, or the text is unchanged
	 */
	public static boolean editMessage(UUID message, UUID user, String newText, long timestamp) {
		Message current = MessageIndex.getInstance().get(message);
		if (current == null || newText == null || UserDAO.getInstance().getByUUID(user) == null) return false;
		if (!user.equals(current.poster()) || newText.equals(current.message())) return false;

		Message edited = new Message(current.id(), current.poster(), current.thread(), current.timestamp(), newText);
		Post post = MessageIndex.getInstance().getPostOf(current);
		post.replaceMessage(current, edited);
		history.computeIfAbsent(message, k -> new ArrayList<>()).add(new Edit(current.message(), timestamp));
		return true;
	}

	/** @return previous versions, oldest first (empty if never edited) */
	public static List<Edit> getHistory(UUID message) {
		return Collections.unmodifiableList(history.getOrDefault(message, List.of()));
	}

	public static int getEditCount(UUID message) {
		return history.getOrDefault(message, List.of()).size();
	}

	public static boolean isEdited(UUID message) {
		return getEditCount(message) > 0;
	}

	/** Undo the last edit (back to the previous text). Author only. */
	public static boolean undoLastEdit(UUID message, UUID user) {
		List<Edit> edits = history.get(message);
		Message current = MessageIndex.getInstance().get(message);
		if (edits == null || edits.isEmpty() || current == null || !user.equals(current.poster())) return false;
		Edit last = edits.remove(edits.size() - 1);
		Message restored = new Message(current.id(), current.poster(), current.thread(), current.timestamp(), last.text());
		MessageIndex.getInstance().getPostOf(current).replaceMessage(current, restored);
		if (edits.isEmpty()) history.remove(message);
		return true;
	}

	/** persistence: message -> previous versions */
	public static Map<UUID, List<Edit>> all() {
		return Collections.unmodifiableMap(history);
	}

	public static void restore(UUID message, Edit edit) {
		history.computeIfAbsent(message, k -> new ArrayList<>()).add(edit);
	}

	public static void clear() {
		history.clear();
	}
}
```

---------------------------------------------------------------------

## 16. Account deletion, account changes, roles, moderators

Deletion follows the week 2 briefing: the `User` goes, their messages stay (still holding the poster UUID, shown as "[deleted]"),
and every store forgets them. `User` is an immutable record, so a change = remove + add a new record with the **same UUID**.

**`app/src/accounts/AccountTools.java`** (full file)

```java
package accounts;

import bans.BanTools;
import dao.UserDAO;
import dao.model.User;
import friends.FriendTools;
import mentions.MentionTools;
import moderation.ReportStore;
import notifications.NotificationTools;
import polls.PollTools;
import reactions.ReactionStore;
import readtracking.ReadTracker;
import relations.SocialTools;
import votes.VoteTools;

import java.util.UUID;

/**
 * PREDICTED Task 1 theme: account deletion and account changes.
 * Deletion follows the week 2 briefing: the User is removed (privacy / GDPR), but their
 * Messages stay, still holding the poster UUID, shown as "[deleted]".
 * Every feature store gets a forget/remove call so nothing points at a ghost user.
 */
public final class AccountTools {
	public static final String DELETED_NAME = "[deleted]";

	private AccountTools() {}

	/** Needs the correct password. @return false if unknown user or wrong password */
	public static boolean deleteAccount(UUID user, String password) {
		User u = UserDAO.getInstance().getByUUID(user);
		if (u == null || password == null || !password.equals(u.password())) return false;
		UserDAO.getInstance().remove(u);
		ReportStore.getInstance().removeAllByUser(user);
		ReactionStore.getInstance().removeAllByUser(user);
		VoteTools.removeAllByUser(user);
		PollTools.removeVotesBy(user);
		SocialTools.forgetUser(user);
		FriendTools.forgetUser(user);
		NotificationTools.forgetUser(user);
		MentionTools.forgetUser(user);
		ReadTracker.forgetUser(user);
		BanTools.forgetUser(user);
		return true;
	}

	/** @return the username, or "[deleted]" for a removed / unknown user */
	public static String displayName(UUID user) {
		User u = UserDAO.getInstance().getByUUID(user);
		return u == null ? DELETED_NAME : u.username();
	}

	/**
	 * User is an immutable record: a change = remove the old record + add a new one
	 * with the SAME UUID (so messages, reports... still point at them).
	 */
	public static boolean changePassword(UUID user, String oldPassword, String newPassword) {
		User u = UserDAO.getInstance().getByUUID(user);
		if (u == null || !u.password().equals(oldPassword) || !UserDAO.isValidPassword(newPassword)) return false;
		return replace(u, new User(u.id(), u.role(), u.username(), newPassword));
	}

	/** @return false if the name is invalid or taken by someone else (case-insensitive) */
	public static boolean changeUsername(UUID user, String newUsername) {
		User u = UserDAO.getInstance().getByUUID(user);
		if (u == null || !UserDAO.isValidUsername(newUsername)) return false;
		User existing = UserDAO.getInstance().get(new User(newUsername));
		if (existing != null && !existing.id().equals(user)) return false;
		return replace(u, new User(u.id(), u.role(), newUsername, u.password()));
	}

	/** Admin promotes / demotes another user. */
	public static boolean setRole(UUID admin, UUID user, User.Role role) {
		User a = UserDAO.getInstance().getByUUID(admin);
		User u = UserDAO.getInstance().getByUUID(user);
		if (a == null || u == null || role == null || a.role() != User.Role.Admin || a.id().equals(user)) return false;
		return replace(u, new User(u.id(), role, u.username(), u.password()));
	}

	private static boolean replace(User oldUser, User newUser) {
		UserDAO.getInstance().remove(oldUser);
		return UserDAO.getInstance().add(newUser);
	}
}
```

**`app/src/accounts/Permissions.java`** (full file)

```java
package accounts;

import dao.UserDAO;
import dao.model.User;

import java.util.HashSet;
import java.util.Set;
import java.util.UUID;

/**
 * PREDICTED theme: a Moderator role between Member and Admin.
 * Adding a constant to User.Role would also need changes in UserSerializer (its switch),
 * GuestState.createStateFromUser and every "== Admin" check. Keeping moderators as a
 * separate set avoids touching the enum, and all permission checks live in ONE place.
 * If the spec says "add a Moderator role to User.Role", do that instead and update those spots.
 */
public final class Permissions {
	private static final Set<UUID> moderators = new HashSet<>();

	private Permissions() {}

	public static boolean isAdmin(UUID user) {
		User u = UserDAO.getInstance().getByUUID(user);
		return u != null && u.role() == User.Role.Admin;
	}

	/** Admins can always moderate; moderators too. */
	public static boolean canModerate(UUID user) {
		return isAdmin(user) || (moderators.contains(user) && UserDAO.getInstance().getByUUID(user) != null);
	}

	/** Admin only; cannot target admins. */
	public static boolean setModerator(UUID admin, UUID user, boolean moderator) {
		if (!isAdmin(admin) || UserDAO.getInstance().getByUUID(user) == null || isAdmin(user)) return false;
		return moderator ? moderators.add(user) : moderators.remove(user);
	}

	public static Set<UUID> moderators() {
		return java.util.Collections.unmodifiableSet(moderators);
	}

	/** persistence */
	public static void restore(UUID user) {
		moderators.add(user);
	}

	public static void clear() {
		moderators.clear();
	}
}
```

---------------------------------------------------------------------

## 17. Rate limiting (sliding window)

**`app/src/ratelimit/RateLimiter.java`** (full file)

```java
package ratelimit;

import java.util.ArrayDeque;
import java.util.Deque;
import java.util.HashMap;
import java.util.Map;
import java.util.UUID;

/**
 * PREDICTED Task 1 theme: spam protection, "at most N actions per W milliseconds per user".
 * SLIDING WINDOW: each user has a queue of their recent action times. Old times fall off
 * the front, new ones go on the back. Each call is O(1) amortised (every time is added
 * once and removed once).
 */
public class RateLimiter {
	private final int maxActions;
	private final long windowMillis;
	private final Map<UUID, Deque<Long>> recent = new HashMap<>();

	public RateLimiter(int maxActions, long windowMillis) {
		if (maxActions <= 0 || windowMillis <= 0) throw new IllegalArgumentException("limits must be positive");
		this.maxActions = maxActions;
		this.windowMillis = windowMillis;
	}

	/**
	 * Records an action if allowed.
	 * @param now current time; calls must not go backwards in time per user
	 * @return true if the user is under the limit (and the action is counted)
	 */
	public boolean tryAcquire(UUID user, long now) {
		Deque<Long> times = recent.computeIfAbsent(user, k -> new ArrayDeque<>());
		while (!times.isEmpty() && times.peekFirst() <= now - windowMillis) times.pollFirst(); // drop expired
		if (times.size() >= maxActions) return false;
		times.addLast(now);
		return true;
	}

	/** @return actions the user may still do right now */
	public int remaining(UUID user, long now) {
		Deque<Long> times = recent.get(user);
		if (times == null) return maxActions;
		int active = 0;
		for (long t : times) if (t > now - windowMillis) active++;
		return maxActions - active;
	}

	public void reset(UUID user) {
		recent.remove(user);
	}
}
```

---------------------------------------------------------------------

## 18. Shared helper used by every "top k" query

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

---------------------------------------------------------------------

## 19. Recipe for an unseen theme (5 questions)

1. **Key?** Usually the message/post UUID: `HashMap<UUID, Group>`.
2. **Inside one group?** Who did it: `HashMap<UUID user, Record>` (O(1) duplicate check).
3. **What will Task 4 ask?** Counts: keep an `int`/`EnumMap` updated. Oldest/newest: `TreeSet` by time.
4. **Who may do it?** Anyone existing: `exists(...)`. Admin: `role() == Admin`. Author: `message.poster().equals(user)`.
5. **What survives what?** Keep stores independent so one action never wipes another (reports survive hiding).

---------------------------------------------------------------------

## 20. Trap checklist (Task 1)

- [ ] Every "does X exist" check is O(1) (`MessageIndex`, indexed `UserDAO.getByUUID`); `PostDAO.get(new Post(id))` is O(log n).
- [ ] Duplicate add returns **false and changes nothing** (no second count increment).
- [ ] Remove of something absent, or with unknown UUIDs, returns false.
- [ ] `null` arguments return false / throw `IllegalArgumentException`, never `NullPointerException` (the tests caught one in `ReactionType.valueOf(null)`).
- [ ] Empty groups are deleted after the last remove (otherwise Task 4 returns zero-count items).
- [ ] Unused parameters in a given signature (like `removeReport`'s timestamp): keep them and javadoc why.
- [ ] Stores are Singletons/static with a `clear()` for tests and for loading.
- [ ] Never add state to the `Message`/`User` records; use a store keyed by UUID, or replace the record keeping its UUID.

---------------------------------------------------------------------

## 21. How this was verified

| Feature | Tests | Highlights |
|---|---|---|
| Base fixes | course tests from your miniproject + `ModerationVerify` | UserDAO singleton style test now passes; CSVReader 15/15 after the Task 3 fix |
| Reports | 19 + 29 (Task 5) | duplicates, null/unknown ids, re-report after remove, reports survive hiding, 20,000 x 4 operations under 3 s |
| Reactions | 5 + 15 (Task 5) | add/change/remove/counts/rankings; found and fixed a `null` strategy `NullPointerException` |
| Votes | 2 + 17 (Task 5) | switching votes, same vote twice, rankings |
| Follow/block/bookmark/likes | 4 + 6 (Task 5) | |
| Notifications | 1 | not notified about your own message; edits do not notify |
| Tags, polls, read tracking, bans, friends, mentions, edits, rate limit, accounts | 1 test method each (`FeaturesVerify`), many assertions | e.g. account deletion cleans 10 stores; poll results are a copy |
| Save + load of every feature | `AllPersistenceVerify` | all 27 feature files, CSV and JSON |
