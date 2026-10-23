# 🔄 Keeping the data current

Everything a character *is* — collections, achievements, item level, reputations, the artwork — is read
live per request, and those caches expire on their own (a shelf in two hours, a profile in ten minutes,
the Armoury journal a mounts tile is measured against in a day). What a patch moves are the numbers the
site pays for once and ships with the build. One command runs all of them, in the order they are read:

Every one of those reads is made with a token of the site's own: a client-credentials token per region
(`server/utils/blizzard`), minted at that region's OAuth host and cached in the process until an hour before it
expires. A token the API *refuses* is not waited out - the refusal drops it, a fresh one is minted and the call is
sent once more, so a rotation on Blizzard's side costs one request rather than a day of failed reads that only a
restart ends. A *credential* refused by the token endpoint itself is remembered for a minute
(`TOKEN_REFUSAL_BACKOFF_MS`), so one wrong secret is not asked about once per document. Either way the failure is
counted, logged (`server/utils/adminDiagnostics`) and filed on the day's row (`server/utils/blizzardFailures`), which
is the copy a restart does not erase.

```bash
npm run refresh:all                     # every step, read with the character it remembers
npm run refresh:all eu gordunni name    # the same, read with another character
npm run refresh:all --only=collections,reputations
npm run refresh:all --only=achievements # just the achievement atlas
npm run refresh:all -- --dry-run        # print the commands and run nothing
npm run deploy                          # nothing is live until this runs
```

| Step | Writes | What goes stale without it |
| --- | --- | --- |
| `collections` | `server/utils/collections-data.json` | an item a patch added, a new expansion's bucket, a new source or title — a shelf keeps the tree of the last run |
| `achievements` | `server/utils/achievements-data.json` | an achievement a patch added, a new category, a renamed heading — a category keeps the tree of the last run |
| `armoury` | `server/utils/armoury-totals.json` | the two fallbacks: a mount tile is measured by its Collections shelf first, and the Armoury journal (live, then this snapshot) is the fallback for a shelf the atlas cannot size |
| `meta` | `server/utils/achievement-meta.json` | the feed cards' wording, points and icons — a new achievement is read live, so it is slower rather than wrong; `refresh:achievement-flags` fills in the one bit the fingerprint needs (`is_account_wide`) without re-reading nine thousand achievements |
| `points` | `server/utils/achievement-points.json` | the fallback total for achievement points |
| `reputations` | `server/utils/reputations-data.json` | how many factions each side can work on, the tree the reputation pane is organized by, and the top and steps of every ladder |

The `collections` step downloads SimpleArmory's own lists — mounts, pets, battle pets, toys, decorations and
titles — into a cache in the OS temp directory before it builds, so the atlas is made from what
the site answered this run rather than from anything a previous run left; a file the site will not
answer for falls back to that cached copy. The pets are one shelf whatever they are read from:
`pets.json` and `battlepets.json` are merged into a single tree, because a character's journal does not
tell the two apart either.

The three walks (`meta`, `points`, `reputations`) run with `--fresh`, which drops the progress cache each
of them keeps in the OS temp directory and re-reads every achievement and faction: minutes rather than
seconds, which is what a patch wants.

A heading a patch introduces reads in English until it is named in `SIMPLEARMORY_LABELS_RU`
(`shared/data/collectionsSchema.ts`) — the one regular hand edit, and the run prints the headings that
map has not got yet:

```text
  ⚠ not in the Russian label map yet (they read in English): Midnight: Season 2
```

The achievements step needs no such edit: it reads Blizzard's own achievement-category index as it
builds, so a `cat` or a `subcat` the index names is already written in Russian. Only a heading the
index has not got (a boss, a zone) falls back to English, and one that should read differently can be
named in `ACHIEVEMENT_LABELS_RU` (`shared/data/achievementsSchema.ts`).

