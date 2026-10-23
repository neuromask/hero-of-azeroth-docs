# Bilingual and SEO
## 🌍 Bilingual

An address carries its language and its region: `/region-eu/gordunni/<name>` is English, the same page
under `/ru` is Russian, and the front door stays on `/`. Only the non-default language spells itself
out, so links stay short — i18n runs in `no_prefix` mode, the pages read the language off the address
and `app/middleware/lang.global.ts` applies it. Headings that come from SimpleArmory are translated by a
label map rather than by message keys: a new expansion costs one line there, not a key in every locale
file.

Every page carries the same pair of plates in its header (`app/components/LocaleSwitch.vue`): a link to
the same page one segment away, with the language being read warmed in the brand gold. It is a link
rather than a router push because the two addresses are one route — a navigation to the route the
browser is already on is dropped as a duplicate - so the pair reads for a crawler too, and the header
switches with no JavaScript at all.

## 🔎 SEO & sharing

- `NUXT_PUBLIC_SITE_URL` is what canonical links, `og:url`, the hreflang alternates, the JSON-LD nodes
  and `sitemap.xml` are built on, so a staging copy never advertises the production host —
  `public/robots.txt` names the same host literally, so the two are changed together.
- Every character page is `og:image`d with its own card (`summary_large_image`), and `/api/card/` is
  deliberately left crawlable in `robots.txt`: a blocked preview path would leave every shared link
  blank.
- `/sitemap.xml` lists the pages of the site itself — the front door (which is the hall of fame), the fourteen
  rankings under it (`/leaderboards/<figure>`, the roots only: a region or a realm under a ranking is a slice of the
  same table), the about page, the policy and the legal notice — and the characters the site has actually rendered
  (kept in `server/data/`, capped at 50,000 URLs) with `xhtml:link` alternates per language. A browser opening it
  gets the page the map doubles as (`public/sitemap.xsl`): who has been looked up and what they were — level, class,
  item level, rating — plus the breakdowns worth reading (level, class, realm, the last two weeks, the biggest
  collections), every figure counted off the map itself, so the page and the file a crawler reads can never disagree.
- The structured data is one node per kind of page, and none of it can buy a sitelink — those are Google's own and
  automated (`developers.google.com/search/docs/appearance/sitelinks`): the front door publishes an `Organization`
  and a `WebSite` joined by `@id` (`siteGraph` in `app/composables/seo.ts`, the site's only entity node), the about
  page an `AboutPage` naming the site as its subject, and a ranking a `BreadcrumbList` through its scopes. Both
  builders are functions rather than objects written inside a page, so they are asserted on
  (`test/unit/seoJsonLd.test.ts`): an `@id` that resolves, an absolute `logo`, no brand said twice in a title.
- Google Tag Manager is wired in `nuxt.config.ts` — the loader in `<head>`, the `<noscript>` frame right
  after `<body>` — and it is wired *behind a consent*: two inline scripts run ahead of the container in the
  same head, the first setting the defaults of Google's Consent Mode, the second handing over the answer of a
  reader who has already given one, read straight out of their own storage — the key is `cookie_consent` and
  the answer is `'true'` or `'false'` (`shared/utils/consent.ts`), with the key absent meaning nobody has been
  asked yet. `localStorage` on purpose: it is not a cookie, so it rides on no request and is never sent to the
  server, and the site's one cookie stays the session the sign-in hands out. The order is the point of it: a
  `default` may only be said once, so a page that lets the container load first has already lost the argument
  — and a reader who said no is never counted at all rather than corrected a moment later. Both scripts are
  built from `shared/utils/consent.ts`, so the key the banner writes and the key the page reads cannot drift
  apart, and the defaults keep `functionality_storage` and `security_storage` granted: those *are* the
  Battle.net sign-in and the state its OAuth round trip carries, and a website cannot ask permission for the
  thing that keeps a reader signed in.
- The question is one line at the foot of the page (`app/components/CookieNotice.vue`, drawn by the root in
  `app/app.vue`, answered through `app/composables/cookieConsent.ts`): a mark, one sentence that says the
  sign-in cookie is not optional and the counter is, a link to the policy, and the two answers — it opens
  with a short step up rather than springing into place, and it is asked exactly once: the answer is
  remembered in the reader's own storage, so the next visit is a page with no question on it, and there is no
  way back into it from anywhere on the site. What it points at is the
  policy page itself (`app/pages/privacy.vue`, `/privacy` and `/ru/privacy`), which is written against what
  the site actually does rather than copied from a template: the sign-in section names the id, the BattleTag
  and the roster the account stores and the profile switch that takes them out of the table, and the
  analytics section says that nothing reaches the reader's device until the question is answered. `denied` is
  not "nothing runs" but "nothing of yours is touched" — the tags still load and count what Google can count
  without cookies. Because the site runs no ads, the question really is one switch: the three advertising
  consent switches keep their default and stay denied. Two things are worth knowing when working on this: the
  tags in the container have to be the kind that honour consent mode (a Google tag or a GA4 tag does, the
  retired Universal Analytics one does not), and Wowhead's tooltip widget — which a shelf loads when it is
  read — is deliberately outside the question, as the figures are not readable without it.

