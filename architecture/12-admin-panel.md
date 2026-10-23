# 🛠 The admin panel

A very small panel, and deliberately so: it exists for the handful of figures the site *states* rather than reads,
and it is built as a store rather than as a form, so that the next thing an operator needs is a section rather
than a migration.

**The panel is one file per tab.** `/admin` is a shell - the tab strip in the navigation row, the sign-in gate and the
timer that keeps the diagnostics fresh (`app/composables/adminDiagnostics`) - and each of its seven tabs is a component of
its own under `app/components/admin/`, in the order the panel is read in: System (the process, the database, the log and
the two maintenance actions), Scan (character scanning and the guild roster crawl, **one card each**), Guest accounts (the
guests, the pass and the backfill), Statistics (the site's own four figures, drawn by the front page's `SiteStatistics`
too), Season (the Mythic+ boundary and the leaderboard it feeds), Content (the score's weights, the stated maxima and the
stored copy) and
Rights (the administrator list, for the owner alone). A tab that is a file is a tab that can be found, and the prose
explaining a tab lives with it.

The Scan tab's two cards are the two jobs the site schedules, and they are two cards rather than one with two halves
because the jobs have nothing to do with each other: different sources, different budgets, different reasons to be on.
Each carries its own state, its own cadence, its own way to run one pass by hand and its own save button - the endpoint
behind them takes both jobs in one body (the scan is required, and a body that says nothing about the crawl leaves it as
it is), so the write is one and only the *answer* is printed under the button that was pressed. A shared card had also
left the crawl's own switch under the scan's label, which is what a shared card invites.

**Every number those two cards offer is bounded in one place** (`shared/data/automationSettings`), because a cadence or a
batch the jobs were not built for is a way to spend the site's Blizzard quota quietly for ever: a pass reads between one and
fifty characters (the shipped batch is **ten**, and the ceiling is a *burst* limit rather than a daily one - fifty full reads
is six hundred requests, and what keeps the site inside the hour is its own guard, `server/utils/blizzardBudget`, which holds
thirty thousand of the thirty-six thousand Blizzard allows), the cadence is one of six intervals from three to sixty minutes,
and a crawl is never allowed a bigger burst than the scan. Raising the ceiling is what lets a quiet site drain a stale
population overnight; it does not change what a copy of the site does on its first day, and the panel is where the bigger
number is asked for.

The System tab is where the process says what it is doing, and its last section is the **log** - the last hundred entries of
the in-process buffer, newest first, with a **strip of tabs by kind** (`DIAGNOSTIC_CATEGORIES`: Blizzard, the sync pass, the
database, an administrator's action) and a count on each, so that an operator reads *where* the failures are before opening
one. A kind nothing was ever filed under has no tab at all, rather than a tab that opens onto an empty list. Twenty lines are
shown and **"show N more"** adds twenty at a time up to the hundred there are - a number rather than a request, because the
whole buffer is already in the answer the panel holds, and the hundred is where it stops (`DIAGNOSTIC_LOG_LIMIT`).

`/admin` (and `/ru/admin`) is reachable only from the Battle.net menu, and only for an account the site lists as
an administrator - the entry is simply absent for everybody else (`app/components/AuthMenu.vue`, drawn from the
`isAdmin` flag on `/api/profile`). Hiding the entry is a courtesy, not the lock: `/api/admin/settings` asks the
question again and answers a 403 to a signed-in stranger, and a 401 to a visitor with no session at all.

What the panel edits is **the maxima the comparison board measures a row against**. Almost every figure the site
prints carries its own denominator - a collection's total is the size its own shelf prints (`server/utils/collectionAtlas`,
read by both the shelf and the tiles; the Armoury journal and its snapshot are the fallback for a shelf the atlas
cannot size) -
but three of them are published nowhere: how many quests have been
written, how many appearance sources exist, and how many recipes a book can hold. Those are stated by an operator
and corrected when a patch moves them, and they are kept in the `settings` table (`server/utils/siteSettings`):
one row per namespace with the value held as JSON, and the values a copy of the site ships with living in front of
it (`shared/data/siteSettings`), so a checkout that has never been opened in a panel still draws its bars.
Clearing a field is a real answer rather than an empty one - it is how a figure is left without a scale, because
the board draws a bar only where a size is *known* and prints the number alone everywhere else (the score, the
honorable kills and the honor level are the three no panel can help with).

