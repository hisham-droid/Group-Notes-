## What Task 4 depends on

**Task 1 (reporting) is the blocker.** `getReportedMessages` has to read the reports, so it needs Task 1's report storage before it can return anything real. For each reported message you need three things:

- which messages currently have at least one active report
- how many active reports each message has (for `"MOST"`)
- the timestamp of each message's oldest active report (for `"OLDEST"`)

You don't need to wait for Task 1 to be finished, only for its design to be agreed. Settle with the Task 1 person early on how reports are stored and what you can call to read them. For example, something like `Map<UUID, Collection<Report>>`, where each `Report` holds a user and a timestamp. Make sure removed reports really are deleted, not just flagged. Otherwise "oldest *non-removed* report" and "*active* reports" both break. Once that's agreed, you can write your code against it in parallel.

**Not dependencies:**

- **Task 2 (hiding):** The spec never says to leave hidden messages out, and Task 1 says reports stay on hidden messages. So you can ignore hiding.
- **Task 3 (persistence):** It saves whatever Task 1 and Task 2 hold in memory. It doesn't change how you read it.

**What depends on you:** Task 5's second half, the black-box tests for `getReportedMessages`. Those can be written from the spec straight away, but they can't pass until your code works. That makes you a mid-chain bottleneck: blocked by Task 1, blocking Task 5.

## Basic knowledge you'd need

**Java concepts**

- **The `Iterator<T>` interface:** `hasNext()` and `next()`, and how to write your own class that implements it. The project already has two you can copy: `sorteddata/avltree/AVLIterator.java` and `sorteddata/sortedarraylist/SortedArrayListIterator.java`. The `count` field in those is a good model for stopping after `amount` items.
- **`Comparator<T>`:** writing comparators and chaining them for tie-breaks. `dao/MessageComparator.java` is a good reference.
- **Collections:** `HashMap`, `ArrayList`, `PriorityQueue` and `Collections.sort`. You'll be gathering messages, sorting them and taking the first `amount`.
- **Exceptions:** throwing `IllegalArgumentException` when `strategy` isn't `"OLDEST"` or `"MOST"`, or when `amount <= 0`. Check this before doing any other work.
- **Records and `UUID`:** `Message` is a record (`message.id()`, `message.timestamp()`, etc.). Reports will likely be keyed by message UUID, so you'll need to turn a UUID back into a `Message`. Check how `PostDAO` does lookups.

**Design patterns (40% of the marks, both are mandatory)**

- **Factory:** an object or static method that decides which concrete thing to build, so the caller doesn't have to. `sorteddata/SortedDataFactory.java` is the example already in the codebase. A natural fit here is a factory that takes the strategy string and returns the matching comparator or iterator, throwing on unknown strings.
- **Iterator:** your function must return an `Iterator<Message>`. The most "pattern-like" approach is your own iterator class, for example one that walks the sorted results and stops after `amount` items. Wrapping `list.iterator()` will likely score lower.
- **Strategy (bonus vocabulary):** the parameter is literally called `strategy`, and `"OLDEST"` and `"MOST"` are two interchangeable ordering strategies. Being able to name this helps in the design discussion and the UML.

**One trap to know about**

If you put messages into the project's `SortedData` (via `SortedDataFactory`), it treats any two items the comparator calls equal as duplicates and drops one. A comparator that only compares report counts would silently lose messages that tie. Always add a final tie-break, such as `message.id()`. Even though the spec says ties can come back in any order, they all still have to come back.

**Also worth knowing:** basic Big-O, so you can explain why your approach is efficient. Sorting all n reported messages costs O(n log n). Keeping a size-`amount` heap costs O(n log k) instead.


-------------------------


## What Task 4 is, in plain words

Moderators want a to-do list: "show me the reported messages I should look at first." You write one function:

```java
Iterator<Message> getReportedMessages(String strategy, int amount)
```

- **`strategy`** says how to rank the messages:
  - `"OLDEST"`: the message whose oldest active report has the earliest timestamp comes first ("who's been waiting longest?").
  - `"MOST"`: the message with the most active reports comes first ("what's the biggest problem?").
- **`amount`** is how many messages to give back at most.
- You return an **Iterator**, which the moderator steps through one message at a time.

**Rules**

- If `strategy` isn't `"OLDEST"` or `"MOST"`, or `amount` is 0 or negative, throw an exception.
- Never include a message with 0 active reports. Removed reports don't count.
- Each message appears at most once, even if 5 people reported it.
- If there are fewer reported messages than `amount`, return what you have.
- Ties can come out in any order.

