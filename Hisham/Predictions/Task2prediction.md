# Task 2 prediction: state on a model class + trees

> **Every prediction below has complete, tested code.** The trees were checked against `java.util.TreeSet`:
> 100 random rounds comparing rank / floor / ceiling / ranges / getRange (both directions, any start, any count) on
> AVL, SortedArrayList, red-black and BST, plus 60,000 random AVL insert/delete operations with the balance rule checked after each.

## 1. What Task 2 looks like (from the practice hackathon)

| Part | Marks | What earns it |
|---|---|---|
| The tools method (e.g. `setHidden(msg, user, hidden)`) | 40% | validation (exists? Admin?) + state change |
| The model method (e.g. `Post.getVisibleMessages(isAdmin)` returning a `SortedData`) | 40% | correct filtered view that **reacts** to later changes |
| Quality | 20% | |

"Data structures, trees" in the title means they want you in the tree code. Week 4 already said the tree
"should be designed so it can be extended to support deletion".

## 2. Every Task 2 prediction (all coded)

| # | Prediction | Tools method | Model / tree method | Section |
|---|---|---|---|---|
| 1 | Hide / unhide messages (practice) | `ModerationTools.setHidden` | `Post.getVisibleMessages(isAdmin)` | 4 |
| 2 | Soft-delete / restore (author or admin) | `ContentTools.deleteMessage`, `restoreMessage` | `Post.setDeleted`, `isVisibleToMembers` | 4, 5 |
| 3 | Lock a post (members cannot reply, admins can) | `ContentTools.setLocked`, `reply` | `Post.isLocked`, `MemberState.canReplyTo` hook | 4, 5, 6 |
| 4 | Pinned messages listed first | `ContentTools.setPinned`, `getMessagesPinnedFirst` | `Post.setPinned` (third tree) | 4, 5 |
| 5 | Blocked authors hidden for one viewer | `SocialTools.block` | `SocialTools.getMessagesFor` | Task1prediction.md 6 |
| 6 | Hard delete (admin) | `ContentTools.removeMessage` | `Post.removeMessage` -> `AVLTree.remove` | 5, 7 |
| 7 | AVL delete | | `AVLTree.remove` | 7 |
| 8 | SortedArrayList remove | | `SortedArrayList.remove` | 7 |
| 9 | Order statistics | `MessageQueries.getPosition` | `rank`, `floor`, `ceiling` (AVL, array list, red-black) | 8 |
| 10 | Time-range query / count, pagination | `getMessagesBetween`, `countMessagesBetween`, `getPage` | `SortedDataRanges.range`, `count` | 9 |
| 11 | `BSTree.getRange` (was `return null`) | | `BSIterator` | 10 |
| 12 | Red-black tree | | `RedBlackTree` (left-leaning) | 11 |
| 13 | Move a message to another post | `MoveTools.moveMessage` | remove + insert copy with new thread | 12 |
| 14 | Edit a message (record is immutable) | `EditTools.editMessage` (Task 1 file) | `Post.replaceMessage` | 4 |
| 15 | SortedArrayList: backwards iteration (used to throw) + `getAtIndex` out of range returns null | | `SortedArrayList.getRange(..., true)` | 7 |
| 16 | Merge / split posts | `PostTools` (ExtraPredictions.md 12) | `getRange(first, -1, false)` | ExtraPredictions.md |

## 3. Two ways to build a "visible" view

| | A: filtered copy every call | B: second tree kept in sync (used below) |
|---|---|---|
| `getVisibleMessages(false)` | O(n log n) per call | O(1) |
| hide / delete / restore | O(1) | O(log n) (tree delete / insert) |
| extra memory | none | one more tree per post |
| needs tree delete? | no | **yes** (section 7) |

Approach A (tested: swapping it into `Post` passes all 60 visibility/moderation tests):

```java
public SortedData<Message> getVisibleMessages(boolean isAdmin) {
	if (isAdmin) return messages;
	return SortedDataViews.filter(messages, MessageComparator.getInstance(), m -> isVisibleToMembers(m.id()));
}
```

**`app/src/sorteddata/SortedDataViews.java`** (full file)

```java
package sorteddata;

import java.util.Comparator;
import java.util.Iterator;
import java.util.function.Predicate;

/**
 * HACKATHON (Task 2, simple alternative): build a NEW SortedData containing only
 * the elements that pass a filter. O(n log n) per call, no extra state to keep in sync.
 * Use this if you cannot (or do not want to) implement remove() on the tree.
 */
public final class SortedDataViews {
	private SortedDataViews() {}

	public static <T> SortedData<T> filter(SortedData<T> source, Comparator<T> comparator, Predicate<T> keep) {
		SortedData<T> result = SortedDataFactory.makeSortedData(comparator);
		for (Iterator<T> it = source.getAll(); it.hasNext(); ) {
			T element = it.next();
			if (keep.test(element)) result.insert(element);
		}
		return result;
	}
}
```

## 4. The model: `Post` (hidden, deleted, pinned, locked, replace, remove)

**`app/src/dao/model/Post.java`** (full file)