The same store holds **the score's weights** (`scoreWeights`), and they are the one setting on this page the site
*computes* with: ten multipliers, one per figure of the overall rating, edited in the Content tab's first section. The
form draws them in the two groups a reader meets the figures in - what a character has collected, then what it has earned -
each under the figure's own name and icon, and under them a **preview**: the administrator's own main, scored by the draft
and by the table in force, term by term. That preview is the point of the section, because a multiplier is a claim about how
a category scales (a mount is 1,200 things, a quest log is 27,500) and the figures it scales are somebody's. Two behaviours
are worth knowing before pressing save: the weights are laid over the shipped table rather than replacing it (a field outside
the bounds is clamped, so a hand-edited row cannot leave the site unable to rank), and a save that really changes the table
**re-ranks the whole hall of fame** in one automatic pass - a rating is stored with its row, so the weights carry a revision
the rebuild compares (`scoreVersion` in `server/utils/playerScore`, `PRAGMA user_version` in `server/utils/leaderboardRows`),
and a save that changes nothing does not move it. The same revision brings the **stored history** with it
(`refreshScoreHistory`, beside the rebuild): the newest point of every character is re-rated and every older one loses its
rating while keeping its figures, so a chart begins where the current table begins instead of stepping on the day of the
save. The section says so under the form rather than leaving an operator to find the curve changed by itself.

The second section of the page is the foundation the store was cut for: a page's copy, keyed `page.slug`, stored
and served by the same endpoint. Nothing reads those keys yet - a page still prints its own words - so the section
establishes the shape, the validation and the way back in, and the first page that reads a key is a page that
chooses to (`/api/settings` already serves the copy to every reader).

The third tab is the **Season** panel, and it is the one write on the page that changes what every stored table
prints. A Mythic+ season boundary is kept in the same `settings` store (`mplusSeason`) and loaded into the process
at startup; a stored rating read before it counts as zero, a rating read after it counts as itself, and the history
in `snapshots` is never deleted (`server/utils/mplus`). A reset writes the boundary, rebuilds the materialised
`leaderboard` in one transaction, invalidates the profile and shelf caches, and clears the rating from the
sitemap's index. The tab also offers a leaderboard rebuild on its own and prints the process's limits and data
paths, so an operator can see what is in force without reading the source - those values are read-only, because
each was chosen against a measured load and a field that raised one could take the site past Blizzard's quota.

The fourth tab is the **Statistics** one, and it is the only read-only section of the page: four cards of what the
site holds now, one track per figure over a window of 7, 30, 90 or 365 days, and the day's Blizzard failures beside
them. The figures are written once a day by a timer in the process (`server/utils/siteStatistics`); the failures are
counted as they happen into a table of their own (`blizzard_failures`, `server/utils/blizzardFailures`), because the
System tab's counters are the process's own - and a restart, which is exactly when an operator wants to know what
failed before it, empties those.

| Metric | Stated maximum | Where a total would come from otherwise |
| --- | --- | --- |
| `transmogs` | `50 000` | nothing: the wardrobe's own document carries a count and no size of the wardrobe |
| `quests` | `27 500` | nothing: Blizzard publishes no quest index to add up |
| `recipes` | `11 000` | nothing: a recipe book has no published size |

