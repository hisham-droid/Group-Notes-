Confirmed: the file reader has a hidden date check, so loading has silently failed since 1 Dec 2025. I'll fix that as part of Task 3, then build and test a full save → wipe → load round trip.

Earlier checks still pass. Now the real test: save everything, wipe memory, load it back, and compare. I'm including tricky message text (commas, quotes, newlines) and empty files.

Task 3 is built on top of your Task 1 and Task 2 code, and tested. I saved about 25,000 messages, 501 reports and 21 hidden messages, wiped everything from memory, and loaded it back. Everything came back exactly, including a message containing commas, quotes and a line break.

# Task 3: Saving reports and hidden messages

## What it is

When the app closes and reopens, reports (Task 1) and hidden messages (Task 2) must still be there. Right now they only live in memory, so they vanish. You need to:

1. **(60%)** Save and load them using the project's existing save/load system (`persistentdata`), in a **portable** format. Other programs in other languages will read these files, so no Java-only formats like Java serialization.
2. **(40%)** Make the code **very clean**. The spec explicitly says to look for code smells and refactor them, and that includes smells in the existing persistence code.

## First: the existing loading code is broken (twice)

Before writing anything new, I found two bugs that stop saved data from loading **at all**. Fixing them is part of making persistence work, and it's exactly what the "check for code smells" marks are about.

**Bug 1: a hidden date check in `ComputerIOFactory.reader`.**

```java
// Weird trick for some Linux distributions. No effect on Windows/Mac/other flavours of Linux
if (System.currentTimeMillis() > 1764620398492L) {
    throw new IOException("Incompatible operating system detected.");
}
```

The comment lies. That number is a timestamp: **1 December 2025**. After that date, every read throws, the exception is silently swallowed, and the method returns `null`. Nothing ever loads, and you get no error message. It's a planted time bomb. Delete it.

**Bug 2: `CSVReader.hasNext()` always returns `true`.** The loading loop is `while (hasNext()) getNext()`, so after the last row it calls `getNext()` again and crashes with "Already reached end of file." An empty file (for example, no reports yet) crashes immediately. The fix is to peek one character ahead: if there's nothing left, return `false`.

## The design (the thinking part)

**Format: CSV.** The project already saves users, posts and messages as CSV. CSV is plain text, so any language (Python, JavaScript, Excel…) can read it. It's portable and consistent with the existing code.

**Two new files, one row per item:**

`saved/reports.txt`: one row per active report

| message UUID | user UUID | timestamp |
|---|---|---|
| `730c5f33-…` | `49792a43-…` | `677` |

`saved/hidden_messages.txt`: one row per hidden message

| post UUID | message UUID |
|---|---|
| `969c436d-…` | `72800c2c-…` |

**Why separate files instead of adding a "hidden" column to `messages.txt`?**

- **It doesn't break other programs.** Other apps already read `messages.txt` expecting 5 columns. Adding a 6th would break them. New files leave the old ones untouched.
- **Only hidden messages are stored.** Out of 25,000 messages, maybe 20 are hidden, so there's no point writing "false" 24,980 times.
- **Each file has one job.** That's good design.

**Why store the post UUID in the hidden file?** When loading, we need to find the message's post to call `post.setHidden(...)`. With the post UUID in the row, that's one quick lookup. Without it, we'd have to search all 25,000 messages for each hidden one.

**Load order matters.** Reports and hidden messages refer to messages, and messages belong to posts. So the load order is users → posts → messages → hidden messages → reports.

**Reports are loaded through `ModerationTools.addReport`.** This reuses all of Task 1's checks. If a saved report points to a user or message that no longer exists, it's skipped automatically instead of corrupting the data. No duplicate validation code.

**How it fits the existing system.** The project already has a clean structure:

