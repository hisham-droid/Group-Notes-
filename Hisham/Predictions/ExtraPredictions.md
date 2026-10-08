# Extra predictions (themes from the overview that are not one specific task)

> All code here is compiled and tested (`ExtrasVerify` and `MoreExtrasVerify`, 19 test methods, plus the all-features save/load test).
> Any of these could be tomorrow's theme; the task shapes in Task1-5 still apply (tools class, validation, efficiency, patterns).

| Theme | Class | Pattern / structure | Section |
|---|---|---|---|
| Message search | `SearchTools` | inverted index word -> messages, Observer (also re-indexes edits) | 1 |
| Username autocomplete | `Trie` | trie (prefix tree) | 1 |
| Trending posts, busiest/active posts | `TrendingTools` | rank-based counting + `TopKIterator` | Task4prediction.md 7 |
| Leaderboards / karma | `LeaderboardTools` | Factory + `TopKIterator` | Task4prediction.md 8 |
| Statistics | `StatsTools` | one pass over all messages | 2 |
| Profanity censor (week 8 has a censor refactor) | `Censor`, `CensorFactory`, `CensorStrategy` | Strategy + Factory + Decorator | 3 |
| Audit log + undo of admin actions | `AdminActions`, `AdminCommand`, `AuditEntry` | Command | 4 |
| Direct / private messages | `DirectMessageTools`, `DirectMessage` | one SortedData per pair of users | 5 |
| Scheduled messages | `ScheduledMessages` | min-heap + lazy deletion | 6 |
| Boards / categories / subforums | `Board`, `BoardTools` | Composite | 7 |
| Links in messages, duplicate-message detection | `ContentChecks` | regex, HashMap of last post times | 8 |
| Moderator role | `Permissions` | one place for permission checks | Task1prediction.md 16 |
| Move a message to another post | `MoveTools` | remove + insert copy | Task2prediction.md 12 |
| Pagination / time ranges | `MessageQueries` | `getAtIndex`, `rank` | Task2prediction.md 9 |
| Replies to a message, nested threads, quoting | `ThreadTools` | tree of parent/child links, depth-first walk | 9 |
| Private / invite-only posts | `AccessTools` | access control list (`RelationStore` post -> user) | 10 |
| Login lockout + login history | `LoginTools` | failure counter, lock time, bounded `Deque` | 11 |
| Merge / split posts | `PostTools` | remove + insert copies, `getRange` from a message | 12 |
| Emoji shortcodes, markdown to plain text, previews | `TextFormat` | regex | 13 |

## 1. Search: inverted index + trie

**`app/src/search/SearchTools.java`** (full file)

```java
package search;

import dao.MessageComparator;
import dao.MessageIndex;
import dao.UserDAO;
import dao.model.Message;
import dao.model.User;
import sorteddata.SortedDataSubject;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;

/**
 * PREDICTED theme: search.
 *  - Messages: INVERTED INDEX word -> message ids, filled once per new message (Observer).
 *    search("design patterns") returns messages containing ALL words (AND), newest first.
 *    We intersect starting from the rarest word, so the cost depends on the result size,
 *    not on the number of messages.
 *  - Usernames: Trie for autocomplete (rebuilt from UserDAO on demand).
 */
public final class SearchTools {
	private static final Map<String, Set<UUID>> index = new HashMap<>();
	private static final SortedDataSubject<Message> LISTENER = SearchTools::indexMessage;
	static {
		MessageIndex.getInstance().addChangeListener(LISTENER); // new AND edited messages
	}

	private SearchTools() {}

	/** Call at start-up so early messages are indexed. */
	public static void init() {}

	/** lower-case words made of letters/digits */
	static List<String> words(String text) {
		List<String> result = new ArrayList<>();
		if (text == null) return result;
		for (String w : text.toLowerCase().split("[^\\p{L}\\p{N}]+")) if (!w.isEmpty()) result.add(w);
		return result;
	}

	private static void indexMessage(Message message) {
		for (String word : words(message.message())) index.computeIfAbsent(word, k -> new HashSet<>()).add(message.id());
	}

	/** @return messages containing every word of the query, newest first (deleted/edited-away hits dropped) */
	public static List<Message> search(String query) {
		List<String> terms = words(query);
		List<Message> result = new ArrayList<>();
		if (terms.isEmpty()) return result;
		terms.sort((x, y) -> Integer.compare(index.getOrDefault(x, Set.of()).size(), index.getOrDefault(y, Set.of()).size()));
		Set<UUID> candidates = new HashSet<>(index.getOrDefault(terms.get(0), Set.of()));
		for (int i = 1; i < terms.size() && !candidates.isEmpty(); i++) candidates.retainAll(index.getOrDefault(terms.get(i), Set.of()));
		for (UUID id : candidates) {
			Message m = MessageIndex.getInstance().get(id);
			// re-check the CURRENT text: an edit may have removed the word
			if (m != null && new HashSet<>(words(m.message())).containsAll(terms)) result.add(m);
		}
		result.sort(MessageComparator.getInstance().reversed());
		return result;
	}

	/** Username autocomplete, alphabetical, case-insensitive. */
	public static List<String> autocompleteUsername(String prefix, int limit) {
		Trie trie = new Trie();
		for (Iterator<User> it = UserDAO.getInstance().getAll(); it.hasNext(); ) trie.add(it.next().username());
		return trie.startingWith(prefix, limit);
	}

	public static void clear() {
		index.clear();
	}
}
```

