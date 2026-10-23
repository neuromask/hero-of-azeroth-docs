# 🏆 The hall of fame

The site's front door *is* its hall of fame, and it is a summary rather than a table: `/` opens with a banner - the
sentence the page is read under ("your story in Azeroth"), the two doors into the site (the overall ranking, and the
card of the character this browser opened last, or the search that finds one) and the size of the hall, over a
drawing of the hall itself - then the menu of the site's fourteen
rankings, the reader's own place in the overall rating, five plates about the population — who is on it right now,
and the leader of mounts, M+, achievements and decor — and, under the plates, the population as a whole: the realms it is spread over (every one of them a link into that realm's own ranking of the overall rating), the classes it plays, the sides it plays for, and everything its members have gathered between them, printed under the day the table was last written. The rows are the rankings the menu leads to, one page per figure and scope, and
they are built so that opening the site costs the rest of it nothing:

- **Nothing is aggregated per request.** The population is materialised in the site's own database
  (`leaderboard`, `server/utils/leaderboardRows`): one row per member, holding the figures the rankings draw and
  the overall rating computed when the row was written. Every order a ranking is drawn in is an index over that table
  (`server/utils/db`), so a read is a page of an indexed query rather than a sort of the population — the same
  cost whether the table holds ten players or two hundred thousand, and a deep page is an index scan. **A page is
  twenty-five rows** (`LEADERBOARD_PER_PAGE`, and the server's own default is that same number rather than a second spelling
  of it) — which is a number about the *scroll*, not about the query: one indexed `LIMIT` costs the same at twenty-five or
  at fifty, a row here is about sixty pixels, and the first page is rendered into the document for readers who never scroll
  past the tenth. Twenty-five is enough because the table is not a directory: a name is typed into the filter bar, a side, a
  region and a realm narrow the ranking, and a `?rank=` link opens it at a place — so a page is the context around a row.
- **The rating is stored, not derived.** It is computed once, by the formula the site's own code holds
  (`server/utils/playerScore`), and a build that changes the weights — or what a row holds — stamps the table
  and rebuilds the population once at startup (`SCORE_VERSION`, `ROW_VERSION`).
- **The endpoints are cached for an hour** (`defineCachedEventHandler`, `maxAge: 3600`, `swr: false`), so
  a burst of readers is answered from Nitro's memory in single-digit milliseconds. The first page of a ranking
  and the front door's own plates are rendered on the server and arrive inside the document.
- **A write is one row.** A reading filed, an account's totals filed, a character claimed or a main named
  rewrites exactly the member it moved (`refreshGuestRow`, `refreshAccountRow`), inside a single statement
  rather than by rewriting a document.
- **It is fed by the pages themselves.** Every character page a reader opens files a reading
  (`server/utils/profileFeed`), and that reading is what puts a character in the table or moves its row — so a
  name nobody has looked up is a name the table does not hold. The sitemap's own thinner index
  (`server/utils/characterIndex`) is deliberately not a source: it lists addresses a crawler may be told about,
  and a character known only from that list is not a member of any population.

The figures a reader is most likely to have moved themselves are the ones on the page they are looking at: every
character page anybody renders hands the table that character's numbers, so a reader who has just looked somebody up
has very likely changed the totals a ranking is about to print. Every table therefore asks to be answered rather than
taken out of a browser's store (`cache-control: no-cache` on the request), while an answer's own freshness is Nitro's
to write from the endpoint's hour (`max-age`) — and the answer the server rendered during the page's own render is
the one that arrives with the document, so the first paint is the table rather than a spinner.

The storage sits behind `server/utils/leaderboardStorage.ts`, which is a thin layer over the table itself: the
rows *are* the hall of fame's table in the database (`server/utils/leaderboardRows`), and a table read from a
database has no business holding itself in a process.

A database that has never held it builds the whole population once, at startup, from the characters and their
readings (`rebuildPopulation`, `server/plugins/statisticsMigration`), so the page opens with the site's history
in it rather than empty.

The weights are one line per category in `shared/utils/leaderboardScore.ts` (`SCORE_WEIGHTS`), each
chosen so that a top-end character contributes the same order of magnitude to every one of them — and the table is an
**operator's setting** rather than a constant: the panel's Content tab edits all ten (`AdminContentTab`), what the site
computes with is the stored table laid over the shipped one (`getScoreWeights` in `server/utils/siteSettings`), and the
legend a reader opens prints the very numbers in force (`app/components/ScoreLegend.vue`). Two consequences follow from
that, and both are deliberate. A rating is *stored* with its row rather than derived on read, so the weights carry a
revision: a save that changes one re-derives every rating in the table once, automatically, and a save that changes
nothing does not (a hundred thousand members re-ranked because somebody pressed save twice is work nobody asked for).
And adding a category is a line in the table plus a field in `ScoreInput` — the typecheck finds the five places a new
figure has to be read, which is what the module's own history is written around.
Adding a category is a line there plus a column on the table's own row; the endpoint, the page and
the components do not have to learn about it.

A rating is frozen *at write time*, and that is what makes a history a record rather than a re-ranking — so a change to
the weights leaves points behind on a scale this build no longer computes, and a chart that steps by tens of thousands in
the middle reads as a bug rather than as a decision. `refreshScoreHistory` (`server/utils/leaderboardRows`) is the pass
that settles it, once per revision, at startup and on the first read after an operator's save: every **older** point
keeps its figures and loses its rating (the rule the legacy migration already follows for a point it cannot honestly
reproduce, `server/utils/statisticsMigration`), and the **newest** point of each character and of each account is rated
with the table in force where it carries every figure the formula reads. The newest point is the one the distributions
are drawn from (`readSnapshotDistribution`), so the comparison boards keep their population through a weight change
instead of losing it until every character has been read again — and a point filed before a category existed keeps no
rating rather than a smaller one. The marker is the weights' own revision (`getScoreHistoryRevision` in
`server/utils/siteSettings`), so a start that finds it reads one row and touches nothing.