```
Serializer        object  ⇄ String[] row      (e.g. UserSerializer)
FormattedFactory  row     ⇄ CSV text          (CSVReader / CSVWriter)
IOFactory         text    ⇄ file on disk      (ComputerIOFactory)
DataPipeline      glues those three together for one file
DataManager       runs all the pipelines: readAll() / writeAll()
```

So adding a new saved type means writing **one Serializer** and **one pipeline line** in `DataManager`. That's the point of the design.

## The code

### Small change to Task 1: `Report` now knows its message

To save a report as one row, the report needs to contain the message ID.

**`moderation/Report.java`**

```java
package moderation;

import java.util.UUID;

/**
 * A single report: one user flagging one message at a point in time.
 */
public record Report(UUID message, UUID user, long timestamp) {}
```

**`moderation/MessageReports.java`:** change one line in `add`:

```java
		return reportsByUser.putIfAbsent(user, new Report(message.id(), user, timestamp)) == null;
```

**`moderation/ReportStore.java`:** add `import java.util.Iterator;` and this method (above `clear()`):

```java
	/** @return every active report, across all messages, in no particular order */
	public Iterator<Report> getAllReports() {
		return reportsByMessage.values().stream()
				.flatMap(reports -> reports.getReports().stream())
				.iterator();
	}
```

`flatMap` turns "a collection of collections of reports" into one flat stream of reports. `.iterator()` gives it the `Iterator` shape the pipeline expects.

### New: `dao/model/HiddenMessage.java`

```java
package dao.model;

import java.util.UUID;

/**
 * Records that a message has been hidden by a moderator. The post is stored
 * alongside the message so the message can be located without searching every post.
 */
public record HiddenMessage(UUID post, UUID message) {}
```

### `dao/PostDAO.java`: add a method and imports

Imports to add:

```java
import dao.model.HiddenMessage;
import java.util.ArrayList;
import java.util.List;
```

Method (below `getMessageByUUID`):

```java
	/**
	 * Lists every hidden message across all posts.
	 * @return an Iterator over the hidden messages, in no particular order
	 */
	public Iterator<HiddenMessage> getAllHiddenMessages() {
		List<HiddenMessage> hidden = new ArrayList<>();
		for (Iterator<Post> it = getAll(); it.hasNext(); ) {
			Post post = it.next();
			for (UUID messageId : post.getHiddenMessageIds())
				hidden.add(new HiddenMessage(post.id, messageId));
		}
		return hidden.iterator();
	}
```

### New: `persistentdata/serialization/ReportSerializer.java`

```java
package persistentdata.serialization;

import moderation.Report;

import java.util.UUID;

/**
 * Converts between Reports and String[].
 * Schema (3 columns): message UUID, reporting user UUID, timestamp (milliseconds since the Unix epoch)
 */
public class ReportSerializer implements Serializer<Report, String[]> {
	public static final int COLUMN_COUNT = 3;

	@Override
	public String[] serialize(Report report) {
		return new String[] {report.message().toString(), report.user().toString(), String.valueOf(report.timestamp())};
	}

	@Override
	public Report deserialize(String[] data) {
		return new Report(UUID.fromString(data[0]), UUID.fromString(data[1]), Long.parseLong(data[2]));
	}
}
```

### New: `persistentdata/serialization/HiddenMessageSerializer.java`

```java
package persistentdata.serialization;

import dao.model.HiddenMessage;

import java.util.UUID;

/**
 * Converts between HiddenMessages and String[].
 * Schema (2 columns): post UUID, message UUID.
 * Only hidden messages are stored; any message not listed is visible.
 */
public class HiddenMessageSerializer implements Serializer<HiddenMessage, String[]> {
	public static final int COLUMN_COUNT = 2;

	@Override
	public String[] serialize(HiddenMessage hidden) {
		return new String[] {hidden.post().toString(), hidden.message().toString()};
	}

	@Override
	public HiddenMessage deserialize(String[] data) {
		return new HiddenMessage(UUID.fromString(data[0]), UUID.fromString(data[1]));
	}
}
```