**`app/src/search/Trie.java`** (full file)

```java
package search;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.TreeMap;

/**
 * PREDICTED: prefix search (e.g. username autocomplete). A trie stores words letter by letter;
 * all words with a prefix live under the prefix's node. Lookup is O(prefix length + results).
 * Lower-cases everything; children in a TreeMap so results come out alphabetically.
 */
public class Trie {
	private static final class Node {
		final Map<Character, Node> children = new TreeMap<>();
		String word; // non-null if a word ends here (stored with its original case)
	}

	private final Node root = new Node();

	public void add(String word) {
		Node node = root;
		for (char c : word.toLowerCase().toCharArray()) node = node.children.computeIfAbsent(c, k -> new Node());
		node.word = word;
	}

	/** @return false if the word was not stored */
	public boolean remove(String word) {
		Node node = find(word.toLowerCase());
		if (node == null || node.word == null) return false;
		node.word = null; // (empty branches are left in place: simpler, still correct)
		return true;
	}

	public boolean contains(String word) {
		Node node = find(word.toLowerCase());
		return node != null && node.word != null;
	}

	/** @return up to `limit` stored words starting with prefix (case-insensitive), alphabetical */
	public List<String> startingWith(String prefix, int limit) {
		List<String> result = new ArrayList<>();
		Node node = find(prefix.toLowerCase());
		if (node != null) collect(node, result, limit);
		return result;
	}

	private Node find(String key) {
		Node node = root;
		for (char c : key.toCharArray()) {
			node = node.children.get(c);
			if (node == null) return null;
		}
		return node;
	}

	private void collect(Node node, List<String> out, int limit) {
		if (out.size() >= limit) return;
		if (node.word != null) out.add(node.word);
		for (Node child : node.children.values()) collect(child, out, limit);
	}
}
```

## 2. Statistics

**`app/src/stats/StatsTools.java`** (full file)

```java
package stats;

import dao.PostDAO;
import dao.model.Message;
import dao.model.Post;

import java.time.Instant;
import java.time.LocalDate;
import java.time.ZoneOffset;
import java.util.HashMap;
import java.util.Iterator;
import java.util.Map;
import java.util.TreeMap;
import java.util.UUID;

/**
 * PREDICTED theme: statistics. Each method is one pass over all messages (O(n)).
 */
public final class StatsTools {
	private StatsTools() {}

	public static int totalMessages() {
		int total = 0;
		for (Iterator<Post> it = PostDAO.getInstance().getAll(); it.hasNext(); ) total += it.next().messages.size();
		return total;
	}

	/** user -> number of messages written (null posters ignored) */
	public static Map<UUID, Integer> messagesPerUser() {
		Map<UUID, Integer> counts = new HashMap<>();
		forEachMessage(m -> { if (m.poster() != null) counts.merge(m.poster(), 1, Integer::sum); });
		return counts;
	}

	/** UTC day -> messages that day, in date order */
	public static Map<LocalDate, Integer> messagesPerDay() {
		Map<LocalDate, Integer> counts = new TreeMap<>();
		forEachMessage(m -> counts.merge(LocalDate.ofInstant(Instant.ofEpochMilli(m.timestamp()), ZoneOffset.UTC), 1, Integer::sum));
		return counts;
	}

	/** @return average text length, 0 if there are no messages */
	public static double averageMessageLength() {
		long[] sumAndCount = new long[2];
		forEachMessage(m -> { sumAndCount[0] += m.message() == null ? 0 : m.message().length(); sumAndCount[1]++; });
		return sumAndCount[1] == 0 ? 0 : (double) sumAndCount[0] / sumAndCount[1];
	}

	/** @return the post with the most messages, or null if there are none */
	public static Post busiestPost() {
		Post best = null;
		for (Iterator<Post> it = PostDAO.getInstance().getAll(); it.hasNext(); ) {
			Post p = it.next();
			if (best == null || p.messages.size() > best.messages.size()) best = p;
		}
		return best == null || best.messages.size() == 0 ? null : best;
	}

	private static void forEachMessage(java.util.function.Consumer<Message> action) {
		for (Iterator<Post> it = PostDAO.getInstance().getAll(); it.hasNext(); ) it.next().messages.getAll().forEachRemaining(action);
	}
}
```

## 3. Censor: Strategy + Factory + Decorator

**`app/src/censor/CensorStrategy.java`** (full file)

```java
package censor;

/** STRATEGY: how a banned word is replaced. */
public interface CensorStrategy {
	String replace(String word);
}
```

**`app/src/censor/CensorFactory.java`** (full file)

```java
package censor;

/**
 * FACTORY for censor strategies:
 *  "STAR"    -> "****" (same length)
 *  "FIRST"   -> "d***" (keep the first letter)
 *  "REMOVE"  -> ""
 *  "REPLACE" -> "[censored]"
 */
public final class CensorFactory {
	private CensorFactory() {}

	public static CensorStrategy create(String name) {
		if (name == null) throw new IllegalArgumentException("strategy must not be null");
		return switch (name) {
			case "STAR" -> word -> "*".repeat(word.length());
			case "FIRST" -> word -> word.charAt(0) + "*".repeat(word.length() - 1);
			case "REMOVE" -> word -> "";
			case "REPLACE" -> word -> "[censored]";
			default -> throw new IllegalArgumentException("Unknown strategy: " + name);
		};
	}
}
```

**`app/src/censor/Censor.java`** (full file)

