# ⇄ Comparing two characters

A character page's sixth view is a comparison: the character's own figures beside a second character's, row by
row. It is not a tab of its own — it is the second half of the character's own entry in the navigation, the
half that carries the two arrows and nothing else, so the entry holding their name holds the way to read them
against somebody else. Its address is the character's own with `/compare` after it — `/region-eu/gordunni/neromask/compare`
opens on the search, and picking somebody writes them into the address (`/compare/eu/azuregos/dzheykb`) — so a comparison is a
link that can be shared like any other page, and the two characters are read from the very endpoints their own
pages are.

The board (`app/components/CharacterVersus.vue`) is two columns of the site's glass, each under its own card,
with the same thirteen figures in the same order below them: the site's score, the collections (achievements,
mounts, toys, pets, decor, titles), the reputations, the wardrobe, the
quests, the recipes, the honorable kills and the honor level. Every row says whose figure it is — the account's
own in the brand gold, the character's own in grey — because a reader who came with an account is shown the
account's numbers and a stranger is shown the character's, which is the rule every tile of the site already
prints. The higher of the two figures is lit in green with the `+X` it won by beside it, and a figure that is
not there to print is a dash rather than a zero: a wardrobe nobody has read and an empty one are different
answers.

The wardrobe is the one figure of the fourteen that no profile carries — 2.5 MB of Blizzard's own — so it is
read by the endpoint that keeps it off a page (`/api/transmog`), in the browser, and its two rows fill in a
moment after the board opens. That reading is *filed* as the character's own (`fileTransmogReading`), because it
is the only place the wardrobe is ever read: without it the one figure a reader has just looked at would be the
one figure the site's record of them never holds. Blizzard counts the wardrobe for the account, so the badge
beside it says so for everybody — as it does for the pets, the toys, the decor and the achievements — while a
character of an account on the site is placed in the distribution by the account's own figure, the best of its
roster's readings.

Inside the open row, above the charts, stand the two blocks that make the row readable on its own. The first is
the **fill of the collection** (`app/components/VersusMetricScale.vue`): the bar the tiles of a character page
already draw, with the two figures it was worked out from (`1 234 / 5 925`) and the share they make - offered
for every figure whose size is *known*. That is the seven collections, whose totals travel with the reading, and
the three figures an operator states a ceiling for (see **The admin panel** below). A figure whose end nobody
states prints its number alone, and the score, the honorable kills and the honor level are exactly
that: there is no honest denominator to measure them against, and - as the tiles of a character page say - a bar
drawn to an invented one would be a lie. The second is the **places** the row already had
(`MetricRankLinks`: the world, the region, the realm), now drawn in the same dark plate as the charts beneath it,
so an open row reads as three blocks of one panel rather than one block and two strangers.

Under each row stands that metric's panel (`app/components/MetricHistoryPanel.vue`), and it is two charts of the
site's own record. The **history** is the column's character, one point a *day* - the day is the smallest thing
the record holds, so a second reading inside a day rewrites that day's point rather than filing another one beside
it (`fileDailyReading` in `server/utils/characterReading`), and the reader folds whatever an older build left to
one point a day as well - drawn as a line over
the last week, month, quarter, year or everything, with a table view of the same points; a character the site
has read once is drawn as that single point rather than as a notice, because it is where the curve starts. The
**distribution** is the metric over the whole population — everyone the site has read, in twenty-four buckets the
server cuts (`/api/distribution`) — and it is one answer shared by both columns, so the two characters are marked
on one histogram. The figure a panel marks and the place it prints come from the *member* the character stands
for: a guest's own newest reading, or the account's own totals, which is what makes a main and each of its alts
the same player — the same curve, the same bucket, the same place. The bucket the column falls into wears its
class colour; a figure nobody has read is a dash, and no bucket is marked for it.

Each card carries the same pair of buttons (`app/components/CharacterCardActions.vue`): the star that saves the
character for a quick compare, and the refresh that reads them from Blizzard again. The first character's
refresh *is* the button the header carries — the same act, the same minute of cooldown, the same toast, handed
to the board through the shell's own context (`app/composables/characterView.ts`), so two buttons cannot race
each other into a 429 (`server/utils/refreshBudget`) — while the second character's is the board's own read.
Under the buttons stands when that character's figures were last read, which is the moment the endpoint fetched
them (`fetchedAt`) rather than the moment the page was opened: a profile is answered from a two-hour cache, so
that moment is the only honest answer, and a reader who has just pressed the refresh sees it read "just now".

A saved character is one press from the board, and the star is not the only place the list appears. The list
is called **Favorites** in the interface, and the star that fills it stands beside the names the site draws:
every row of the hall of fame wears one, a character's own page
carries one as a button in its header, to the left of Refresh, the rows under a search field each carry one,
and so do the two cards of the board itself (`app/components/FavoriteStar.vue` for the first three,
`app/components/CharacterCardActions.vue` for the cards). The search field's own list holds the saved
characters beside the ones this browser looked up, as one list
(`mergeSearchEntries`), and the account page carries the list itself as one tile of pills
(`app/pages/profile.vue`): a pill opens the character, the cross beside it drops them - a list a reader can
only fill would not be a list they own. The list is the browser's own
(`app/composables/versusFavorites.ts`), like the
search history - and for a reader who is signed in it is the account's as well, so the same twelve characters
are waiting on any device that reader signs in on (`/api/saved`). The second character of a comparison is read
by the board rather than by a page of their own
(`/api/character`), so the board is also what remembers the read: a character found by the search under a name
field goes into the search history exactly as a character's own page writes it
(`app/pages/[region]/[realm]/[name].vue`), and a read that found nobody writes nothing — which is what puts the
character a comparison looked up back into the list that field offers.

