# COMP2100 Hackathon: predicted themes and cheat notes (open book)

Built from: the practice hackathon (moderation tools), the miniproject manuals (weeks 2-8), and a read of the base code.
These are predictions, not leaks. Skim section 1 for the template, section 2 for every theme **and where its tested code is**, sections 3-9 for tools you will reuse.

> **Updated:** every theme in section 2 now has complete, tested code in the `Predictions` folder (49 of 51; the 2 exceptions say why). Section 3's traps now say where each one is fixed.

---------------------------------------------------------------------

## 1. The template (this repeats, only the theme changes)

Practice hackathon structure:
- A new package with a static tools class (here `moderation.ModerationTools`) full of `// TODO: task N` stubs.
- One stub added to an existing model class (here `Post.getVisibleMessages`).
- Two empty test classes (`...AddReportTests`, `...GetReportsTests`).
- `uml.png` placeholder for the group UML task.
- "Do not modify enum X" line (the `ReactionType` enum is mentioned but missing: leftover from a reactions version).

| Task | Skill | Typical marks |
|---|---|---|
| 1 | Core feature: add / remove / has. Data structure choice + performance | 50% behaviour, 30% efficiency, 20% quality |
| 2 | State on a model class + a derived `SortedData` view (trees) | 40 / 40 / 20 |
| 3 | Persistence + refactoring. Portable format (not Java serialization) | 60% persistence, 40% refactor/code smells |
| 4 | Query method with a strategy string. MUST use Iterator + Factory | 40 / 40 / 20 |
| 5 | JUnit4: white-box branch coverage on one method, black-box vs faulty impls on another | 40 / 40 / 20 |
| UML | 5 classes, private/protected/public field, 2 static + 2 non-static methods, composition + aggregation + association | 50 content / 50 clarity |

Spec habits to expect:
- "If X or Y does not exist, do nothing and return false."
- Duplicate action returns false (already reported / already reacted).
- Remove returns false if there is nothing to remove, and no record of removal is kept.
- Strategy strings ("OLDEST", "MOST") with invalid input => throw an exception.
- Positive `amount` only. Return fewer if not enough. No zero-count items. No duplicates in output.
- Ties: "arbitrary order" (do not waste time on tie-breaking).
- "Performance is important" applies ONLY to new code, not old code.
- Data "must persist across runs" and be "portable to other languages": CSV or JSON/XML.

---------------------------------------------------------------------

## 2. Every predicted theme and WHERE ITS CODE IS

All code is compiled and tested (201 tests, see `Predictions/00_README.md`). `T1` = Task1prediction.md, `T2` = Task2prediction.md, `T3` = Task3prediction.md, `T4` = Task4prediction.md, `T5` = Task5prediction.md, `Extra` = ExtraPredictions.md (all in the `Predictions` folder). `s5` = section 5.
Use `Predictions/HackathonIndex.pdf` during the hackathon: it has the same list sorted by the keywords a question would use.

### A. Social interactions on messages

| # | Theme | Words in the question | Code (file + section) | Main classes / methods |
|---|---|---|---|---|
| 1 | Reactions | react, reaction, like/love/laugh, ReactionType, emoji reaction | T1 s5; T4 s4; T5 s5, s8 | ReactionTools.addReaction / changeReaction / countReactions / getTopMessages |
| 2 | Up/down votes, score | vote, upvote, downvote, score, rating, karma | T1 s7; T4 s6; T5 s6, s9 | VoteTools.vote / getScore / getTopMessages |
| 3 | Reports / moderation (the practice) | report, flag, abusive, moderate, moderator review | T1 s4; T4 s3, s5; T5 s2, s3 | ModerationTools.addReport / getReportedMessages; ReportQueries |
| 4 | Hide / soft delete / restore | hide, hidden, visible, visibility, delete, soft delete, restore, remove message | T2 s4, s5 | Post.setHidden / setDeleted / getVisibleMessages; ContentTools |
| 5 | Edit + edit history (+ undo) | edit, modify message, history, revision, previous version | T1 s15 | EditTools.editMessage / getHistory / undoLastEdit; Post.replaceMessage |
| 6 | Replies to a message, threads, quoting | reply to a message, thread, nested, parent, child, quote | Extra s9 | ThreadTools.reply / getReplies / getThread / quote |
| 7 | @mentions | mention, @username, tagged user | T1 s14 | MentionTools.extractUsernames / getMentions |
| 8 | Pin messages | pin, pinned, sticky, featured, shown first | T2 s5 | ContentTools.setPinned / getMessagesPinnedFirst |
| 9 | Bookmarks / favourites | bookmark, save, saved, favourite | T1 s6 | SocialTools.bookmark / getBookmarks |
| 10 | Polls | poll, option, choice, survey, ballot, results, winner | T1 s10 | PollTools.createPoll / vote / changeVote / getResults / getWinners |