```java
package censor;

import java.util.HashSet;
import java.util.Set;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

/**
 * PREDICTED theme (week 8 has a censor refactor): replaces banned WHOLE words,
 * case-insensitively ("class" does not censor "classic"), using a strategy.
 * Censors can be chained (DECORATOR): new Censor(words2, strategy, new Censor(words1, ...)).
 */
public class Censor {
	private static final Pattern WORD = Pattern.compile("[\\p{L}\\p{N}']+");

	private final Set<String> banned = new HashSet<>();
	private final CensorStrategy strategy;
	private final Censor inner; // the censor applied first (may be null)

	public Censor(Set<String> bannedWords, CensorStrategy strategy) {
		this(bannedWords, strategy, null);
	}

	public Censor(Set<String> bannedWords, CensorStrategy strategy, Censor inner) {
		for (String w : bannedWords) banned.add(w.toLowerCase());
		this.strategy = strategy;
		this.inner = inner;
	}

	public String apply(String text) {
		if (text == null) return null;
		String input = inner == null ? text : inner.apply(text);
		StringBuilder out = new StringBuilder();
		Matcher m = WORD.matcher(input);
		int last = 0;
		while (m.find()) {
			out.append(input, last, m.start());
			String word = m.group();
			out.append(banned.contains(word.toLowerCase()) ? strategy.replace(word) : word);
			last = m.end();
		}
		return out.append(input.substring(last)).toString();
	}

	/** @return true if the text contains any banned word */
	public boolean containsBanned(String text) {
		if (text == null) return false;
		Matcher m = WORD.matcher(text);
		while (m.find()) if (banned.contains(m.group().toLowerCase())) return true;
		return inner != null && inner.containsBanned(text);
	}
}
```

## 4. Audit log + undo: Command

**`app/src/audit/AdminCommand.java`** (full file)

```java
package audit;

/** COMMAND pattern: an admin action that can be undone. */
public interface AdminCommand {
	/** @return false if the action could not be done (nothing changes) */
	boolean execute();

	/** reverses a successful execute() */
	void undo();

	/** human-readable, for the audit log */
	String describe();
}
```

**`app/src/audit/AuditEntry.java`** (full file)

```java
package audit;

import java.util.UUID;

/** One line of the audit log: who did what, when. */
public record AuditEntry(UUID admin, String action, long timestamp) {}
```

**`app/src/audit/AdminActions.java`** (full file)

```java
package audit;

import dao.MessageIndex;
import dao.PostDAO;
import dao.UserDAO;
import dao.model.Message;
import dao.model.Post;
import dao.model.User;

import java.util.ArrayDeque;
import java.util.ArrayList;
import java.util.Collections;
import java.util.Deque;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;

/**
 * PREDICTED theme: audit log + undo for admin actions (COMMAND pattern).
 * Each admin has a stack of executed commands (undo pops the last one).
 * Every successful action and undo is appended to an append-only audit log.
 */
public final class AdminActions {
	private static final Map<UUID, Deque<AdminCommand>> history = new HashMap<>();
	private static final List<AuditEntry> log = new ArrayList<>();

	private AdminActions() {}

	/** @return false if not an admin or the command failed (nothing is logged) */
	public static boolean run(UUID admin, AdminCommand command, long timestamp) {
		User a = UserDAO.getInstance().getByUUID(admin);
		if (a == null || a.role() != User.Role.Admin || !command.execute()) return false;
		history.computeIfAbsent(admin, k -> new ArrayDeque<>()).push(command);
		log.add(new AuditEntry(admin, command.describe(), timestamp));
		return true;
	}

	/** Undo this admin's most recent action. @return false if there is nothing to undo */
	public static boolean undo(UUID admin, long timestamp) {
		Deque<AdminCommand> stack = history.get(admin);
		if (stack == null || stack.isEmpty()) return false;
		AdminCommand command = stack.pop();
		command.undo();
		log.add(new AuditEntry(admin, "UNDO " + command.describe(), timestamp));
		return true;
	}

	/** @return the whole log, oldest first (read-only) */
	public static List<AuditEntry> getLog() {
		return Collections.unmodifiableList(log);
	}

	/** persistence: the log is saved; undo stacks are not (undo only works within one run) */
	public static void restoreLog(AuditEntry entry) {
		log.add(entry);
	}

	public static void clear() {
		history.clear();
		log.clear();
	}

	// ----------------------------------------------------- concrete commands

	/** Hide (or unhide) a message, remembering the previous state for undo. */
	public static AdminCommand hide(UUID messageId, boolean hidden) {
		return new AdminCommand() {
			private Message message;
			private boolean previous;

			public boolean execute() {
				message = MessageIndex.getInstance().get(messageId);
				if (message == null) return false;
				Post post = MessageIndex.getInstance().getPostOf(message);
				previous = post.isHidden(messageId);
				post.setHidden(message, hidden);
				return true;
			}

			public void undo() {
				MessageIndex.getInstance().getPostOf(message).setHidden(message, previous);
			}

			public String describe() {
				return (hidden ? "HIDE " : "UNHIDE ") + messageId;
			}
		};
	}

	/** Lock (or unlock) a post. */
	public static AdminCommand lock(UUID postId, boolean locked) {
		return new AdminCommand() {
			private Post post;
			private boolean previous;

			public boolean execute() {
				post = PostDAO.getInstance().get(new Post(postId));
				if (post == null) return false;
				previous = post.isLocked();
				post.setLocked(locked);
				return true;
			}

			public void undo() {
				post.setLocked(previous);
			}

			public String describe() {
				return (locked ? "LOCK " : "UNLOCK ") + postId;
			}
		};
	}
}
```