### Cleaned up: the three existing serializers

Each one gets a `COLUMN_COUNT` constant, so the column count lives next to the schema it describes instead of as a magic number in `DataManager`. Each also gets proper schema docs. `DataManager` below uses these constants, so these changes are required for it to compile.

**`persistentdata/serialization/UserSerializer.java`**

```java
package persistentdata.serialization;

import dao.model.User;
import persistentdata.PersistentDataException;

import java.util.UUID;

/**
 * Converts between Users and String[].
 * Schema (4 columns): user UUID, role ("member" or "admin"), username, password
 */
public class UserSerializer implements Serializer<User, String[]> {
	public static final int COLUMN_COUNT = 4;

	@Override
	public String[] serialize(User user) {
		return new String[] {user.id().toString(), roleToString(user.role()), user.username(), user.password()};
	}

	@Override
	public User deserialize(String[] data) {
		return new User(UUID.fromString(data[0]), roleFromString(data[1]), data[2], data[3]);
	}

	private static String roleToString(User.Role role) {
		return switch (role) {
			case Member -> "member";
			case Admin -> "admin";
		};
	}

	private static User.Role roleFromString(String role) {
		return switch (role) {
			case "member" -> User.Role.Member;
			case "admin" -> User.Role.Admin;
			default -> throw new PersistentDataException("Unknown user role: " + role);
		};
	}
}
```

**`persistentdata/serialization/PostSerializer.java`**

```java
package persistentdata.serialization;

import dao.model.Post;

import java.util.UUID;

/**
 * Converts between Posts and String[].
 * Schema (3 columns): post UUID, poster UUID, topic
 */
public class PostSerializer implements Serializer<Post, String[]> {
	public static final int COLUMN_COUNT = 3;

	@Override
	public String[] serialize(Post post) {
		return new String[] {post.id.toString(), post.poster.toString(), post.topic};
	}

	@Override
	public Post deserialize(String[] data) {
		return new Post(UUID.fromString(data[0]), UUID.fromString(data[1]), data[2]);
	}
}
```

**`persistentdata/serialization/MessageSerializer.java`**

```java
package persistentdata.serialization;

import dao.model.Message;

import java.util.UUID;

/**
 * Converts between Messages and String[].
 * Schema (5 columns): message UUID, poster UUID, thread (post) UUID, timestamp, message text
 */
public class MessageSerializer implements Serializer<Message, String[]> {
	public static final int COLUMN_COUNT = 5;

	@Override
	public String[] serialize(Message message) {
		return new String[] {message.id().toString(), message.poster().toString(), message.thread().toString(),
				String.valueOf(message.timestamp()), message.message()};
	}

	@Override
	public Message deserialize(String[] data) {
		return new Message(UUID.fromString(data[0]), UUID.fromString(data[1]), UUID.fromString(data[2]),
				Long.parseLong(data[3]), data[4]);
	}
}
```

### `persistentdata/DataManager.java` (full file)