### B. User relationships and access

| # | Theme | Words in the question | Code (file + section) | Main classes / methods |
|---|---|---|---|---|
| 11 | Block / mute users | block, mute, ignore, hide user | T1 s6 | SocialTools.block / isBlocked / getMessagesFor |
| 12 | Follow users | follow, follower, following, feed | T1 s6; T5 s7 | SocialTools.follow / getFollowers |
| 12b | Subscribe + notifications | subscribe, notify, notification, inbox, alert, unread notifications | T1 s8 | NotificationTools.subscribe / getNotifications / markAllRead |
| 13 | Friend requests | friend, friend request, accept, decline | T1 s13 | FriendTools.sendRequest / accept / areFriends |
| 14 | Direct / private messages | direct message, DM, private message, conversation, chat between two users | Extra s5 | DirectMessageTools.send / getConversation / getLatest |
| 15 | Private / invite-only posts | private post, invite, access, who can view, permission to read | Extra s10 | AccessTools.setPrivate / invite / canView |
| 16 | Moderator role, roles | moderator, role, promote, demote, permission | T1 s16 | Permissions.setModerator / canModerate; AccountTools.setRole |
| 17 | Bans with expiry | ban, suspend, suspension, timeout, until | T1 s12 | BanTools.ban / unban / isBanned |
| 18 | Account deletion | delete account, remove user, GDPR, [deleted] | T1 s16 | AccountTools.deleteAccount / displayName |
| 19 | Change password / username | change password, rename, change username | T1 s16 | AccountTools.changePassword / changeUsername |
| 20 | Login lockout + login history | failed login, lockout, attempts, login history, session | Extra s11 | LoginTools.login / isLocked / getHistory / unlock |

### C. Content organisation

| # | Theme | Words in the question | Code (file + section) | Main classes / methods |
|---|---|---|---|---|
| 21 | Tags / hashtags | tag, hashtag, #, label, category label | T1 s9 | TagTools.addTag / getPostsWithTag / getPopularTags |
| 22 | Boards / categories / subforums | board, category, subforum, section, hierarchy | Extra s7 | Board (Composite), BoardTools |
| 23 | Search + autocomplete | search, keyword, find, query, autocomplete, prefix | Extra s1 | SearchTools.search / autocompleteUsername; Trie |
| 24 | Trending / busiest / active posts | trending, hot, popular, recent activity, busiest | T4 s7 | TrendingTools.getTrendingPosts / getTopPosts |
| 25 | Leaderboards / karma | leaderboard, top users, reputation, karma, most active user | T4 s8 | LeaderboardTools.getTopUsers |
| 26 | Statistics | statistics, stats, average, per day, per user, analytics | Extra s2 | StatsTools |
| 27 | Read / unread tracking | read, unread, seen, last read, new since | T1 s11 | ReadTracker.markRead / getUnreadCount |
| 28 | Scheduled / delayed messages | schedule, scheduled, delayed, later, publish at, draft | Extra s6 | ScheduledMessages.schedule / publishDue / cancel |
| 29 | Lock / close a post | lock, locked, close, closed, read-only, archive | T2 s5, s6 | ContentTools.setLocked / reply; MemberState.canReplyTo |
| 30 | Move message / merge / split posts | move, merge, split, combine posts | T2 s12; Extra s12 | MoveTools.moveMessage; PostTools.mergePosts / splitPost |
| 31 | Pagination | page, page size, paginate, scroll, slice of messages | T2 s9 | MessageQueries.getPage |
| 32 | Time-range queries / counts | between, from ... to, since, time window, range, count in period | T2 s9 | MessageQueries.getMessagesBetween / countMessagesBetween |

### D. Content safety