## 5. Direct messages

**`app/src/dms/DirectMessage.java`** (full file)

```java
package dms;

import java.util.UUID;

/** A private message between two users. */
public record DirectMessage(UUID id, UUID from, UUID to, long timestamp, String text) {}
```

**`app/src/dms/DirectMessageTools.java`** (full file)

```java
package dms;

import dao.UserDAO;
import relations.SocialTools;
import sorteddata.SortedData;
import sorteddata.SortedDataFactory;

import java.util.ArrayList;
import java.util.Comparator;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.UUID;

/**
 * PREDICTED theme: private / direct messages.
 * One conversation per PAIR of users, keyed the same way regardless of who sends
 * (smaller UUID first). Each conversation is a SortedData ordered by time, so
 * reading the latest k messages is O(log n + k) with getRange(..., backwards).
 * Privacy: only the two participants may read; blocked senders are refused.
 */
public final class DirectMessageTools {
	private static final Comparator<DirectMessage> BY_TIME =
			Comparator.comparingLong(DirectMessage::timestamp).thenComparing(DirectMessage::id);
	private static final Map<String, SortedData<DirectMessage>> conversations = new HashMap<>();

	private DirectMessageTools() {}

	private static String key(UUID a, UUID b) {
		return a.compareTo(b) <= 0 ? a + ":" + b : b + ":" + a;
	}

	/** @return the message, or null if a user is missing, self-message, empty text, or the receiver blocked the sender */
	public static DirectMessage send(UUID from, UUID to, String text, long timestamp) {
		if (UserDAO.getInstance().getByUUID(from) == null || UserDAO.getInstance().getByUUID(to) == null) return null;
		if (from.equals(to) || text == null || text.isBlank() || SocialTools.isBlocked(to, from)) return null;
		DirectMessage dm = new DirectMessage(UUID.randomUUID(), from, to, timestamp, text);
		conversations.computeIfAbsent(key(from, to), k -> SortedDataFactory.makeSortedData(BY_TIME)).insert(dm);
		return dm;
	}

	/** @return the conversation oldest first; empty if `viewer` is not one of the two users */
	public static List<DirectMessage> getConversation(UUID viewer, UUID other) {
		List<DirectMessage> result = new ArrayList<>();
		SortedData<DirectMessage> c = conversations.get(key(viewer, other));
		if (c != null) c.getAll().forEachRemaining(result::add);
		return result;
	}

	/** @return the newest `count` messages, newest first */
	public static List<DirectMessage> getLatest(UUID viewer, UUID other, int count) {
		List<DirectMessage> result = new ArrayList<>();
		SortedData<DirectMessage> c = conversations.get(key(viewer, other));
		if (c != null) c.getRange(null, count, true).forEachRemaining(result::add);
		return result;
	}

	/** @return everyone this user has a conversation with */
	public static List<UUID> getPartners(UUID user) {
		List<UUID> partners = new ArrayList<>();
		for (Map.Entry<String, SortedData<DirectMessage>> e : conversations.entrySet()) {
			Iterator<DirectMessage> it = e.getValue().getAll();
			if (!it.hasNext()) continue;
			DirectMessage any = it.next();
			if (any.from().equals(user)) partners.add(any.to());
			else if (any.to().equals(user)) partners.add(any.from());
		}
		return partners;
	}

	/** persistence: every direct message */
	public static List<DirectMessage> all() {
		List<DirectMessage> result = new ArrayList<>();
		for (SortedData<DirectMessage> c : conversations.values()) c.getAll().forEachRemaining(result::add);
		return result;
	}

	/** persistence: restore without checks */
	public static void restore(DirectMessage dm) {
		conversations.computeIfAbsent(key(dm.from(), dm.to()), k -> SortedDataFactory.makeSortedData(BY_TIME)).insert(dm);
	}

	public static void clear() {
		conversations.clear();
	}
}
```

## 6. Scheduled messages

**`app/src/scheduled/ScheduledMessages.java`** (full file)

```java
package scheduled;

import dao.PostDAO;
import dao.UserDAO;
import dao.model.Message;
import dao.model.Post;

import java.util.ArrayList;
import java.util.HashSet;
import java.util.List;
import java.util.PriorityQueue;
import java.util.Set;
import java.util.UUID;
import java.util.Comparator;

/**
 * PREDICTED theme: scheduled (delayed) messages.
 * A min-heap ordered by publish time: the next one due is always at the top, so
 * publishDue(now) only looks at messages that are actually due: O(k log n).
 * Cancelling marks the id; cancelled entries are skipped when they reach the top
 * ("lazy deletion": avoids an O(n) removal from the heap).
 */
public final class ScheduledMessages {
	private static final PriorityQueue<Message> queue =
			new PriorityQueue<>(Comparator.comparingLong(Message::timestamp).thenComparing(Message::id));
	private static final Set<UUID> cancelled = new HashSet<>();

	private ScheduledMessages() {}

	/** @return the id of the scheduled message, or null if post/user missing */
	public static UUID schedule(UUID post, UUID user, String text, long publishAt) {
		if (PostDAO.getInstance().get(new Post(post)) == null || UserDAO.getInstance().getByUUID(user) == null || text == null) return null;
		Message message = new Message(UUID.randomUUID(), user, post, publishAt, text);
		queue.add(message);
		return message.id();
	}

	public static boolean cancel(UUID scheduledId) {
		for (Message m : queue) {
			if (m.id().equals(scheduledId) && !cancelled.contains(scheduledId)) return cancelled.add(scheduledId);
		}
		return false;
	}

	/** Publishes every message with publish time <= now into its post. @return what was published */
	public static List<Message> publishDue(long now) {
		List<Message> published = new ArrayList<>();
		while (!queue.isEmpty() && queue.peek().timestamp() <= now) {
			Message m = queue.poll();
			if (cancelled.remove(m.id())) continue;
			Post post = PostDAO.getInstance().get(new Post(m.thread()));
			if (post != null && post.messages.insert(m)) published.add(m);
		}
		return published;
	}

	public static int pendingCount() {
		return queue.size() - cancelled.size();
	}

	/** persistence: pending (not cancelled) scheduled messages */
	public static List<Message> pending() {
		List<Message> result = new ArrayList<>();
		for (Message m : queue) if (!cancelled.contains(m.id())) result.add(m);
		return result;
	}

	public static void restore(Message message) {
		queue.add(message);
	}

	public static void clear() {
		queue.clear();
		cancelled.clear();
	}
}
```

