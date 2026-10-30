# Seeds Tycoon — handoff

Influencer affiliate program for **amsterdammarijunaseeds.com** where each influencer's dashboard is a cartoon "grow your garden" tycoon game. Influencers put a link in their Instagram bio; followers land on a curated seed shop; clicks and orders make the influencer's garden grow and earn them commission.

Status: design approved (**Sunny Garden** cartoon style), clickable prototype done, backend not started.

- `prototype/index.html`: standalone clickable prototype (vanilla JS, demo data, no backend). Open it in a browser.
- `design/*.dc.html`: design source from the Claude design canvas (Sunny Garden desktop, mobile, prototype). Reference only.

---

## 1. Decisions so far

- The **influencer dashboard** is the game (Sunny Garden: sky blue, thick dark outlines, Lilita One + Nunito, plants with faces).
- The **customer-facing shop** that followers land on stays a normal, fast, trustworthy, grown-up seed shop in the influencer's branding. Not gamified. **18+ age gate** on every influencer shop.
- The game is a skin over real numbers. Every number in the game must come from real clicks, orders and commissions. Nothing fake.
- Attribution uses first-party tracking only (no third-party cookies).
- Build **on top of the existing affiliate module** in the seed platform (referral codes + commission tracking already exist), don't create a second one.

## 2. Game rules

### Metaphor

| Game | Real thing |
|---|---|
| Plot / plant | A strain the influencer features in their shop |
| Water | Clicks on the influencer's link for that strain (last 7 days) |
| Growth % | Orders since the last harvest ÷ harvest target |
| Harvest | Plant reaches 100%: influencer claims XP, plant resets to seedling |
| XP / level | Progress; level sets the commission rate and unlocks perks |
| Coin jar | Commission wallet (pending / available / paid) |

### Plant stages (from growth %)

`<30` Seedling · `30–59` Growing · `60–99` Flowering · `100` Ready.
Faces: sleepy (seedling) → smile → big grin + "Ready!" sticker. Show "Thirsty!" when the plot had 0 clicks in the last 48 h.

Proposed rule: **harvest target = 3 orders** of that strain (tune later). Growth must be computed server-side from orders; the prototype's "simulate a day" growth is fake.

### XP sources (proposal, tune later)

| Event | XP |
|---|---|
| Order attributed (counted when the commission is approved, not at checkout) | +40 |
| Harvest a plant | +120 |
| Quest reward | per quest (50–300) |

XP is an append-only event log (`xp_event`), summed for the total. Never stored only as a mutable counter. Reversals (refunds) add a negative event.

### Levels (starting point from prototype)

| Level | XP from | Title | Commission | Perks |
|---|---|---|---|---|
| 7 | 2,000 | Master Grower | 12% | – |
| 8 | 3,000 | Garden Boss | 15% | +1 plot, golden watering can badge |
| 9 | 4,200 | Seed Baron | 15% | custom shop banner/colours, early access to drops |
| 10 | 5,600 | Seeds Tycoon | 18% | exclusive strains, season bonus eligibility |

Levels 1–6 still need defining (suggested: 0 / 100 / 300 / 600 / 1,000 / 1,500 XP, 10% commission, 3–6 plots). Level is computed from total XP; the commission rate is **snapshotted onto each commission** at order time.

### Quests

Weekly set, reset Monday 00:00 Europe/Amsterdam. Examples: share link in a Story (+150), 5 orders this week (+300), first 10 clicks today (+50), keep bio link live 7 days (+200). Rewards are claimed manually (the "Claim reward" moment is part of the fun).
Story detection: give influencers a Story-specific link (`/r/{handle}?s=story`) and complete the quest on the first click from it. No honour-system button in production.

### Streak

A day counts if the influencer's link got ≥1 click that day. Show it, maybe reward 7/30-day streaks with XP.

### Seasons and leaderboard

Monthly season, ranked by XP earned in that season. Show real handles only if influencers opt in; otherwise "Creator A/B/C". Top 3 prize: [TO DECIDE].

## 3. Money

- Commission = rate (snapshot) × order subtotal excl. VAT and shipping (to confirm).
- Ledger states: `pending` (order placed) → `approved` (delivered + 14-day return window) → `paid` (in a payout). Also `reversed` (refund, chargeback, cancellation).
- Payout minimum €50, paid monthly (or on request). Use Stripe Connect (already in the platform) or SEPA bank transfer.
- Self-billing invoices for influencers (NL/EU); store KvK/VAT number where applicable.
- Show the influencer: available, pending, earned this month, payout history.

## 4. Attribution

