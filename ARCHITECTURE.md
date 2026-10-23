> **This is the technical README** — the long read about how the site is built: what every table holds, why the
> score is what it is, where Blizzard's data disagrees with itself, and every trap that has already been stepped in.
> It is meant to be read, and kept up to date, while working on the site itself.
>
> Just visiting the project? The friendly page is [`../README.md`](../README.md).

<div align="center">

<img src="../app/assets/img/emblem.png" width="104" alt="Hero of Azeroth" />

# ⚔️ Hero of Azeroth — architecture

**Everything a World of Warcraft character is — on one page, and in one image.**

[🌐 heroofazeroth.com](https://heroofazeroth.com) · 🇬🇧 English · 🇷🇺 Русский · 🌍 EU & US realms

</div>

---

## Contents

**This file is the index.** The documentation was one 1600-line file, which made a question like "how does the guest
merge decide?" a search through everything; it is fourteen files now, one per subject, so that the answer is usually the
file whose *name* is the question. Nothing was rewritten on the way — the prose is the same, moved.

Read them in order for the whole picture, or jump to the one you need. Every file stands on its own.

| File | What it holds |
| --- | --- |
| [`01-overview.md`](architecture/01-overview.md) | What the site does, a real card, and how to run it locally |
| [`02-how-it-works.md`](architecture/02-how-it-works.md) | The request path, the stores, the background jobs, the shape of a page |
| [`03-project-map.md`](architecture/03-project-map.md) | Every file that matters and what it is for |
| [`04-hall-of-fame.md`](architecture/04-hall-of-fame.md) | The front door: the ranking, the score, the rows and the tables |
| [`05-collections.md`](architecture/05-collections.md) | Where the collection numbers come from |
| [`06-reputations.md`](architecture/06-reputations.md) | The journal of one character |
| [`07-comparison.md`](architecture/07-comparison.md) | Two characters side by side: the board, the history, the distribution |
| [`08-data-freshness.md`](architecture/08-data-freshness.md) | The passes that keep the data current, and what they cost |
| [`09-i18n-and-seo.md`](architecture/09-i18n-and-seo.md) | Two languages, and how a page is shared and indexed |
| [`10-accounts-and-alts.md`](architecture/10-accounts-and-alts.md) | Sign-in, rosters, the main, and what an account is |
| [`11-guest-merge.md`](architecture/11-guest-merge.md) | **Guests, the account signature, merging alts, and the backfill** — the biggest single subject on the site |
| [`12-admin-panel.md`](architecture/12-admin-panel.md) | The panel: what each tab does, and what it may do |
| [`13-configuration-and-deploy.md`](architecture/13-configuration-and-deploy.md) | Environment, limits, backups, and how the site is deployed |
| [`14-about.md`](architecture/14-about.md) | Support and the disclaimer |

### Where to look for what

A few questions that come up often, and the file each is answered in:

- *Why does an alt show its account's rating?* — [`04-hall-of-fame.md`](architecture/04-hall-of-fame.md) and
  [`10-accounts-and-alts.md`](architecture/10-accounts-and-alts.md): a ranking holds one row per account.
- *How does the site know two characters are one person?* — [`11-guest-merge.md`](architecture/11-guest-merge.md):
  the three anchors of an achievement log.
- *Why is a figure a dash rather than a zero?* — [`02-how-it-works.md`](architecture/02-how-it-works.md) and
  [`05-collections.md`](architecture/05-collections.md): the site prints what Blizzard answered.
- *What is a background job allowed to do on its own?* — [`08-data-freshness.md`](architecture/08-data-freshness.md)
  and [`12-admin-panel.md`](architecture/12-admin-panel.md).
