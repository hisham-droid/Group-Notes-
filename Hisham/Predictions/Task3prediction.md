# Task 3 prediction: persistence + refactoring

> All code here passed save-then-load round-trip tests (including one that saves and reloads all 27 predicted features): in memory, on a real temporary folder, with
> commas, quotes, newlines and null fields, and again with the JSON format swapped in.

## 1. What Task 3 looks like (from the practice hackathon)

| Part | Marks | What earns it |
|---|---|---|
| Persistence | 60% | the new Task 1 + Task 2 data survives `writeAll()` then `readAll()` |
| Quality, "beyond the level expected for the other tasks" | 40% | find and fix code smells in the persistence code |

"Read by other applications ... in other languages ... select a **portable** representation" means:
CSV or JSON with plain strings and numbers. **Not** Java `Serializable` / `ObjectOutputStream`.

## 2. FIRST: two bugs in the provided base that break ALL persistence

Check these before writing anything. In the practice base both were present:

1. **`CSVReader.hasNext()` always returned `true`**: reading any file ends in "Already reached end of file". The miniproject's own `CSVReaderTests` failed 11 of 15 because of it.
2. **`ComputerIOFactory.reader()` had a date check** (`if (System.currentTimeMillis() > 1764620398492L) throw ...`), so after December 2025 every read returned `null` and **nothing ever loaded**. It also never created the `saved/` folder, so writing failed on a fresh clone.

### 2.1 CSVReader.hasNext fix (peek one character)

> How to read a diff block: lines starting with `+` are new, lines starting with `-` are deleted, other lines are unchanged context so you can find the spot. `@@` lines just mark where the next chunk starts.

**`app/src/persistentdata/formatted/CSVReader.java`** (changes only: `+` = add this line, `-` = remove this line)

```diff
@@ -3,20 +3,39 @@
 import persistentdata.PersistentDataException;
 
 import java.io.IOException;
+import java.io.PushbackReader;
 import java.io.Reader;
 
 public class CSVReader implements FormattedReader<String[]> {
 	private final CSVFormat format;
-	private final Reader reader;
+	// HACKATHON: PushbackReader lets hasNext() PEEK at one character and put it back
+	private final PushbackReader reader;
 
 	public CSVReader(CSVFormat format, Reader reader) {
 		this.format = format;
-		this.reader = reader;
+		this.reader = new PushbackReader(reader);
 	}
-	
+
 	private boolean eof = false;
+
+	/**
+	 * HACKATHON FIX: the provided version always returned true, so reading any file
+	 * ended in an exception. Now: true only if at least one more character exists.
+	 * (A trailing newline at the very end of the file therefore does not count as a row.)
+	 */
 	public boolean hasNext() {
-		return true;
+		if (eof) return false;
+		try {
+			int c = reader.read();
+			if (c == -1) {
+				eof = true;
+				return false;
+			}
+			reader.unread(c);
+			return true;
+		} catch (IOException e) {
+			throw new CSVIOException(e.getMessage());
+		}
 	}
 
 	// These format strings are provided to give you some ideas about what error cases might be encountered,
```

### 2.2 ComputerIOFactory without the date check (and testable)

**`app/src/persistentdata/io/ComputerIOFactory.java`** (full file)

```java
package persistentdata.io;

import java.io.*;

/**
 * Reads/writes text files on a desktop OS.
 * HACKATHON changes:
 *  - removed the "if (System.currentTimeMillis() > ...) throw" check. It made every
 *    read return null after Dec 2025, so NOTHING was ever loaded.
 *  - the folder is created if missing (FileWriter fails if "saved/" does not exist)
 *  - the folder is a constructor parameter so tests can use a temporary folder
 */
public class ComputerIOFactory implements IOFactory {
	private static final String DEFAULT_FOLDER = "saved";
	private static final String FILE_EXTENSION = ".txt";

	private final File folder;

	public ComputerIOFactory() {
		this(new File(DEFAULT_FOLDER));
	}

	public ComputerIOFactory(File folder) {
		this.folder = folder;
	}

	private File fileFor(String name) {
		return new File(folder, name + FILE_EXTENSION);
	}

	@Override
	public Writer writer(String filename) {
		try {
			if (!folder.exists() && !folder.mkdirs()) return null;
			return new FileWriter(fileFor(filename));
		} catch (IOException ignored) {
			return null;
		}
	}

	@Override
	public Reader reader(String filename) {
		try {
			return new FileReader(fileFor(filename));
		} catch (IOException ignored) {
			return null; // no file yet = nothing saved yet
		}
	}
}
```

## 3. Every Task 3 prediction

| What must persist | Portable representation (used below) | Alternative |
|---|---|---|
| Reports (practice) | `reports.txt`: one row per active report `message,user,timestamp` | JSON |
| Hidden flags (practice) | `hidden.txt`: one row per hidden message `message,post` | 6th column in messages.txt (changes the existing format, so other readers break) |
| Reactions | `reactions.txt`: `message,user,TYPE,timestamp` (store the enum **name**, not `ordinal()`) | |
| Follows / blocks / bookmarks | one file per relation: `source,target` | |
| Votes / polls | `votes.txt`: `poll,user,option` | |
| A list inside one record (tags, edit history) | **one row per item** (`post,tag`), never a list in one cell | JSON nested arrays |

The rule: **one row per fact**, plain strings, and parents load before children.

## 4. Serializers (one class per row type)

Shared null handling (posts made with `new Post(uuid)` have a null poster/topic, test messages often a null poster;
the original serializers crashed on both):

**`app/src/persistentdata/serialization/Nullable.java`** (full file)

```java
package persistentdata.serialization;

import java.util.UUID;

/**
 * HACKATHON (Task 3 refactor): shared null handling for serializers.
 * Posts made with new Post(uuid) have a null poster and topic, and test messages often
 * have a null poster: the original serializers crashed on those with NullPointerException.
 * Convention: null is written as an empty field.
 */
final class Nullable {
	private Nullable() {}

	static String write(Object value) {
		return value == null ? "" : value.toString();
	}

	static UUID readUUID(String field) {
		return field.isEmpty() ? null : UUID.fromString(field);
	}

	static String readString(String field) {
		return field.isEmpty() ? null : field;
	}
}
```