## 7. Boards: Composite

**`app/src/boards/Board.java`** (full file)

```java
package boards;

import java.util.ArrayList;
import java.util.Collections;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.Set;
import java.util.UUID;

/**
 * PREDICTED theme: categories / boards / subforums (COMPOSITE pattern).
 * A board holds posts AND sub-boards; "all posts" and "count" work the same on a
 * leaf board and on a whole tree of boards.
 */
public class Board {
	private final String name;
	private final List<Board> children = new ArrayList<>();
	private final Set<UUID> posts = new LinkedHashSet<>();

	public Board(String name) {
		this.name = name;
	}

	public Board addChild(Board child) {
		children.add(child);
		return child;
	}

	public boolean addPost(UUID post) { return posts.add(post); }
	public boolean removePost(UUID post) { return posts.remove(post); }
	public String getName() { return name; }
	/** posts directly on this board (not sub-boards) */
	public Set<UUID> getOwnPosts() { return Collections.unmodifiableSet(posts); }
	public List<Board> getChildren() { return Collections.unmodifiableList(children); }

	/** posts on this board and every board below it */
	public List<UUID> getAllPosts() {
		List<UUID> result = new ArrayList<>(posts);
		for (Board child : children) result.addAll(child.getAllPosts());
		return result;
	}

	public int countPosts() {
		int total = posts.size();
		for (Board child : children) total += child.countPosts();
		return total;
	}

	/** @return the board with this name in the subtree (depth-first), or null */
	public Board find(String boardName) {
		if (name.equalsIgnoreCase(boardName)) return this;
		for (Board child : children) {
			Board found = child.find(boardName);
			if (found != null) return found;
		}
		return null;
	}

	/** Moves a post from wherever it is in this subtree to `target`. */
	public boolean movePost(UUID post, Board target) {
		if (!removeFromSubtree(post)) return false;
		return target.addPost(post);
	}

	private boolean removeFromSubtree(UUID post) {
		if (posts.remove(post)) return true;
		for (Board child : children) if (child.removeFromSubtree(post)) return true;
		return false;
	}
}
```

**`app/src/boards/BoardTools.java`** (full file)

```java
package boards;

import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

/**
 * Singleton home for the board tree (so it can be saved / loaded).
 * Board names are assumed unique (find() searches by name).
 */
public final class BoardTools {
	public static final String ROOT_NAME = "All";
	private static Board root = new Board(ROOT_NAME);

	private BoardTools() {}

	public static Board root() {
		return root;
	}

	/** @return the new board, or null if the parent is missing or the name is taken */
	public static Board createBoard(String name, String parentName) {
		Board parent = root.find(parentName);
		if (parent == null || name == null || name.isBlank() || root.find(name) != null) return null;
		return parent.addChild(new Board(name));
	}

	/** persistence: (board, parent) for every board except the root, parents before children */
	public static List<String[]> boardRows() {
		List<String[]> rows = new ArrayList<>();
		addRows(root, rows);
		return rows;
	}

	private static void addRows(Board board, List<String[]> rows) {
		for (Board child : board.getChildren()) {
			rows.add(new String[] {child.getName(), board.getName()});
			addRows(child, rows);
		}
	}

	/** persistence: (board, post) for every post placed on a board */
	public static List<String[]> postRows() {
		List<String[]> rows = new ArrayList<>();
		addPostRows(root, rows);
		return rows;
	}

	private static void addPostRows(Board board, List<String[]> rows) {
		for (UUID post : board.getOwnPosts()) rows.add(new String[] {board.getName(), post.toString()});
		for (Board child : board.getChildren()) addPostRows(child, rows);
	}

	public static void clear() {
		root = new Board(ROOT_NAME);
	}
}
```

## 8. Links and duplicate detection

**`app/src/links/ContentChecks.java`** (full file)