### The metric leaderboards

Beside the table there is a ranking *per figure*: `/leaderboards/mounts` is the world's mount ranking,
`/leaderboards/mounts/eu` the European one, `/leaderboards/mounts/eu/gordunni` that realm's — the same
three addresses for each of the thirteen figures the comparison board prints. The list lives in one place
(`METRIC_LEADERBOARDS` in `shared/data/leaderboardSchema.ts`): a slug, the figure it ranks, the column of the
materialised row the order is read from, the index that carries that order, and the words it is drawn in. Nothing a
caller types reaches SQL — a slug this build does not serve is a 404, and the *entry* is what the query is built
from.

A ranking is a total order, and that is the whole of its correctness: `metricOrder` in
`server/utils/leaderboardRows` writes the order (the figure, then the rating, then the name, then the region, the
realm and the name key a row is keyed by), the indices in `server/utils/db` carry that very order, and a place is
answered by *counting the rows above a member* with the same comparison written as a predicate — so a page and the
place a reader is shown can never disagree about who is ahead of whom. The comparison is written as a flat chain of
`AND`s and one `OR`, which SQLite turns into a handful of index seeks; nested `OR`s make it count the whole index
instead (4 ms against 40 ms for one place at two hundred thousand members).

A page of a ranking is laid out as the front door is, because it answers the same kind of question - down to the bar
they both wear (`LeaderboardFilters.vue`), which is one component on both pages. The bar leads the page, because
choosing the table is what a reader does before reading one: it opens with the menu of the site's fourteen rankings,
one figure each, a press each, keeping the region and realm the reader is already looking at, since a query parameter
would make one address stand for fourteen different ranks. A hairline of the brand gold parts that menu from the
filters under it: which side, which region, which realm and a name, with the legend of the score beside the field
wherever the score is what the table shows (the front door draws no table, so it wears the menu alone and is told so
- `filters`). Those filters are the last thing in the bar and the first thing over the rows they narrow, and the rows
follow straight after them: the reader's own place is worn in the corner of the header above them as one plate of two
sections (`app/components/SelectedCharacterChip.vue`) - who this browser selected, and the press that reads their
place in this very ranking and opens it turned to their row - which is the same control, with the same figure and
scope, that the hall of fame wears about the overall rating.
The region chips and the realm menu are *scopes* here as much as filters - the region and the realm a ranking is
drawn for are part of its address - so the menu offers the realms this ranking actually holds members in, with what
each holds (`readMetricRealms`), "every realm" is the world ranking, and the reset button steps back to it, which is
what makes it appear at all once a realm or a region is chosen. The rows below wear the marks every row of the site
wears: the place in a medal for the first three, the side as a badge beside the name, the portrait with its account
halo and an online dot, the name with the star that saves a character for the favorites beside it and the two
badges that say which row is the reader's own main (`lbYouBadge`) and which row the chip in the corner is
about (`lbSelectedBadge`), the class, the level and - for a character who has finished the climb - the
item level under it, and the guild in the column the table keeps for it.
The halo is **two colours, because it is two marks**: Battle.net's own blue for the main of an account somebody signed
in with (the same blue the header's plate wears, and the tooltip says which), and a lilac for a row the merge panel built
out of several guests (`aggregated`, `server/utils/accountMerge`) - an account nobody ever logged in as, whose figures are
an operator's statement about strangers rather than a person's own reading. The two are both `isAccount`, and until the
lilac was drawn a reader could not tell "this row is three characters" from "this is one account's main" - which is the
very thing the row version was raised for (`ROW_VERSION` in `server/utils/leaderboardRows`).
The columns are the table's own and not its content's (`table-fixed`): the place takes a fixed slice of the left
edge, the player the most of what is left - a nickname, the star, the faction badge and the two account badges all
stand in that cell - and the guild, the realm and the figure share the rest. The realm is the narrowest of them on
purpose: it carries only its region badge and the realm's name, both of which truncate with a tooltip -
a column that grew with the longest name in the table would shift every column beside it, and a reader scanning for
their own row would lose it as the columns moved. The realm prints the region it belongs to as a badge of its own
(`EU`, `US` - the game's own shorthand, the way `M+` is): a ranking of the *world* holds both sides of the
Atlantic side by side and a realm's name is the same word on either of them, so the slug alone would be a name a
reader could be wrong about.

The guild is a fact about the character rather than a figure of its readings, and it rides on the very profile the
site already reads for everything else: a reading files it on the character and rewrites the member's row from it
(`fileDailyReading` in `server/utils/characterReading`), so a row holds it from the day that character is read
again rather than from the day the row was first written.

A page is named for both halves of what it shows - "Зал славы - Счёт - Hero of Azeroth" (`lbMetricTitle`,
`lbMetricTitleScoped`): the site's own table, the figure it is read for, and - on a scoped page - the scope,
which is the *page's* address rather than a filter inside it. A realm's pets and the world's pets are two
tables, not two views of one. The header prints the same two halves in large type, joined the way a heading
joins a pair rather than the way a document title does - "Зал славы / Счёт" (`lbMetricHeading`) - because the
fourteen rankings share one name and a reader has to know which table they opened; the scope is what stays in
the pills under it, along with the count of members standing in it.

Every part of a title is joined by the one separator the whole site joins titles with (`TITLE_SEPARATOR` in
`app/composables/seo.ts`, ` - `), and the brand is the last of them: a page names itself, `usePageSeo` appends the
brand (`titleWithBrand`), and a page that writes its title whole - the character pages do, theirs ending in the
brand - writes it with the very same join.

A figure nobody has read is not a nought: the five figures that arrive beside the collections (`titles`, `quests`,
`recipes`, `transmogs`, `honorLevel`) are `NULL` on a row until something reads them, and a member holding no
figure for a metric is *left out of that one ranking* rather than placed at the bottom of it. A real zero — an
account whose own totals say it holds none — is ranked like any other figure. The five columns are filled from the
rows the table is already made of: a guest's own newest reading, an account's own totals
(`readAccountTotals` in `server/utils/accountPool`), so the figure a metric page prints for a character is the
figure the comparison board prints for them.

The comparison board puts the three places of the open row above its charts (`MetricRankLinks.vue`), one line per
side, each a link into the ranking it names with `?rank=` on it: `/leaderboards/mounts/eu?rank=1102` opens the
European ranking at the page holding the 1 102nd member and marks their row. One request answers all three places
of both sides (`server/api/leaderboard/metric-ranks`), it is made only while a row is open, and the transmogs row
waits for the wardrobe it is about to file before it asks — a place computed from a figure the site does not hold
yet would be a place of nothing. A character who is not in the population at all — an alt that is neither its
account's main nor a row of its own — is told so rather than given a place.

The metric pages are cached exactly as the front door is (`server/api/leaderboard/metrics`, an hour keyed on the
question and on the stamp of the data), the first page of one is rendered on the server, and the realms, regions
and figures a reader may reach are the site's own: there is no page for the item level (which the site draws no
history of) and none for the Mythic+ rating (which the site's own read counts in the season Blizzard is
running).

A ranking's answer is filed under the question that asked for it, and that is the page's own rule rather than the
endpoint's (`requestKey` in `app/composables/metricLeaderboard.ts`): the name a fetch is cached under is its address
together with the narrowing and the page, so a page the router has rebuilt for another ranking — another figure,
another region, another realm, another page of one — *asks*, rather than being handed the rows of the ranking the
reader has just left. `useAsyncData` does not fetch again for a key it already holds an answer for, so a single
fixed key is exactly what a table looks like when it changes only on a full reload: the address moved, the request
did not. The same name on both sides of the wire is what lets the server's answer ride in with the document
(`useAsyncData`'s payload) instead of being asked for twice.

The bar above a table is where the site's tables are chosen: the menu of the fourteen of them, one figure each
(`METRIC_LEADERBOARDS`), stands over the filters, parted from them by a hairline of the brand gold. Each entry is an
address rather than a query parameter, so each has a page of its own and a crawler can find it, and the entry a reader
is reading wears the gold and is not a link to itself. The hub is not an entry of this menu: it is the tab of the
navigation row above, whose caret opens this same list from anywhere on the site (`HallOfFameMenu.vue`), and the front
door wears the menu without the filters under it (`filters`), because it draws no table to narrow. Under the menu are the filters: which side, which
region, which realm, and a name, with the legend of the score (`?`, "how the score is calculated") beside the field,
where the figure it explains is the one in the table (its multipliers are read off the formula itself, so a panel can
never promise something the server does not compute). The realm names come from the site's realm list rather than from
whichever language the record happened to be written in, and an item level is read beside the level under a row's
name rather than as a figure of its own — a figure that says nothing among characters still climbing is printed only
for one at the current level cap, and the comparison board and a character's own header print it unconditionally,
because neither ranks anybody. An account has no item level of its own at all: it is always the character's, so a
row shows the main's — the very character the row is named after — and the account's page prints the main's too. The
Mythic+ rating is counted in the season Blizzard is running (`server/utils/mplus`): a score read before a season
began counts as nothing, so a table never prints the season that has just ended.

The page carries the site's one search in two places, and both of them are the same two fields
(`app/components/CharacterSearchForm.vue`) - the site's own glass, with the rim of a field warming to the brand
gold while it holds the cursor, and the button the search is pressed through cut from the amber glass a
character's own download button wears: the material's frosted pane over the deep amber fill, its rim warmed from
amber by the one variable every rim of the site reads, and its glow widening under the pointer. The geometry is
the layout's, so the button is exactly the size it always was - what its three shapes share is the material, not
the shape: in the navigation row, as one row, in the place a character's own entries take while this browser knows
nobody and nobody is signed in (`compact`); and the sheet the row drops
(`app/components/CharacterSearchDialog.vue`, on `Ctrl`/`⌘`+`K` as well). There used to be a third, under an empty
table, offering the name that matched nothing; the table it stood under is a set of rankings now, and a ranking
that holds nobody for a realm says so rather than asking for a search.

The list that opens under the name field is one list holding two kinds of remembered character: the ones this
browser looked up, and the ones it saved with a star for a comparison (`mergeSearchEntries`,
`app/composables/searchHistory.ts`). They are drawn together because a reader looking for a character does not
care which of the two remembered it, and each row carries the star, so the list a reader can see is the list a
reader can change. Those same stars are how the list is filled from the rest of the site: every row of a ranking
wears one, a character's own page carries one as a
button in its header, beside Refresh (`app/components/FavoriteStar.vue`, `app/pages/[region]/[realm]/[name].vue`),
and the account page keeps the whole list in one tile, where a pill is a way to the character and the cross
beside it a way out of the list. The saved ones stand first, and the pane draws at most twenty of the pair: a field's list is a
way back to a character rather than an archive to browse, and what is typed still narrows the whole list before
that cut is made.

A ranking is drawn a row at a time in the one table of the site (`app/components/leaderboard/MetricLeaderboardTable.vue`),
which is wider than a phone on purpose - a place, a name, a guild, a realm and a figure do not fit a column of glass -
so it scrolls sideways under a bar of the site's own (`.hoa-scroll-x`, `app/assets/css/main.css`): the one scroller of
the site that *draws* its bar, because a table whose right-hand columns are a drag away has to say so rather than look
complete. The place and the figure are as wide as their own numbers, the name is measured for its own line with the
star and *both* marks a row can wear on it, and the rest spread across the panel, so a name is never cut and no mark
drops to a line of its own. The side a row is on is one more badge beside the name - the word and its dot in the
side's own colour - so the table says it once per row in words rather than only as a colour at the table's edge.

A reader who is signed in has that same list kept for their account (`/api/saved`), so it is the list they had on
their other device. The browser's copy is what a field still draws first - it is written before anything is asked
of the server, so a field never waits for a request to have a list - and it is brought into step with the
account's once per page: `/api/saved` is read, the browser's own two lists are handed over the first time on that
browser (`hoa_saved_account` records it, so only the first account to sign in on it gets them), and the account's
lists are then adopted as the browser's copy, which is what leaves a reader who signs out holding what they were
just looking at. A visitor notices none of it - their lists are the browser's own, exactly as they were before
the account existed - and a request that fails changes nothing at all.

The character the page is *about* is the one this browser opened last, or - for a reader who is signed in and has
looked nobody up on this device - the main of their account: `/api/profile` is read for the row's own badge in any
case, so the main costs the page no second request, and a reader who signed in somewhere else finds their
character waiting in the bar instead of a pair of empty fields. The row under the page's name is the site's own
navigation and the very component a character page wears (`app/components/SiteNav.vue`), drawn only once there is
somebody for its six character entries to point at; the chip in the corner about the overall rating
(`app/components/SelectedCharacterChip.vue`) is the browser's own last lookup - the character this browser selected,
whether or not an account is signed in - and it carries the press that opens that rating turned to their row.

The rankings are a *destination* rather than only a list, and there are two ways into them from what a reader is
looking at. Every stat tile on a character page - and every figure of the drawer under its summary plate - links
into the **ranking of the figure it prints**, whole: `/leaderboards/<figure>`, with no scope on the address, because
what a tile is a door into is the *table*, and the table the site keeps is the population's. A realm-scoped page is a
handful of rows - the site has read a few characters on most realms - so it answers "who else is here" rather than
"where do I stand among everybody"; the address is built by `metricLeaderboardPath`, the composable every ranking page
is addressed by, so a tile and the page it opens cannot spell the same scope differently, and a reader who wants
their own corner of the ranking narrows it from the filters that page carries.
Every one of those doors also carries the **place** the character holds in the table it opens: the address ends in
`?rank=N`, so the table lands on the page that holds the row and the row is brought into view rather than the reader
being shown the top of a ranking they are thousands of rows down. The places come back in one answer for the whole
page (`/api/leaderboard/character-ranks`, the fourteen of them at once, cached under the stamp of the data), and they
are the *world's* - the scope of the table a tile opens - with `null` for a figure the table holds no reading of,
which leaves that door pointing at the ranking itself. `app/composables/characterRanks.ts` is where a page writes both
halves of such an address, and its answer is read with the page rather than after it, so the links are in the HTML.
That walk is the `?rank=` mechanism itself: the place is turned into the page that holds it
(`app/composables/metricLeaderboard.ts`), the row is scrolled to once it is drawn, and a character the table does not
know (an alt of an account whose main is the row it enters) leaves the page exactly as it was. The address is read
again whenever the query moves under a page the router keeps: the leaderboard pages are keyed by their route's name,
so a press of "find in the table" on a table that is already open changes only `?rank=` and never rebuilds the
component - a state read once at setup would leave that press working only after a document reload. A second press
about the same place is a counter in the app's state rather than a second address (`router` resolves a repeated
navigation to nothing), so it still walks back to the row. `data-key` on every row
is what makes a row an address a page can walk to rather than a position it has to count to. And the plates of the
front door's own tiles name a leader each: pressing one opens that leader's page, which is where the figures behind
the plate are read in full.