```java
package dao.model;

import dao.MessageComparator;
import dao.MessageIndex;
import sorteddata.SortedData;
import sorteddata.SortedDataFactory;

import java.util.Collections;
import java.util.HashSet;
import java.util.Iterator;
import java.util.Set;
import java.util.UUID;

/**
 * A forum post and its messages.
 * HACKATHON additions (Task 2 predictions), all on the post because the post owns its messages:
 *  - hidden messages        (admin moderation)        -> not visible to non-admins
 *  - soft-deleted messages  (author or admin)         -> not visible to non-admins, can be restored
 *  - pinned messages        (admin)                   -> listed first
 *  - locked post            (admin)                   -> members cannot reply
 *  - replace / remove a message (edit history, hard delete)
 * A message is visible to non-admins when it is neither hidden nor deleted.
 */
public class Post implements HasUUID {
	public final UUID id;
	public final UUID poster;
	public final String topic;
	public final SortedData<Message> messages;

	// Second tree with only the messages non-admins may see, kept in sync by
	// listening to `messages` and by refreshVisibility(...).
	//   getVisibleMessages(false) O(1); hide/delete/restore O(log n); costs extra memory.
	private final SortedData<Message> visibleMessages;
	private final SortedData<Message> pinnedMessages;
	private final Set<UUID> hiddenMessageIds = new HashSet<>();
	private final Set<UUID> deletedMessageIds = new HashSet<>();
	private boolean locked = false;

	public Post(UUID id, UUID poster, String topic) {
		this.id = id;
		this.poster = poster;
		this.topic = topic;
		this.messages = SortedDataFactory.makeSortedData(MessageComparator.getInstance());
		this.visibleMessages = SortedDataFactory.makeSortedData(MessageComparator.getInstance());
		this.pinnedMessages = SortedDataFactory.makeSortedData(MessageComparator.getInstance());

		// every new message: index it by UUID, and show it unless it is hidden/deleted
		this.messages.registerListener(MessageIndex.getInstance());
		this.messages.registerListener(this::refreshVisibility);
	}

	public Post(UUID id) {
		this(id, null, null);
	}

	// ------------------------------------------------------------ visibility

	/** @return true if non-admins may see this message (not hidden and not deleted) */
	public boolean isVisibleToMembers(UUID messageId) {
		return !hiddenMessageIds.contains(messageId) && !deletedMessageIds.contains(messageId);
	}

	// puts the message into / takes it out of the visible tree to match its flags.
	// insert of an existing element and remove of a missing one both do nothing, so this is idempotent.
	private void refreshVisibility(Message message) {
		if (isVisibleToMembers(message.id())) visibleMessages.insert(message);
		else visibleMessages.remove(message);
	}

	/** Hide or unhide (admin moderation). Doing it twice is harmless. */
	public void setHidden(Message message, boolean hidden) {
		if (hidden) hiddenMessageIds.add(message.id());
		else hiddenMessageIds.remove(message.id());
		refreshVisibility(message);
	}

	/** Soft-delete or restore. The message stays in `messages`, so admins still see it. */
	public void setDeleted(Message message, boolean deleted) {
		if (deleted) deletedMessageIds.add(message.id());
		else deletedMessageIds.remove(message.id());
		refreshVisibility(message);
	}

	public boolean isHidden(UUID messageId) {
		return hiddenMessageIds.contains(messageId);
	}

	public boolean isDeleted(UUID messageId) {
		return deletedMessageIds.contains(messageId);
	}

	/** @return read-only view of hidden message ids (for persistence) */
	public Set<UUID> getHiddenMessageIds() {
		return Collections.unmodifiableSet(hiddenMessageIds);
	}

	/** @return read-only view of soft-deleted message ids (for persistence) */
	public Set<UUID> getDeletedMessageIds() {
		return Collections.unmodifiableSet(deletedMessageIds);
	}

	/**
	 * Admins see everything; everyone else sees only non-hidden, non-deleted messages.
	 * The returned structure is live: do not insert into it directly.
	 */
	public SortedData<Message> getVisibleMessages(boolean isAdmin) {
		return isAdmin ? messages : visibleMessages;
	}

	// ------------------------------------------------------------ pinning

	public void setPinned(Message message, boolean pinned) {
		if (pinned) pinnedMessages.insert(message);
		else pinnedMessages.remove(message);
	}

	public boolean isPinned(Message message) {
		return pinnedMessages.get(message) != null;
	}

	/** @return pinned messages in time order (includes hidden ones; callers filter for non-admins) */
	public Iterator<Message> getPinnedMessages() {
		return pinnedMessages.getAll();
	}

	// ------------------------------------------------------------ locking

	public boolean isLocked() {
		return locked;
	}

	public void setLocked(boolean locked) {
		this.locked = locked;
	}

	// ------------------------------------------------------------ replace / remove

	/**
	 * Permanently removes a message from every structure of this post.
	 * @return false if the message was not in this post
	 */
	public boolean removeMessage(Message message) {
		if (!messages.remove(message)) return false;
		visibleMessages.remove(message);
		pinnedMessages.remove(message);
		hiddenMessageIds.remove(message.id());
		deletedMessageIds.remove(message.id());
		return true;
	}

	/**
	 * Swaps a message for a new version with the same id/timestamp/thread/poster (an edit).
	 * Needed because Message is an immutable record: an edit is a NEW record.
	 * The comparator ignores the text, so old and new compare as equal: we must remove
	 * the old one first, otherwise insert() thinks it is a duplicate and does nothing.
	 * Flags (hidden/deleted/pinned) are kept because they are stored by id.
	 */
	public void replaceMessage(Message oldVersion, Message newVersion) {
		boolean wasPinned = isPinned(oldVersion);
		messages.remove(oldVersion);
		visibleMessages.remove(oldVersion);
		pinnedMessages.remove(oldVersion);
		messages.insert(newVersion);  // listeners re-index it and refresh visibility
		if (wasPinned) pinnedMessages.insert(newVersion);
	}

	public UUID getUUID() { return id; }
}
```

The practice tools method (from `ModerationTools`, full file in Task1prediction.md):