```java
package persistentdata;

import dao.PostDAO;
import dao.UserDAO;
import dao.model.HiddenMessage;
import dao.model.Message;
import dao.model.Post;
import dao.model.User;
import moderation.ModerationTools;
import moderation.Report;
import moderation.ReportStore;
import persistentdata.formatted.CSVFormat;
import persistentdata.formatted.CSVFormattedFactory;
import persistentdata.io.ComputerIOFactory;
import persistentdata.io.IOFactory;
import persistentdata.serialization.HiddenMessageSerializer;
import persistentdata.serialization.MessageSerializer;
import persistentdata.serialization.PostSerializer;
import persistentdata.serialization.ReportSerializer;
import persistentdata.serialization.Serializer;
import persistentdata.serialization.UserSerializer;

/**
 * Saves and loads all application data. Each kind of data is stored in its own CSV file.
 */
public class DataManager {
	private static DataManager instance;

	public static DataManager getInstance() {
		if (instance == null)
			instance = new DataManager();
		return instance;
	}

	private DataManager() {}

	private final IOFactory io = new ComputerIOFactory();

	private final DataPipeline<User, String[]> userPipeline =
			csvPipeline(new UserSerializer(), UserSerializer.COLUMN_COUNT, "users");
	private final DataPipeline<Post, String[]> postPipeline =
			csvPipeline(new PostSerializer(), PostSerializer.COLUMN_COUNT, "posts");
	private final DataPipeline<Message, String[]> messagePipeline =
			csvPipeline(new MessageSerializer(), MessageSerializer.COLUMN_COUNT, "messages");
	private final DataPipeline<HiddenMessage, String[]> hiddenPipeline =
			csvPipeline(new HiddenMessageSerializer(), HiddenMessageSerializer.COLUMN_COUNT, "hidden_messages");
	private final DataPipeline<Report, String[]> reportPipeline =
			csvPipeline(new ReportSerializer(), ReportSerializer.COLUMN_COUNT, "reports");

	private final UserDAO users = UserDAO.getInstance();
	private final PostDAO posts = PostDAO.getInstance();
	private final ReportStore reports = ReportStore.getInstance();

	private <T> DataPipeline<T, String[]> csvPipeline(Serializer<T, String[]> serializer, int columns, String filename) {
		return new DataPipeline<>(io, new CSVFormattedFactory(new CSVFormat(columns)), serializer, filename);
	}

	/**
	 * Replaces all data in memory with the saved data. Files are read in dependency order:
	 * hidden messages and reports refer to messages, which refer to posts and users.
	 */
	public void readAll() {
		users.clear();
		posts.clear();
		reports.clear();
		userPipeline.readTo(users::add);
		postPipeline.readTo(posts::add);
		messagePipeline.readTo(message -> posts.get(new Post(message.thread())).messages.insert(message));
		hiddenPipeline.readTo(this::restoreHidden);
		reportPipeline.readTo(this::restoreReport);
	}

	public void writeAll() {
		userPipeline.writeFrom(users.getAll());
		postPipeline.writeFrom(posts.getAll());
		messagePipeline.writeFrom(posts.getAllMessages());
		hiddenPipeline.writeFrom(posts.getAllHiddenMessages());
		reportPipeline.writeFrom(reports.getAllReports());
	}

	private void restoreHidden(HiddenMessage hidden) {
		Post post = posts.get(new Post(hidden.post()));
		if (post != null) post.setHidden(hidden.message(), true);
	}

	/** Goes through addReport so that reports on deleted users or messages are skipped. */
	private void restoreReport(Report report) {
		ModerationTools.addReport(report.message(), report.user(), report.timestamp());
	}
}
```

What changed and why:

- **`csvPipeline(...)` helper.** Before, each pipeline repeated `new DataPipeline<>(IO, new CSVFormattedFactory(new CSVFormat(n)), ...)`. Now that's written once (removes duplication).
- **`reports.clear()` in `readAll`.** Without it, loading twice would mix old and new reports.
- **`private DataManager() {}`.** It's a singleton, but the constructor was public, so anyone could make a second one. Now they can't.
- **`IO` renamed to `io`.** Java naming conventions reserve ALL_CAPS for constants.

### `persistentdata/DataPipeline.java` (full file, refactored)

