# A guest who is somebody's alt

The site cannot know which guests are one person's characters: a guest has no account, and every account-wide read of
the site is keyed by `user_id` — so a guest's collections are its own, while the game keeps most of a collector's
things for the whole account, and the hall of fame holds two rows where the game holds one person. What the site *can*
read is the one thing the game wrote down about the account rather than about the character: an achievement the
account owns carries **the account's completion second**, so every character of one account reports the very same
moment for it, while a character's own achievement carries whichever character earned it.

**That is the whole idea of the fingerprint** (`server/utils/achievementFingerprint`,
`server/utils/characterFingerprint`, `#shared/data/accountFingerprint`). A character's log is reduced to the moments its
*account* owns, and out of those the site reads the three achievements that are the account's **signature**
(`SIGNATURE_ANCHORS`): two characters reporting the same three seconds are one person, exactly, with no threshold to set
and no verdict to explain. Collections cannot do this — they grow, so two readings of one account are never equal — and
that is why this replaced the collection sketch the feature was first built on.

Three filters turn a log into that set, each measured rather than guessed. **Only what the account owns**:
`is_account_wide`, Blizzard's own answer, kept in the catalogue the site already ships
(`server/utils/achievement-meta.json`, filled by `npm run refresh:achievement-meta`; `npm run
refresh:achievement-flags` fills it into a catalogue built before the flag was read). It does most of the work: a
raid's kills are a character's own achievements, and a whole raid group earns them at the same second. **Not a burst**:
a second in which one character earned several achievements is a hand-out of the service — a patch that converts
exploration, a transfer, a warband conversion — and not a play. **Not handed out**: a moment that two **accounts** answer to is a service hand-out or a boss a group killed
together, and it says nothing about whose alts two characters are. Accounts, deliberately, and never guests: two
guests holding one moment is exactly what a genuine match looks like, since the characters being placed hold it by
definition. It is read off the *signature registry* rather than off everyone's stored logs
(`server/utils/accountSignatures`), so applying it costs no log at all: a moment two accounts' signatures carry is a
grant, a moment one account's carries is that account's own.

**What the anchors replaced, and why the measurement is still here.** The first version of this feature compared two
characters by the *share* of their histories, and it separated perfectly on the sample it was measured against — 262
characters of 35 real accounts, 898 pairs of alts (Blizzard's own rosters, not a guess) and 3487 pairs of strangers on the
same realm, where the *weakest* alt pair shares 35.4% of its smaller side while the closest stranger pair shares 0.0%. A
count cannot separate them: two people who raided together share one night's achievements at the same second (121
moments), more than the weakest real alt pair shares (28), while a share is 3% of a history against 35%. A fence in the
middle of that gap (`FINGERPRINT_MIN_OVERLAP` 0.25, over the floors `FINGERPRINT_MIN_SHARED` 20 and
`FINGERPRINT_MIN_ITEMS` 20) confirms every one of the 898 alt pairs and none of the 3487 stranger pairs — and it is still
a *judgement*: a number somebody chooses, and a knob an operator can move. The anchors make it an identity instead, so the
fences are gone from the code, and what survives of these three numbers is the measurement that says why they were not
shipped.

The moments cost nothing to collect. The site reads a character's whole achievement log on three paths, and none of them
pays for this on its own: the Activity view and the Achievements shelf read it to draw what they draw (`./activity`,
`./achievementCollections`), and a profile read of a character the site has **never held** reads it for the three anchors
that make that character placeable (`fileCharacterSignature` in `./profileFeed`). The first two are on-demand and cached;
the third is one call on top of the fifteen a profile read already spends, asked once per character, because a guest
without a signature can never be placed and a character that is never placed never joins an account's figures. Both kinds
of read file what they read as a by-product, the way a page view files a reading. **The log itself is not stored.** What
is kept is the answer it was read for: the three anchors as `id@at|id@at|id@at`, forty-odd characters in one column
(`character_fingerprints.signature`), and NULL for a log that holds fewer than three. The eleven kilobytes of moments
this feature first kept answered exactly one question, and that question is answered by the string. The value
**never travels in a response**: it is the site's own claim about a stranger.