```java
public static boolean setHidden(UUID message, UUID user, boolean hidden) {
	Message target = MessageIndex.getInstance().get(message);          // O(1) exists check
	User moderator = UserDAO.getInstance().getByUUID(user);             // O(1) exists check
	if (target == null || moderator == null || moderator.role() != User.Role.Admin) return false;
	Post post = MessageIndex.getInstance().getPostOf(target);          // O(log n)
	post.setHidden(target, hidden);                                     // O(log n) tree update
	return true;
}
```

## 5. Tools for predictions 2, 3, 4, 6: `ContentTools`

**`app/src/content/ContentTools.java`** (full file)

```java
package content;

import dao.MessageIndex;
import dao.PostDAO;
import dao.UserDAO;
import dao.model.Message;
import dao.model.Post;
import dao.model.User;

import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;
import java.util.UUID;

/**
 * PREDICTED Task 2 facades (same shape as ModerationTools.setHidden):
 *  soft delete / restore, hard delete, lock / unlock + reply, pin / unpin + pinned-first listing.
 * Every method: validate (exists? allowed?) -> return false without changing anything, else act.
 */
public final class ContentTools {
	private ContentTools() {}

	// ------------------------------------------------------------ soft delete

	/** Author or Admin may soft-delete. @return false if ids missing or not allowed */
	public static boolean deleteMessage(UUID message, UUID user) {
		return setDeleted(message, user, true);
	}

	/** Author or Admin may restore a soft-deleted message. */
	public static boolean restoreMessage(UUID message, UUID user) {
		return setDeleted(message, user, false);
	}

	private static boolean setDeleted(UUID message, UUID user, boolean deleted) {
		Message target = MessageIndex.getInstance().get(message);
		User actor = UserDAO.getInstance().getByUUID(user);
		if (target == null || actor == null || !isAuthorOrAdmin(target, actor)) return false;
		MessageIndex.getInstance().getPostOf(target).setDeleted(target, deleted);
		return true;
	}

	// ------------------------------------------------------------ hard delete

	/** Admin only. The message is gone for everyone (reports etc. may also need cleaning). */
	public static boolean removeMessage(UUID message, UUID admin) {
		Message target = MessageIndex.getInstance().get(message);
		if (target == null || !isAdmin(admin)) return false;
		MessageIndex.getInstance().getPostOf(target).removeMessage(target);
		MessageIndex.getInstance().forget(message);
		return true;
	}

	// ------------------------------------------------------------ locking

	/** Admin only. @return false if the post or admin does not exist / is not an admin */
	public static boolean setLocked(UUID post, UUID admin, boolean locked) {
		Post target = findPost(post);
		if (target == null || !isAdmin(admin)) return false;
		target.setLocked(locked);
		return true;
	}

	/**
	 * Adds a reply. Members cannot reply to a locked post; admins can.
	 * @return the new message, or null if not allowed
	 */
	public static Message reply(UUID post, UUID user, String text, long timestamp) {
		Post target = findPost(post);
		User author = UserDAO.getInstance().getByUUID(user);
		if (target == null || author == null || text == null) return null;
		if (target.isLocked() && author.role() != User.Role.Admin) return null;
		if (bans.BanTools.isBanned(user, timestamp)) return null; // banned users cannot post
		Message message = new Message(UUID.randomUUID(), user, post, timestamp, text);
		target.messages.insert(message);
		return message;
	}

	// ------------------------------------------------------------ pinning

	/** Admin only. */
	public static boolean setPinned(UUID message, UUID admin, boolean pinned) {
		Message target = MessageIndex.getInstance().get(message);
		if (target == null || !isAdmin(admin)) return false;
		MessageIndex.getInstance().getPostOf(target).setPinned(target, pinned);
		return true;
	}

	/**
	 * Pinned messages first (in time order), then every other message (in time order).
	 * Non-admins only see visible ones. Each message appears once.
	 */
	public static Iterator<Message> getMessagesPinnedFirst(UUID post, boolean isAdmin) {
		Post target = findPost(post);
		List<Message> result = new ArrayList<>();
		if (target == null) return result.iterator();
		for (Iterator<Message> it = target.getPinnedMessages(); it.hasNext(); ) {
			Message m = it.next();
			if (isAdmin || target.isVisibleToMembers(m.id())) result.add(m);
		}
		for (Iterator<Message> it = target.getVisibleMessages(isAdmin).getAll(); it.hasNext(); ) {
			Message m = it.next();
			if (!target.isPinned(m)) result.add(m);
		}
		return result.iterator();
	}

	// ------------------------------------------------------------ helpers

	private static Post findPost(UUID post) {
		return post == null ? null : PostDAO.getInstance().get(new Post(post));
	}

	static boolean isAdmin(UUID user) {
		User u = UserDAO.getInstance().getByUUID(user);
		return u != null && u.role() == User.Role.Admin;
	}

	private static boolean isAuthorOrAdmin(Message message, User user) {
		return user.role() == User.Role.Admin || user.id().equals(message.poster());
	}
}
```

## 6. Locked posts in the State pattern (`userstate`)

The practice base had no `userstate` package; this is your miniproject's version with a **Template Method hook**:
members cannot reply to a locked post, admins override the hook.

**`app/src/userstate/MemberState.java`** (full file)

```java
package userstate;

import dao.model.Message;
import dao.model.Post;
import dao.model.User;

import java.util.UUID;

public class MemberState extends UserState {
	public final User user;
	public MemberState(User user) {
		this.user = user;
	}

	@Override
	public boolean isLoggedIn() {
		return true;
	}


	@Override
	public UserState register(String username, String password) {
		return this;
	}

	@Override
	public UserState logout() {
		return new GuestState();
	}

	@Override
	public boolean addReply(Post post, String content) {
		if (!canReplyTo(post)) return false; // HACKATHON: locked posts
		post.messages.insert(new Message(UUID.randomUUID(), user.getUUID(), post.getUUID(), System.currentTimeMillis(), content));
		return true;
	}

	/**
	 * HACKATHON: hook that subclasses override (Template Method + State pattern).
	 * Members may not reply to a locked post.
	 */
	protected boolean canReplyTo(Post post) {
		return !post.isLocked();
	}
}
```