| # | Theme | Words in the question | Code (file + section) | Main classes / methods |
|---|---|---|---|---|
| 33 | Profanity censor / word filter | censor, profanity, swear, filter words, banned words | Extra s3 | Censor, CensorFactory, CensorStrategy |
| 34 | Spam / rate limiting | spam, rate limit, too many, per minute, flood | T1 s17 | RateLimiter.tryAcquire |
| 35 | Duplicate-message detection | duplicate, same message again, repeated | Extra s8 | ContentChecks.isDuplicate |

### E. Infrastructure-flavoured

| # | Theme | Words in the question | Code (file + section) | Main classes / methods |
|---|---|---|---|---|
| 36 | Notifications inbox | notification, inbox, alert | T1 s8 | NotificationTools |
| 37 | Export / import, JSON | export, import, JSON, XML, portable, other languages | T3 s6, s7 | JSONFormattedFactory / JSONWriter / JSONReader; Features |
| 38 | Undo / redo admin actions | undo, redo, revert, command | Extra s4 | AdminActions.run / undo; AdminCommand |
| 39 | O(1) lookup index | efficient, performance, lookup by id, does not exist | T1 s3 | MessageIndex; UserDAO.getByUUID (indexed) |
| 40 | Audit log | audit, log, record of admin actions | Extra s4 | AdminActions.getLog; AuditEntry |
| 41 | Links / emoji / markdown / previews | link, url, attachment, emoji, markdown, preview | Extra s8, s13 | ContentChecks.extractLinks; TextFormat |

### F. Data-structure heavy (Task 2 style, can appear in any theme)