```java
package links;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

/**
 * PREDICTED small themes:
 *  - extract links from a message (attachments / previews)
 *  - duplicate-message detection: the same user posting the same text again within a time window
 */
public final class ContentChecks {
	private static final Pattern URL = Pattern.compile("https?://[^\\s]+");
	private static final Map<UUID, Map<String, Long>> lastPosted = new HashMap<>(); // user -> normalised text -> time

	private ContentChecks() {}

	/** @return every http(s) link in the text, in order (trailing . , ) ! ? removed) */
	public static List<String> extractLinks(String text) {
		List<String> links = new ArrayList<>();
		if (text == null) return links;
		Matcher m = URL.matcher(text);
		while (m.find()) links.add(m.group().replaceAll("[.,)!?]+$", ""));
		return links;
	}

	/**
	 * Records the post and says whether it duplicates one from the same user within the window.
	 * Text is compared ignoring case and repeated spaces.
	 */
	public static boolean isDuplicate(UUID user, String text, long now, long windowMillis) {
		String key = text == null ? "" : text.trim().replaceAll("\\s+", " ").toLowerCase();
		Map<String, Long> mine = lastPosted.computeIfAbsent(user, k -> new HashMap<>());
		Long previous = mine.put(key, now);
		return previous != null && now - previous < windowMillis;
	}

	public static void clear() {
		lastPosted.clear();
	}
}
```

## 9. Replies and nested threads

**`app/src/threads/ThreadTools.java`** (full file)

```java
package threads;

import dao.MessageIndex;
import dao.UserDAO;
import dao.model.Message;
import dao.model.Post;

import java.util.ArrayList;
import java.util.Comparator;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;

/**
 * PREDICTED theme: replies to a specific message (nested threads) and quoting.
 * The reply is a normal Message in the same post (so everything else keeps working);
 * we only store the parent/child links:  child -> parent  and  parent -> children.
 * The messages form a TREE; getThread walks it depth-first (pre-order), like a forum view.
 */
public final class ThreadTools {
	private static final Map<UUID, UUID> parentOf = new HashMap<>();
	private static final Map<UUID, List<UUID>> childrenOf = new HashMap<>();

	private ThreadTools() {}

	/** One line of a thread view: the message and how deep it is (0 = the root you asked for). */
	public record ThreadEntry(Message message, int depth) {}

	/** @return the new reply, or null if the parent/user is missing or text is null */
	public static Message reply(UUID parent, UUID user, String text, long timestamp) {
		Message target = MessageIndex.getInstance().get(parent);
		if (target == null || text == null || UserDAO.getInstance().getByUUID(user) == null) return null;
		Post post = MessageIndex.getInstance().getPostOf(target);
		Message reply = new Message(UUID.randomUUID(), user, target.thread(), timestamp, text);
		post.messages.insert(reply);
		link(parent, reply.id());
		return reply;
	}

	/** persistence + reply(): record that child replies to parent */
	public static void link(UUID parent, UUID child) {
		parentOf.put(child, parent);
		childrenOf.computeIfAbsent(parent, k -> new ArrayList<>()).add(child);
	}

	/** @return the message this one replies to, or null for a top-level message */
	public static UUID getParent(UUID message) {
		return parentOf.get(message);
	}

	/** @return direct replies, oldest first (deleted ones skipped) */
	public static List<Message> getReplies(UUID message) {
		List<Message> result = new ArrayList<>();
		for (UUID id : childrenOf.getOrDefault(message, List.of())) {
			Message m = MessageIndex.getInstance().get(id);
			if (m != null) result.add(m);
		}
		result.sort(Comparator.comparingLong(Message::timestamp));
		return result;
	}

	/** @return the message and all replies below it, depth-first (pre-order), oldest replies first */
	public static List<ThreadEntry> getThread(UUID root) {
		List<ThreadEntry> result = new ArrayList<>();
		Message m = MessageIndex.getInstance().get(root);
		if (m != null) walk(m, 0, result);
		return result;
	}

	private static void walk(Message message, int depth, List<ThreadEntry> out) {
		out.add(new ThreadEntry(message, depth));
		for (Message child : getReplies(message.id())) walk(child, depth + 1, out);
	}

	/** @return how many replies deep this message is (0 = top level) */
	public static int getDepth(UUID message) {
		int depth = 0;
		for (UUID p = parentOf.get(message); p != null; p = parentOf.get(p)) depth++;
		return depth;
	}

	/** @return the text of a reply that quotes the message: "> original\n\nyour text" */
	public static String quote(UUID message, String yourText) {
		Message m = MessageIndex.getInstance().get(message);
		if (m == null) return yourText;
		StringBuilder quoted = new StringBuilder();
		for (String line : m.message().split("\n", -1)) quoted.append("> ").append(line).append('\n');
		return quoted.append('\n').append(yourText).toString();
	}

	/** persistence: child -> parent */
	public static Map<UUID, UUID> allLinks() {
		return java.util.Collections.unmodifiableMap(parentOf);
	}

	public static void clear() {
		parentOf.clear();
		childrenOf.clear();
	}
}
```

## 10. Private posts

**`app/src/access/AccessTools.java`** (full file)

