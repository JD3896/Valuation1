# DCF Valuation Lab — Hosted SaaS Setup

This folder is a ready-to-deploy Vercel project:
- `index.html` — your DCF tool, now with a "Fetch Live Price" button
- `api/quote.js` — serverless function that securely fetches live US stock
  quotes from Finnhub (your API key never touches the browser)
- `api/check-subscription.js` — stub for gating access to paying users

Nothing here needs a server you manage — Vercel runs the `api/` functions
for you on-demand.

---

## Step 1 — Get it live (no billing/login yet)

1. Create a free account at https://vercel.com and https://finnhub.io
2. Push this folder to a new GitHub repo
3. In Vercel: "Add New Project" → import that repo → Deploy
4. In Vercel Project Settings → Environment Variables, add:
   `FINNHUB_API_KEY = <your Finnhub key>`
5. Redeploy (env vars only take effect on a fresh deploy)

At this point you have a live URL (e.g. `yourproject.vercel.app`) where
anyone can use the tool and pull live US stock prices. No login, no
paywall yet — that's steps 2-3.

**Optional: your own domain.** Buy one (~$10-15/yr, e.g. via Namecheap or
Google Domains) and point it at Vercel under Project Settings → Domains.

---

## Step 2 — Add login (Clerk)

1. Sign up at https://clerk.com (free up to 10k monthly active users)
2. Create an application, grab your Publishable Key + Secret Key
3. Add both to Vercel env vars (see `.env.example`)
4. Follow Clerk's "Add to any website" quickstart
   (https://clerk.com/docs) — it's a `<script>` include plus a
   `<div id="clerk-*">` mount point, similar in spirit to what's already
   in `index.html`. It gives you a `<SignIn>` widget and a JS API to check
   "is someone logged in" (`window.Clerk.user`).
5. Wrap the DCF tool's main content so it only renders once
   `window.Clerk.user` is truthy; otherwise show the sign-in widget.

---

## Step 3 — Add subscription billing (Stripe)

1. Sign up at https://stripe.com
2. Create a Product (e.g. "DCF Lab — Monthly") with a recurring price
3. Use a **Stripe Payment Link** (Dashboard → Payment Links) — no code
   needed for the checkout page itself
4. Set up a webhook endpoint pointing at a new `/api/stripe-webhook.js`
   function (not included yet — Stripe's docs have a copy-paste example
   for `checkout.session.completed` and `customer.subscription.deleted`)
5. On each webhook event, write the user's status ("active" /
   "cancelled") into a Supabase table `subscriptions(user_id, status)`
6. That's exactly what `api/check-subscription.js` reads from — fill in
   `SUPABASE_URL` and `SUPABASE_SERVICE_KEY` in your env vars once your
   Supabase project exists (free tier at https://supabase.com)

**Linking Clerk users to Stripe customers:** the simplest approach is to
pass the Clerk user's ID as `client_reference_id` when redirecting to the
Stripe Payment Link, so the webhook payload tells you which app user just
subscribed.

---

## Step 4 — Gate the tool

In `index.html`, before rendering the DCF sections, call:

```js
const res = await fetch(`/api/check-subscription?userId=${window.Clerk.user.id}`);
const { active } = await res.json();
if (!active) {
  // show a "Subscribe" button linking to your Stripe Payment Link instead
}
```

---

## Important: live data licensing

Finnhub's free tier (and most competitors' free/cheap tiers) license data
for personal or internal use. Reselling that data as part of a paid
product typically needs a commercial/redistribution license from the
provider or the underlying exchange. Before you charge customers, it's
worth reading Finnhub's terms (or Polygon.io's, or Twelve Data's) for
their commercial-use clause, or emailing their sales team to confirm your
use case is covered. This is a licensing question, not a coding one — I'm
not a lawyer, so treat this as a flag to check rather than a final answer.

---

## Order of operations if you want to build this incrementally

1. Ship step 1 today — live tool, no paywall, gets you a shareable link
2. Add Clerk (step 2) — now you know who's using it
3. Add Stripe + Supabase (step 3-4) — now you can charge for it

Happy to help wire up any single step in more depth — e.g. the actual
Clerk mount code, or the Stripe webhook handler — once you're ready for it.