**`app/src/userstate/AdminState.java`** (full file)

```java
package userstate;

import dao.model.Post;
import dao.model.User;

public class AdminState extends MemberState {
	public AdminState(User user) {
		super(user);
	}

	/** HACKATHON: admins may reply even to locked posts. */
	@Override
	protected boolean canReplyTo(Post post) {
		return true;
	}
}
```

## 7. Tree deletion

> How to read a diff block: lines starting with `+` are new, lines starting with `-` are deleted, other lines are unchanged context so you can find the spot. `@@` lines just mark where the next chunk starts.

**Sign trap:** in this base `balance = left.height - right.height` (positive = left-heavy). Your GitLab `main`
uses `right.height - left.height`. Flip every comparison if your base uses the other convention.

Algorithm: walk down rebuilding each node on the path (immutable); at the target, 0 or 1 child -> return the child,
2 children -> copy the in-order successor here and delete it from the right; on the way up call `rebalance()`
(same 4 rotations as insert, but a child can have balance 0, hence `< 0` / `> 0`).

**`app/src/sorteddata/avltree/AVLNode.java`** (changes only: `+` = add this line, `-` = remove this line)

```diff
@@ -74,4 +74,21 @@
 	 * @return true if the element exists in the tree, false otherwise
 	 */
 	public abstract boolean contains(T element);
+
+	/**
+	 * HACKATHON: removes an element (if present) and returns the rebalanced tree.
+	 * Like insert, this is immutable: the current tree is not modified.
+	 * @param element the element to remove
+	 * @return the resulting tree (may be empty)
+	 */
+	public abstract AVLNode<T> delete(T element);
+
+	/** HACKATHON: number of elements strictly smaller than element */
+	public abstract int rank(T element);
+
+	/** HACKATHON: largest element <= target, or null */
+	public abstract T floor(T target);
+
+	/** HACKATHON: smallest element >= target, or null */
+	public abstract T ceiling(T target);
 }
```

**`app/src/sorteddata/avltree/AVLNodeEmpty.java`** (changes only: `+` = add this line, `-` = remove this line)

```diff
@@ -38,4 +38,15 @@
 	public T get(T element) {
 		return null;
 	}
+
+	// HACKATHON: deleting from an empty tree changes nothing
+	public AVLNode<T> delete(T element) {
+		return this;
+	}
+
+	public int rank(T element) { return 0; }
+
+	public T floor(T target) { return null; }
+
+	public T ceiling(T target) { return null; }
 }
```

**`app/src/sorteddata/avltree/AVLNodeFilled.java`** (changes only: `+` = add this line, `-` = remove this line)

```diff
@@ -102,4 +102,85 @@
 		}
 		return value;
 	}
+
+	// ---------------------------------------------------------------------
+	// HACKATHON: immutable AVL delete
+	// NOTE: in THIS codebase balance = left.height - right.height
+	//       (positive = left-heavy). If your base uses right - left, flip the signs.
+	// ---------------------------------------------------------------------
+	public AVLNode<T> delete(T element) {
+		int cmp = comparator.compare(element, value);
+		AVLNodeFilled<T> result;
+		if (cmp < 0) {
+			// target is in the left subtree: rebuild this node with the new left side
+			result = new AVLNodeFilled<>(comparator, value, left.delete(element), right);
+		} else if (cmp > 0) {
+			// target is in the right subtree
+			result = new AVLNodeFilled<>(comparator, value, left, right.delete(element));
+		} else {
+			// found it. Case 1 and 2: zero or one child -> the child replaces this node
+			if (left instanceof AVLNodeEmpty<T>) return right;
+			if (right instanceof AVLNodeEmpty<T>) return left;
+			// Case 3: two children -> copy the in-order successor (smallest on the right)
+			// into this position, then delete the successor from the right subtree
+			T successor = ((AVLNodeFilled<T>) right).minValue();
+			result = new AVLNodeFilled<>(comparator, successor, left, right.delete(successor));
+		}
+		return result.rebalance();
+	}
+
+	/** @return the smallest value in this subtree (keep going left) */
+	private T minValue() {
+		AVLNodeFilled<T> current = this;
+		while (current.left instanceof AVLNodeFilled<T> next) current = next;
+		return current.value;
+	}
+
+	/**
+	 * Restores the AVL property at this node after a delete.
+	 * Same four cases as insertion (LL, LR, RR, RL), but a delete can leave a
+	 * child with balance 0, which is why we test "< 0" / "> 0" (not "<= 0").
+	 */
+	private AVLNodeFilled<T> rebalance() {
+		if (balance > 1) { // left-heavy
+			AVLNodeFilled<T> l = (AVLNodeFilled<T>) left;
+			AVLNodeFilled<T> node = this;
+			if (l.balance < 0) // left-right case: rotate the child first
+				node = new AVLNodeFilled<>(comparator, value, l.leftRotate(), right);
+			return node.rightRotate();
+		}
+		if (balance < -1) { // right-heavy
+			AVLNodeFilled<T> r = (AVLNodeFilled<T>) right;
+			AVLNodeFilled<T> node = this;
+			if (r.balance > 0) // right-left case
+				node = new AVLNodeFilled<>(comparator, value, left, r.rightRotate());
+			return node.leftRotate();
+		}
+		return this; // already balanced
+	}
+
+	// ---------------------------------------------------------------------
+	// HACKATHON: order statistics. Each node stores size, so we can count
+	// "how many are smaller" on the way down without visiting every node.
+	// ---------------------------------------------------------------------
+	public int rank(T element) {
+		if (comparator.compare(element, value) <= 0) return left.rank(element);   // everything here is in the left subtree
+		return left.size() + 1 + right.rank(element);                              // whole left side + this node + part of the right
+	}
+
+	public T floor(T target) {
+		int cmp = comparator.compare(target, value);
+		if (cmp == 0) return value;
+		if (cmp < 0) return left.floor(target);
+		T better = right.floor(target);          // maybe something on the right is still <= target
+		return better != null ? better : value;
+	}
+
+	public T ceiling(T target) {
+		int cmp = comparator.compare(target, value);
+		if (cmp == 0) return value;
+		if (cmp > 0) return right.ceiling(target);
+		T better = left.ceiling(target);
+		return better != null ? better : value;
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

**`app/src/sorteddata/sortedarraylist/SortedArrayList.java`** (changes only: `+` = add this line, `-` = remove this line)

```diff
@@ -32,6 +32,7 @@
 		if (i < list.size() && comparator.compare(list.get(i), value) == 0)
 			return false;
 		list.add(i, value);
