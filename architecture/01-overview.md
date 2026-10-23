# Overview

> **This is the technical README** — the long read about how the site is built: what every table holds, why the
> score is what it is, where Blizzard's data disagrees with itself, and every trap that has already been stepped in.
> It is meant to be read, and kept up to date, while working on the site itself.
>
> Just visiting the project? The friendly page is [`../../README.md`](../../README.md).


<div align="center">

<img src="../../app/assets/img/emblem.png" width="104" alt="Hero of Azeroth" />

# ⚔️ Hero of Azeroth

**Everything a World of Warcraft character is — on one page, and in one image.**

Item level · M+ score · achievements · mounts · pets · toys · decor · reputations

[![Nuxt](https://img.shields.io/badge/Nuxt-4-00DC82?logo=nuxtdotjs&logoColor=white)](https://nuxt.com)
[![Node](https://img.shields.io/badge/node-%E2%89%A520-5FA04E?logo=nodedotjs&logoColor=white)](https://nodejs.org)
[![Data](https://img.shields.io/badge/data-Blizzard%20API%20%2B%20Armoury-0074D9)](https://develop.battle.net)
[![Locales](https://img.shields.io/badge/locales-en%20%7C%20ru-9b59b6)](09-i18n-and-seo.md)
[![Unofficial](https://img.shields.io/badge/fan%20project-unofficial-orange)](14-about.md)

[🌐 heroofazeroth.com](https://heroofazeroth.com) · 🇬🇧 English · 🇷🇺 Русский · 🌍 EU & US realms

<a href="https://heroofazeroth.com"><img src="../../app/assets/img/hoa-prev.jpg" alt="Hero of Azeroth" width="880" /></a>

</div>

---

## ✨ What it does

- 🎴 **Character cards** — a profile rendered as a 1200×630 image. Every page carries its own card as
  its link preview, and the header hands you the file to download or the link to copy.
- 🔎 **Look up anyone** — EU and US realms, realm autocomplete, a search history that remembers what
  you looked at, and addresses that are stable enough to share.
- 🏅 **Overview tab** — item level, M+ score, achievement points against what is reachable, mounts,
  pets, toys, decor and reputations, each tile marked `account-wide` or `per character`.
- 🏆 **The hall of fame** — every character the site has looked up, ranked. The front door sums the population up:
  the menu of the site's fourteen rankings, the reader's own place in the overall rating, a banner that opens the
  hall, five plates under it - who is online, and the leaders of mounts, M+, toys and decor - and then the
  population as a whole: where it lives (every realm a link into that realm's own
  ranking), what it plays, which side it plays for, and everything it has gathered between them. Its own entry in
  the navigation row is a control in two halves, the way the collections tab is: the tab opens the hub, and the caret
  beside it drops the same fourteen rankings from any page of the site. The rows are the fourteen rankings — mounts,
  pets, toys, decor, titles, reputations, transmogs, quests, recipes, achievements, honorable kills, honour level,
  M+ score and the overall rating — each at the world's scope, a region's or a realm's, drawn with the portraits,
  the medals and the sides a reader knows from the rest of the site.
- 🐎 **Collection shelves** — mounts, pets, toys, decor and titles drawn against the **whole game**: every
  expansion and event bucket with its own bar, every source ("Raid Drop", "Vendor", a zone) as a block
  of tiles beside its neighbours, and a rail of anchors to jump between them.
- 🧭 **Numbers that mean something** — a shelf counts what can still be obtained (SimpleArmory's own numbers),
  and two switches uncover what it cannot: the retired items and the ones the game has not shipped yet.
  The character view filters every shelf by that character's faction, so a reader sees what this hero can
  actually use; the account view counts both sides. Pets, toys and decor are the account's collections by
  nature, but a character page still filters them, while a guest's account view can read their account-wide
  lists directly from Blizzard; mounts and titles need a Battle.net sign-in for their account view. The
  tiles and the comparison board measure their bars by **the same** shelf total
  (`server/utils/collectionAtlas`), so a figure and the page that lists it never disagree; a short note under
  the bar says where every number comes from.
- 📜 **Activity feed** — the hundred achievements a character earned most recently, with icons and
  filters for all / achievements / PvE / collections / PvP.
- 🏆 **Achievements & reputations** — reputations counted the way the game counts them (an Exalted faction, the
  last renown level, the highest companion rank), and as the account's own union of finished factions across the
  roster, the number the journal and the tile both print, with this character's own ladders beside it.
- 🌍 **Bilingual** — English at the plain address, Russian under `/ru`: pages, SEO alternates, the
  card image, and the collection headings themselves.
- ⚡ **Built to be fast** — Blizzard is read once per character and answered from a stale-while-
  revalidate cache; the game's whole catalogue (collections tree, achievement wording, faction
  ladders) is paid for offline and shipped with the build.
- 🎨 **A look of its own** — dark fantasy theme, the character's own Armoury artwork, Wowhead tooltips
  on hover, mobile-first layout.

---

## 🖼 A real card — live, not a screenshot

<div align="center">

<a href="https://heroofazeroth.com/ru/region-eu/gordunni/%D0%BD%D0%B5%D0%B9%D1%80%D0%BE%D0%BC%D0%B0%D1%81%D0%BA">
<img src="https://heroofazeroth.com/api/card/eu/gordunni/%D0%BD%D0%B5%D0%B9%D1%80%D0%BE%D0%BC%D0%B0%D1%81%D0%BA?locale=ru_RU" alt="A character card" width="760" />
</a>

<sub>The picture above is drawn on demand by <code>/api/card/…</code> and served as a JPEG — 1200×630,
the size a social network wants, with the character's own render and class artwork in it.</sub>

</div>

### The pages behind it

| Address | What it shows |
| --- | --- |
| `/` | Front door *and* hall of fame: the summary of the population — the menu of the site's fourteen rankings, the reader's own place in the overall rating, a banner that opens the hall, five plates under it (who is online now, and the leader of mounts, M+, achievements and decor), and the population as a whole under them (the realms it is spread over, the classes it plays, the sides it plays for, and everything it has gathered, with the day the table was last written) — with the search itself standing in the navigation row, in the place a character's own entries take, while this browser knows no character and nobody is signed in. The rows are the rankings the menu leads to |
| `/region-eu/<realm>/<name>` | Overview: the stat tiles, the render, download / share / refresh |
| `/region-eu/<realm>/<name>/collections` | Collections summary: a tile per shelf in two rails around the character's render, the running total pinned to the bottom |
| `/region-eu/<realm>/<name>/collections/mounts` | A shelf: sections with bars, sources side by side, the three switches, the note |
| `/region-eu/<realm>/<name>/achievements` | Achievements summary: a tile per category in two columns on each side of the render, the running total pinned to the bottom |
| `/region-eu/<realm>/<name>/achievements/dungeons` | One achievement category: sections with bars, subcategories side by side, the tile grid |
| `/region-eu/<realm>/<name>/activity` | The achievement feed with its category filters |
| `/ru/…` | The very same pages in Russian |

The hall of fame's old address, `/leaderboard`, is a 301 to `/` (`nuxt.config.ts`, `routeRules`): the two are one
page now, and only the address that no longer names anything redirects - every other page of the site still
refuses to.

### One row of navigation, in two groups — and a panel for a phone

Every page wears the same row (`app/components/SiteNav.vue`), and it is deliberately *grouped*. It is also the only
place the languages, the account and the search are drawn at all - a header of a page that has no row of its own is
handed this very component - so no page can carry them twice. Six of the row's entries are about the character
being read: the profile - which carries the character's own name and portrait rather than the word "character", so
a reader always sees who the bar is about - the collections, the achievements, the reputations, the feed and the
comparison. Two of
them are the site's own and answer a different question: the hall of fame the figures are ranked in, and the search
that opens the next character. Those two stand at the far end of the row, past a hairline, in the corner the
languages and the account already close the row with. A page that is not about a character - a document, the account
page - wears the same row with those entries left out.

Below `lg` that row does not fit, and a row that wraps to three lines costs a phone a whole screen of the page it
came for. So the entries that are navigation move into the panel the burger opens - the six of a character and the
hall of fame, under two headings, with the collections and the achievements as lists indented under the entry that
opens them (a pane dropped inside a pinned bar would cover the panel it came from, and a phone reads an indented
list better than a second layer of glass). What stays in the row is the search, the languages and the account: the
one thing a phone must always have within reach, and the corner every row of the site ends in. The button carries
`aria-expanded` and swaps its label between opening and closing in the reader's own language, Escape and a press
outside the panel close it, and picking an entry closes it too - the link that was just followed is the page being
read.

## 🚀 Quick start

```bash
npm install
```

Create a `.env` with a Blizzard API client
([Creator portal](https://develop.battle.net/access/clients) → client credentials):

```bash
NUXT_BLIZZARD_CLIENT_ID=...
NUXT_BLIZZARD_CLIENT_SECRET=...
```

Then:

```bash
npm run dev       # development server on http://localhost:3100
npm run build     # production build into .output
npm run preview   # run the built server locally
npm run deploy    # build, push the sources, upload the build to the host (see Deploying)
```

The development port is **pinned to 3100**, and it is not a preference. `nuxt dev` resolves its port
as `--port`, then `NUXT_PORT`, then `NITRO_PORT`, then `PORT`, and only then `devServer.port` - and it
reads those variables before `nuxt.config.ts` is loaded, so a stray `PORT` inherited from another
project or an editor would quietly move the dev server elsewhere. Both scripts therefore pass the
port on the command line (`nuxt dev --port 3100`), which nothing else can override, and
`npm run dev` first checks the port is free (`scripts/check-dev-port.mjs`) so a second dev server
fails with a message that names the cause instead of drifting to another port - which would leave the
Battle.net callback, registered against `http://localhost:3100/api/auth/callback`, refused on the
way back.

---

