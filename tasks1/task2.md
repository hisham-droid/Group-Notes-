I'll check how the sorted structures insert and iterate, then build Task 2 on top of the Task 1 code and test it.

I built Task 2 on top of the Task 1 code. It compiles, and 13 new checks pass (Task 1's checks still pass too).

# Task 2: Hiding messages

## What it is

Admins can hide a bad message so normal users stop seeing it. Admins can still see it, and can unhide it later. There are two parts:

**Part A: `ModerationTools.setHidden(message, user, hidden)`**

- Return `false` and do nothing if the message doesn't exist, the user doesn't exist, or the user isn't an **Admin**.
- Otherwise hide the message (`hidden = true`) or unhide it (`hidden = false`) and return `true`.
- New messages start visible.

**Part B: `Post.getVisibleMessages(isAdmin)`**

- Admins (`isAdmin = true`) get **all** of the post's messages.
- Everyone else (`isAdmin = false`) gets only the **non-hidden** messages.
- It returns a `SortedData<Message>`, still in time order like normal.

## The design (the thinking part)

**The big question: where does "hidden" live?** There are three options.

1. **Add a `hidden` field to `Message`.** This doesn't work well. `Message` is a **record**, and records can't be changed after they're created. To flip the flag you'd have to swap in a new `Message`, but `SortedData` has no remove method. You'd also have to change every `new Message(...)` call and the CSV format.
2. **One global set of hidden message IDs** (for example in a singleton). This works, but then `Post` has to reach out to a moderation singleton to answer "which of my messages are visible?" That ties the model to the moderation code.
3. **Each `Post` keeps a `HashSet<UUID>` of its own hidden messages.** ✅ The post owns its messages, so it should own which of them are hidden. `getVisibleMessages` lives in `Post`, so everything it needs is right there.

We go with **option 3**.

**Why a `HashSet`?** "Is this message hidden?" gets asked once per message every time a non-admin views a post. A `HashSet` answers in O(1). Hiding (`add`) and unhiding (`remove`) are O(1) too.

**Why store UUIDs, not `Message` objects?** UUIDs are what `setHidden` receives, they're small, and Task 3 can save them straight to a file.

**How does `getVisibleMessages(false)` build its result?** It walks through all the post's messages in order and copies the non-hidden ones into a **new** `SortedData`, made with the same `SortedDataFactory` and comparator, so it's still sorted by time. It never touches the original `messages`, so hidden messages aren't lost.

**One shortcut:** if nothing in the post is hidden (the usual case), it returns `messages` directly instead of copying. That's O(1).

**Reports aren't affected.** Task 1 keeps reports in `ReportStore`, completely separate from this. Hiding a message never touches its reports, which is what the spec requires.

## The code

### `dao/model/Post.java` (full updated file)

```java
package dao.model;

import dao.MessageComparator;
import sorteddata.SortedData;
import sorteddata.SortedDataFactory;

import java.util.Collections;
import java.util.HashSet;
import java.util.Iterator;
import java.util.Set;
import java.util.UUID;

public class Post implements HasUUID {
	public final UUID id;
	public final UUID poster;
	public final String topic;
	public final SortedData<Message> messages;
	private final Set<UUID> hiddenMessageIds = new HashSet<>();

	public Post(UUID id, UUID poster, String topic) {
		this.id = id;
		this.poster = poster;
		this.topic = topic;
		this.messages = SortedDataFactory.makeSortedData(MessageComparator.getInstance());
	}

	public Post(UUID id) {
		this(id, null, null);
	}

	/**
	 * Hides or unhides one of this post's messages.
	 * @param messageId the UUID of the message
	 * @param hidden true to hide the message, false to make it visible again
	 */
	public void setHidden(UUID messageId, boolean hidden) {
		if (hidden) hiddenMessageIds.add(messageId);
		else hiddenMessageIds.remove(messageId);
	}

	/** @return true if the given message has been hidden by a moderator */
	public boolean isHidden(UUID messageId) {
		return hiddenMessageIds.contains(messageId);
	}

	/** @return a read-only view of the UUIDs of this post's hidden messages */
	public Set<UUID> getHiddenMessageIds() {
		return Collections.unmodifiableSet(hiddenMessageIds);
	}

	/**
	 * Gets the messages that a viewer is allowed to see.
	 * @param isAdmin whether the viewer is an Admin
	 * @return every message if isAdmin is true; otherwise only the messages that are not hidden
	 */
	public SortedData<Message> getVisibleMessages(boolean isAdmin) {
		if (isAdmin || hiddenMessageIds.isEmpty()) return messages;

		SortedData<Message> visible = SortedDataFactory.makeSortedData(MessageComparator.getInstance());
		for (Iterator<Message> it = messages.getAll(); it.hasNext(); ) {
			Message message = it.next();
			if (!isHidden(message.id())) visible.insert(message);
		}
		return visible;
	}

	public UUID getUUID() { return id; }
}
```

- `hiddenMessageIds` is **private**, so only `Post` can change it, through `setHidden`. That's encapsulation, and it's good for code-quality marks.
- `getHiddenMessageIds()` returns a **read-only** view. Task 3 can read it to save to disk, but can't accidentally modify it.

### `moderation/ModerationTools.java` (add `setHidden` and two imports)

Add these imports next to the ones from Task 1:

```java
import dao.model.Post;
import dao.model.User;
```

Replace the `setHidden` stub:

```java
	/**
	 * Hides or unhides a message. Only Admins may do this.
	 * @return true if the message's state was updated; false if the message or
	 * user does not exist, or the user is not an Admin
	 */
	public static boolean setHidden(UUID message, UUID user, boolean hidden) {
		User moderator = UserDAO.getInstance().getByUUID(user);
		if (moderator == null || moderator.role() != User.Role.Admin) return false;

		Message target = PostDAO.getInstance().getMessageByUUID(message);
		if (target == null) return false;

		Post post = PostDAO.getInstance().get(new Post(target.thread()));
		post.setHidden(message, hidden);
		return true;
	}
```

This uses `PostDAO.getMessageByUUID`, which you already added in Task 1.

## How `setHidden` works, step by step

1. **Find the user and check they're an Admin.** `getByUUID` returns `null` for an unknown user, and `||` stops early, so `moderator.role()` is never called on `null`. This check comes first because it's cheaper than searching every message.
2. **Find the message.** If it doesn't exist, return `false`.
3. **Find the message's post.** Every `Message` has a `thread()` field, which is the UUID of the post it belongs to. `PostDAO` compares posts by UUID only, so `new Post(target.thread())` works as a "search key." It's a blank post with just the ID filled in, and `get` returns the real post with that ID. That's the same trick `DataManager.readAll()` already uses.
4. **Tell the post to hide or unhide it** and return `true`.

Hiding an already hidden message (or unhiding a visible one) still returns `true`. The spec says to return `false` only when the checks fail. After the checks pass, it just says "update the state," and the state ends up correct either way.

## Performance

| Operation | Cost |
|---|---|
| `Post.setHidden` / `isHidden` | O(1) (HashSet) |
| `getVisibleMessages(true)` (admin) | O(1): returns the existing structure |
| `getVisibleMessages(false)` with nothing hidden | O(1): same shortcut |
| `getVisibleMessages(false)` with something hidden | O(n log n): copies the n messages into a new sorted structure |
| `ModerationTools.setHidden` | dominated by the existing user and message lookups |

## How this connects to the other tasks

- **Task 1:** no changes needed. Reports and hiding are stored separately, so "reports stay even if the message is hidden" is automatic.
- **Task 3 (saving to disk):** for each post, save `post.getHiddenMessageIds()`. A simple portable format is a CSV file with one message UUID per line. When loading, after posts and messages are loaded, look up each message's post and call `post.setHidden(id, true)`. Saving only the hidden IDs (not a flag for every message) keeps the file small, and it means the existing messages CSV doesn't need to change.
- **Task 4:** no connection. Hidden messages can still show up in the reported list, which makes sense, since moderators need to see what they've hidden.
- **Task 5:** nothing required for Task 2 (the tests are for `addReport` and `getReportedMessages`).

## The "trees" angle, if your group wants to go further

The task title says *data structures, trees*. The solution above is correct and simple, but every non-admin view copies the list when something is hidden. A faster design keeps a **second** sorted structure of visible messages inside `Post`, updated as you go:

- Hiding removes the message from the visible structure, and unhiding inserts it back. With an AVL tree, that's O(log n).
- `getVisibleMessages(false)` then just returns the visible structure: O(1) every time.

The catch is that `SortedData` has **no remove method**. You'd have to add `remove` to `SortedData` and implement it in `AVLTree` (AVL deletion with rebalancing), `BSTree` and `SortedArrayList`. You'd also have to make `messages` private with an `addMessage()` method, so every new message goes into both structures. That's a much bigger job. If marks reward tree work, it could be worth it. Otherwise the version above fully meets the spec.

## One thing to confirm with your group

Right now `messages` is still a `public` field, so code elsewhere could hand non-admins `post.messages` directly and skip the hidden check. Anywhere your app displays messages to users should go through `getVisibleMessages(isAdmin)` instead.