**`app/src/persistentdata/serialization/PostSerializer.java`** (full file)

```java
package persistentdata.serialization;
import dao.model.Post;

import java.util.UUID;

/**
 * Converts between Posts and String[]: (UUID, poster, topic).
 * HACKATHON: poster/topic may be null, written as an empty field.
 */
public class PostSerializer implements Serializer<Post, String[]> {

	@Override
	public String[] serialize(Post object) {
		return new String[] {object.id.toString(), Nullable.write(object.poster), Nullable.write(object.topic)};
	}

	@Override
	public Post deserialize(String[] data) {
		return new Post(UUID.fromString(data[0]), Nullable.readUUID(data[1]), Nullable.readString(data[2]));
	}
}
```

**`app/src/persistentdata/serialization/MessageSerializer.java`** (full file)

```java
package persistentdata.serialization;

import dao.model.Message;

import java.util.UUID;

/**
 * Converts between Messages and String[]: (UUID, poster, thread, timestamp, message).
 * HACKATHON: poster may be null (e.g. deleted user, test data), written as an empty field.
 */
public class MessageSerializer implements Serializer<Message, String[]> {

	@Override
	public String[] serialize(Message object) {
		return new String[] {object.id().toString(), Nullable.write(object.poster()), object.thread().toString(),
				String.valueOf(object.timestamp()), Nullable.write(object.message())};
	}

	@Override
	public Message deserialize(String[] data) {
		return new Message(UUID.fromString(data[0]), Nullable.readUUID(data[1]), UUID.fromString(data[2]),
				Long.parseLong(data[3]), Nullable.readString(data[4]));
	}
}
```

**`app/src/persistentdata/serialization/ReportSerializer.java`** (full file)

```java
package persistentdata.serialization;

import moderation.Report;

import java.util.UUID;

/**
 * HACKATHON (Task 3): one CSV row per active report.
 * Schema (3 columns): messageUUID, userUUID, timestamp (UNIX ms as a decimal string)
 * Plain strings and numbers only, so any language can read it.
 */
public class ReportSerializer implements Serializer<Report, String[]> {
	public static final int COLUMNS = 3;

	@Override
	public String[] serialize(Report report) {
		return new String[] {report.message().toString(), report.user().toString(), Long.toString(report.timestamp())};
	}

	@Override
	public Report deserialize(String[] data) {
		return new Report(UUID.fromString(data[0]), UUID.fromString(data[1]), Long.parseLong(data[2]));
	}
}
```

**`app/src/persistentdata/serialization/HiddenMessage.java`** (full file)

```java
package persistentdata.serialization;

import java.util.UUID;

/**
 * HACKATHON (Task 3): "this message is hidden". We store the post id too, so on
 * load we can find the post in O(log n) without scanning for the message.
 * @param message the hidden message's UUID
 * @param post    the UUID of the post that message belongs to
 */
public record HiddenMessage(UUID message, UUID post) {}
```

**`app/src/persistentdata/serialization/HiddenMessageSerializer.java`** (full file)

```java
package persistentdata.serialization;

import java.util.UUID;

/**
 * HACKATHON (Task 3): one CSV row per hidden message.
 * Schema (2 columns): messageUUID, postUUID
 * A separate file (instead of a 6th "hidden" column in messages.txt) means the
 * existing messages file format does not change, so other programs reading it keep working.
 */
public class HiddenMessageSerializer implements Serializer<HiddenMessage, String[]> {
	public static final int COLUMNS = 2;

	@Override
	public String[] serialize(HiddenMessage hidden) {
		return new String[] {hidden.message().toString(), hidden.post().toString()};
	}

	@Override
	public HiddenMessage deserialize(String[] data) {
		return new HiddenMessage(UUID.fromString(data[0]), UUID.fromString(data[1]));
	}
}
```

Every other feature (reactions, votes, polls, ...) is saved through the feature list in section 6, not with one serializer class each.

## 5. DataPipeline and DataManager (refactored)

Code smells fixed (say these in a comment or commit message; that is where the 40% comes from):

| Smell | Where | Fix |
|---|---|---|
| Dead code / hidden coupling | `DataPipeline` had unused static `UserDAO`/`PostDAO` fields | removed |
| Resource leak | `writer.close()` skipped when an exception is thrown | try-with-resources |
| Unused interface methods | `putHeader`/`putFooter` never called | called (JSON needs them) |
| Reinvented standard type | custom `AddToDAO` interface | `java.util.function.Consumer` |
| Magic numbers | `new CSVFormat(4)`, `(3)`, `(5)` | named constants |
| Duplicated code | three copy-pasted `new DataPipeline<>(IO, new CSVFormattedFactory(...))` | one `pipeline(...)` helper |
| Untestable singleton | IOFactory hard-coded | constructor takes an `IOFactory` |
| Null crashes | serializers | `Nullable` helper |
| Time-bomb / hidden behaviour | `ComputerIOFactory` date check | removed |

**`app/src/persistentdata/DataPipeline.java`** (full file)

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
 * Moves objects of type T to/from one file:  IO  <->  format (e.g. CSV)  <->  serializer.
 * HACKATHON refactor (Task 3 "code smells"):
 *  - removed unused static UserDAO/PostDAO fields (dead code + hidden coupling)
 *  - try-with-resources: the file is closed even when an exception is thrown
 *  - putHeader()/putFooter() are now actually called (needed by formats like JSON)
 *  - the custom AddToDAO interface is replaced by the standard java.util.function.Consumer
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

	/** Writes every element of the iterator to this pipeline's file (replacing it). */
	public void writeFrom(Iterator<T> iterator) {
		try (Writer writer = ioFactory.writer(filename)) {
			if (writer == null) throw new PersistentDataException("Cannot open " + filename + " for writing");
			FormattedWriter<S> formattedWriter = formattedFactory.writer(writer);
			formattedWriter.putHeader();
			while (iterator.hasNext()) {
				formattedWriter.putNext(serializer.serialize(iterator.next()));
			}
			formattedWriter.putFooter();
		} catch (IOException e) {
			throw new PersistentDataException(e.getMessage());
		}
	}

	/** Reads every entry in this pipeline's file and hands each object to the callback. */
	public void readTo(Consumer<T> callback) {
		Reader reader = ioFactory.reader(filename);
		if (reader == null) return; // file does not exist yet: nothing to load
		try (reader) {
			FormattedReader<S> formattedReader = formattedFactory.reader(reader);
			while (formattedReader.hasNext()) {
				callback.accept(serializer.deserialize(formattedReader.getNext()));
			}
		} catch (IOException e) {
			throw new PersistentDataException(e.getMessage());
		}
	}
}
```

**`app/src/persistentdata/DataManager.java`** (full file)

```java
package persistentdata;

