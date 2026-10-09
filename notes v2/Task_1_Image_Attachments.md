# Task 1 — Image attachments (commented solution)

Based on the updated ZIP: `Post.getAttachment()` and `setAttachment()` are TODOs, `Image` does not extend `Attachment`, and persistence currently handles only users, posts, and messages. **All three need changes.**

## 1. Change `dao/model/Post.java`

Add a field and replace the two TODOs:

```java
private Attachment attachment; // Reference, NOT image bytes; only one per post.

public Attachment getAttachment() {
    return attachment; // May be null when no attachment exists.
}
public void setAttachment(Attachment attachment) {
    this.attachment = attachment; // Replaces the old one; null removes it.
}
```

Two posts can point to the same Image instance. Do not save an image inside `setAttachment`.

## 2. Replace `attachments/Image.java`

```java
package attachments;
import javax.imageio.ImageIO;
import java.awt.Graphics2D;
import java.awt.image.BufferedImage;
import java.io.IOException;
import java.nio.file.*;
import java.util.*;

public final class Image extends Attachment {
    private static final Path DIR = Path.of("saved", "img");
    private final String caption;
    private final BufferedImage pixels;

    public Image(java.awt.Image source, String caption) {
        this(UUID.randomUUID(), source, caption, true);
    }
    private Image(UUID id, java.awt.Image source, String caption, boolean save) {
        super(id); // Attachment owns stable UUID.
        Objects.requireNonNull(source, "source image");
        this.caption = Objects.requireNonNull(caption, "caption");
        int w = source.getWidth(null), h = source.getHeight(null);
        if (w <= 0 || h <= 0) throw new IllegalArgumentException("Bad dimensions");
        pixels = new BufferedImage(w, h, BufferedImage.TYPE_INT_ARGB);
        Graphics2D g = pixels.createGraphics();
        try { g.drawImage(source, 0, 0, null); }
        finally { g.dispose(); } // Release graphics resources.
        if (save) saveToDisk(); // Save at construction, not at attachment.
    }
    private Path path() { return DIR.resolve(getUUID() + ".png"); }
    private void saveToDisk() {
        try {
            Files.createDirectories(DIR);
            // CREATE_NEW prevents accidental overwrites.
            try (var stream = Files.newOutputStream(path(), StandardOpenOption.CREATE_NEW)) {
                if (!ImageIO.write(pixels, "png", stream)) throw new IOException("No PNG writer");
            }
        } catch (IOException ex) {
            throw new IllegalStateException("Cannot save image " + getUUID(), ex);
        }
    }
    public static Image load(UUID id, String caption) {
        try {
            BufferedImage b = ImageIO.read(DIR.resolve(id + ".png").toFile());
            if (b == null) throw new IOException("Invalid image file");
            return new Image(id, b, caption, false); // No duplicate write.
        } catch (IOException ex) {
            throw new IllegalStateException("Cannot load image " + id, ex);
        }
    }
    public String getCaption() { return caption; }
    public BufferedImage getPixels() { return pixels; }
}
```

## 3. Persist references and captions

The existing `PostSerializer` writes **3 columns only** (`id`, `poster`, `topic`). Keep those columns, and add a **separate attachments pipeline** so existing posts still load.

Recommended `saved/attachments.txt` six-column rows:

`postUUID, type, attachmentUUID, captionOrQuestion, encodedOptions, correctAnswer`

For images: `postUUID,image,imageUUID,caption,,`. Use the existing `DataPipeline` and `CSVFormattedFactory(new CSVFormat(6))`, plus an `AttachmentSerializer` (see Task 2). On `DataManager.readAll()`, load posts first, then attachments and call `post.setAttachment(image)`. On `writeAll()`, iterate posts and write a row for each attached post. **Reuse the same loaded object for repeated attachment UUIDs** using `Map<UUID,Attachment>`.

Avoid a CSV schema that assumes captions never contain commas/newlines. Use Base64 for free-text fields, or a properly working CSV escaping layer.

## 4. Checks

- New Image writes `saved/img/<UUID>.png` immediately.
- Two posts share the same `Image` object and one PNG file.
- Replacing an attachment changes only the reference, not the old image file.
- Save, restart, reload: identical UUID, caption, and pixel contents.
- An unreadable image should throw an informative exception.

**Dependency:** `Attachment` must have a UUID constructor (Task 2/5). The supplied `ComputerIOFactory` has a date-dependent read failure and `CSVReader.hasNext()` always returns true; see Task 2 warnings. **This guide is reference code, not a tested end-to-end patch.**