**The guild crawl is a fourth door onto that third path, and it is the one nobody expects.** A crawl reads the members of
a guild nobody has opened a page for (`./guildCrawler`), and each of those reads is a profile read with `reading: true` -
so a member the site has never held is filed as a guest *and* has its log read for the three anchors, exactly as a
stranger's first page view does. That is how a population of guests grows on a site whose pages nobody visits: a crawled
guild hands the panel a set of characters, some of which carry a signature that matches an account the site already has
(they are **attached** on the spot, `placeCharacterFromLog`) and some of which name nobody - and those are the ones that
turn up as **groups** waiting to be folded. The crawl never merges: it reads, files and places, and building an account
out of a group stays an operator's decision (`./accountMerge`). Two members it does not touch are worth naming, because
both look like a gap and neither is one: a character an account already holds is left to that account's own scan, and a
guest read less than a day ago is `cached` rather than re-read - so a log that named nobody is not re-read by the crawl
either, which is what the backfill below is for.

A row exists **only for a guest**, and that is the shape the whole feature is built on now: a character whose owner the
site knows has no question left, so its signature goes on the account instead (`users.signature`) and no row is written
for it at all. The account's own signature is the same value in the same format — every alt of one person reports the
same three moments — and it is what makes an account one the site can *recognise*. It is filled by the first read that
can tell, by a claim that copies a guest's own signature over (`claimGuestCharacter` in `./accountSync`), and by a merge
that writes the group's on the account it builds. From then on placing a new guest is one lookup in that registry
rather than a scan of the population.

The hand-out rule is applied when a signature is **read**, never when it is written (`placedSignature`), and that is
why a row keeps the three raw anchors rather than a filtered signature: what counts as a hand-out grows as the site
recognises more accounts, so a value filtered on the day of the read would never learn that one of its anchors had
since been granted to somebody else.

**That is what the site does about it** (`server/utils/guestAccounts`, `server/utils/accountMerge`,
`#shared/data/guestMerge`, `#shared/data/accountFingerprint`). A guest whose log carries the three anchors is identified
*exactly*: the three seconds are the account's own, so two characters reporting them are one person with no threshold, no
share and no coincidence. A guest whose log does not — fewer than three anchors, which is a character read before an
anchor was earned or an account that never earned one — is **left alone**: it is counted on the panel
(`GuestMergeTotals.unanchored`) and merged with nothing, because the site has nothing to say about whose account it is.
A group carries its three anchors as the evidence it rests on, and once merged
the characters are one row of the hall of fame. The population question — which guests go together at all, and which of them are somebody's already — costs one
pass and one lookup each: every guest's signature is a column beside its own row, the characters are bucketed by that
string, and each bucket is looked up once in the registry of account signatures. No pair is measured, no graph is
walked and nothing is thresholded. What the read holds per character is one short string rather than a copy of a
history, and what it compares against is one small column rather than a second copy of the population — so it reads the
whole fingerprinted population rather than a slice of it, and the ceiling that used to stand here went with the log it
was a ceiling on.

**Two answers, because there are two situations.** A signature names either nobody — and then the guests carrying it are
one account the site has yet to **build** — or an account that already exists, and then those guests are its **alts**.
That second case is why the panel carries two lists rather than one: a guest that names an account the site has is not a
member of any group to be built, and leaving it in the group list is exactly what would let the panel build a *second*
account for somebody who already has one. The write refuses the mistake from either direction — a merge whose signature
an account already answers to is refused (`signature-taken`), an attach with nothing to attach to is refused
(`no-standing-account`) — so a guest is only ever in one of the two lists.