+		notifyAdded(value); // HACKATHON: tell listeners (Observer)
 		return true;
 	}
 
@@ -43,8 +44,29 @@
 
 	@Override
 	public Iterator<T> getRange(T start, int count, boolean backwards) {
-		if (backwards)
-			throw new IllegalArgumentException("SortedArrayList does not yet support backwards iteration");
+		// HACKATHON: backwards iteration (was "does not yet support backwards iteration").
+		// Start at the last element <= start (floor) and walk towards index 0. O(log n) to find the start.
+		if (backwards) {
+			int i = start == null ? list.size() - 1 : binarySearch(start);
+			if (start != null && (i >= list.size() || comparator.compare(list.get(i), start) > 0)) i--; // step back to the floor
+			int first = i;
+			return new Iterator<>() {
+				private int index = first;
+				private int remaining = count;
+
+				@Override
+				public boolean hasNext() {
+					return index >= 0 && remaining != 0;
+				}
+
+				@Override
+				public T next() {
+					if (!hasNext()) throw new NoSuchElementException();
+					if (remaining > 0) remaining--;
+					return list.get(index--);
+				}
+			};
+		}
 		return new SortedArrayListIterator<>(list, comparator, start, count);
 	}
 
@@ -60,8 +82,9 @@
 
 	private static final Random random = new Random();
 
+	// HACKATHON: return null out of range (like AVLTree) instead of throwing
 	public T getAtIndex(int i) {
-		return list.get(i);
+		return i >= 0 && i < list.size() ? list.get(i) : null;
 	}
 
 	@Override
@@ -73,4 +96,39 @@
 	public Iterator<T> getAll() {
 		return new SortedArrayListIterator<>(list, comparator, null, -1);
 	}
+
+	// HACKATHON: O(log n) search + O(n) shift, same cost profile as insert
+	@Override
+	public boolean remove(T value) {
+		int i = binarySearch(value);
+		if (i < list.size() && comparator.compare(list.get(i), value) == 0) {
+			list.remove(i);
+			return true;
+		}
+		return false;
+	}
+
+	@Override
+	public int size() {
+		return list.size();
+	}
+
+	// HACKATHON: binarySearch already returns "how many are smaller"
+	@Override
+	public int rank(T value) {
+		return binarySearch(value);
+	}
+
+	@Override
+	public T floor(T value) {
+		int i = binarySearch(value);
+		if (i < list.size() && comparator.compare(list.get(i), value) == 0) return list.get(i);
+		return i == 0 ? null : list.get(i - 1);
+	}
+
+	@Override
+	public T ceiling(T value) {
+		int i = binarySearch(value);
+		return i < list.size() ? list.get(i) : null;
+	}
 }
```

`SortedData` gets default `remove`/`rank`/`floor`/`ceiling` that throw, plus abstract `size()` (diff in Task1prediction.md 3.2).
The factory switches every DAO and Post to the AVL tree:

**`app/src/sorteddata/SortedDataFactory.java`** (full file)

```java
package sorteddata;

import java.util.Comparator;

/**
 * HACKATHON: the practice base shipped with SortedArrayList here (O(n) inserts).
 * Switching this one line to AVLTree makes every DAO and every Post use the
 * O(log n) tree, and AVLTree is the one that supports remove() in O(log n).
 */
public class SortedDataFactory {
	public static <T> SortedData<T> makeSortedData(Comparator<T> comparator) {
		return new sorteddata.avltree.AVLTree<>(comparator);
	}
}
```

## 8. Order statistics: rank / floor / ceiling

Each node stores `size`, so "how many are smaller than x" adds up left-subtree sizes on the way down: O(log n).
`rank(x)` is also the index `x` has (or would have). The AVL and array-list code is in the diffs above.

## 9. Time ranges, counts, position, pages

The trick: `new Message(null, null, null, t, null)` is a **probe** that sorts before every real message at time `t`
(thanks to the null-safe comparator), so the normal tree search finds "first message at or after t".

**`app/src/sorteddata/SortedDataRanges.java`** (full file)

```java
package sorteddata;

import java.util.Comparator;
import java.util.Iterator;
import java.util.NoSuchElementException;

/**
 * HACKATHON (Task 2 prediction): range queries on any SortedData.
 */
public final class SortedDataRanges {
	private SortedDataRanges() {}