```java
package persistentdata;

import persistentdata.formatted.FormattedFactory;
import persistentdata.formatted.FormattedReader;
import persistentdata.formatted.FormattedWriter;
import persistentdata.io.IOFactory;
import persistentdata.serialization.Serializer;

import java.io.IOException;
import java.io.Reader;
import java.io.Writer;
import java.util.Iterator;
import java.util.function.Consumer;

/**
 * Moves one kind of object between the application and one persistent file,
 * by combining a Serializer (object to row), a FormattedFactory (row to text)
 * and an IOFactory (text to storage).
 * @param <T> the type of object stored
 * @param <S> the serialized form of each object
 */
public class DataPipeline<T, S> {
	private final IOFactory ioFactory;
	private final FormattedFactory<S> formattedFactory;
	private final Serializer<T, S> serializer;
	private final String filename;

	public DataPipeline(IOFactory ioFactory, FormattedFactory<S> formattedFactory, Serializer<T, S> serializer, String filename) {
		this.ioFactory = ioFactory;
		this.formattedFactory = formattedFactory;
		this.serializer = serializer;
		this.filename = filename;
	}

	/**
	 * Writes every object from the iterator to storage, replacing the file's previous contents.
	 */
	public void writeFrom(Iterator<T> iterator) {
		try (Writer writer = ioFactory.writer(filename)) {
			FormattedWriter<S> formattedWriter = formattedFactory.writer(writer);
			while (iterator.hasNext())
				formattedWriter.putNext(serializer.serialize(iterator.next()));
		} catch (IOException e) {
			throw new PersistentDataException(e.getMessage());
		}
	}

	/**
	 * Reads every object from storage and passes each one to the callback.
	 * Does nothing if no file has been saved yet.
	 */
	public void readTo(Consumer<T> callback) {
		Reader reader = ioFactory.reader(filename);
		if (reader == null) return;
		try (reader) {
			FormattedReader<S> formattedReader = formattedFactory.reader(reader);
			while (formattedReader.hasNext())
				callback.accept(serializer.deserialize(formattedReader.getNext()));
		} catch (IOException e) {
			throw new PersistentDataException(e.getMessage());
		}
	}
}
```

What changed and why:

- **Removed the unused `users` and `posts` fields and `import dao.*`.** That was dead code, and it tied a general-purpose pipeline to specific DAOs.
- **Replaced the custom `AddToDAO<T>` interface with Java's built-in `Consumer<T>`.** It did exactly the same thing, so the custom one was reinventing the wheel. The callers (`users::add`, lambdas) work unchanged.
- **`try (...)` (try-with-resources).** Before, if anything threw mid-write, `writer.close()` was skipped and the file was left open (a resource leak). Now Java closes it automatically, even on errors.

### `persistentdata/io/ComputerIOFactory.java` (full file, time bomb removed)

```java
package persistentdata.io;

import persistentdata.PersistentDataException;

import java.io.File;
import java.io.FileReader;
import java.io.FileWriter;
import java.io.IOException;
import java.io.Reader;
import java.io.Writer;

/**
 * Stores each named file as a text file inside the "saved" folder.
 */
public class ComputerIOFactory implements IOFactory {
	private static final String FOLDER = "saved";
	private static final String FULL_FILENAME_TEMPLATE = FOLDER + "/%s.txt";

	private static File fileFor(String filename) {
		return new File(FULL_FILENAME_TEMPLATE.formatted(filename));
	}

	/**
	 * @throws PersistentDataException if the file cannot be created
	 */
	@Override
	public Writer writer(String filename) {
		try {
			new File(FOLDER).mkdirs();
			return new FileWriter(fileFor(filename));
		} catch (IOException e) {
			throw new PersistentDataException("Could not write " + filename + ": " + e.getMessage());
		}
	}

	/**
	 * @return a Reader for the file, or null if it has not been saved yet
	 * @throws PersistentDataException if the file exists but cannot be opened
	 */
	@Override
	public Reader reader(String filename) {
		File file = fileFor(filename);
		if (!file.exists()) return null;
		try {
			return new FileReader(file);
		} catch (IOException e) {
			throw new PersistentDataException("Could not read " + filename + ": " + e.getMessage());
		}
	}
}
```

What changed and why:

- **Removed the date check.** It was a fake "OS check" that made every load fail after 1 Dec 2025.
- **Stopped swallowing errors.** Before, a failed save returned `null` and `DataPipeline` silently skipped it. You'd lose data with no warning. Now a real failure throws a clear `PersistentDataException`. "No file yet" (first run) is still allowed and returns `null`.
- **`mkdirs()` creates the `saved` folder if it's missing.** Before, a fresh clone had no `saved/` folder (it's in `.gitignore`), so saving silently did nothing.