The writes keep five rules. **Never somebody's account**: only characters with no owner are ever moved, so the site's
record of a real player is not a candidate for anything — an attach moves a character *into* an account, never out of one.
**Evidence before action**: both paths measure first and file what they measured on the record (`account_merges`) — the
characters and the signature they were accepted by — so what is on the row is what was true when it was written, and there
is nothing in it for a caller to override: a set the signature does not describe is refused whether a pass or a person
asked. **The account says who it is**: a merge writes the group's signature on the account it builds, which is what makes
it recognisable to the next guest that turns up for it. **One transaction**: the account, the roster move, its first
reading, the guest rows leaving the table and the record go in together, so a failure leaves no account holding a third of
a group. **An undo is a first-class move**, and what it means is read off the record's own `kind`: a *merge* detaches its
roster before deleting the account it built (the characters cascade with it), while an *attach* detaches the character and
leaves the account exactly where it was — signature, readings and row in the hall of fame — because that account is
somebody's, and a rollback has no business touching it. Either way the members return as guests and the record stays on
file marked undone: a history rather than a delete.

**And the account a merge built is one a person can turn out to own.** A guest is a character with no owner, so a merge
cannot tell whose it is beyond the signature - but the person behind it can prove it: the day they sign in with Battle.net,
their roster (Blizzard's own answer, fetched with their own token) names the characters. That is the third write of this
story, and it lives with the roster rather than here (`server/utils/accountSync`). A reader with no account *adopts* the row
the merge built: the identity is written onto it, nothing moves, and the roster read simply completes it. A reader who
already had an account gets the other half: the derived account's characters move into theirs, and what is left is a shell
that is marked (`absorbed_into`), empty, and stripped of the signature the characters carried over - a signature two
accounts spell is one the registry answers nobody with. Both halves exist for one reason: a character belongs on one row,
and `characters` is unique per *account* rather than per address, so a second account beside the first would put one
character in two places - two histories for one character, and the same collection in two rows of the table. The invented
subject a merge wrote is kept as `derived_subject` on the account that adopted it: the panel prints it, nothing public does.

**And a character that changes owner is read again.** The figures an account keeps as its own - mounts, titles, quests, recipes -
are folded in by a read taken *while it holds* the character: Blizzard publishes a title, a quest and a recipe per character
only, so an account's own number for those three is the union of its characters' ids (`account_pool`, `syncPoolFigures`), and a
read taken while the character was nobody's files its own reading and folds nothing into anything. So a merge, an attach and an
absorption set `characters.owes_account_read` on the characters they move, which asks the background pass for one full read of
each of them. A flag of its own rather than `stats_read_at = NULL`, and that is not a detail: every read writes that column, a
page view included, so a mark kept there was gone before the pass - the only reader that folds the five lists - ever ran.
Without either of them, a character read a moment before it changed hands kept the fresh snapshot it already carried and stayed
out of the pass for a day, and the merged account's mounts, titles, quests and recipes looked like one character's for exactly
that long.

**The automatic pass ships off**, and its switch is the one setting here that is a decision rather than a memory: it runs
on its own timer (`server/utils/guestMergeScheduler`), takes the groups a signature found, and at most a handful per pass.
**It is the only thing on the site that folds a group without a person**: every other write about strangers - a group's own
button, and the pass run by hand - is two presses, and the panel's pass button asks first and writes second
(`planGuestMergePass`, which is the same reading as the pass and none of the writing). There is no rule among its settings
to move — a group is one account by construction, so the only question a setting can answer is how much work the job may do
unprompted.

**Which character an account leads with is kept in step with the rule, and a person's own choice is the one thing that stops
it.** The row a merged account holds in the hall of fame is named after the character it leads with (`leaderboardRows`,
`ORDER BY is_main DESC, level DESC, name ASC`), and the *rule* for picking one is the strongest character of the account - the
highest level, then the best gear, then the name, then the realm (`leaderOf` in `./guestAccounts`; the second step is the one
that matters, because a level is capped and almost every character of an account is at it). Every write that moves a roster applies it
again (`refreshAccountMain` in `./accountSync`: a merge, an attach, an absorption, a roster read), because the case it exists
for cannot be foreseen at the merge: a stronger alt attached a week later, a level that arrives when the backfill finally reads
a log. A row named by a running computation would move under the account silently, and a row named once would stay wrong for
ever - so the rule *writes* its pick and re-applies it, and the one thing that overrides it is a person (`setAccountMain`,
which pins the choice: `users.main_pinned`).

The panel is where that happens and where it is undone: the merge tab lists the accounts a merge built, one line each with the
character leading it and the roster behind a fold, says when a pinned leader is no longer the strongest, and offers both a
*dot in the table* that names a character the leader and a "follow the rule again" button beside the account's name. The tab
is read through **three views of its own** (the groups, the accounts a merge built, and the merges on file), and the two
lists of things that already happened carry a **search by name** beside their page — the name of a character, of a BattleTag,
of an account — because the question an operator arrives with is "what did the site do with this person", and a site that has
been merging for a while has thousands of rows to answer it from. That dot
is the one control on this screen that writes in a single click, and the difference is what the write is: a merge moves
characters between accounts, while naming a leader moves none - it changes which of the account's *own* characters its row is
named after, and the next click moves it back (an account's owner does exactly this on their own profile, `/api/profile/main`).
An account somebody signs in with is refused by the write (`not-derived`): its main is its owner's to choose.

**And the whole population can be folded in one press.** The scheduled pass takes the handful the settings name, so that an
armed switch is a drip rather than a flood; the button beside it takes every group there is, which is the same reading, the
same write and the same answer with the ceiling lifted (`all: true` in the body, `server/api/admin/guests`). Its first press
is the same check the other button makes - the figures an operator decides on - so "merge every group" is still two presses
and still says how much it is about to write.

**The backfill is the other half of that**, and it is the one job that reads a log nobody asked for
(`server/utils/signatureBackfillQueue` for the queue, `server/utils/signatureBackfill` for the pass). A guest is placed by
its signature, so a guest nobody ever opens a page for is one the site can never place - and its readings never join any
account's figures, however long the site holds them. Measured on the production database, that is **902 guests at the first
deploy**, plus 35 accounts with no signature of their own, which no guest could ever be attached to. The queue is those
three states and only those: a guest with no log on file at all, a guest whose log named nobody and whose retry window
(thirty days) has passed - an account *earns* those anchors over time - and an account with no signature, marked asked
(`users.signature_asked_at`) so that a young one is not read on every pass. The pass reads one batch of those, hands each
guest to the account its log names, and **never builds an account**: it makes the same call a page makes
(`placeCharacterFromLog`), so it places and files, and the one write that stays a person's is the merge. It ships **on**,
unlike the automatic merge, and the difference is the reason: a merge writes rows of the site's public table and an
operator should watch that work once, while a backfill moves nothing and changes no page - it files forty characters beside
a row, one call per character, once ever. Its batch is two presses like everything else here, and the first press only
counts the queue.

**The switch reaches the merge and not the attach**, and that is deliberate rather than an omission. Two things place a
guest without anybody pressing that guest's own button: the pass, and the *read* that produced the evidence in the first
place — `placeCharacterFromLog` files what a log says and, when its signature names an account the site knows, hands the
character over in the same call (`./activity`, `./achievementCollections`). A page view placing an alt is the conservative
half of the same idea, because it names a character the site could not place rather than inventing an account for one;
widening `autoMerge` to cover attaching as well would change what an operator agreed to when they turned it on, so it is a
decision to make on purpose. The thresholds that were the whole feature in its first version (`FINGERPRINT_MIN_OVERLAP`,
`FINGERPRINT_MIN_SHARED`, `FINGERPRINT_MIN_ITEMS`) are **gone from the code**: they are the measurement that proves a share
separates alts from strangers, and they are not a rule the site applies any more.

**"Held by every alt" and "held at the same moment" are two different facts**, and the gap between them is where every
hash idea dies. Measured on the same sample: **72 achievements are held by every alt of all 35 accounts** — the
`Explore <zone>` family (Dun Morogh, Durotar, Mulgore, Elwynn, Teldrassil… about forty of them), `50/100/250/500
Quests Completed`, `Got My Mind On My Money`, `The Bread Winner`, `Stable Keeper`, `Can I Keep Him?`, the `Fashionista`
family — so at *id* level the answer to "is there such an achievement at all" is yes, and comfortably so (7567 of 8021
achievements in the sample are held by every alt of at least one account; the median achievement is complete in 16 of the
35). But a fingerprint compares **moments**, and there the same achievements fall apart: `Explore Dun Morogh` is held by
every alt of all 35 accounts yet at one and the same second in only **18** of them — inside a single account the
seconds read `1224409449, 1224409440, 1286902759, …`, i.e. each character carries *its own* completion time for an
achievement the account owns. Hashing that set splits one account into several values.

Exactly **three** achievements are held by every alt of all 35 accounts *and* at one and the same second:
`Fledgling Hero of Warcraft` (10657), `Fashionista: Weapon & Off-Hand` (10689) and `A New Friend` (15794); a further 78
are stable in at least 30 accounts (the `Fashionista` family, `50/200 World Quests Completed`, `Toying Around`,
`Explore Azj-Kahet`…). Their three seconds give **35 distinct signatures for 35 accounts, with every alt of every
account agreeing** — an exact membership test with no threshold at all, and **the identity the site now leads with**
(`SIGNATURE_ANCHORS`). One thing had to be fixed for it to be usable: the burst rule ate the anchors themselves (a
level-20 ding lands with other achievements), taking `10657` from 262/262 characters to 225 and leaving a complete
signature for only 16 of the 35 accounts — so an anchor is exempt from that rule, because it is *read as a constant*
rather than compared as evidence. What the signature cannot do is *find* an account: it compares one character against a
known account, and a character that has earned none of the three has no signature at all — which is exactly the young
account with no history to merge, and the one case the site leaves alone rather than guessing about.

**Why a hash of that *core* cannot be the identity** — the same question in its stricter form, and what the measurement
says. The tempting shape is: take the achievements *every* alt of an account holds, hash them, and add a new alt by
looking its hash up. Of the 184 achievements **every one of the 35 sampled accounts** holds, **not one** is held by both
sides of every alt pair: the best is shared by 877 of the 898
pairs (97.7%), the median by 651 (72.5%), the worst by 29. Per account it is worse still — *every* alt holding it is
true for no account at all across all 184, only a median of 21 of the 35 — because an alt differs from its siblings by
*what it plays*: the M+ season, the delve tier, the event. Hashing each character's own core and comparing hashes
routes 262 characters into 194 distinct values (so it invents no false matches, which a moment-bearing hash never
will) but recognises **125 of 898 real alt pairs — 13.9%**. It is a recall failure, not a precision one, and it is
structural: the record a hash compares is a set of *moments*, and moments differ the moment a character is younger
than its siblings.

What a stored core *can* do is **confirm**: holding a character against an account's core is 0.538 of it at the
median and never below 0.176 for a real alt, while against a stranger's core the best of 8908 pairs is **0.000**. That
is a useful fifth stage — "is this guest an alt of account #N?" — and it belongs *after* the merge that creates the
account, never before it: the intersection it needs is a property of a group, and a group is what the signature above
establishes in the first place. The intersection itself is never tiny (median 653 moments, and 28 at its smallest
across the sample), so nothing here is a matter of too little data.

Register the callback in the Battle.net developer portal before the first sign-in:
`https://heroofazeroth.com/api/auth/callback` (plus a localhost copy for development). It must match
`NUXT_PUBLIC_SITE_URL` exactly.