	/**
	 * Elements x with  from <= x < toExclusive, in order. Lazily stops at the first
	 * element past the range, so it costs O(log n + k) for k results on an AVL tree.
	 * @param toExclusive null means "no upper bound"
	 */
	public static <T> Iterator<T> range(SortedData<T> data, T from, T toExclusive, Comparator<T> comparator) {
		Iterator<T> source = data.getRange(from, -1, false); // starts at the first element >= from
		return new Iterator<>() {
			private T next = advance();

			private T advance() {
				if (!source.hasNext()) return null;
				T candidate = source.next();
				if (toExclusive != null && comparator.compare(candidate, toExclusive) >= 0) return null;
				return candidate;
			}

			@Override
			public boolean hasNext() {
				return next != null;
			}

			@Override
			public T next() {
				if (next == null) throw new NoSuchElementException();
				T result = next;
				next = advance();
				return result;
			}
		};
	}

	/** Number of elements with from <= x < toExclusive, in O(log n) using rank. */
	public static <T> int count(SortedData<T> data, T from, T toExclusive) {
		int upper = toExclusive == null ? data.size() : data.rank(toExclusive);
		return Math.max(0, upper - data.rank(from));
	}
}
```

**`app/src/queries/MessageQueries.java`** (full file)

```java
package queries;

import dao.MessageComparator;
import dao.MessageIndex;
import dao.PostDAO;
import dao.model.Message;
import dao.model.Post;
import sorteddata.SortedData;
import sorteddata.SortedDataRanges;

import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;
import java.util.UUID;

/**
 * PREDICTED Task 2 queries on a post's messages: by time range, position, pages.
 * <p>
 * Trick: MessageComparator orders by timestamp first and (after our null-safe fix) puts null
 * fields first, so new Message(null, null, null, t, null) is a "probe" that sorts BEFORE every
 * real message with timestamp t. That lets us search by time with the normal tree methods.
 */
public final class MessageQueries {
	private MessageQueries() {}

	/** A fake message that sorts before every real message with this timestamp. */
	static Message probe(long timestamp) {
		return new Message(null, null, null, timestamp, null);
	}

	// probe just after every message with timestamp t (null = no upper bound)
	private static Message probeAfter(long timestamp) {
		return timestamp == Long.MAX_VALUE ? null : probe(timestamp + 1);
	}

	/** Messages of a post with from <= timestamp <= to (inclusive), oldest first. */
	public static Iterator<Message> getMessagesBetween(UUID post, long from, long to, boolean isAdmin) {
		Post target = PostDAO.getInstance().get(new Post(post));
		if (target == null || from > to) return new ArrayList<Message>().iterator();
		return SortedDataRanges.range(target.getVisibleMessages(isAdmin), probe(from), probeAfter(to), MessageComparator.getInstance());
	}

	/** How many messages have from <= timestamp <= to, in O(log n). */
	public static int countMessagesBetween(UUID post, long from, long to, boolean isAdmin) {
		Post target = PostDAO.getInstance().get(new Post(post));
		if (target == null || from > to) return 0;
		return SortedDataRanges.count(target.getVisibleMessages(isAdmin), probe(from), probeAfter(to));
	}

	/** @return 0-based position of the message in its post (oldest = 0), or -1 if unknown. O(log n). */
	public static int getPosition(UUID message) {
		Message target = MessageIndex.getInstance().get(message);
		if (target == null) return -1;
		return MessageIndex.getInstance().getPostOf(target).messages.rank(target);
	}

	/**
	 * Pagination: page `page` (0-based) of `pageSize` messages, oldest first.
	 * Uses getAtIndex (O(log n) each) so earlier pages are never walked.
	 */
	public static List<Message> getPage(UUID post, int page, int pageSize, boolean isAdmin) {
		if (page < 0 || pageSize <= 0) throw new IllegalArgumentException("page must be >= 0 and pageSize > 0");
		List<Message> result = new ArrayList<>();
		Post target = PostDAO.getInstance().get(new Post(post));
		if (target == null) return result;
		SortedData<Message> source = target.getVisibleMessages(isAdmin);
		int start = page * pageSize;
		for (int i = start; i < Math.min(start + pageSize, source.size()); i++) result.add(source.getAtIndex(i));
		return result;
	}
}
```

## 10. `BSTree.getRange` (was `return null`)

**`app/src/sorteddata/bstree/BSIterator.java`** (full file)

```java
package sorteddata.bstree;

import java.util.Comparator;
import java.util.Iterator;
import java.util.NoSuchElementException;
import java.util.Stack;

/**
 * HACKATHON (Task 2 prediction): BSTree.getRange used to return null.
 * Same algorithm as AVLIterator: a stack holds the path still to visit.
 *  - forwards: start at the first element >= begin (or the smallest if begin == null)
 *  - backwards: start at the last element <= begin (or the largest if begin == null)
 *  - stop after `count` elements (negative = unlimited)
 */
class BSIterator<T> implements Iterator<T> {
	private final Stack<BSNodeFilled<T>> stack = new Stack<>();
	private final boolean backwards;
	private int remaining;

	BSIterator(T begin, BSNode<T> root, Comparator<T> comparator, int count, boolean backwards) {
		this.backwards = backwards;
		this.remaining = count;
		BSNode<T> current = root;
		while (current instanceof BSNodeFilled<T> node) {
			int cmp = begin == null ? (backwards ? 1 : -1) : comparator.compare(begin, node.value);
			if (cmp == 0) {               // exact match: it is the first result
				stack.push(node);
				break;
			}
			boolean goLeft = cmp < 0;
			// a node is a future result if it lies on the side we are moving towards
			if (goLeft != backwards) stack.push(node);
			current = goLeft ? node.left : node.right;
		}
	}

	@Override
	public boolean hasNext() {
		return remaining != 0 && !stack.isEmpty();
	}

