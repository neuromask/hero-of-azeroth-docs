# 🎖 Reputations: the journal of one character

`/…/reputations` is the faction journal of the character whose page is open — the tab beside Collections
and Achievements. It is drawn from `server/utils/reputations-data.json`, the atlas
`npm run refresh:reputations` builds: Blizzard's own reputation-faction tree (the expansions, the headings
under them — a side, a garrison, a cluster of city factions — and the factions in those), the side that may
raise each one, and the steps of every ladder (the classic eight, a renown faction's levels, a delve
companion's hundred ranks). The twelve factions no heading of that tree reached — the Shattrath and
Steamwheedle reputations of the Burning Crusade — are collected under one group of the shelf's own.

The journal has two readings, which the switch under its bar asks for. This character's own is the one it opens
on: the factions their side may raise, counted against what that side can reach. The account's reading is the
account's journal rather than this character's — the pool of ladders *some* character of the account has finished
(`server/utils/accountPool`) marks and counts the rows, and **both sides are drawn** for it, because a faction an
alt of the other faction finished is part of the account and a number the reader could not find on the list below
would be no number at all. Its total is then every ladder of the atlas, which is the same pair of figures the
tile above the shelf prints, so the shelf and the tile agree. Every stage and every span stays the character's
own: the pool holds maxima and nothing else.

A row *is* the measure: one block, the faction's name set into its left and the stage it stands at — with
the numbers inside that stage (`Почтение · 751 / 21 000`) — set into its right, and the fill standing behind
the words. Two hundred of them therefore read down the page as one column of measures rather than a wall of
stripes, and hovering one names the stage and the span that stage covers (`Honored · 9 000 – 21 000`) — the
steps of every ladder travel with the journal as a ledger, read once per ladder and shared by every faction
standing on it. The fills are translucent and muted by state rather than bright everywhere: emerald where a
faction is finished, gold while a reputation is being raised, sky while a renown ladder is being climbed, and
a finished stage prints no numbers at all — which is what keeps a journal that is mostly done from turning
into a rash. The search box and the two switches — the factions not started, the factions finished — filter
rows the journal already holds, so none of them spends a request or moves a count.

The journal wears the collections shelf's own furniture: the pinned rail of quick anchors down its left
(one chip per expansion, the one being read lit, a click gliding the page to it) and the same blocks a
subcategory is drawn in, so the two shelves are walked the same way. A journal narrowed to a single
expansion drops the rail and takes the whole width.

Blizzard's own placeholders are left out of the atlas entirely: `[PH]` names a patch has not shipped and
`[DNT]` marks a test entry (`[DNT] AC Major Faction Child Renown Test` was filed under the Maruuk Centaur),
and neither is a reputation a player can raise — neither is drawn, and neither is counted in a total.

### Why this page and the tile above it disagree

The journal is read two ways, and the tile is the account's. With the character's own reading on, the shelf is
what somebody standing in the game on that character sees: the factions the other side may raise are not on it at
all, and no other character of the account adds to it - so it counts fewer factions than the tile, and the note
under its bar says so in the reader's own words. Flip the switch to the account's reading and the two agree: the
account's pool of finished factions is what the rows are marked and counted from, both sides are drawn for it
(a faction an alt of the other faction finished is part of the account), and the shelf's total becomes every
ladder of the atlas - the very numbers the tile prints (`reputationReachableTotal`,
`server/utils/reputations`).

