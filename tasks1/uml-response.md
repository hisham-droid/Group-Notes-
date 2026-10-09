I'll make this a Markdown file with a Mermaid class diagram. GitHub draws Mermaid natively, so it shows up as a picture right on the repo page. I'll also render it to `uml.png`, since the spec asks you to replace that file. First, a check that the rendering tools are available:

The top-to-bottom version is much clearer: `ModerationTools` at the top, the classes it uses in the middle, and the data records at the bottom. I'll use that one.

I've made the UML diagram as two files:

- **`UML.md`** is the one to read on GitHub. Put it in your repo's root folder. GitHub draws the diagram as a picture right on the page, and you can zoom in on the classes. Under the diagram there's a key for the symbols, a checklist showing where each requirement is met, and a short summary of how Tasks 1–4 fit together. That makes it handy for revision in the exam.
- **`uml.png`** is the same diagram as an image. Use it to replace the `uml.png` file in your repo, which is what the spec asks you to submit.

**Requirements covered:**

| Requirement | In the diagram |
|---|---|
| At least 5 classes, including `ModerationTools` | 18 classes |
| Private, protected and public fields | private: `ReportStore.reportsByMessage`; protected: `DAO.data`; public: `Post.id` |
| Two static and two non-static methods | static (underlined): `addReport`, `ReportOrderingFactory.create`; non-static: `ReportStore.add`, `MessageReports.count` |
| Composition (filled diamond) | `ReportStore` → `MessageReports`, `MessageReports` → `Report`, `Post` → `Message` |
| Aggregation (hollow diamond) | `MessageReports` → `Message`, `PostDAO` → `Post`, `UserDAO` → `User` |
| Association (solid arrow) | `ModerationTools` → `ReportStore`, `UserDAO`, `PostDAO` |

The Factory pattern (`ReportOrderingFactory`) and the Iterator pattern (`ReportedMessageIterator` implementing `Iterator<Message>`) are both visible in the diagram.

The diagram matches the code from Tasks 1–5 in this chat. If your group's code ends up different (other class names, an extra method), change the text in `UML.md`; GitHub redraws the picture automatically. `uml.png` won't change with it, though, so I'd need to re-render the image for you.
