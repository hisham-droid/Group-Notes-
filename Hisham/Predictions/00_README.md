# Hackathon prediction pack (read this first)

Made from the practice hackathon (`comp2100-hackathon-practice`, moderation tools) and your miniproject.
They are **predictions**: tomorrow's theme will differ, but the task shapes repeat.

**Every prediction in every file now has complete code**, and every piece of code was compiled and tested.
There are no "recipe only" rows left.

| File | What is inside |
|---|---|
| `Task1prediction.md` | 19 core-feature themes, each with full code: reports, reactions, votes, follow / block / bookmark / post likes, subscriptions + notifications, tags, polls, read tracking, bans, friends, mentions, edit history + undo, account deletion and changes, moderator role, rate limiting; plus the O(1) index fixes every theme needs |
| `Task2prediction.md` | 14 model/tree predictions: hide, soft delete, lock (+ State pattern hook), pin, hard delete, edit/replace, move, AVL delete, rank / floor / ceiling, time ranges + counts + pagination, `BSTree.getRange`, red-black tree, two "visible view" designs |
| `Task3prediction.md` | the two base bugs that break all loading, portable CSV for every feature (27 feature files via one `PersistentFeature` list), refactor / code-smell table, JSON with no library |
| `Task4prediction.md` | Iterator + Factory (+ Strategy) queries: reports (7 strategies), reactions, votes, posts, trending, leaderboards, reusable `TopKIterator` |
| `Task5prediction.md` | 7 test classes (white-box branch maps + black-box), 39 broken versions all caught, JUnit cheat sheet |
| `ExtraPredictions.md` | search (inverted index + trie), statistics, censor, audit log + undo, DMs, scheduled messages, boards, links + duplicates, replies and threads, private posts, login lockout + history, merge / split posts, emoji + markdown |
| `HackathonIndex.pdf` | **the one-page-per-task index to keep open during the hackathon**: keywords from the question -> where the code is |
| `UMLprediction.md` + `uml.png` + `uml_reactions.png` | the group UML task: notation, requirement checklist, two rendered examples with Mermaid source |
| `HACKATHON_PREDICTIONS.md` (one folder up) | the earlier one-page overview |
| `verified-code.zip` | the complete working code (`app/src` 122 files, about 7,100 lines, + `app/test` + `verify`) that every file quotes from |

## What "verified" means here

Compiled with JDK 21 and run with JUnit 4.13.2 (the course versions) on top of the practice base:

* **201 tests pass** in 26 test classes: the 7 Task 5 classes, 12 verification classes, and 7 of your miniproject's own course test classes.
* **39 deliberately broken implementations**, every one caught by the Task 5 tests.
* Trees compared with `java.util.TreeSet` on random data (AVL, red-black, sorted array list, BST).
* Save + load of **every** feature at once, in CSV and in JSON.
* Speed: 20,000 messages x 4 report operations; 200,000 AVL inserts + 100,000 deletes; 500,000 red-black inserts: each well under 3 s.
* `SortedDataEfficiencyTests` fails only for BSTree / SortedArrayList on the massive tests, exactly as week 4 intends.

Bugs the tests found while building this (now fixed, and worth remembering tomorrow): `ReactionType.valueOf(null)`
throws `NullPointerException` (spec wants `IllegalArgumentException`); an edit re-inserts a message, which would have
sent duplicate notifications and left the search index stale.

## Order to work tomorrow (team of 4 to 5)

1. **First 10 minutes, together:** read all tasks; agree on the data model (store classes, maps, where extra state
   lives, file formats). Task 3 and Task 4 depend on Task 1's classes.
2. **Base fixes first** (Task1 section 3, Task3 section 2): indexes, null-safe comparator, CSV `hasNext`, IO date check.
   Push them before splitting up so everyone builds on them.
3. Split tasks. Commit small, pull often, push to `main`.
4. **The final commit on `main` must compile**, or the whole team gets zero for code. Compile and run the tests before the last push.
5. Last 20 minutes: javadoc, remove dead code and smells (20% quality on most tasks), `uml.png`.

## Before you rely on this

Check the hackathon's rules on prepared notes and generative AI. Your `Integrity.md` asks you to declare AI use,
so say how these notes were made if you use them.