```java
package access;

import dao.PostDAO;
import dao.UserDAO;
import dao.model.Post;
import dao.model.User;
import relations.RelationStore;

import java.util.Collections;
import java.util.HashSet;
import java.util.Set;
import java.util.UUID;

/**
 * PREDICTED theme: private / invite-only posts (an access control list per post).
 * A post is public unless marked private. A private post can be seen by its author,
 * admins, and invited users. invites is a RelationStore post -> user (O(1) checks).
 */
public final class AccessTools {
	private static final Set<UUID> privatePosts = new HashSet<>();
	private static final RelationStore<UUID, UUID> invites = new RelationStore<>(); // post -> user

	private AccessTools() {}

	/** Author or Admin. */
	public static boolean setPrivate(UUID post, UUID user, boolean isPrivate) {
		if (!isOwnerOrAdmin(post, user)) return false;
		if (isPrivate) privatePosts.add(post);
		else privatePosts.remove(post);
		return true;
	}

	/** Author or Admin invites someone. @return false if not allowed, user missing, or already invited */
	public static boolean invite(UUID post, UUID owner, UUID guest) {
		if (!isOwnerOrAdmin(post, owner) || UserDAO.getInstance().getByUUID(guest) == null) return false;
		return invites.add(post, guest);
	}

	public static boolean uninvite(UUID post, UUID owner, UUID guest) {
		return isOwnerOrAdmin(post, owner) && invites.remove(post, guest);
	}

	/** @return true if this user (null = guest) may read the post */
	public static boolean canView(UUID post, UUID user) {
		Post p = post == null ? null : PostDAO.getInstance().get(new Post(post));
		if (p == null) return false;
		if (!privatePosts.contains(post)) return true;
		if (user == null) return false;
		User u = UserDAO.getInstance().getByUUID(user);
		if (u == null) return false;
		return u.role() == User.Role.Admin || u.id().equals(p.poster) || invites.contains(post, user);
	}

	public static boolean isPrivate(UUID post) {
		return privatePosts.contains(post);
	}

	/** persistence */
	public static Set<UUID> privatePosts() { return Collections.unmodifiableSet(privatePosts); }
	public static RelationStore<UUID, UUID> invites() { return invites; }
	public static void restorePrivate(UUID post) { privatePosts.add(post); }

	public static void clear() {
		privatePosts.clear();
		invites.clear();
	}

	private static boolean isOwnerOrAdmin(UUID post, UUID user) {
		Post p = post == null ? null : PostDAO.getInstance().get(new Post(post));
		User u = UserDAO.getInstance().getByUUID(user);
		return p != null && u != null && (u.role() == User.Role.Admin || u.id().equals(p.poster));
	}
}
```

## 11. Login lockout and history

**`app/src/login/LoginTools.java`** (full file)

```java
package login;

import dao.UserDAO;
import dao.model.User;

import java.util.ArrayDeque;
import java.util.ArrayList;
import java.util.Deque;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;

/**
 * PREDICTED theme: login security.
 *  - lockout: after MAX_FAILURES wrong passwords in a row, the account is locked for LOCK_MILLIS
 *  - history: the last HISTORY_SIZE login attempts per user (newest first, a bounded Deque)
 * Uses UserDAO.login for the actual check, so the existing rules are reused, not copied.
 */
public final class LoginTools {
	public static final int MAX_FAILURES = 3;
	public static final long LOCK_MILLIS = 5 * 60 * 1000L;
	public static final int HISTORY_SIZE = 10;

	/** One login attempt. */
	public record LoginAttempt(UUID user, long timestamp, boolean success) {}

	private static final Map<UUID, Integer> failuresInARow = new HashMap<>();
	private static final Map<UUID, Long> lockedUntil = new HashMap<>();
	private static final Map<UUID, Deque<LoginAttempt>> history = new HashMap<>();

	private LoginTools() {}

	/** @return the user on success; null if unknown, locked, or wrong password */
	public static User login(String username, String password, long now) {
		User user = username == null ? null : UserDAO.getInstance().get(new User(username));
		if (user == null || user.id() == null) return null;
		if (isLocked(user.id(), now)) {
			record(user.id(), now, false);
			return null;
		}
		boolean ok = UserDAO.getInstance().login(username, password) != null;
		record(user.id(), now, ok);
		if (ok) {
			failuresInARow.remove(user.id());
			return user;
		}
		int failures = failuresInARow.merge(user.id(), 1, Integer::sum);
		if (failures >= MAX_FAILURES) {
			lockedUntil.put(user.id(), now + LOCK_MILLIS);
			failuresInARow.remove(user.id());
		}
		return null;
	}

	public static boolean isLocked(UUID user, long now) {
		Long until = lockedUntil.get(user);
		return until != null && now < until;
	}

	/** Admin unlocks an account early. */
	public static boolean unlock(UUID admin, UUID user) {
		User a = UserDAO.getInstance().getByUUID(admin);
		if (a == null || a.role() != User.Role.Admin) return false;
		failuresInARow.remove(user);
		return lockedUntil.remove(user) != null;
	}

	/** @return newest first */
	public static List<LoginAttempt> getHistory(UUID user) {
		return new ArrayList<>(history.getOrDefault(user, new ArrayDeque<>()));
	}

	private static void record(UUID user, long now, boolean success) {
		Deque<LoginAttempt> attempts = history.computeIfAbsent(user, k -> new ArrayDeque<>());
		attempts.addFirst(new LoginAttempt(user, now, success));
		if (attempts.size() > HISTORY_SIZE) attempts.removeLast(); // keep it bounded
	}

	public static void clear() {
		failuresInARow.clear();
		lockedUntil.clear();
		history.clear();
	}
}
```

## 12. Merge / split posts

**`app/src/posts/PostTools.java`** (full file)