	@Override
	public T next() {
		if (!hasNext()) throw new NoSuchElementException();
		if (remaining > 0) remaining--;
		BSNodeFilled<T> node = stack.pop();
		// the next result is the extreme node of the subtree on the "moving" side
		BSNode<T> child = backwards ? node.left : node.right;
		while (child instanceof BSNodeFilled<T> filled) {
			stack.push(filled);
			child = backwards ? filled.right : filled.left;
		}
		return node.value;
	}
}
```

**`app/src/sorteddata/bstree/BSTree.java`** (changes only: `+` = add this line, `-` = remove this line)

```diff
@@ -25,6 +25,7 @@
 	public boolean insert(T element) {
 		if (root.contains(element)) return false;
 		root = root.insert(element);
+		notifyAdded(element); // HACKATHON: tell listeners (Observer)
 		return true;
 	}
 
@@ -46,7 +47,13 @@
 		return root.getAtIndex(random.nextInt(root.size()));
 	}
 
+	// HACKATHON: was "return null"
 	public Iterator<T> getRange(T start, int count, boolean backwards) {
-		return null;
+		return new BSIterator<>(start, root, comparator, count, backwards);
+	}
+
+	@Override
+	public int size() {
+		return root.size();
 	}
 }
```

## 11. Red-black tree (left-leaning)

Only if asked to "implement a red-black tree" (the practice readme names it as an example). `isValid()` checks all
three rules; 500,000 inserts in increasing order stay valid and take well under 3 seconds.

**`app/src/sorteddata/redblack/RedBlackTree.java`** (full file)

```java
package sorteddata.redblack;

import sorteddata.SortedData;

import java.util.Comparator;
import java.util.Iterator;
import java.util.NoSuchElementException;
import java.util.Random;
import java.util.Stack;

/**
 * HACKATHON (Task 2 prediction, "implement a red-black tree"): a LEFT-LEANING red-black tree
 * (Sedgewick). It is the easiest correct red-black variant: insert is 3 cases on the way back up.
 * Rules it keeps:
 *  1. links are red or black; red links only lean LEFT
 *  2. no node has two red links in a row
 *  3. every path from the root to an empty link has the same number of black links
 * => height <= 2 log2(n), so every operation is O(log n), like AVL.
 * Each node also stores its subtree size, for getAtIndex / rank in O(log n).
 * (Mutable, unlike the AVL tree. Delete is not implemented: remove() throws, like SortedData's default.)
 */
public class RedBlackTree<T> extends SortedData<T> {
	private static final boolean RED = true;
	private static final boolean BLACK = false;
	private static final Random random = new Random();

	private static final class Node<T> {
		T value;
		Node<T> left, right;
		boolean color = RED; // the link from the parent; new nodes are always red
		int size = 1;

		Node(T value) {
			this.value = value;
		}
	}

	private final Comparator<T> comparator;
	private Node<T> root;

	public RedBlackTree(Comparator<T> comparator) {
		this.comparator = comparator;
	}

	// ------------------------------------------------------------ helpers

	private static boolean isRed(Node<?> node) {
		return node != null && node.color == RED;
	}

	private static int size(Node<?> node) {
		return node == null ? 0 : node.size;
	}

	// right-leaning red link -> lean it left
	private Node<T> rotateLeft(Node<T> h) {
		Node<T> x = h.right;
		h.right = x.left;
		x.left = h;
		x.color = h.color;
		h.color = RED;
		x.size = h.size;
		h.size = 1 + size(h.left) + size(h.right);
		return x;
	}

	// two left reds in a row -> rotate right
	private Node<T> rotateRight(Node<T> h) {
		Node<T> x = h.left;
		h.left = x.right;
		x.right = h;
		x.color = h.color;
		h.color = RED;
		x.size = h.size;
		h.size = 1 + size(h.left) + size(h.right);
		return x;
	}

	// both children red -> push the red up (this is the 2-3 tree "split")
	private void flipColors(Node<T> h) {
		h.color = RED;
		h.left.color = BLACK;
		h.right.color = BLACK;
	}

	// ------------------------------------------------------------ SortedData

	@Override
	public boolean insert(T value) {
		if (get(value) != null) return false; // no duplicates
		root = insert(root, value);
		root.color = BLACK;
		notifyAdded(value);
		return true;
	}

	private Node<T> insert(Node<T> h, T value) {
		if (h == null) return new Node<>(value);
		if (comparator.compare(value, h.value) < 0) h.left = insert(h.left, value);
		else h.right = insert(h.right, value);

		// fix-up on the way back up: the three cases, in this order
		if (isRed(h.right) && !isRed(h.left)) h = rotateLeft(h);
		if (isRed(h.left) && isRed(h.left.left)) h = rotateRight(h);
		if (isRed(h.left) && isRed(h.right)) flipColors(h);
		h.size = 1 + size(h.left) + size(h.right);
		return h;
	}

	@Override
	public T get(T value) {
		Node<T> node = root;
		while (node != null) {
			int cmp = comparator.compare(value, node.value);
			if (cmp == 0) return node.value;
			node = cmp < 0 ? node.left : node.right;
		}
		return null;
	}

	@Override
	public T getAtIndex(int i) {
		if (i < 0 || i >= size()) return null;
		Node<T> node = root;
		while (true) {
			int leftSize = size(node.left);
			if (i < leftSize) node = node.left;
			else if (i == leftSize) return node.value;
			else {
				i -= leftSize + 1;
				node = node.right;
			}
		}
	}

	@Override
	public int rank(T value) {
		int result = 0;
		Node<T> node = root;
		while (node != null) {
			if (comparator.compare(value, node.value) <= 0) node = node.left;
			else {
				result += size(node.left) + 1;
				node = node.right;
			}
		}
		return result;
	}

