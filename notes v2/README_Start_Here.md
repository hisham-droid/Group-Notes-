# COMP2100 Mock — five commented task guides

These guides are based on **`hackathon-main (1).zip`** and share the same proposed architecture. They are **study/reference implementations**, not a tested complete project. In particular, Task 2 persistence includes integration patterns requiring new serializers and repairs to the provided CSV reader; it is not fully drop-in.

1. `Task_1_Image_Attachments.md` — image storage, Post reference, persistence strategy.
2. `Task_2_Factory_And_Persistence.md` — factory, metadata/response serialization, DataManager integration, existing persistence bugs.
3. `Task_3_Voting_Data_Structures.md` — shared choice base, Poll/Quiz code, O(1) voting.
4. `Task_4_JUnit_Tests.md` — 20-call JUnit test suite and condition coverage.
5. `Task_5_Code_Quality_SOLID.md` — QuizMarker refactor, SOLID answers, UML plan, review checklist.

**Recommended merge order:** agree on `Attachment` and `ChoiceAttachment` interfaces; Task 3 + Task 2 factory; Task 1; Task 2 persistence; Task 4; Task 5 and UML. Remember the updated ZIP's Quiz constructor does not compile as supplied because it omits `super(UUID)`.
