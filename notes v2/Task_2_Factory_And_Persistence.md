# Task 2 — Poll/quiz factory and persistence

This is coordinated with Task 3's `ChoiceAttachment`, `Poll`, and `Quiz` classes. Implement **Task 3 first**, or agree on their constructors/interfaces before merging.

## 1. Replace `attachments/ChoiceAttachmentFactory.java`

```java
package attachments;
/** Factory hides which concrete class callers instantiate. */
public final class ChoiceAttachmentFactory {
    public Attachment makeAttachment(String question,String[] options,int correctAnswer) {
        if (options == null || options.length < 2)
            throw new IllegalArgumentException("At least two options required");
        // Valid answer index => Quiz; invalid => Poll (per task sheet).
        if (correctAnswer >= 0 && correctAnswer < options.length)
            return new Quiz(question,options,correctAnswer);
        return new Poll(question,options);
    }
}
```

**Constructor validation** happens in `ChoiceAttachment`: nonblank question, at least two nonblank option labels. The correct-answer index is zero-based. `correctAnswer=-1` creates a poll; it is **not** an exception.

## 2. Persist attachment metadata

Add a separate `AttachmentSerializer` implementing the existing `Serializer<T,String[]>` interface. Store one row **per post/attachment association**, not one row per unique attachment, so that shared images remain attached to multiple posts. Suggested six columns:

| Field | Example |
|---|---|
| postUUID | post UUID |
| type | `image`, `poll`, `quiz` |
| attachmentUUID | stable UUID |
| text | Base64-encoded caption/question |
| options | comma-separated Base64-encoded labels |
| correct | quiz index, otherwise blank |

```java
// In AttachmentSerializer, these helpers make CSV-safe string payloads:
private static String encode(String text) {
    return java.util.Base64.getEncoder().encodeToString(
        text.getBytes(java.nio.charset.StandardCharsets.UTF_8));
}
private static String decode(String text) {
    return new String(java.util.Base64.getDecoder().decode(text),
        java.nio.charset.StandardCharsets.UTF_8);
}

// Example serialization logic for an existing attachment:
String[] serialize(UUID postId, Attachment a) {
    if (a instanceof Image image)
        return new String[]{postId.toString(), "image", a.getUUID().toString(),
                            encode(image.getCaption()), "", ""};
    ChoiceAttachment choice = (ChoiceAttachment) a;
    String labels = String.join(",", java.util.Arrays.stream(choice.getOptions())
        .map(this::encode).toArray(String[]::new));
    return new String[]{postId.toString(), a instanceof Quiz ? "quiz" : "poll",
        a.getUUID().toString(), encode(choice.getQuestion()), labels,
        a instanceof Quiz q ? Integer.toString(q.getCorrectAnswer()) : ""};
}
```

This snippet belongs **inside your serializer class**, with imports and an appropriate `Entry` record for `(postId, attachment)`; it is not a standalone Java file. Deserialization reverses each conversion and uses `Image.load(uuid,caption)`, `new Poll(uuid,question,options)`, or `new Quiz(uuid,question,options,correct)`.

## 3. Persist responses in separate rows

Store each `(attachmentUUID,userUUID,optionIndex)` in `saved/responses.txt`. Do not serialize `HashMap.toString()`: it is not a stable data format.

```java
// Serialization:
String[] row = {attachmentId.toString(), userId.toString(), Integer.toString(index)};
// Deserialization:
UUID attachmentId = UUID.fromString(row[0]);
UUID userId = UUID.fromString(row[1]);
int index = Integer.parseInt(row[2]);
// Once the choice object is found by attachmentId:
choice.restoreAnswer(userId, index); // Rebuilds counts as well as map.
```

## 4. Integrate into `persistentdata/DataManager.java`

```java
// Add alongside existing user/post/message pipelines:
private final DataPipeline<AttachmentEntry,String[]> attachmentPipeline =
    new DataPipeline<>(IO,new CSVFormattedFactory(new CSVFormat(6)),
                       new AttachmentSerializer(),"attachments");
private final DataPipeline<ResponseEntry,String[]> responsePipeline =
    new DataPipeline<>(IO,new CSVFormattedFactory(new CSVFormat(3)),
                       new ResponseSerializer(),"responses");
```

`AttachmentEntry` and `ResponseEntry` are **new record types you must define** (e.g. nested inside their serializers). The code above is an integration pattern, not a drop-in compiling block until those types and serializers exist.

**Write order:** users, posts, messages, attachment associations, responses. **Read order:** users, posts, messages, attachment associations, responses. Keep a `Map<UUID,Attachment>` while reading to reuse objects and a `Map<UUID,ChoiceAttachment>` to connect response rows to the correct poll/quiz. Deduplicate response output by attachment UUID if an attachment belongs to multiple posts.

## 5. Critical bugs in the uploaded persistence layer

- `persistentdata/formatted/CSVReader.hasNext()` **always returns `true`**, so `DataPipeline.readTo()` can run past EOF. Fix using a one-record lookahead (`hasNext` loads the next record, `getNext` consumes it), and ensure EOF after a final unterminated row is supported.
- `ComputerIOFactory.reader()` deliberately throws after **1 December 2025** (`System.currentTimeMillis() > 1764620398492L`), which prevents all loading now. Remove that date check.
- `ComputerIOFactory.writer()` silently returns null when `saved/` does not exist. Create the directory before opening files and propagate real I/O errors.
- Keep attachment text/option labels lossless across commas, quotes, Unicode, and line breaks.

## 6. End-to-end verification

1. Create users, two posts, one image shared across posts, a poll, and a quiz.
2. Vote with multiple users, change a poll vote, and attempt a duplicate quiz answer.
3. Call `DataManager.writeAll()`; restart or clear in-memory DAOs; call `readAll()`.
4. Check post associations, attachment UUIDs, PNG pixels, captions, options, correct answer, vote counts, and duplicate quiz protection after reload.

**Status:** This guide provides factory code and the persistence design/code patterns; the complete serializers, CSV repair, and DataManager wiring still need implementation and testing. Do not present it as a fully verified solution.