```java
package posts;

import dao.MessageIndex;
import dao.PostDAO;
import dao.model.Message;
import dao.model.Post;

import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;
import java.util.UUID;

/**
 * PREDICTED theme: merge two posts / split a post (admin).
 * Messages keep their ids, so reports, reactions, votes and edits stay attached.
 * A message's post id (thread) is part of the immutable record, so each move is
 * remove + insert of a copy with the new thread (same idea as MoveTools).
 */
public final class PostTools {
	private PostTools() {}

	/** Moves every message of `source` into `target`, then deletes `source`. Admin only. */
	public static boolean mergePosts(UUID source, UUID target, UUID admin) {
		Post from = find(source), to = find(target);
		if (from == null || to == null || source.equals(target) || !isAdmin(admin)) return false;
		for (Message m : snapshot(from.messages.getAll())) moveInto(from, to, m);
		PostDAO.getInstance().remove(from);
		return true;
	}

	/**
	 * Moves `firstMessage` and every LATER message of its post into a new post.
	 * @return the new post's id, or null if not allowed
	 */
	public static UUID splitPost(UUID firstMessage, String newTopic, UUID admin) {
		Message first = MessageIndex.getInstance().get(firstMessage);
		if (first == null || newTopic == null || !isAdmin(admin)) return null;
		Post from = MessageIndex.getInstance().getPostOf(first);
		Post to = new Post(UUID.randomUUID(), first.poster(), newTopic);
		PostDAO.getInstance().add(to);
		// getRange(first, -1, false) = first and everything after it, in order
		for (Message m : snapshot(from.messages.getRange(first, -1, false))) moveInto(from, to, m);
		return to.id;
	}

	// copy first: we must not change a tree while iterating it
	private static List<Message> snapshot(Iterator<Message> it) {
		List<Message> list = new ArrayList<>();
		it.forEachRemaining(list::add);
		return list;
	}

	private static void moveInto(Post from, Post to, Message m) {
		boolean hidden = from.isHidden(m.id()), deleted = from.isDeleted(m.id());
		from.removeMessage(m);
		Message moved = new Message(m.id(), m.poster(), to.id, m.timestamp(), m.message());
		to.messages.insert(moved);
		if (hidden) to.setHidden(moved, true);
		if (deleted) to.setDeleted(moved, true);
	}

	private static Post find(UUID id) {
		return id == null ? null : PostDAO.getInstance().get(new Post(id));
	}

	private static boolean isAdmin(UUID user) {
		return accounts.Permissions.isAdmin(user); // one shared permission check
	}
}
```

## 13. Emoji, markdown, previews

**`app/src/textformat/TextFormat.java`** (full file)

```java
package textformat;

import java.util.Map;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

/**
 * PREDICTED theme: rich text in messages.
 *  - emoji shortcodes ":smile:" -> "😄" (unknown codes are left as they are)
 *  - markdown to plain text: **bold**, *italic*, `code`, [text](url) -> text (url)
 *  - preview: first N characters, cut at a word boundary, with "..."
 */
public final class TextFormat {
	private static final Map<String, String> EMOJI = Map.of(
			"smile", "😄", "heart", "❤️", "thumbsup", "👍",
			"laugh", "😂", "sad", "😢", "fire", "🔥");
	private static final Pattern SHORTCODE = Pattern.compile(":([a-z]+):");

	private TextFormat() {}

	public static String replaceEmoji(String text) {
		if (text == null) return null;
		Matcher m = SHORTCODE.matcher(text);
		StringBuilder out = new StringBuilder();
		while (m.find()) m.appendReplacement(out, Matcher.quoteReplacement(EMOJI.getOrDefault(m.group(1), m.group())));
		m.appendTail(out);
		return out.toString();
	}

	public static String markdownToPlain(String text) {
		if (text == null) return null;
		return text.replaceAll("\\[([^\\]]+)\\]\\(([^)]+)\\)", "$1 ($2)")
				.replaceAll("\\*\\*(.+?)\\*\\*", "$1")
				.replaceAll("\\*(.+?)\\*", "$1")
				.replaceAll("`(.+?)`", "$1");
	}

	public static String preview(String text, int maxLength) {
		if (text == null || text.length() <= maxLength) return text;
		int cut = text.lastIndexOf(' ', maxLength);
		if (cut <= 0) cut = maxLength;
		return text.substring(0, cut) + "...";
	}
}
```

## 14. How this was verified

| Check | Result |
|---|---|
| search: AND of words, newest first, edits remove old words AND add new ones, deleted messages vanish | pass |
| trie: prefix, limit, case-insensitive, remove; username autocomplete | pass |
| trending window, BUSY / ACTIVE, bad strategy throws | pass |
| leaderboards: MESSAGES, KARMA (only positive totals), REACTIONS, POSTS | pass |
| stats: totals, per user, per UTC day, average length, busiest post, empty cases | pass |
| censor: whole words only (`classic` untouched), 4 strategies, chained censors, `null` text | pass |
| audit/undo: member refused, failed command not logged, undo order, log contents | pass |
| DMs: privacy, blocked sender refused, latest-k newest first, partners | pass |
| scheduled: publish order, cancel (lazy deletion), pending count | pass |
| boards: recursive count / posts, find, move between sub-boards | pass |
| links (trailing punctuation stripped), duplicates within / after the window, per user | pass |
| replies: direct replies, depth-first thread with depths, quoting, unknown parent/user | pass |
| private posts: owner / admin / invited / guest / uninvite | pass |
| login: lock after 3 failures (even with the right password), expiry, admin unlock, history bounded to 10 | pass |
| merge (reports and hidden flags follow), split from a message onward | pass |
| emoji, markdown, previews | pass |
| everything above survives save + load (CSV and JSON) | pass |