| # | Theme | Words in the question | Code (file + section) | Main classes / methods |
|---|---|---|---|---|
| F1 | AVL delete | delete from tree, remove, rebalance, deletion | T2 s7 | AVLTree.remove; AVLNodeFilled.delete / rebalance |
| F2 | BSTree.getRange (was null) | binary search tree iterator, BSTree | T2 s10 | BSIterator |
| F3 | SortedArrayList backwards + bounds | SortedArrayList, backwards | T2 s7 | SortedArrayList.getRange(..., true) / getAtIndex |
| F4 | Red-black tree | red-black, balanced tree, new SortedData | T2 s11 | RedBlackTree (left-leaning) |
| F5 | Order statistics | rank, position, index of, k-th, how many before | T2 s8, s9 | rank / floor / ceiling; MessageQueries.getPosition |
| F6 | floor / ceiling / successor / predecessor | floor, ceiling, nearest, next, previous | T2 s8 | SortedData.floor / ceiling |
| F7 | Filtered view of a SortedData | view, filter, visible to | T2 s3 | SortedDataViews.filter; Post visible tree |
| F8 | Skip list / heap / hash table / B-tree / Fenwick tree | skip list, heap, hash table, B-tree, segment tree | **not coded**: T2 s11 (copy RedBlackTree's shape) | not built: only if the spec names one. RedBlackTree shows every SortedData method to implement |
| F9 | SortedDataSlice / AVLTreeSlice | slice, pinned, unpinned, shiftForward, shiftBackward | **not coded**: week 6 lab (hidden implementation) | not built: the course hides it; reuse your week 6 AVLSliceTests |

---------------------------------------------------------------------

## 3. Traps in YOUR base code (found by reading it)

1. **No global Message lookup by UUID.** Messages live in `Post.messages` (SortedData). Comparator = (timestamp, thread, poster, id), so you can't `get` by id alone. UUID -> Message needs an index map or a scan. Same for the existence check "if the message does not exist". **Fixed:** `MessageIndex` (T1 s3.2).
2. **`UserDAO.getByUUID` is a linear scan.** `UserDAO.get(new User(username))` is O(log n) by username (case-insensitive). By-UUID existence checks are slow unless you index. **Fixed:** UUID index in `UserDAO` (T1 s3.1).
3. **`PostDAO` compares by UUID only** (`Comparator.comparing(HasUUID::getUUID)`), so `posts.get(new Post(uuid))` is the O(log n) lookup. Posts have `getAtIndex`, ordered by UUID (not by time!) despite the javadoc.
4. **`Message` is an immutable record**, `Post` fields are final. New state must live elsewhere or be copy-on-write.
5. **`Post(UUID)` has null poster/topic**: `PostSerializer` would NPE on those (test posts). `Message` tests use null poster too, which would NPE in `MessageSerializer` and in `MessageComparator` (poster().compareTo). **Fixed:** `Nullable` serializers (T3 s4) + null-safe `MessageComparator` (T1 s3.3).
6. **`DataManager`** hard-codes column counts (4, 3, 5) and `readAll` order matters (users, posts, then messages need posts). `DataPipeline` has unused static `users`/`posts` fields (code smell). CSV cannot hold variable-length lists: use a separate file/rows per item (one row per report/reaction) or JSON. **Fixed:** named constants, one pipeline helper, `PersistentFeature` list (T3 s5, s6).
7. **`ComputerIOFactory.reader`** has a date check that forces `null` (time-bomb), and writes to `saved/%s.txt` (folder must exist). If persistence "doesn't load", look here first. **Fixed:** date check removed, folder created (T3 s2.2). Also `CSVReader.hasNext` always returned true: fixed in T3 s2.1.
8. **`DataPipeline.writeFrom`** never calls `putHeader` / `putFooter`. **Fixed:** T3 s5.
9. **`StateManager`** is a static global: tests share state. Reset with logout; clear DAOs in `@Before`.
10. **`SortedDataFactory`** swaps implementations in one line. In the practice base it was the array list; AVL is the fast one. Performance tests may need AVL. **Fixed:** AVL in the factory (T2 s7).
11. **`AVLTree.insert`** returns false on duplicates, notifies listeners on success. `getRange(start, count, backwards)`: negative count = unlimited; null start = from the beginning (or end if backwards).
12. **Generics**: `SortedData<T>` needs a `Comparator<T>`; for derived views build a new comparator, not a new class where possible.
13. **Record equality**: `User` and `Message` records use field equality; two users with the same fields are `equals`.
14. **Admin check**: `user.role() == User.Role.Admin` (AdminState extends MemberState).
15. Test-file caveat: tests are in the default package, so only public API is usable.

---------------------------------------------------------------------

## 4. Data structure cheat sheet (Task 1 performance marks)

| Need | Use | Cost |
|---|---|---|
| exists / lookup by UUID | `HashMap<UUID, X>` | O(1) |
| has user done X to msg | `HashMap<UUID, HashSet<UUID>>` or composite key record(msg,user) in a HashSet | O(1) |
| count per category | `EnumMap<Enum,Integer>` or `HashMap<K,Integer>` maintained incrementally | O(1) |
| oldest item | `PriorityQueue` or `TreeMap`/AVL keyed by timestamp (lazy deletion or `remove`) | O(log n) |
| top-K by count | sort or heap of size K; or keep a `SortedData` keyed by (count,id) | O(n log K) |
| ordered by time | `SortedData` (AVL) with comparator | O(log n) |
| remove with ordering | needs AVL delete, or lazy deletion flag | |
| tags / words | `HashMap<String, Set<UUID>>` (inverted index) | O(1) avg |
| prefix search | trie | O(len) |
| sliding window per user | `ArrayDeque<Long>` of timestamps | O(1) amortised |
| FIFO queue of due items | `PriorityQueue` by time | O(log n) |

Rule of thumb for "maximise efficiency": no scans over all messages/users/posts inside add/remove/has. Say so in a comment (quality marks).

---------------------------------------------------------------------

## 5. Task 4 patterns (Iterator + Factory are "mandatory" in the practice)

- **Iterator**: write a class implementing `Iterator<T>` with `hasNext/next`; precompute a lazy position; handle `amount`; no duplicates. Template already in `PostDAO.getAllMessages` (anonymous class) and `AVLIterator` (stack-based).
- **Factory**: `ReportOrderingFactory.create(strategy)` returns the ordering; `ModerationTools.getReportedMessages` throws `IllegalArgumentException` for unknown strings or non-positive amount (T4 s3). Reusable: `util.TopKIterator` (T4 s4).
- **Strategy**: comparator or ordering chosen by string ("OLDEST", "MOST", "NEWEST", "SCORE").
- **Observer**: AVLTree listeners (`SortedDataSubject.onAdd`).
- **Singleton**: DAOs, `MessageComparator`.
- **State**: `userstate` package, or locked/unlocked post.
- **Template**: persistence pipeline (IO -> format -> serializer).
- **Builder**: `AVLTestBuilder` (tests).
- **Decorator / Chain of Responsibility**: filters (censor, visibility).
- **Command / Memento**: undo, edit history.
- **Adapter**: format conversion. **Facade**: `ModerationTools` itself acts as a facade over the module.
- **Composite**: board/category tree, reply tree.

Order of "OLDEST"-type strategies: sort ascending by key; "MOST": sort descending by count. Always filter out zero-count items first.

---------------------------------------------------------------------

## 6. Task 3 persistence checklist

- Portable = CSV (already built), JSON, XML. Avoid Java `Serializable`.
- Add a new `Serializer<T,String[]>` + `DataPipeline` + file name in `DataManager`, wire into `readAll` AND `writeAll`. Cleaner for many features: one `PersistentFeature` per file in a list (T3 s6).
- One row per record (report, reaction, hidden flag); never a list inside a cell (or if so, document the delimiter and escape it).
- Hidden flag: add a column to messages (5 -> 6, update `CSVFormat(6)` and both serializer directions) or a separate "hidden.csv" of UUIDs. The CSV approach says "adding a column breaks existing data", note this in a comment.
- Read order: parents before children (users, posts, messages, then reports/reactions).
- Writer escaping: fields with comma, newline or quote must be quoted (already in `CSVWriter.writeEntry`).
- gson-2.8.6 appears in your IntelliJ module (`.idea`): JSON via Gson might be allowed. Check the spec; if "no libraries", hand-write a tiny JSON writer.
- Refactor / code smell list to scan for: magic numbers (column counts), duplicate code (three near-identical pipelines), unused fields/imports (`DataPipeline` statics, `IOException` import in CSVWriter/Reader), long methods (CSVReader.getNext), static global state, null returns where exceptions/Optional fit, god class (`ModerationTools` doing everything), missing javadoc, public mutable fields (`Post.messages`), primitive obsession.

---------------------------------------------------------------------

## 7. Task 5 testing checklist

White-box (branch coverage of one method, like `addReport`):
- List every `if`/`&&`/`||`/ternary/loop in the method AND helpers you wrote; one test per true/false outcome.
- addReport-style branches: message missing, user missing, duplicate, success (and "both missing" if separately checked).
- Set up real data: add users to `UserDAO`, posts to `PostDAO`, messages via `post.messages.insert(...)`. Reset in `@Before` (`PostDAO.getInstance().clear()`, `UserDAO.getInstance().clear()`).
- Use `Assert.assertTrue/False/Equals`, `@Test(timeout=...)`, `@Test(expected=...)`.
- Do not assume the method is correct: compute expected values by hand.

Black-box (kill faulty implementations of e.g. `getReportedMessages`):
- Invalid strategy string, `amount` = 0 and negative (expect exception); null strategy.
- Amount larger than available; amount 1; exactly equal.
- Zero reports excluded; removed reports excluded (the remove makes the "oldest" change!).
- Same message reported by many users appears once.
- Ordering for both strategies with distinct values; ties allowed to be arbitrary (assert sets or only the unambiguous positions).
- Reports on hidden messages still counted.
- Empty system returns an empty iterator (not null).
- Iterator contract: `hasNext` stable, `next()` after exhaustion throws `NoSuchElementException`.
- Calling twice gives independent iterators.
- Think of typical faults: sort ascending vs descending swapped, off-by-one on amount, counting removed reports, duplicates, mutating shared state, forgetting zero-count filter.

---------------------------------------------------------------------

## 8. UML checklist (group task, 100% in 10 minutes)

Must show: at least five classes incl. the tools class; one private (-), one protected (#), one public (+) field; two static (underlined) and two non-static methods; one composition (filled diamond: part dies with whole, e.g. Tools owns ReportStore), one aggregation (hollow diamond: e.g. Post has Messages), one association (plain line with multiplicity/label).
Make it readable: only the new module + the 2-3 existing classes it touches, interfaces for patterns (`Iterator`, `Factory`), relationships labelled. Photo or screenshot replaces `uml.png`.

---------------------------------------------------------------------

## 9. Time plan for the hackathon (group, open book)

1. First 10 min: skim ALL tasks, agree on the data model together (report/reaction class, which maps, where hidden state lives, persistence format). Decide who owns what because Task 3 (persistence) and Task 4 (queries) depend on Task 1 and 2 data shapes.
2. Code the shared model classes first, push, then split tasks.
3. Commit small, push to `main` (marking = final commit on `main`, must COMPILE: run a compile check before the last push).
4. Last 20 min: javadoc + remove smells (quality marks are 20% on nearly every task), delete dead code, run all tests, update `uml.png`.