import dao.MessageIndex;
import dao.PostDAO;
import dao.UserDAO;
import dao.model.Message;
import dao.model.Post;
import dao.model.User;
import moderation.Report;
import moderation.ReportStore;
import persistentdata.formatted.CSVFormat;
import persistentdata.features.Features;
import persistentdata.features.PersistentFeature;
import persistentdata.features.RowSerializer;
import persistentdata.formatted.CSVFormattedFactory;
import persistentdata.io.ComputerIOFactory;
import persistentdata.io.IOFactory;
import persistentdata.serialization.HiddenMessage;
import persistentdata.serialization.HiddenMessageSerializer;
import persistentdata.serialization.MessageSerializer;
import persistentdata.serialization.PostSerializer;
import persistentdata.serialization.ReportSerializer;
import persistentdata.serialization.Serializer;
import persistentdata.serialization.UserSerializer;

import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;
import java.util.UUID;

/**
 * Saves and loads the whole application.
 * HACKATHON (Task 3): adds reports and hidden flags, and refactors:
 *  - column counts are named constants (no magic numbers)
 *  - one helper builds every pipeline (no copy-pasted constructor calls)
 *  - the IOFactory can be injected, so tests can save to memory or a temp folder
 *  - load order is documented, because later files refer to earlier ones
 */
public class DataManager {
	private static final int USER_COLUMNS = 4;    // id, role, username, password
	private static final int POST_COLUMNS = 3;    // id, poster, topic
	private static final int MESSAGE_COLUMNS = 5; // id, poster, thread, timestamp, text

	private static DataManager instance;

	public static DataManager getInstance() {
		if (instance == null) instance = new DataManager(new ComputerIOFactory());
		return instance;
	}

	private final DataPipeline<User, String[]> userPipeline;
	private final DataPipeline<Post, String[]> postPipeline;
	private final DataPipeline<Message, String[]> messagePipeline;
	private final DataPipeline<Report, String[]> reportPipeline;
	private final DataPipeline<HiddenMessage, String[]> hiddenPipeline;

	// every other feature: one pipeline each, built from the Features list
	private final List<PersistentFeature> features = Features.all();
	private final List<DataPipeline<String[], String[]>> featurePipelines = new ArrayList<>();

	private final UserDAO users = UserDAO.getInstance();
	private final PostDAO posts = PostDAO.getInstance();
	private final ReportStore reports = ReportStore.getInstance();

	/** @param io where files live (public so tests can pass an in-memory implementation) */
	public DataManager(IOFactory io) {
		userPipeline = pipeline(io, new UserSerializer(), USER_COLUMNS, "users");
		postPipeline = pipeline(io, new PostSerializer(), POST_COLUMNS, "posts");
		messagePipeline = pipeline(io, new MessageSerializer(), MESSAGE_COLUMNS, "messages");
		reportPipeline = pipeline(io, new ReportSerializer(), ReportSerializer.COLUMNS, "reports");
		hiddenPipeline = pipeline(io, new HiddenMessageSerializer(), HiddenMessageSerializer.COLUMNS, "hidden");
		for (PersistentFeature feature : features) {
			featurePipelines.add(pipeline(io, new RowSerializer(), feature.columns(), feature.fileName()));
		}
	}

	private static <T> DataPipeline<T, String[]> pipeline(IOFactory io, Serializer<T, String[]> serializer, int columns, String file) {
		return new DataPipeline<>(io, new CSVFormattedFactory(new CSVFormat(columns)), serializer, file);
	}

	/**
	 * Load order matters: users and posts first, then messages (they need their post),
	 * then hidden flags (need the message's post), then reports.
	 */
	public void readAll() {
		users.clear();
		posts.clear();
		reports.clear();
		MessageIndex.getInstance().clear(); // so reloaded messages count as new (rebuilds derived data like mentions)
		mentions.MentionTools.clear();      // derived data: rebuilt from the messages as they load
		search.SearchTools.clear();
		userPipeline.readTo(users::add);
		postPipeline.readTo(posts::add);
		messagePipeline.readTo(this::restoreMessage);
		hiddenPipeline.readTo(this::restoreHidden);
		reportPipeline.readTo(reports::add);
		for (int i = 0; i < features.size(); i++) {
			PersistentFeature feature = features.get(i);
			feature.clear();
			featurePipelines.get(i).readTo(feature::restore);
			feature.afterLoad();
		}
	}

	public void writeAll() {
		userPipeline.writeFrom(users.getAll());
		postPipeline.writeFrom(posts.getAll());
		messagePipeline.writeFrom(posts.getAllMessages());
		hiddenPipeline.writeFrom(allHiddenMessages());
		reportPipeline.writeFrom(reports.getAllReports());
		for (int i = 0; i < features.size(); i++) {
			featurePipelines.get(i).writeFrom(features.get(i).rows());
		}
	}

	private void restoreMessage(Message message) {
		Post post = posts.get(new Post(message.thread()));
		if (post != null) post.messages.insert(message); // skip orphans instead of crashing
	}

	private void restoreHidden(HiddenMessage hidden) {
		Post post = posts.get(new Post(hidden.post()));
		// messages were loaded first, so the index already knows this message (O(1))
		Message message = MessageIndex.getInstance().get(hidden.message());
		if (post != null && message != null) post.setHidden(message, true);
	}