	@Override
	public T floor(T target) {
		T best = null;
		Node<T> node = root;
		while (node != null) {
			int cmp = comparator.compare(target, node.value);
			if (cmp == 0) return node.value;
			if (cmp < 0) node = node.left;
			else {
				best = node.value; // candidate; maybe something bigger on the right is still <= target
				node = node.right;
			}
		}
		return best;
	}

	@Override
	public T ceiling(T target) {
		T best = null;
		Node<T> node = root;
		while (node != null) {
			int cmp = comparator.compare(target, node.value);
			if (cmp == 0) return node.value;
			if (cmp > 0) node = node.right;
			else {
				best = node.value;
				node = node.left;
			}
		}
		return best;
	}

	@Override
	public T getRandom() {
		return size() == 0 ? null : getAtIndex(random.nextInt(size()));
	}

	@Override
	public int size() {
		return size(root);
	}

	/** Same contract as AVLTree.getRange (stack-based in-order walk). */
	@Override
	public Iterator<T> getRange(T start, int count, boolean backwards) {
		Stack<Node<T>> stack = new Stack<>();
		Node<T> current = root;
		while (current != null) {
			int cmp = start == null ? (backwards ? 1 : -1) : comparator.compare(start, current.value);
			if (cmp == 0) {
				stack.push(current);
				break;
			}
			boolean goLeft = cmp < 0;
			if (goLeft != backwards) stack.push(current);
			current = goLeft ? current.left : current.right;
		}
		return new Iterator<>() {
			private int remaining = count;

			@Override
			public boolean hasNext() {
				return remaining != 0 && !stack.isEmpty();
			}

			@Override
			public T next() {
				if (!hasNext()) throw new NoSuchElementException();
				if (remaining > 0) remaining--;
				Node<T> node = stack.pop();
				Node<T> child = backwards ? node.left : node.right;
				while (child != null) {
					stack.push(child);
					child = backwards ? child.right : child.left;
				}
				return node.value;
			}
		};
	}

	// ------------------------------------------------------------ invariant check (for tests)

	/** @return true if all three red-black rules and the size fields hold */
	public boolean isValid() {
		return !isRed(root) && blackHeight(root) >= 0;
	}

	// returns the black height, or -1 if any rule is broken below this node
	private int blackHeight(Node<T> node) {
		if (node == null) return 0;
		if (isRed(node.right)) return -1;                       // rule 1: no right-leaning red
		if (isRed(node) && isRed(node.left)) return -1;          // rule 2: no two reds in a row
		if (node.size != 1 + size(node.left) + size(node.right)) return -1;
		int l = blackHeight(node.left), r = blackHeight(node.right);
		if (l < 0 || r < 0 || l != r) return -1;                 // rule 3: perfect black balance
		return l + (isRed(node) ? 0 : 1);
	}
}
```

## 12. Move a message to another post

**`app/src/content/MoveTools.java`** (full file)

```java
package content;

import dao.MessageIndex;
import dao.PostDAO;
import dao.model.Message;
import dao.model.Post;

import java.util.UUID;

/**
 * PREDICTED theme: move a message to another post (admin).
 * Message is immutable and its post id (thread) is part of the record, so moving =
 * remove from the old post + insert a copy with the new thread. Same id, so reports,
 * reactions, votes and edit history (all keyed by message id) still apply.
 */
public final class MoveTools {
	private MoveTools() {}

	/** @return false if message/post missing, not admin, or already in that post */
	public static boolean moveMessage(UUID message, UUID targetPost, UUID admin) {
		Message current = MessageIndex.getInstance().get(message);
		Post target = targetPost == null ? null : PostDAO.getInstance().get(new Post(targetPost));
		if (current == null || target == null || !ContentTools.isAdmin(admin) || targetPost.equals(current.thread())) return false;
		Post source = MessageIndex.getInstance().getPostOf(current);
		boolean wasHidden = source.isHidden(message);
		source.removeMessage(current);
		Message moved = new Message(current.id(), current.poster(), targetPost, current.timestamp(), current.message());
		target.messages.insert(moved);
		if (wasHidden) target.setHidden(moved, true); // keep moderation state
		return true;
	}
}
```

## 13. Trap checklist (Task 2)

- [ ] Admin check uses the stored user's role (`UserDAO.getByUUID(user).role()`), not `StateManager`.
- [ ] Return false **before** changing anything when a check fails.
- [ ] Views must reflect later changes (new messages are visible by default; the listener handles it).
- [ ] Hidden + deleted combine: a message is visible only when it is neither.
- [ ] Editing a record: remove the old one **first** (the comparator ignores the text, so insert would see a duplicate).
- [ ] If you touch the tree, rerun `AVLTreeTests` and `SortedDataEfficiencyTests`.
- [ ] Keep `remove`/`rank` defaults throwing in `SortedData` so classes without them still compile.

## 14. How this was verified

| Check | Result |
|---|---|
| AVL delete: 200 rounds x 300 random operations vs `TreeSet`, invariants after every operation | pass |
| rank / floor / ceiling / range / count on AVL, array list, red-black: 100 rounds x 40 random queries vs `TreeSet` | pass |
| getRange forwards/backwards, null start, counts -1..10 on AVL, red-black, BST and sorted array list | pass |
| red-black: 500,000 sorted inserts, `isValid()` | pass, under 3 s |
| soft delete (author / admin / other member / unknown), restore, hidden + deleted together | pass |
| hard delete, lock + `reply` + `MemberState`/`AdminState`, banned users cannot reply | pass |
| pinned first (hidden pins only for admins), replace keeps flags and order | pass |
| time range (inclusive), count O(log n) incl. `Long.MIN/MAX`, position, partial last page, bad page throws | pass |
| move keeps hidden flag and reports | pass |