### `persistentdata/formatted/CSVReader.java`: fix `hasNext`

Add `import java.io.PushbackReader;`, then replace the `reader` field, constructor, `eof` field and `hasNext()` (the top of the class) with:

```java
	private final PushbackReader reader;
	private boolean eof = false;

	public CSVReader(CSVFormat format, Reader reader) {
		this.format = format;
		this.reader = new PushbackReader(reader);
	}

	/**
	 * Peeks one character ahead, so that an empty file or a trailing newline
	 * is not mistaken for another row.
	 */
	@Override
	public boolean hasNext() {
		if (eof) return false;
		try {
			int next = reader.read();
			if (next == -1) {
				eof = true;
				return false;
			}
			reader.unread(next);
			return true;
		} catch (IOException e) {
			throw new CSVIOException(e.getMessage());
		}
	}
```

A `PushbackReader` lets you read one character and then "put it back," so `getNext()` still sees it. That's how we peek without losing data. The rest of `CSVReader` stays the same.

## Code smells fixed (for the 40% quality marks)

Listing these in your commit messages or to your marker shows you deliberately refactored:

| Smell | Where | Fix |
|---|---|---|
| Hidden time bomb disguised as an OS check | `ComputerIOFactory.reader` | Removed |
| Swallowed exceptions → silent data loss | `ComputerIOFactory` | Throw `PersistentDataException` with a message |
| `hasNext()` always `true` | `CSVReader` | Peek ahead with `PushbackReader` |
| Unused fields and imports | `DataPipeline` | Removed |
| Reinventing a standard interface (`AddToDAO`) | `DataPipeline` | `java.util.function.Consumer` |
| Resource leak (file not closed on error) | `DataPipeline` | try-with-resources |
| Duplicated pipeline construction | `DataManager` | `csvPipeline(...)` helper |
| Magic numbers (column counts) | `DataManager` | `COLUMN_COUNT` constants in each serializer |
| Public constructor on a singleton | `DataManager` | Private constructor |
| Non-constant named in ALL_CAPS (`IO`) | `DataManager` | Renamed `io` |
| Leftover TODO comment / copy-pasted wrong doc ("each field of Post" on Message) | Serializers | Real schema docs |
| Confusing overloaded method names (`serialize(Role)` next to `serialize(User)`) | `UserSerializer` | `roleToString` / `roleFromString` |
| `throw new RuntimeException()` with no message | `UserSerializer` | `PersistentDataException("Unknown user role: …")` |
| `Long.valueOf` where a primitive is needed (needless boxing) | `MessageSerializer` | `Long.parseLong` |

## What I tested

| Test | Result |
|---|---|
| Save and load with no reports or hidden messages (empty files) | ✅ |
| 25,001 messages restored | ✅ |
| 501 reports restored exactly (same message, user, timestamp) | ✅ |
| 21 hidden messages restored | ✅ |
| Message with commas, quotes and a newline survives | ✅ |
| Non-admins still can't see hidden messages after reloading | ✅ |
| All Task 1 and Task 2 checks still pass | ✅ |

## Things to mention to your group

- **The `Report` change touches Task 1.** Whoever owns Task 1 should know `Report` now has three fields. Nothing else in Task 1 changes, and Task 4 isn't affected.
- **Restoring reports through `addReport` is safe but not the fastest.** Each never-seen message gets searched for once. With hundreds of reported messages that's fine. If performance on load ever matters, you could build a `HashMap<UUID, Message>` once during loading instead.
- **If someone prefers a "hidden" column in `messages.txt`,** the trade-off is that it breaks other programs that already read the 5-column file. The separate file avoids that. This is a good point to put in your design discussion.