	private Iterator<HiddenMessage> allHiddenMessages() {
		List<HiddenMessage> result = new ArrayList<>();
		for (Iterator<Post> it = posts.getAll(); it.hasNext(); ) {
			Post post = it.next();
			for (UUID messageId : post.getHiddenMessageIds()) {
				result.add(new HiddenMessage(messageId, post.id));
			}
		}
		return result.iterator();
	}
}
```

### What the files look like (real output from the test)

```
reports.txt  (message, user, timestamp)
95fd40a6-29ff-4420-b752-f08a77e44a99,362cf8fd-6de2-4fe5-a72a-3456e853cc42,75
89642b2c-579e-4a06-8a7d-a2f4ffc8c3db,c93ba8bf-b6c7-489e-ad55-18c9d4688a2b,50

hidden.txt   (message, post)
95fd40a6-29ff-4420-b752-f08a77e44a99,a61f017d-47bd-461a-8a46-02b1cad8714a

messages.txt (id, poster, thread, timestamp, text): note the empty poster and the quoted multi-line text
95fd40a6-...,,a61f017d-...,2,"line1
line2 ""q"""
```

## 6. Persistence for EVERY predicted feature (one list, 27 files)

Writing a serializer + pipeline + read call + write call for each of 27 features would be the "duplicated code"
smell the 40% quality mark is about. Instead each feature is a `PersistentFeature` (Strategy pattern):
it turns its state into rows and back. `DataManager` loops over `Features.all()`. Adding a feature = one entry.

| File | Columns (in order) | Restored into |
|---|---|---|
| `deleted.txt` | message, post | `Post.setDeleted` |
| `pinned.txt` | message, post | `Post.setPinned` |
| `locked.txt` | post | `Post.setLocked` |
| `reactions.txt` | message, user, TYPE (enum name), timestamp | `ReactionStore` |
| `votes.txt` | message, user, +1/-1, latest timestamp | `VoteTools.restore` |
| `edits.txt` | message, index, old text, edited at | `EditTools.restore` |
| `follows.txt`, `blocks.txt`, `bookmarks.txt`, `postlikes.txt`, `subscriptions.txt`, `friendrequests.txt`, `friends.txt`, `tags.txt` | source, target | the matching `RelationStore` (one generic class) |
| `polls.txt` | poll, post, creator, option index, option text (**one row per option**: CSV has fixed columns) | rebuilt in `afterLoad` |
| `pollvotes.txt` | poll, user, option | `PollTools.restoreVote` |
| `readmarkers.txt` | user, post, timestamp | `ReadTracker.restore` |
| `bans.txt` | user, banned until | `BanTools.restore` |
| `moderators.txt` | user | `Permissions.restore` |
| `directmessages.txt` | id, from, to, timestamp, text | `DirectMessageTools.restore` |
| `scheduled.txt` | id, user, post, publish at, text (cancelled ones are not saved) | `ScheduledMessages.restore` |
| `auditlog.txt` | admin, action, timestamp | `AdminActions.restoreLog` |
| `boards.txt`, `boardposts.txt` | board, parent / board, post | `BoardTools` |
| `replies.txt` | parent message, reply message | `ThreadTools.link` |
| `privateposts.txt`, `invites.txt` | post / post, invited user | `AccessTools` |

**Not saved on purpose** (say why in javadoc): notifications and rate-limit windows are temporary; mentions and the
search index are *derived* from messages and rebuild themselves when messages load (`readAll` empties them first).

**`app/src/persistentdata/features/PersistentFeature.java`** (full file)

```java
package persistentdata.features;

import java.util.Iterator;

/**
 * HACKATHON (Task 3 refactor): one saved file = one feature.
 * A feature turns its state into rows of strings and back. DataManager loops over a
 * list of features, so adding persistence for a new feature is ONE new class + ONE line
 * in the list, instead of a new serializer, pipeline field, read call and write call.
 * (Strategy pattern: each feature is a strategy for saving one kind of data.)
 */
public interface PersistentFeature {
	/** file name without extension, e.g. "votes" */
	String fileName();

	/** number of columns in every row */
	int columns();

	/** the rows to save */
	Iterator<String[]> rows();

	/** rebuild state from one saved row */
	void restore(String[] row);

	/** empty the feature before loading */
	void clear();

	/** called after all rows were restored (for features that need every row first) */
	default void afterLoad() {}
}
```

**`app/src/persistentdata/features/RowSerializer.java`** (full file)

```java
package persistentdata.features;

import persistentdata.serialization.Serializer;

/** Rows are already String[], so serializing is the identity. */
public class RowSerializer implements Serializer<String[], String[]> {
	@Override
	public String[] serialize(String[] row) {
		return row;
	}

	@Override
	public String[] deserialize(String[] row) {
		return row;
	}
}
```

**`app/src/persistentdata/features/RelationFeature.java`** (full file)

```java
package persistentdata.features;

import relations.RelationStore;

import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;
import java.util.function.Function;

/**
 * Saves ANY RelationStore (follows, blocks, bookmarks, likes, tags, subscriptions, friends...)
 * as rows "source,target". One generic class instead of eight copies.
 */
public class RelationFeature<A, B> implements PersistentFeature {
	private final String fileName;
	private final RelationStore<A, B> store;
	private final Function<String, A> parseSource;
	private final Function<String, B> parseTarget;

	public RelationFeature(String fileName, RelationStore<A, B> store, Function<String, A> parseSource, Function<String, B> parseTarget) {
		this.fileName = fileName;
		this.store = store;
		this.parseSource = parseSource;
		this.parseTarget = parseTarget;
	}

	@Override public String fileName() { return fileName; }
	@Override public int columns() { return 2; }

	@Override
	public Iterator<String[]> rows() {
		List<String[]> rows = new ArrayList<>();
		for (A source : store.sources())
			for (B target : store.targetsOf(source)) rows.add(new String[] {source.toString(), target.toString()});
		return rows.iterator();
	}

	@Override
	public void restore(String[] row) {
		store.add(parseSource.apply(row[0]), parseTarget.apply(row[1]));
	}

	@Override
	public void clear() {
		store.clear();
	}
}
```

**`app/src/persistentdata/features/SimpleFeature.java`** (full file)

```java
package persistentdata.features;

import java.util.Iterator;
import java.util.function.Consumer;
import java.util.function.Supplier;

/**
 * A feature built from three lambdas (rows / restore / clear), for the small cases.
 * Keeps the feature list in Features.java short and readable.
 */
public class SimpleFeature implements PersistentFeature {
	private final String fileName;
	private final int columns;
	private final Supplier<Iterator<String[]>> rows;
	private final Consumer<String[]> restore;
	private final Runnable clear;
	private final Runnable afterLoad;

	public SimpleFeature(String fileName, int columns, Supplier<Iterator<String[]>> rows, Consumer<String[]> restore, Runnable clear) {
		this(fileName, columns, rows, restore, clear, () -> {});
	}

	public SimpleFeature(String fileName, int columns, Supplier<Iterator<String[]>> rows, Consumer<String[]> restore, Runnable clear, Runnable afterLoad) {
		this.fileName = fileName;
		this.columns = columns;
		this.rows = rows;
		this.restore = restore;
		this.clear = clear;
		this.afterLoad = afterLoad;
	}

	@Override public String fileName() { return fileName; }
	@Override public int columns() { return columns; }
	@Override public Iterator<String[]> rows() { return rows.get(); }
	@Override public void restore(String[] row) { restore.accept(row); }
	@Override public void clear() { clear.run(); }
	@Override public void afterLoad() { afterLoad.run(); }
}
```

**`app/src/persistentdata/features/Features.java`** (full file)

```java
package persistentdata.features;

import accounts.Permissions;
import audit.AdminActions;
import audit.AuditEntry;
import bans.BanTools;
import boards.Board;
import boards.BoardTools;
import dao.MessageIndex;
import dao.PostDAO;
import dao.model.Message;
import dao.model.Post;
import dms.DirectMessage;
import dms.DirectMessageTools;
import edits.Edit;
import edits.EditTools;
import friends.FriendTools;
import notifications.NotificationTools;
import polls.Poll;
import polls.PollTools;
import reactions.MessageReactions;
import reactions.Reaction;
import reactions.ReactionStore;
import reactions.ReactionType;
import readtracking.ReadTracker;
import relations.SocialTools;
import scheduled.ScheduledMessages;
import tags.TagTools;
import votes.MessageVotes;
import votes.VoteTools;

import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.TreeMap;
import java.util.UUID;
import java.util.function.Function;

/**
 * HACKATHON (Task 3): every predicted feature's persistence, in LOAD ORDER.
 * Each file is plain CSV of strings/numbers (portable). Enums are saved by name().
 * NOT saved on purpose: notifications and rate-limit/duplicate windows (temporary),
 * mentions and the search index (rebuilt automatically when messages load).
 */
public final class Features {
	private Features() {}

	private static final Function<String, UUID> ID = UUID::fromString;

	public static List<PersistentFeature> all() {
		List<PersistentFeature> f = new ArrayList<>();

		// ---- message flags (need posts + messages, which DataManager loads first) ----
		f.add(new SimpleFeature("deleted", 2, () -> postFlagRows(Post::getDeletedMessageIds),
				row -> withMessage(row, (post, m) -> post.setDeleted(m, true)), () -> {}));
		f.add(new SimpleFeature("pinned", 2, Features::pinnedRows,
				row -> withMessage(row, (post, m) -> post.setPinned(m, true)), () -> {}));
		f.add(new SimpleFeature("locked", 1, Features::lockedRows,
				row -> { Post p = PostDAO.getInstance().get(new Post(ID.apply(row[0]))); if (p != null) p.setLocked(true); }, () -> {}));

		// ---- per-message, per-user data ----
		f.add(new SimpleFeature("reactions", 4, Features::reactionRows,
				row -> ReactionStore.getInstance().forMessage(ID.apply(row[0]))
						.add(new Reaction(ID.apply(row[0]), ID.apply(row[1]), ReactionType.valueOf(row[2]), Long.parseLong(row[3]))),
				ReactionStore.getInstance()::clear));
		f.add(new SimpleFeature("votes", 4, Features::voteRows,
				row -> VoteTools.restore(ID.apply(row[0]), ID.apply(row[1]), Integer.parseInt(row[2]), Long.parseLong(row[3])),
				VoteTools::clear));
		f.add(new SimpleFeature("edits", 4, Features::editRows,
				row -> EditTools.restore(ID.apply(row[0]), new Edit(row[2], Long.parseLong(row[3]))), EditTools::clear));

		// ---- relations: one generic class for all of them ----
		f.add(new RelationFeature<>("follows", SocialTools.follows(), ID, ID));
		f.add(new RelationFeature<>("blocks", SocialTools.blocks(), ID, ID));
		f.add(new RelationFeature<>("bookmarks", SocialTools.bookmarks(), ID, ID));
		f.add(new RelationFeature<>("postlikes", SocialTools.postLikes(), ID, ID));
		f.add(new RelationFeature<>("subscriptions", NotificationTools.subscriptions(), ID, ID));
		f.add(new RelationFeature<>("friendrequests", FriendTools.requests(), ID, ID));
		f.add(new RelationFeature<>("friends", FriendTools.friends(), ID, ID));
		f.add(new RelationFeature<>("tags", TagTools.store(), ID, s -> s));

		// ---- polls: options first (built in afterLoad), then votes ----
		Map<UUID, List<String[]>> pendingPolls = new TreeMap<>();
		f.add(new SimpleFeature("polls", 5, Features::pollRows,
				row -> pendingPolls.computeIfAbsent(ID.apply(row[0]), k -> new ArrayList<>()).add(row),
				() -> { PollTools.clear(); pendingPolls.clear(); },
				() -> { buildPolls(pendingPolls); pendingPolls.clear(); }));
		f.add(new SimpleFeature("pollvotes", 3, Features::pollVoteRows,
				row -> PollTools.restoreVote(ID.apply(row[0]), ID.apply(row[1]), Integer.parseInt(row[2])), () -> {}));

		// ---- per-user data ----
		f.add(new SimpleFeature("readmarkers", 3, Features::readRows,
				row -> ReadTracker.restore(ID.apply(row[0]), ID.apply(row[1]), Long.parseLong(row[2])), ReadTracker::clear));
		f.add(new SimpleFeature("bans", 2, () -> BanTools.all().entrySet().stream()
				.map(e -> new String[] {e.getKey().toString(), e.getValue().toString()}).iterator(),
				row -> BanTools.restore(ID.apply(row[0]), Long.parseLong(row[1])), BanTools::clear));
		f.add(new SimpleFeature("moderators", 1, () -> Permissions.moderators().stream().map(u -> new String[] {u.toString()}).iterator(),
				row -> Permissions.restore(ID.apply(row[0])), Permissions::clear));

		// ---- other stores ----
		f.add(new SimpleFeature("directmessages", 5, () -> DirectMessageTools.all().stream()
				.map(d -> new String[] {d.id().toString(), d.from().toString(), d.to().toString(), Long.toString(d.timestamp()), d.text()}).iterator(),
				row -> DirectMessageTools.restore(new DirectMessage(ID.apply(row[0]), ID.apply(row[1]), ID.apply(row[2]), Long.parseLong(row[3]), row[4])),
				DirectMessageTools::clear));
		f.add(new SimpleFeature("scheduled", 5, () -> ScheduledMessages.pending().stream()
				.map(m -> new String[] {m.id().toString(), m.poster().toString(), m.thread().toString(), Long.toString(m.timestamp()), m.message()}).iterator(),
				row -> ScheduledMessages.restore(new Message(ID.apply(row[0]), ID.apply(row[1]), ID.apply(row[2]), Long.parseLong(row[3]), row[4])),
				ScheduledMessages::clear));
		f.add(new SimpleFeature("auditlog", 3, () -> AdminActions.getLog().stream()
				.map(e -> new String[] {e.admin().toString(), e.action(), Long.toString(e.timestamp())}).iterator(),
				row -> AdminActions.restoreLog(new AuditEntry(ID.apply(row[0]), row[1], Long.parseLong(row[2]))), AdminActions::clear));
		f.add(new SimpleFeature("boards", 2, () -> BoardTools.boardRows().iterator(),
				row -> BoardTools.createBoard(row[0], row[1]), BoardTools::clear));
		f.add(new SimpleFeature("boardposts", 2, () -> BoardTools.postRows().iterator(),
				row -> { Board b = BoardTools.root().find(row[0]); if (b != null) b.addPost(ID.apply(row[1])); }, () -> {}));
		f.add(new SimpleFeature("replies", 2, () -> threads.ThreadTools.allLinks().entrySet().stream()
				.map(e -> new String[] {e.getValue().toString(), e.getKey().toString()}).iterator(),
				row -> threads.ThreadTools.link(ID.apply(row[0]), ID.apply(row[1])), threads.ThreadTools::clear));
		f.add(new SimpleFeature("privateposts", 1, () -> access.AccessTools.privatePosts().stream().map(p -> new String[] {p.toString()}).iterator(),
				row -> access.AccessTools.restorePrivate(ID.apply(row[0])), access.AccessTools::clear));
		f.add(new RelationFeature<>("invites", access.AccessTools.invites(), ID, ID));
		return f;
	}

	// ------------------------------------------------------------ row builders

	private static Iterator<String[]> postFlagRows(Function<Post, java.util.Set<UUID>> flags) {
		List<String[]> rows = new ArrayList<>();
		for (Iterator<Post> it = PostDAO.getInstance().getAll(); it.hasNext(); ) {
			Post post = it.next();
			for (UUID id : flags.apply(post)) rows.add(new String[] {id.toString(), post.id.toString()});
		}
		return rows.iterator();
	}

	private static Iterator<String[]> pinnedRows() {
		List<String[]> rows = new ArrayList<>();
		for (Iterator<Post> it = PostDAO.getInstance().getAll(); it.hasNext(); ) {
			Post post = it.next();
			for (Iterator<Message> pins = post.getPinnedMessages(); pins.hasNext(); )
				rows.add(new String[] {pins.next().id().toString(), post.id.toString()});
		}
		return rows.iterator();
	}

	private static Iterator<String[]> lockedRows() {
		List<String[]> rows = new ArrayList<>();
		for (Iterator<Post> it = PostDAO.getInstance().getAll(); it.hasNext(); ) {
			Post post = it.next();
			if (post.isLocked()) rows.add(new String[] {post.id.toString()});
		}
		return rows.iterator();
	}

	private static Iterator<String[]> reactionRows() {
		List<String[]> rows = new ArrayList<>();
		for (MessageReactions group : ReactionStore.getInstance().all())
			for (Reaction r : group.getReactions())
				rows.add(new String[] {r.message().toString(), r.user().toString(), r.type().name(), Long.toString(r.timestamp())});
		return rows.iterator();
	}

	private static Iterator<String[]> voteRows() {
		List<String[]> rows = new ArrayList<>();
		for (MessageVotes votes : VoteTools.all())
			for (Map.Entry<UUID, Integer> v : votes.getVotes().entrySet())
				rows.add(new String[] {votes.getMessageId().toString(), v.getKey().toString(), v.getValue().toString(), Long.toString(votes.latestTimestamp())});
		return rows.iterator();
	}

	private static Iterator<String[]> editRows() {
		List<String[]> rows = new ArrayList<>();
		for (Map.Entry<UUID, List<Edit>> e : EditTools.all().entrySet())
			for (int i = 0; i < e.getValue().size(); i++) // index column keeps the order readable for other programs
				rows.add(new String[] {e.getKey().toString(), Integer.toString(i), e.getValue().get(i).text(), Long.toString(e.getValue().get(i).editedAt())});
		return rows.iterator();
	}

	// one row per OPTION (CSV has fixed columns, a poll has a variable number of options)
	private static Iterator<String[]> pollRows() {
		List<String[]> rows = new ArrayList<>();
		for (Poll poll : PollTools.all())
			for (int i = 0; i < poll.getOptions().size(); i++)
				rows.add(new String[] {poll.getId().toString(), poll.getPost().toString(), poll.getCreator().toString(), Integer.toString(i), poll.getOptions().get(i)});
		return rows.iterator();
	}

	private static void buildPolls(Map<UUID, List<String[]>> rowsByPoll) {
		for (Map.Entry<UUID, List<String[]>> e : rowsByPoll.entrySet()) {
			List<String[]> rows = new ArrayList<>(e.getValue());
			rows.sort((x, y) -> Integer.compare(Integer.parseInt(x[3]), Integer.parseInt(y[3])));
			List<String> options = new ArrayList<>();
			for (String[] row : rows) options.add(row[4]);
			String[] first = rows.get(0);
			PollTools.restore(new Poll(e.getKey(), ID.apply(first[1]), ID.apply(first[2]), options));
		}
	}

	private static Iterator<String[]> pollVoteRows() {
		List<String[]> rows = new ArrayList<>();
		for (Poll poll : PollTools.all())
			for (Map.Entry<UUID, Integer> v : poll.getVotes().entrySet())
				rows.add(new String[] {poll.getId().toString(), v.getKey().toString(), v.getValue().toString()});
		return rows.iterator();
	}

	private static Iterator<String[]> readRows() {
		List<String[]> rows = new ArrayList<>();
		for (Map.Entry<UUID, Map<UUID, Long>> user : ReadTracker.all().entrySet())
			for (Map.Entry<UUID, Long> post : user.getValue().entrySet())
				rows.add(new String[] {user.getKey().toString(), post.getKey().toString(), post.getValue().toString()});
		return rows.iterator();
	}

	// finds the message by id and hands it to the action together with its post
	private static void withMessage(String[] row, java.util.function.BiConsumer<Post, Message> action) {
		Message m = MessageIndex.getInstance().get(ID.apply(row[0]));
		Post post = PostDAO.getInstance().get(new Post(ID.apply(row[1])));
		if (m != null && post != null) action.accept(post, m);
	}
}
```

The `DataManager` in section 5 already contains the loop (`features` / `featurePipelines`).

## 7. Alternative format: JSON with no library

Because `DataPipeline` only talks to a `FormattedFactory<String[]>`, switching **every** file to JSON is one line
in `DataManager.pipeline(...)` (tested, all round-trips still pass):

```java
// CSV:
return new DataPipeline<>(io, new CSVFormattedFactory(new CSVFormat(columns)), serializer, file);
// JSON:
return new DataPipeline<>(io, new JSONFormattedFactory(columns), serializer, file);
```

Output: `[ ["id","member","alice","pw"], ["id2","admin","bob","pw2"] ]`

**`app/src/persistentdata/formatted/JSONFormattedFactory.java`** (full file)

```java
package persistentdata.formatted;

import java.io.Reader;
import java.io.Writer;

/**
 * HACKATHON (Task 3 alternative): JSON instead of CSV, with NO library.
 * Because DataPipeline only talks to FormattedFactory<String[]>, swapping CSV for JSON
 * is ONE line in DataManager:  new CSVFormattedFactory(new CSVFormat(n))  ->  new JSONFormattedFactory(n)
 * File shape (an array of rows, each row an array of strings):
 *   [
 *   ["id1","member","alice","pw"],
 *   ["id2","admin","bob","pw2"]
 *   ]
 */
public class JSONFormattedFactory implements FormattedFactory<String[]> {
	private final int columns;

	public JSONFormattedFactory(int columns) {
		this.columns = columns;
	}

	@Override
	public FormattedWriter<String[]> writer(Writer documentWriter) {
		return new JSONWriter(documentWriter, columns);
	}

	@Override
	public FormattedReader<String[]> reader(Reader documentReader) {
		return new JSONReader(documentReader, columns);
	}
}
```

**`app/src/persistentdata/formatted/JSONWriter.java`** (full file)

```java
package persistentdata.formatted;

import persistentdata.PersistentDataException;

import java.io.IOException;
import java.io.Writer;

/**
 * HACKATHON: writes rows as a JSON array of string arrays.
 * putHeader writes "[", putFooter writes "]" (this is why DataPipeline must call them).
 */
public class JSONWriter implements FormattedWriter<String[]> {
	private final Writer writer;
	private final int columns;
	private boolean firstRow = true;

	public JSONWriter(Writer writer, int columns) {
		this.writer = writer;
		this.columns = columns;
	}

	@Override
	public void putHeader() {
		write("[");
	}

	@Override
	public void putNext(String[] row) {
		if (row.length != columns) throw new PersistentDataException("Expected " + columns + " columns, got " + row.length);
		StringBuilder line = new StringBuilder(firstRow ? "\n[" : ",\n[");
		firstRow = false;
		for (int i = 0; i < row.length; i++) {
			if (i > 0) line.append(',');
			line.append(quote(row[i]));
		}
		write(line.append(']').toString());
	}

	@Override
	public void putFooter() {
		write("\n]\n");
	}

	// JSON string escaping: backslash, quote and control characters
	static String quote(String value) {
		StringBuilder out = new StringBuilder("\"");
		for (char c : value.toCharArray()) {
			switch (c) {
				case '"' -> out.append("\\\"");
				case '\\' -> out.append("\\\\");
				case '\n' -> out.append("\\n");
				case '\r' -> out.append("\\r");
				case '\t' -> out.append("\\t");
				default -> {
					if (c < 0x20) out.append(String.format("\\u%04x", (int) c));
					else out.append(c);
				}
			}
		}
		return out.append('"').toString();
	}

	private void write(String text) {
		try {
			writer.write(text);
		} catch (IOException e) {
			throw new PersistentDataException(e.getMessage());
		}
	}
}
```

**`app/src/persistentdata/formatted/JSONReader.java`** (full file)

```java
package persistentdata.formatted;

import persistentdata.PersistentDataException;

import java.io.IOException;
import java.io.PushbackReader;
import java.io.Reader;
import java.util.ArrayList;
import java.util.List;

/**
 * HACKATHON: reads the format written by JSONWriter: [ [ "a", "b" ], ... ]
 * A tiny hand-written parser (no libraries). Whitespace between tokens is ignored.
 */
public class JSONReader implements FormattedReader<String[]> {
	private final PushbackReader reader;
	private final int columns;
	private boolean started = false; // have we consumed the outer '[' yet?
	private boolean finished = false; // have we seen the outer ']'?

	public JSONReader(Reader reader, int columns) {
		this.reader = new PushbackReader(reader);
		this.columns = columns;
	}

	@Override
	public boolean hasNext() {
		if (finished) return false;
		if (!started) {
			int c = skipWhitespace();
			if (c == -1) { finished = true; return false; } // empty file = no rows
			expect(c, '[');
			started = true;
		}
		int c = skipWhitespace();
		if (c == ',') c = skipWhitespace();
		if (c == ']') { finished = true; return false; }
		if (c == -1) throw new PersistentDataException("Unexpected end of JSON");
		unread(c);
		return true;
	}

	@Override
	public String[] getNext() {
		if (!hasNext()) throw new PersistentDataException("No more rows");
		expect(skipWhitespace(), '[');
		List<String> row = new ArrayList<>();
		int c = skipWhitespace();
		while (c != ']') {
			if (c == ',') c = skipWhitespace();
			expect(c, '"');
			row.add(readString());
			c = skipWhitespace();
		}
		if (row.size() != columns) throw new PersistentDataException("Expected " + columns + " columns, got " + row.size());
		return row.toArray(new String[0]);
	}

	// reads the rest of a string after its opening quote, undoing the escapes
	private String readString() {
		StringBuilder out = new StringBuilder();
		while (true) {
			int c = read();
			if (c == -1) throw new PersistentDataException("Unterminated string");
			if (c == '"') return out.toString();
			if (c != '\\') { out.append((char) c); continue; }
			int e = read();
			switch (e) {
				case '"' -> out.append('"');
				case '\\' -> out.append('\\');
				case '/' -> out.append('/');
				case 'n' -> out.append('\n');
				case 'r' -> out.append('\r');
				case 't' -> out.append('\t');
				case 'b' -> out.append('\b');
				case 'f' -> out.append('\f');
				case 'u' -> {
					char[] hex = new char[4];
					for (int i = 0; i < 4; i++) hex[i] = (char) read();
					out.append((char) Integer.parseInt(new String(hex), 16));
				}
				default -> throw new PersistentDataException("Bad escape \\" + (char) e);
			}
		}
	}

	private int skipWhitespace() {
		int c;
		do { c = read(); } while (c == ' ' || c == '\n' || c == '\r' || c == '\t');
		return c;
	}

	private void expect(int actual, char expected) {
		if (actual != expected) throw new PersistentDataException("Expected '" + expected + "' but found '" + (char) actual + "'");
	}

	private int read() {
		try { return reader.read(); } catch (IOException e) { throw new PersistentDataException(e.getMessage()); }
	}

	private void unread(int c) {
		try { reader.unread(c); } catch (IOException e) { throw new PersistentDataException(e.getMessage()); }
	}
}
```

> Your IntelliJ project lists a `gson-2.8.6` library. If the spec allows libraries, Gson is quicker, but the
> marker's machine may not have it, and the code **must compile** to get any marks. The hand-written version is safe.

## 8. A test you can paste to prove persistence works

This is a shortened template; the full tested versions are `verify/PersistenceVerify.java` and `verify/AllPersistenceVerify.java` in `verified-code.zip`.

```java
// in-memory "disk": file name -> contents. Pass it to new DataManager(io).
static class MemoryIO implements IOFactory {
	final Map<String, StringWriter> files = new HashMap<>();
	public Writer writer(String f) { StringWriter w = new StringWriter(); files.put(f, w); return w; }
	public Reader reader(String f) { StringWriter w = files.get(f); return w == null ? null : new StringReader(w.toString()); }
}

@Test public void roundTrip() {
	// ...set up users, posts, messages, reports, hidden flags...
	DataManager dm = new DataManager(new MemoryIO());
	dm.writeAll();
	UserDAO.getInstance().clear(); PostDAO.getInstance().clear(); ReportStore.getInstance().clear();
	dm.readAll();
	assertTrue(ModerationTools.hasReported(message.id(), user.id()));
	assertTrue(PostDAO.getInstance().get(new Post(post.id)).isHidden(message.id()));
}
```

## 9. Trap checklist (Task 3)

- [ ] Fix the two base bugs in section 2 first, or nothing loads.
- [ ] `readAll` clears **every** store (DAOs and your new stores) before loading.
- [ ] Load order: users, posts, messages, then anything that refers to messages.
- [ ] Removed reports are not written (only active ones).
- [ ] Enums saved by `name()`, never `ordinal()` (reordering the enum would corrupt old files).
- [ ] Text with `,` `"` or newline round-trips (the CSV writer quotes it; test it).
- [ ] Do not change the existing file formats unless you must (other programs read them).
- [ ] Every new class has javadoc stating its schema (column order).

## 10. How this was verified

| Check | Result |
|---|---|
| Round trip in memory: users (password with `,` `"` newline), posts (incl. null poster/topic), messages (null poster, multi-line text), reports, hidden flag, OLDEST order after reload | pass |
| Round trip on disk into a folder that did not exist yet | pass |
| Loading with no files gives an empty app (no crash) | pass |
| Removed reports are not saved | pass |
| Same four tests with JSON swapped in | pass |
| JSON reader/writer: escapes, unicode, control chars, empty file, wrong column count, `hasNext` repeatable | pass |
| Course `CSVReaderTests` / `CSVWriterTests` after the fix | 15/15 and 11/11 |
| **Every feature**: set up all 27 feature stores, save, wipe everything, load, check each one (incl. poll option with a comma, DM with a newline, undo in the audit log, mentions + search rebuilt) | pass, 32 files written |
| The same all-features round trip with JSON swapped in | pass |
