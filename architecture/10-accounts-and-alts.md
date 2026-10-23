# 🔐 Accounts & alts

A reader may sign in with Battle.net, which is what turns the site's one-character lookup into an
account with a roster of alts behind it. The public half of the site is untouched: looking a
character up still costs nothing and needs no account.

- **Sign-in** is OAuth 2.0 Authorization Code Flow (`scope=wow.profile`), served by
  `server/api/auth/{login,callback,logout}.get.ts`. The callback makes exactly *two* Battle.net
  calls - the token exchange and the account summary (`/profile/user/wow`) - and writes them before
  redirecting, so an account with thirty alts never holds the callback open into a gateway timeout.
- **The session** is an h3 sealed cookie (`useSession`, `iron`), and it carries only a session id.
  The tokens themselves live in SQLite, encrypted with AES-256-GCM under `NUXT_SESSION_PASSWORD`
  (`server/utils/tokenCrypto.ts`), so a session can be revoked by deleting a row.
- **Battle.net does not issue refresh tokens.** The stored `refresh_token` column is honoured if one
  ever arrives, but the normal path when the 24-hour access token runs out is a fresh sign-in - which
  Battle.net's own SSO cookie (about 30 days) answers without asking for a password again.
- **A sign-in that stops working says so.** A renewal the token endpoint *refuses* - a rotated client
  secret, a grant Battle.net revoked - is the one failure a reader cannot see: every page still renders
  from what the site already holds, while the roster and the account's collections quietly stop moving.
  That case marks the account (`users.auth_expired`, `server/utils/session`) and the header and the
  profile page turn the mark into a "sign in again" plate. A timeout, a DNS hiccup or a 5xx does not:
  those say nothing about the grant, and the account is left alone for the next request.
- **A sign-in adopts the account a merge already built.** The roster is read *before* anything is written, because it is
  the only thing that can tell the site who the reader already is here: a first sign-in whose characters the site has
  already folded into a derived account (`11-guest-merge.md`) does not build a second account - the Battle.net identity is
  written onto that row (`adoptDerivedAccount` in `server/utils/accountSync`), which is what keeps one character on one
  row (`characters` is unique per *account*, not per address). A reader who signed in *before* their alts were folded gets
  the opposite half of the same rule: the derived account's characters move into theirs (`absorbDerivedAccount`), the shell
  is left standing, marked and empty - and the history says so, because there is nothing left to take back. The invented
  subject a merge wrote is kept on the account as `derived_subject`: the panel prints it, and nothing public does, because
  a reader's own pages show their characters and that is the whole of the answer.
- **Storage** is a single SQLite file, `server/data/hoa.db`, opened through the built-in `node:sqlite`
  driver (`server/utils/db.ts`): accounts, their characters, the readings each character has been through, the
  account's own points of history, and the hall of fame's materialised table. No database server, no native
  dependency, and the schema is created on first open — a database written by an older build is brought up to
  it when the process starts (`migrate`). It sits beside the JSON stores, and `npm run deploy --prune` never
  touches it.
- **The file is compacted at startup, never from a request.** SQLite keeps the pages a delete frees on a free
  list and reuses them, so a file can be much larger than the data in it — the panel prints the free pages
  beside the size, because the size alone cannot tell air from data. One plugin
  (`server/plugins/dbMaintenance.ts`, `server/utils/dbMaintenance.ts`) runs per start: it folds the write-ahead
  log, turns on `auto_vacuum = INCREMENTAL` — which is the one full `VACUUM` that mode change requires, and so
  happens once per file — and afterwards reclaims at most 32 MB of free pages per run (128 MB at startup,
  where no request is waiting). The panel's maintenance
  button does the bounded half (log folded, up to 8192 pages given back) and never a `VACUUM`: nothing a
  request can press rewrites the file under the site's readers. A table a build has taken away is dropped on
  the same start (`migrateRemovedTables`; `account_collections` is the first, superseded by `account_pool`).
- **The site counts itself once a day.** Four figures about the people it holds - the accounts that signed in
  through Battle.net, the characters the site has read, the rows of the hall of fame, and the characters no
  account holds - are written to `site_statistics` as one point per day, keyed by the day's own first second
  (`dayStartOf` in `shared/utils/day`), so a restart rewrites its day rather than filing a second point beside
  it. The recorder is a timer in the process (`server/plugins/siteStatistics.ts`, `server/utils/siteStatistics`):
  it writes at start and at every midnight UTC, which is the one tick it needs, and the admin panel's Statistics
  tab draws the points back as four tracks (`app/components/SiteStatistics.vue`). A year of history is 365 rows.
- **Stats are filled in the background.** The sign-in writes names, not figures; the profile page
  reads a few characters on demand (`server/api/profile/sync.post.ts`) and a server plugin
  (`server/plugins/characterSync.ts`) does the same on a timer, so no page ever waits on a Blizzard
  read per alt. Every read also stamps the moment it happened on the character
  (`characters.stats_read_at`), because a reading is filed for the moment it was taken rather than for the day
  it belongs to - and a page that printed a day would tell a reader who has just pressed Refresh that their
  character was read "20 hours ago". The point that reading is filed as belongs to the day all the same: one point a
  day per character and per account (`snapshots`, `account_snapshots`), so a day's readings cost one row however
  many times it was opened, and a chart drawn from them is a curve of days.
