# Digital Heroes

A subscription-driven golf performance, charity-giving, and monthly
prize-draw platform, built against the *Digital Heroes PRD (Level 1)*.

Live design direction: dark forest ink background, warm amber accent for
prizes, soft green accent for the charity story — deliberately not a
"traditional golf website" per §12 of the PRD.

## Stack

- **Next.js 14** (App Router, TypeScript, Server Actions)
- **Supabase** — Postgres, Auth, Row Level Security, Storage (winner proof
  screenshots)
- **Stripe** — subscription billing (Checkout + webhook)
- **Tailwind CSS**
- **Vercel** for deployment

## 1. Set up Supabase

1. Create a **new** Supabase project (per the PRD's deployment constraints —
   don't reuse a personal/existing one).
2. Open the SQL Editor and run the entire contents of `supabase/schema.sql`.
   This creates every table (`profiles`, `charities`, `subscriptions`,
   `scores`, `draws`, `tickets`, `winners`), the auto-profile-on-signup
   trigger, the "keep only the latest 5 scores" trigger, Row Level Security
   policies, and a private `winner-proofs` storage bucket.
3. Copy your Project URL, anon key, and service role key into `.env.local`
   (see `.env.example`).
4. Sign up through the app once, then in the SQL Editor run:
   ```sql
   update public.profiles set role = 'admin' where id = '<your-user-uuid>';
   ```
   to promote yourself to admin (find the UUID in Authentication → Users).

## 2. Set up Stripe

1. Create a new Stripe account (or a fresh test-mode environment).
2. Create a **Product** ("Digital Heroes Subscription") with two recurring
   **Prices**: monthly and yearly. Copy both price IDs into
   `NEXT_PUBLIC_STRIPE_PRICE_MONTHLY` / `_YEARLY`.
3. Copy your secret key into `STRIPE_SECRET_KEY` and your publishable key
   into `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY`.
4. Add a webhook endpoint pointing at `https://<your-domain>/api/stripe/webhook`
   listening for `checkout.session.completed`, `customer.subscription.updated`,
   and `customer.subscription.deleted`. Copy the signing secret into
   `STRIPE_WEBHOOK_SECRET`.
5. For local testing, use the Stripe CLI: `stripe listen --forward-to
   localhost:3000/api/stripe/webhook`.

## 3. Run locally

```bash
npm install
cp .env.example .env.local   # fill in the values from steps 1–2
npm run dev
```

## 4. Deploy

1. Push this repo to a new GitHub repository.
2. Import it into a **new** Vercel account/project.
3. Add every variable from `.env.example` to the Vercel project's
   Environment Variables (set `NEXT_PUBLIC_SITE_URL` to the deployed URL).
4. Deploy. Re-point the Stripe webhook at the production URL once it's live.

## How the pieces map to the PRD

| PRD section | Where it lives |
|---|---|
| §04 Subscription & payment | `app/dashboard/subscribe`, `app/api/stripe/*` |
| §05 Score management | `components/ScoreEntryForm.tsx`, `trg_trim_scores` trigger in `schema.sql` |
| §06/§07 Draw & prize pool | `lib/draw.ts`, `app/api/draws/run`, `app/admin/draws` |
| §08 Charity system | `components/CharitySelector.tsx`, `app/charities`, `app/admin/charities` |
| §09 Winner verification | `components/ProofUpload.tsx`, `app/winners`, `app/admin/winners` |
| §10 User dashboard | `app/dashboard/page.tsx` |
| §11 Admin dashboard | `app/admin/*` |
| §12 UI/UX | `app/globals.css`, `tailwind.config.js`, `app/page.tsx` |

## Deliberate assumptions (the PRD leaves these open — §16 rewards naming them)

- **Prize pool funding per draw** is modelled as a flat ₹150 contribution per
  active subscriber per month (`POOL_CONTRIBUTION_PER_SUBSCRIBER` in
  `app/api/draws/run/route.ts`), rather than deriving it live from Stripe
  invoice amounts. This keeps the draw engine deterministic and auditable
  for the assignment; swapping in real payment totals only requires changing
  that one constant to a Stripe query.
- **Numbers pool** is 1–49, 5 numbers per ticket, matching the classic
  lottery shape implied by "5-number / 4-number / 3-number match" in §07.
- **Jackpot rollover** is entered manually by the admin when creating next
  month's draw (a "rollover in (₹)" field), rather than auto-chaining, so
  admins can visually confirm the prior month's unclaimed jackpot before
  carrying it forward.
- **Algorithmic draw type**: ticket numbers are weighted toward a
  subscriber's own recent Stableford scores (see `algorithmicTicket` in
  `lib/draw.ts`) — a concrete interpretation of "algorithm-powered" draws
  that still keeps the full 1–49 range reachable.
- **Admin role** is a column on `profiles` (`role = 'admin'`), promoted
  manually via SQL for the first admin, then toggleable from
  `/admin/users`.

## Testing checklist coverage (§16.1)

Signup/login, subscription checkout, rolling 5-score logic, draw
simulate/publish, charity selection + percentage, winner proof upload +
admin verification + payout marking, user dashboard modules, admin panel
modules, and responsive layout are all implemented and exercised through
the flows above. Edge cases handled: duplicate score dates (upsert +
unique constraint), non-subscribers hitting `/dashboard` or `/admin`
(middleware redirect), and non-admins hitting `/admin/*` (middleware
role check + RLS as a second layer).
