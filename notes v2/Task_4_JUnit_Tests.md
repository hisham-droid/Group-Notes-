# Task 4 — JUnit white-box tests (20-call budget)

The updated repository has `app/test/ChoiceAttachmentTests.java` containing only `package attachments;`. Use the Task 3 implementation in the companion guide. The following uses **JUnit 4** (`org.junit.Test`, `org.junit.Assert`).

## Replace `app/test/ChoiceAttachmentTests.java`

```java
package attachments;
import dao.model.User;
import org.junit.Test;
import static org.junit.Assert.*;
import java.util.UUID;

public class ChoiceAttachmentTests {
    private User user() { return new User(UUID.randomUUID()); }

    @Test public void pollCountsChangesAndWithdrawals() {
        Poll p = new Poll("Pick",new String[]{"A","B"});
        User a=user(), b=user();
        assertEquals(0f,p.getResult(0),0.0001f); // 1: empty votes.
        p.makeSelection(a,0);                      // 2: new vote.
        p.makeSelection(b,1);                      // 3: another option.
        assertEquals(0.5f,p.getResult(0),0.0001f);// 4: denominator two.
        p.makeSelection(a,1);                      // 5: change vote.
        assertEquals(1f,p.getResult(1),0.0001f);  // 6: both choose B.
        p.makeSelection(b,-1);                     // 7: withdraw via invalid.
        assertEquals(0f,p.getResult(-1),0.0001f); // 8: invalid query.
        assertEquals(1f,p.getResult(1),0.0001f);  // 9: remaining A chooses B.
    }

    @Test public void quizRejectsDuplicatesAndInvalidSelections() {
        Quiz q = new Quiz("Pick",new String[]{"A","B"},1);
        User a=user(), b=user();
        assertEquals(0f,q.getResult(100),0.0001f); // 10: invalid query.
        try {
            q.makeSelection(a,-1);                  // 11: invalid first vote.
            fail("Expected IllegalArgumentException");
        } catch (IllegalArgumentException expected) { /* Correct */ }
        q.makeSelection(a,1);                       // 12: valid vote.
        try {
            q.makeSelection(a,0);                   // 13: duplicate.
            fail("Expected IllegalStateException");
        } catch (IllegalStateException expected) { /* Correct */ }
        q.makeSelection(b,0);                       // 14: other user's vote.
        assertEquals(0.5f,q.getResult(1),0.0001f); // 15: half correct.
        assertEquals(0.5f,q.getResult(0),0.0001f); // 16: half incorrect.
    }

    @Test public void pollUpperBoundaryAndEmptyAgain() {
        Poll p=new Poll("Pick",new String[]{"A","B"});
        User a=user();
        p.makeSelection(a,1);                      // 17: last valid option.
        p.makeSelection(a,2);                      // 18: index == length withdraws.
        assertEquals(0f,p.getResult(1),0.0001f);  // 19: empty after removal.
        assertEquals(0f,p.getResult(2),0.0001f);  // 20: invalid high index.
    }
}
```

**Total calls to the four target methods: 20**, counting calls inside `try` blocks. Constructor calls, `assertEquals`, and helper calls are not target-method calls.

## White-box condition coverage

| Condition | True case | False case |
|---|---|---|
| `valid(index)` poll | index 0/1 | index -1/2 |
| `old != null` in replace | change/remove | first vote |
| `index != null` in replace | valid vote | withdrawal |
| `hasAnswered(id)` quiz | duplicate | first answer |
| `valid(index)` quiz | valid | invalid |
| `!valid(index)` result | invalid | valid |
| `answers.isEmpty()` result | empty | nonempty |

**Important:** This targets the explicit decisions in our Task 3 implementation. A strict instrumentation tool may count short-circuit operands, validation code, or generated methods differently; run a coverage report and adapt. The test suite is designed to catch wrong denominators, integer division, duplicate voting, wrong boundary checks, stale counters, and incorrect withdrawal behavior. It has **not** been executed here.

## Run locally

Set up JUnit 4 in IntelliJ, mark `app/test` as Test Sources Root, right-click `ChoiceAttachmentTests` -> Run with Coverage. If your project uses JUnit 5, change imports/exception assertions accordingly. Keep the **20-call maximum** in mind when adding tests.
