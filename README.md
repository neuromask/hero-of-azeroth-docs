# Hero of Azeroth — documentation

The architecture documentation of **Hero of Azeroth** — everything a World of Warcraft character is, on one page and in
one image — together with the screenshots that go with it.

- [`ARCHITECTURE.md`](./ARCHITECTURE.md) — the technical README: the long read about how the site is built.
- [`architecture/`](./architecture) — one file per subject, numbered in reading order.
- [`screens/`](./screens) — the screenshots the friendly README prints.

The site is [heroofazeroth.com](https://heroofazeroth.com); its source code lives in a separate, private repository. The
files here are a copy of that project's own `docs/` directory, kept in step by hand — so this is where the documentation
can be read on its own.

## A note on the relative links

The files are copied **verbatim**, so a few links and images point at files that live in the project rather than in this
repository:

- `ARCHITECTURE.md` links to the friendly `../README.md`, and prints `../app/assets/img/emblem.png` as its heading image;
- a page in `architecture/` may name a source file by path (for example `server/utils/playerScore.ts`) — those are
  references to the code, not links to anything here.

Everything inside `architecture/` and `screens/` resolves on its own.