1. Bio link `https://amsterdammarijunaseeds.com/r/{handle}` (optionally a subdomain `{handle}.…`). The route logs a click, sets a **first-party** cookie `aff_ref` (30 days, `SameSite=Lax`, same domain) and redirects to the influencer's shop page.
2. Instagram opens links in its in-app browser; buyers often switch to Safari/Chrome and lose the cookie. So **every influencer also gets a discount/referral code** as the fallback that always works.
3. At checkout: code beats cookie; cookie is last-click within 30 days. Store `affiliate_id` on the cart/order metadata.
4. On the first attributed order, link the **customer** to the affiliate (e.g. 12 months) so repeat orders keep counting.
5. Fraud rules: no commission on self-purchases (same customer/email/payment fingerprint as the affiliate), on codes leaked to coupon sites (flag spikes from non-Instagram referrers), or on cancelled/refunded orders. Store click IP/UA only as salted hashes (GDPR).

## 5. Data model (Medusa v2 module, extending the existing affiliate module)

- `affiliate`: id, handle (unique), code (unique), status (pending/active/suspended), display_name, avatar, rate_override?, payout_method, kvk/vat?, accepted_terms_at, created_at
- `affiliate_plot`: id, affiliate_id, product_id, slot, orders_since_harvest, harvest_target, harvest_count, last_click_at
- `affiliate_click`: id, affiliate_id, plot_id?, source (bio/story/other), ip_hash, ua_hash, created_at
- `commission`: id, affiliate_id, order_id, amount, rate, status, available_at, reversed_reason?
- `xp_event`: id, affiliate_id, type (order/harvest/quest/reversal/streak), amount, ref_id, created_at, unique (type, ref_id) for idempotency
- `quest_template` + `affiliate_quest`: week, progress, goal, reward, claimed_at
- `season`: id, starts_at, ends_at, prize
- `payout`: id, affiliate_id, amount, status (requested/processing/paid/failed), method, reference, created_at

Module links: commission ↔ order, affiliate ↔ customer (referred customers), affiliate_plot ↔ product.

### Workflows, subscribers, jobs

- `order.placed` → resolve affiliate (code > cookie > linked customer) → create `pending` commission, bump plot `orders_since_harvest`, quest progress.
- order canceled / refund / return → reverse commission + negative `xp_event`.
- daily job → approve commissions past `available_at` (+ order XP), update streaks, flag fraud.
- weekly job (Monday) → new quests. Monthly → close season, create payout batch.

### API

- `GET /r/:handle` (storefront route): log click, set cookie, redirect.
- Affiliate auth: own actor type `affiliate` in Medusa v2 auth (email magic link or Instagram login).
- `GET /affiliate/me/dashboard`: everything the game screen needs in one call.
- `POST /affiliate/plots/:id/harvest` (server checks it's at 100%, idempotent).
- `POST /affiliate/quests/:id/claim`.
- `GET/POST /affiliate/payouts`.
- `PUT /affiliate/shop`: choose strains for plots, banner (when unlocked).
- Admin: approve affiliates, see ledger, run payouts, edit levels/quests.

## 6. Frontend

- New Next.js app (or route group) for the dashboard, e.g. `play.amsterdammarijunaseeds.com`, behind affiliate login. Port `prototype/index.html` to React components (Header, LevelCard, Tabs, Garden/Plot, Quests, Ranks, CoinJar, Share, LevelUpModal, Toast).
- Mobile-first: influencers use it on their phone. Prototype already reflows.
- Influencer shop page on the storefront: influencer's name/avatar, their picks (= plots), normal product cards and checkout, 18+ gate.
- Onboarding: sign-up form → admin approval → pick handle + first 3 strains → get link + code.

## 7. Compliance checklist

- Instagram/Meta prohibits promoting the sale of cannabis, including seeds. Neutral bio wording, no product imagery or "buy" wording in posts (shown in the Share tab). The influencer's account is the whole channel.
- 18+ gate on influencer shops; don't target minors; keep the public shop adult-looking.
- Geo-restrict checkout to countries where selling seeds is allowed (platform already has geo-blocking).
- Affiliate content must be labelled as advertising (#ad) under EU rules; put it in the influencer terms.
- GDPR: hashed IP/UA, privacy notice for influencers and buyers.
- Get the influencer terms checked by a lawyer.

## 8. Suggested build order

1. Affiliate entity + `/r/:handle` click tracking + code at checkout + commission ledger (reuse existing module).
2. Plain stats in the dashboard (clicks, orders, commission) to validate the numbers.
3. Game layer on top: plots, growth, harvest, XP log, levels.
4. Quests, streaks, seasons/leaderboard.
5. Payouts + self-billing invoices.
6. Onboarding + admin screens.

## 9. Open decisions

- Harvest target, XP values and the level curve (levels 1–6).
- Commission base (excl. VAT/shipping?) and the customer-lock period.
- Subdomain per influencer vs `/r/{handle}` path.
- Season prize and whether real handles appear on the leaderboard.
- Payout method: Stripe Connect vs SEPA.
