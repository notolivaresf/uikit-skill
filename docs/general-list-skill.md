# General list skill (general1)

Fernando, 28 Oct: the chat-list skill is quite specific to messages; make a copy that fits any list, then test it the same way. `skills/uikit-list/SKILL.md` is a copy of `skills/uikit-chat-list/SKILL.md` (frozen at `v1.0.0`, untouched) with message wording turned into item and row wording. Every rule keeps the behaviour it bought; only the nouns and the chat-only assumptions changed.

Word count: 875 (`wc -w`; the chat-list original is 826, includes frontmatter in both).

| Line | Old wording | New wording | Why |
|---|---|---|---|
| Name, description | `uikit-chat-list`; "long chat or feed list ... live messages pinned to the bottom ... jumping to a message ... (chat rooms, comment threads, logs, timelines)" | `uikit-list`; "long, fast-scrolling list screen ... feeds, inboxes, catalogs, logs, timelines, comment threads, chats ... live items ... jumping to an item ... variable row heights" | Triggers on any long list, chat is one example among many |
| Title and tagline | "UIKit chat list. A list that grows at both ends" | "UIKit list. A list that grows at its ends" | Many lists grow at one end only |
| 1 Edit menu | "return `UIMenu(children: [reply] + ...)`"; "the bubble's context menu" | "If rows show selectable text, return `[custom] + ...`"; "the row's context menu" | Only applies when rows have selectable text; no bubble in other lists |
| 2 TextKit 1 | "Read-only message text" | "Read-only row text" | Same behaviour, any row |
| 3 Layout | "rows keyed by message id" | "rows keyed by the item's stable id" | Every table has an id; says it must be stable |
| 4 Measure | "Measure each message once ... keep the height by message id ... lose the pin ... at the bubble's inner width" | "Measure each row once ... by item id ... lose the position ... at the content's inner width; a row without text measures its fixed parts the same way" | Rows are not always messages or text; "pin" is chat-only; non-text rows needed an instruction |
| 4 neighbour | "When older rows arrive, drop the old first row's height too" | "When a page arrives next to existing rows, drop the neighbouring row's height too" | A page can arrive at either end |
| 6 Live changes | "your own sends apply at once"; "insertItems for new ids"; "per message"; "Keep 'pinned to the bottom' as state ... the layout's target offset is the bottom" | "changes the user makes apply at once"; "per item"; "If the list is anchored to an end (the bottom of a chat or log, the top of a feed), keep 'pinned' as state ... target offset is that end" | Bottom anchoring is a chat and log trait; kept as a conditional with the same state rules |
| 7 Trim | "trimming k rows from the top" | "trimming k rows from the front" | Direction depends on the anchor end |
| 8 Older above | "Older messages above" ... "returns the bottom while pinned" | "A page loaded before the first visible row" ... "returns the pinned end while pinned" | Prepend is general; the pinned end is not always the bottom |
| 9 Jump | "Jump:" "about 10 rows above, 30 below" | "Jump to an item:" "about 10 rows before, 30 after" | Any item, not a message; "above/below" depends on direction |
| Unchanged | rules 5, 10, 11 and the closing paragraph (system symbols, no test-list, rerun only failed tests, timeouts, close simulator) | same | Nothing message-specific; behaviours kept verbatim |

Also: the `.claude-plugin` descriptions mention the new skill. `skills/uikit-chat-list` and tag `v1.0.0` are not changed. Tag for this version: `v1.1.0-general1`.
