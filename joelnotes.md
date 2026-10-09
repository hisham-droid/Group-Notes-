I'll unpack the hackathon zip and read Task 4 and the tasks it builds on.

Now the skeleton and the existing pieces Task 4 would touch.

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