The fifth tab is the **guest merge**, and it is the one section of the panel that writes about strangers. It draws **three**
lists, because a signature names one of two things (`server/utils/guestAccounts`, `server/utils/accountSignatures`) and an
account is a third: the **groups** of guests that are one account the site has yet to build, the **alts** of accounts it
already has, and the **accounts a merge built** — one line each (name, the character leading it, how many it holds, and the
roster behind a fold), because a site that has been running a while has a hundred of them. The tab's own strip switches
between **three views** — the groups (with the counts, the backfill and the automatic pass, which are what act on exactly
those groups), the accounts, and the merges on file — because the three are one subject read three ways, and an operator
moving between them is looking a name up rather than changing what they are doing. The last two views are **paged and
searchable** (`MERGE_PAGE_STEP`, `MERGE_SEARCH_MAX`, `readGuestMergePanel`, `readDerivedAccounts`, `readMergeRecords`):
forty rows and a "show N more" press, with the totals beside them ("showing 40 of 3 481"), because a site that has been
merging for years has thousands of records and an answer that carried them all would be a page that takes a second to
arrive for rows nobody scrolls to. Both are drawn newest first, so the account an operator has just built is on the first
page rather than at the end of a history. A **name** is what the search field takes, and it is matched against everything
the row answers to: a character of the account (by the folded form the site looks names up by, so Cyrillic works in any
case), the BattleTag of the account, and — for a record — the roster as the write recorded it *and* the characters the
account has now. The search is part of the page rather than a filter on what came back, so an account on the fourth page
is found by it, and the count beside the list is counted under it. It is applied on a press (Enter or the button) rather
than on every keystroke, because one read of that endpoint clusters every guest on the site: a search per letter would be
a pass per letter. The leader is the one thing here an
operator can still change after the write, and it is changed *in the table*: a dot in the column the roster is read down names
a character the leader (`choosable` on `<GuestMergeMembers>`), while the account follows the strongest character by itself
until somebody says otherwise — a choice made by hand is *pinned*, and the line then says so, says what the rule would pick
instead, and carries the way back (`setAccountMain`). An
attachment is a guest that names an account with a roster, a standing and a name somebody chose, so its place is that
account rather than a new one beside it — which is the difference that keeps the panel from building a second account for
somebody who already has one. Either tile is one account saying what it is twice over: the **signature** (three chips, one
an anchor, named and dated) and the **members** as an aligned table (character with its class colour and side, realm, level,
item level and the achievement points the board ranks — printed under the site's own name for that figure, one figure one
name — with the row a *group* would be named after marked; an attachment marks no row, because the account already exists,
and an account a merge built marks the row that leads it).
The counts above the lists say how many guests are waiting to join one (`attached`) and how many accounts the site can
recognise at all (`recognised`), so a site whose pages nobody has opened reads zero there rather than looking broken.
There is deliberately no third part to a tile, and no count of a history: a share of two histories, a count of the moments a pair holds in common, a tally of
the sides a group plays for, how much of a log the site has read for a character — each of those was a way of asking whether
a group was *plausible*, and a group is not plausible, it is certain. A figure that invites "is 55% enough?" about a fact
that cannot be otherwise is worse than no figure at all. Each group carries a two-press button: the first press arms it and
says so, the second merges. A group that names a character somebody already owns is refused whatever anybody presses, and so
is a group whose members do not share three identical seconds — there is no rule to argue with and no threshold to move, so
a character the signature cannot describe is simply counted as "without a signature" rather than offered. The same card
carries the switches that decide whether an operator is needed at all:
**auto-merge**, off until somebody arms it, its cadence and how many groups a pass may take - and the pass's own button,
which is two presses like everything else that writes here: the first counts what a pass would fold (`planGuestMergePass`,
answering `taken`, `members` and what the ceiling leaves) and writes nothing at all, the second runs it. Beside it is the same
button with the ceiling lifted - **merge every group**, which counts the whole population on its first press and folds it on
the second (`all: true` in the body) - because a scheduled pass is a drip and an operator who has read the list wants all of
it. A single press
therefore never folds a stranger, whatever the switch says, and the switch is the only thing that ever does. The list of
merges on file is below, each with the signature it was accepted by and a two-press undo, and a merge that was undone stays
in it rather than disappearing: a merge that turned out to be wrong is the most interesting row this panel has. Each record
is **one line** - the account, when it was folded, by whom, and whether it can still be taken back - with the roster and the
signature behind a fold (`<AppCollapse>`, one record open at a time), because a site that has been merging for a while has a
hundred of those lines and the undo button has to stay in the list rather than behind a click.

