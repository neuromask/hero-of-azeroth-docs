# 🎯 Collections: where the numbers come from

Every mount, pet, toy and decoration in the game, in SimpleArmory's own tree: a bucket per expansion or
event, the sources under it ("Raid Drop", "Vendor", a zone), the items under those. The server lays the
character's own ids over that tree, so a shelf looks the same for everybody and the counts belong to
the character.

Two rules make a shelf read **lower** than the counter in the game — and truer for it:

- 🚫 **Retired items are hidden.** An item the game has done away with is drawn only for a character
  who already holds it: a rarity to show off rather than a goal to chase. It is left out of the count
  too, which is what keeps a 1,300-item shelf from reading 1,404.
- ⏳ **Unreleased items wait.** The Trading Post's coming stock and the mounts of a patch that is not
  live are not drawn at all.

Both are SimpleArmory's own settings, and both are yours as well — the two switches under the shelf's
bar, with a note beside them that says the same:

| Switch | What it uncovers |
| --- | --- |
| **Unobtainable** | Every retired item the site knows, drawn as a grey tile. Items that were never released stay hidden, exactly as they do there |
| **Upcoming** | The handful of items Blizzard has in the files but has not shipped |

With a switch on, a shelf reads exactly what SimpleArmory reads with its own setting on — the numbers
were matched item by item, not approximated.

### By account or by character

A WoW collection is shared by every character of a Battle.net account, so the account is the truer unit for a
collector. Every collections page carries the choice: **per character**, which filters every shelf by that
hero's faction and counts only what they can use, or **account-wide**, which draws the shelf from the account's
own ids and counts both sides.

For the account that holds the character, those ids are Blizzard's account-wide collections, read with the
reader's own token and kept in the site's pool (`server/utils/accountPool`). A guest has no pool, but Blizzard
still hands the character endpoint the account-wide lists for pets, toys and decorations, so a guest's account
view changes those three; mounts and titles keep the character's list, and a line under the switch says that
their account view needs a Battle.net sign-in. The account view is the one drawn without the faction filter —
an account plays both sides — and every shelf and card says which reading it answered with. A collection the
site has never read for that account falls back to the character's journal and says so. The five tiles of the
collections page follow the same choice, with the line under the running total naming whose numbers they are.

**Whose number a tile prints and the note under it are one answer**: the account's where the account answered that kind, and the
character's own reading where it did not. An account answers a kind in one of two ways (`server/utils/accountPool`): its pool has
read the ids, or its own total stands on its characters' readings. The second way matters for the three figures Blizzard publishes
per character only — titles, quests and recipes — where an account's own number is the highest its roster has been read at until the
pool has read every character in full. Marking only the pool left a merged account printing its own figure under the note "per
character": two of its characters showed the same quests, and the one holding fewer carried a note claiming the number was its own.

### Titles

Titles are the fifth shelf. Their catalogue is SimpleArmory's `titles.json`, whose rows name the achievement or
quest that grants a title in `id` and the title itself in `titleId`; the atlas stores the latter, because that is
the id Blizzard's title index and a character's own title list speak in. Blizzard publishes that index, so the
names are read in both languages, and a title tile links to its own `title=` page on Wowhead. A character's title
shelf follows its faction, exactly as its mounts do; an account's shelf unions the titles of its roster through the
pool (`syncPoolFigures`), because Blizzard has no account titles endpoint. The profile's title figure now carries
the shelf's own count and denominator, so its bar, its number and the page print one pair.

### Why they differ from the game's own counter

They are meant to: the sources count different sets, and every one of them is right.

| Source | Reads | What it counts |
| --- | --- | --- |
| **In game · Wowhead tracker** | `1215 / 1334` | Everything the character has learned, over the journal the client shows for its faction |
| **Blizzard Armoury** | `1215 / 1387` | The same list, over the Armoury's own journal size — a number that belongs to the character and moves with the patch |
| **Blizzard API** | `1215` learned · `1117` usable | The raw list, ~98 of them unusable mounts of the other faction |
| **Hero of Azeroth — a shelf** | `1163 / 1302` | What this character can still get: SimpleArmory's numbers, to the item |
| **dataforazeroth** | `1162 / 1303` | Their own list, keyed by id rather than by row |

*(one Alliance character, patch 12.1.0 — the absolute numbers move with every patch, the reasons do not)*

The stubborn ±1 against other sites is data, not arithmetic: SimpleArmory counts the **rows** of its own
file, and one mount in it (`Deathtusk Felboar`) sits twice under the same source — a character who owns
it reads one higher there than on any site that keys by id. This project reproduces its rows, because
the tree is its tree.

---