**Example.** Say the reports currently look like this:

| Message | Active reports (timestamps) | Count | Oldest |
|---|---|---|---|
| A | 50, 90 | 2 | 50 |
| B | 10 | 1 | 10 |
| C | 70, 80, 95 | 3 | 70 |
| D | none (its report was removed) | 0 | n/a |

- `getReportedMessages("OLDEST", 2)` returns B, A (oldest reports at 10 and 50).
- `getReportedMessages("MOST", 2)` returns C, A (3 reports, then 2).
- `getReportedMessages("MOST", 10)` returns C, A, B. Only 3 exist, and D is skipped.
- `getReportedMessages("NEWEST", 2)` throws an exception.

## How you'd ideally do it

**Step 0: agree on the data with the Task 1 person.** Your code only reads what Task 1 stores. The cleanest shape is one object per reported message, for example:

```java
class MessageReports {
    Message message;                  // the actual Message object
    Map<UUID, Long> reports;          // user UUID -> report timestamp (active only)

    int count()        { return reports.size(); }
    long oldest()      { return Collections.min(reports.values()); }
}
```

Task 1 keeps a `Map<UUID, MessageReports>` keyed by message UUID. One detail matters: store the `Message` object itself, not just its UUID. The codebase has no "find a message by UUID" method (`PostDAO` can only loop over every message), so without it you'd be searching the whole forum. Task 1 has to find the message anyway to check that it exists, so it can just keep it.

**Step 1: validate the input.** Check `amount > 0` first. Then let the factory reject a bad strategy (step 2). Throw `IllegalArgumentException`.

**Step 2: Factory, to pick the ordering.** Make a small factory that turns the string into a comparator:

```java
public class ReportOrderingFactory {
    public static Comparator<MessageReports> create(String strategy) {
        switch (strategy) {
            case "OLDEST": return Comparator.comparingLong(MessageReports::oldest);
            case "MOST":   return Comparator.comparingInt(MessageReports::count).reversed();
            default: throw new IllegalArgumentException("Unknown strategy: " + strategy);
        }
    }
}
```

`.reversed()` is there because "MOST" wants big numbers first. Also pass `null` through the same check if you like: `switch` on a `null` string throws `NullPointerException`, so test for `null` first if you want a cleaner error.

**Step 3: gather and sort.** Take every `MessageReports` with `count() > 0`, put them in a list and sort with the comparator. Because you're working per message, not per report, each message naturally appears only once.

**Step 4: Iterator, to hand them out.** Write your own iterator that walks the sorted list and stops after `amount` items:

```java
public class ReportedMessageIterator implements Iterator<Message> {
    private final List<MessageReports> sorted;
    private final int limit;
    private int index = 0;

    public ReportedMessageIterator(List<MessageReports> sorted, int amount) {
        this.sorted = sorted;
        this.limit = Math.min(amount, sorted.size());
    }

    @Override public boolean hasNext() { return index < limit; }

    @Override public Message next() {
        if (!hasNext()) throw new NoSuchElementException();
        return sorted.get(index++).message;
    }
}
```

**Step 5: put it together.**

```java
public static Iterator<Message> getReportedMessages(String strategy, int amount) {
    if (amount <= 0) throw new IllegalArgumentException("amount must be positive");
    Comparator<MessageReports> order = ReportOrderingFactory.create(strategy);

    List<MessageReports> active = new ArrayList<>();
    for (MessageReports r : reports.values())      // Task 1's map
        if (r.count() > 0) active.add(r);

    active.sort(order);
    return new ReportedMessageIterator(active, amount);
}
```

Short and readable, with both patterns clearly visible. That covers the 40% for the patterns and helps with the 20% for code quality.

## Things that lose marks

- **Counting removed reports.** Make sure Task 1 actually deletes a report on `removeReport`, or your counts and "oldest" values will be wrong.
- **Leaving 0-report messages in.** A message whose only report got removed must not show up.
- **Using the project's `SortedData` with a comparator that only checks count or time.** It treats ties as duplicates and silently drops one. Plain `ArrayList` + `sort` avoids this. If you do use `SortedData`, add `.thenComparing(r -> r.message.id())` as a tie-break.
- **Wrong direction on `"MOST"`.** Most reports must come first, so it's descending.
- **Forgetting the exception for bad input.** Task 5's tests will almost certainly check this.

**Optional extra for efficiency:** if you want to impress on performance, use a `PriorityQueue` of size `amount` instead of sorting everything. That's O(n log k) instead of O(n log n). The sort version above is perfectly fine for this task though.
