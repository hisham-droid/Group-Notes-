# Task 3 — Large-scale voting (commented solution)

## Updated-repo differences

`Quiz` already extends `Attachment`, but its constructor omits the required `super(UUID)` call. `Poll` has no fields and does not extend `Attachment`. Do **not** paste the old independent `HashMap` solution into both classes; share the voting logic below.

## Add `attachments/ChoiceAttachment.java`

```java
package attachments;
import dao.model.User;
import java.util.*;

public abstract class ChoiceAttachment extends Attachment {
    private final String question;
    private final String[] options;
    private final Map<UUID,Integer> answers = new HashMap<>();
    private final int[] counts; // Cached totals make getResult O(1).

    protected ChoiceAttachment(UUID id, String question, String[] options) {
        super(id);
        if (question == null || question.isBlank()) throw new IllegalArgumentException("Blank question");
        if (options == null || options.length < 2) throw new IllegalArgumentException("Need 2+ options");
        for (String s : options)
            if (s == null || s.isBlank()) throw new IllegalArgumentException("Blank option");
        this.question = question;
        this.options = options.clone(); // Defensive copy.
        counts = new int[options.length];
    }
    public String getQuestion() { return question; }
    public String[] getOptions() { return options.clone(); }
    public Map<UUID,Integer> getAnswers() { return Map.copyOf(answers); }
    protected boolean valid(int i) { return i >= 0 && i < counts.length; }
    protected UUID userId(User user) {
        Objects.requireNonNull(user, "user");
        return Objects.requireNonNull(user.getUUID(), "user ID");
    }
    protected boolean hasAnswered(UUID id) { return answers.containsKey(id); }
    protected Integer answerOf(UUID id) { return answers.get(id); }
    protected void replace(UUID id, Integer index) {
        Integer previous = answers.remove(id);
        if (previous != null) counts[previous]--; // Undo previous vote.
        if (index != null) { answers.put(id,index); counts[index]++; }
    }
    public void restoreAnswer(UUID id, int index) {
        Objects.requireNonNull(id);
        if (!valid(index)) throw new IllegalArgumentException("Corrupt saved vote");
        replace(id,index); // Rebuild cached counts when loading.
    }
    public float getResult(int index) {
        if (!valid(index) || answers.isEmpty()) return 0.0f;
        return (float) counts[index] / answers.size(); // Avoid integer division.
    }
    public abstract void makeSelection(User user,int index);
}
```

## Replace `attachments/Poll.java`

```java
package attachments;
import dao.model.User;
import java.util.UUID;
public final class Poll extends ChoiceAttachment {
    public Poll(String q,String[] options) { this(UUID.randomUUID(),q,options); }
    public Poll(UUID id,String q,String[] options) { super(id,q,options); }
    @Override public void makeSelection(User user,int index) {
        replace(userId(user),valid(index) ? index : null); // Invalid withdraws vote.
    }
}
```

## Replace `attachments/Quiz.java`

```java
package attachments;
import dao.model.User;
import java.util.UUID;
public final class Quiz extends ChoiceAttachment {
    private final int correctAnswer;
    public Quiz(String q,String[] options,int correct) {
        this(UUID.randomUUID(),q,options,correct);
    }
    public Quiz(UUID id,String q,String[] options,int correct) {
        super(id,q,options); // Required: Attachment constructor takes UUID.
        if (!valid(correct)) throw new IllegalArgumentException("Invalid correct answer");
        correctAnswer = correct;
    }
    public int getCorrectAnswer() { return correctAnswer; }
    @Override public void makeSelection(User user,int index) {
        UUID id = userId(user);
        if (hasAnswered(id)) throw new IllegalStateException("Quiz already answered");
        if (!valid(index)) throw new IllegalArgumentException("Invalid quiz option");
        replace(id,index);
    }
    @Override public int getInteractionCorrect(User user) {
        Integer index = answerOf(userId(user));
        return index == null ? -1 : index == correctAnswer ? 1 : 0;
    }
}
```

## Complexity and edge cases

| Operation | Expected time | Why |
|---|---|---|
| First vote | O(1) | HashMap put, array increment |
| Poll change/removal | O(1) | HashMap remove/put, two counter changes |
| Quiz duplicate check | O(1) | HashMap containsKey |
| getResult | O(1) | Direct count / map size |

`HashMap` uses O(number of voters) memory; counters use O(number of options). Runtime is **expected**, not a guaranteed 0.01 seconds. The code is not synchronized for concurrent voting.

**Check:** 3 voters choose 0,1,1 -> result(1)=2/3. First voter changes to 1 -> result(1)=1. Remove voter with invalid poll index -> denominator decreases. Quiz second selection throws. Invalid getResult returns 0.0.