- **The account pool is the source of every figure.** Mounts, pets, toys, decor, titles and the factions
  driven to Exalted are shared across a whole Battle.net account, so the account's numbers are a
  **union of unique ids** (`account_pool`), filled from Blizzard's account collections
  (`/profile/user/wow/collections/*`) and from each character as the roster is read - its own mount
  journal (a mount can belong to a side or a race, so it only ever shows up for a character who can
  use it), its titles, and the factions it has driven to Exalted - never a `SUM` over alts. A character
  page's Refresh folds the ids that read has just fetched as well (`foldReadingIntoPool` in the same
  module): the tile above a shelf takes the best of the journal and the pool (`bestFigures`) while the
  shelf under it is drawn from the pool's ids alone, so a reading filed in one place and not the other
  printed one collection as two numbers - 963 toys on the page and 961 on the shelf. The reputation figure is that
  pool's alone: a stored per-character count is deliberately not merged into it, because until the site counted
  factions for itself that number was Blizzard's account-wide Exalted counter rather than the union
  (`server/utils/blizzard`, `server/utils/reputations`)
  (`server/utils/accountPool.ts`). The pool table is `WITHOUT ROWID` and keeps one *set* of ids per (account,
  kind) - a chunked bitmap in a blob (`server/utils/idSet`) - rather than a row per id, with the size of each
  set stored beside it: one real account (19 585 ids, 13 342 of them quests) measured 445 KiB as rows against
  24 KiB as sets, which is what makes the table viable for a site of many accounts.
  The pool is what the profile tiles, the character pages, the card and the main's row in the hall of fame
  all read; a character that
  belongs to **no** account - a stranger's lookup - keeps Blizzard's per-character numbers, and
  `findOwnerPool` is the one place that rule lives. The overview says which of the two a figure is:
  while a pool answered, `character.pooled` is `true` and every tile reads `account-wide` under its
  label; a character with no pool reads `per character` on mounts and reputations and `account-wide`
  on the collections Blizzard keeps for the account (pets, toys, decor, achievement points). The hall
  of fame *reads* the pool too, on every request that answers with a table
  (`accountTotalsByCharacterKey`): its row for a signed-in player is drawn with the account's figures
  rather than with whatever the row happened to be written with, so the table and that player's own
  page cannot disagree because one of them was looked up before the pool grew. The main
  tile's **Download** draws the account
  card (`/api/profile/card`), **Update all** walks the roster sequentially (700 ms apart, with a
  progress bar), each tile refreshes on its own with a 30-second cooldown, the settings panel (the
  sliders button) holds the Hall of Fame switch, and the bar over the grid carries the roster's order,
  its chips, the class / realm / profession menus, the level window (a rail with a floor and a ceiling,
  which hides everyone outside it) and the search. The main card's figures are doors as well as numbers:
  pressing one opens that figure's fill against everything there is of it (where the site knows that
  size), its places in the site's table, the population's distribution and the character's own history
  under the whole grid, in the comparison board's own fold-out (`AppCollapse` + `MetricDetailPanel`); the
  item level, which the site ranks nowhere and keeps no history of, says so rather than drawing an empty
  chart.
- **One main per account.** A character the reader names as their main (`/api/profile/main`) is the
  only one the public leaderboard shows; every other character of the account stays a personal stat
  (`server/utils/leaderboardExclusions.ts`). The profile draws it twice: as the large card the page
  leads with, and in the roster grid among the others - where its crown is the filled one, while every
  other tile carries the same crown left hollow, and pressing that is the promotion. A crown rather than
  a star on purpose: the star is what the site saves a *favorite* with, and a grid of stars would read as
  a list of saved characters rather than as the three or four candidates for the main. An account may
  also leave the table altogether: the profile's **Hall of Fame** switch (`/api/profile/fame`,
  `users.hide_from_fame`) keeps its main out of it, both from what the endpoint serves and from what
  a later lookup writes.
- **Who is on the site.** Every browser beats every half minute while its tab is visible
  (`POST /api/user/heartbeat`, started once for the whole site by `app/plugins/presence.client.ts`). The beat
  carries the browser's own id, kept in `localStorage`, so two tabs of one reader are one visitor and a reader who
  comes back is the same one; it stamps `users.last_seen_at` when somebody is signed in, and in every case counts the
  browser in the process's own map of beats (`server/utils/presence.ts`). A browser is a reader from its **second**
  beat, and a beat that names a crawler is never written (`server/utils/crawlers`, the same list the crawler throttle
  uses): a page load is not a person, and the renderers that execute the bundle - Googlebot's, Bing's, Yandex's, the
  SEO and AI crawlers - arrive with an empty `localStorage`, so a crawl wave used to read as a crowd on the plate
  (sixty readers beside one visitor in the analytics). The three rules cost half a minute between a reader arriving
  and being counted, and nothing else. The hall of fame prints the count of readers on the first plate, with how many
  of them are signed in under it, and wears the signed-in accounts as a green dot on the portrait of each main
  (`GET /api/leaderboard/presence`, re-read every half minute by `app/composables/presence.ts`). The list is its own
  endpoint on purpose: the table itself is cached for an hour and may be served by a CDN for another, and a dot that
  old says nothing — so no dot ever pulls the hour-long table through the cache again.
- **Which rows are an account's own.** A row that belongs to an account that signed in — that account's
  main — wears a halo in Battle.net's blue on its portrait, the same blue the sign-in plate of the header
  wears, with a tooltip saying what it means: the figures are the account's own totals, kept as a point of
  the account's history rather than read off one character (`server/utils/statisticsPopulation`, and the
  mark itself is written with the row, in `server/utils/leaderboardRows`). Unlike the online
  dot, this fact rides *in* the table's own cached answer: it moves when somebody signs in or names another
  main, which is a handful of times a day, so an hour of cache costs nothing.

