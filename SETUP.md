# Sole Serenity: website and booking system, setup and handover

This folder is Melanie's whole online setup: her **website** (in `public/`) and her **booking system** (in `src/`). Both run together for free on Cloudflare, at her own web address, and connect to Stripe, her Google Calendar and email.

**What it does**
- **Website.** It shows her real free times, which come from her Google Calendar plus existing bookings.
- **Deposits.** A time is held for 30 minutes while the client pays the $40 deposit, and their card is saved.
- **Packages, memberships and gift cards.** After entering their name and email, holders are recognised automatically and book with nothing to pay. Each session is logged against their purchase.
  - Packages and memberships are for the buyer only.
  - Gift cards can be for anyone. Packages and gift cards never expire.
  - Packages and memberships cover a 60-minute Reiki or Reflexology session only. Longer sessions, the Combination and the foot bath are regular bookings with the $40 deposit; the client is told this and offered a switch to 60 minutes.
  - Every booking checks again what's left. Packages and memberships are found by the email on file only, so clients must book with the same email they bought with (capitals and spaces don't matter; the name isn't used). Gift cards are found by the recipient's name, their email or the code. Package holders see how many sessions are left. If nothing is left (all package sessions used, or this month's membership session already booked), the booking is a regular one with the $40 deposit; the booking system enforces this too.
  - Each time someone opens booking, the payment choice starts fresh: a gift card code or package from an earlier attempt is never remembered.
  - Gift cards always pay for the whole session. The client sees their balance and what's left after the visit. If the card doesn't cover it all, the rest is paid by card right away in Stripe Checkout, so nothing is collected afterwards.
- **First-visit offer.** 2 hours of Reflexology + Reiki for $99. Anyone already on file (a past booking, or a membership, package or gift card under their email) is told as soon as they type their email that they are not eligible, and the booking system refuses it too.
- **Emails to clients** (all branded and automatic):
  - a **booking confirmation** with the details and address, as soon as they book
  - a **reminder** about 3 days before, while there's still time to cancel with a refund
  - a **receipt** after Charge balance
  - a **review request** a couple of hours after their first completed session
  - gift card codes, to the buyer and optionally straight to the recipient
- **Emails to Melanie:** an alert for each booking or sale, and a **Monday-morning summary** with last week's sessions and takings, anything not yet charged, and this week's bookings.
- **Admin page** at `/admin`, private to Melanie:
  - **Appointments**, with **Charge balance** (session − $40 deposit, charged to the saved card, one receipt), No-show and Cancel (Refund / No refund).
  - **Packages**, **Memberships**, **Gift cards**: who bought what, each session used, what's left. Memberships switch between Active and Inactive automatically from Stripe.
  - **Download**, for any date range, for bookkeeping: appointments and payments, memberships, packages, gift cards, sessions used from them, or everything in one file.
  - Sessions whose time has passed are greyed out. Past deposit sessions still waiting for **Charge balance** stay highlighted.

It takes about 2 hours to set up once. After that there is nothing to run or update.

---

## 1. Stripe (payments)
1. Create or open Melanie's Stripe account (Canada).
2. Go to **Settings → Customer emails** and turn **off** "Successful payments" and the invoice emails ("Finalized invoices", "Paid invoices").
   - The booking system sends its own branded emails instead, so clients get exactly one receipt, after Charge balance. It shows the session, the deposit, today's payment and the total, and links to Stripe's official PDF receipt.
3. Go to **Developers → API keys** and copy the **Secret key**. Start with the *test mode* key.
4. Create **Payment Links** for things people buy ahead:
   - **3-session package**: one-time price ($279).
   - **The Deep Ten (10-session pass)**: one-time price ($890). In `PURCHASE_LINKS` give it `"total":10`.
   - **Monthly membership**: the only recurring one. When creating its price choose **Recurring → Monthly** ($89), so Stripe charges the member automatically every month. Every other link (packages and gift cards) uses a **One-off** price.
   - When a member asks to cancel (one month's notice, by email or phone), open **Customers → their subscription → Cancel subscription → At the end of the current period**, a month ahead. Stripe stops charging and the booking system ends the membership by itself.
   - **Gift card $99**, **Gift card $198**, and **Gift card, custom amount** ("Customers choose what to pay").
   - On every link, turn on name collection (**Collect customers' names** / require the customer's name) and keep email on, so every purchase has a name and an email.
   - Packages and memberships are only for the person who buys them, so they get no "who is it for" field.
   - On each gift card link, add two **custom fields** (type Text):
     - key `recipient`, label "Who is the gift for?" (required, not optional)
     - key `recipient_email`, label "Their email (optional, we'll send them the code)", marked optional.
   - On each link, under **After payment**, choose **Don't show confirmation page → redirect to your website** and enter `https://<her-domain>/?bought={CHECKOUT_SESSION_ID}`. Buyers then land on the site's own thank-you page with their gift card code or package details.
   - Copy each link's id (starts `plink_`) into `PURCHASE_LINKS` in `wrangler.toml`.
   - Paste each link's URL into `STRIPE_LINKS` near the bottom of `public/index.html` so the Buy buttons work.
5. Go to **Developers → Webhooks → Add endpoint**:
   - URL: `https://<her-domain>/stripe/webhook`
   - Events: `checkout.session.completed`, `checkout.session.expired`, `customer.subscription.updated`, `customer.subscription.deleted`
   - Copy the **Signing secret** (starts `whsec_`).
6. **GST is off for now** (`GST_RATE = "0"`), because Melanie isn't registered. If she registers with the CRA later:
   - set `GST_RATE = "0.05"` and `GST_NUMBER` in `wrangler.toml`
   - change the matching `GST_RATE` line in `public/index.html`
   - redeploy

## 2. Google Calendar
1. Go to https://console.cloud.google.com and create a project called "Sole Serenity booking".
2. Enable the **Google Calendar API**.
3. Under **IAM & Admin → Service accounts**, create a service account. Then go to **Keys → Add key → JSON** and download the file.
4. In Google Calendar, open Melanie's calendar → **Settings and sharing → Share with specific people**. Add the service account's email with **"Make changes to events"**.
5. Put her calendar's email address in `CALENDAR_ID` in `wrangler.toml`.

Anything she adds to that calendar herself (holidays, personal time) automatically blocks those times on the website.

## 3. Email (Resend, free up to 3,000 emails a month)
1. Create a free account at https://resend.com.
2. Add and verify her domain.
3. Create an API key.
4. In `wrangler.toml`, set:
   - `FROM_EMAIL`, for example `Melanie at Sole Serenity <hello@herdomain.ca>`
   - `MELANIE_EMAIL`, where she wants alerts and the Monday summary
   - `STUDIO_ADDRESS`, the exact address, which is shared only in confirmation and reminder emails
   - `STUDIO_DIRECTIONS`, optional, for example "Side door, free street parking"
   - `REVIEW_URL`, her Google review link. In Google Business Profile, choose **Ask for reviews** and copy the link. Leave it empty to skip review emails.

## 4. Cloudflare: the website and the booking system together (free plan)
1. Create a free Cloudflare account and **add her domain** (Cloudflare walks through moving the domain's settings over; her email keeps working).
2. On a computer, in this folder:
```bash
npm install
npx wrangler login
npx wrangler d1 create sole-serenity        # paste the database_id it prints into wrangler.toml
npm run db:init                             # creates the tables
npx wrangler secret put STRIPE_SECRET_KEY
npx wrangler secret put STRIPE_WEBHOOK_SECRET
npx wrangler secret put GOOGLE_SA_EMAIL     # the service account's email
npx wrangler secret put GOOGLE_SA_KEY       # the "private_key" value from the JSON file
npx wrangler secret put RESEND_API_KEY
npx wrangler secret put ADMIN_PASSWORD      # backup sign-in, used only until step 5 is done
npm run deploy
```
3. In Cloudflare, go to **Workers & Pages → sole-serenity-booking → Settings → Domains & Routes → Add → Custom domain** and enter her domain (for example `soleserenity.ca`).
4. Set `SITE_ORIGIN` in `wrangler.toml` to that address and run `npm run deploy` again.

The website is now live at her domain with real booking. There's nothing to switch on: the page talks to the booking system at the same address. The reminder, review and Monday-summary emails run on their own every hour (`[triggers]` in `wrangler.toml`).

To change the website later, edit `public/index.html` and run `npm run deploy`.

## 5. Safer admin sign-in (Cloudflare Access, free)
Instead of a password, Melanie gets a one-time code by email each time she signs in.
1. In Cloudflare, open **Zero Trust** and pick a team name, for example `soleserenity`. The free plan is enough.
2. Go to **Access → Applications → Add an application → Self-hosted**:
   - Name: Sole Serenity admin.
   - Domain: her domain, path `admin`.
   - Add a policy: Action **Allow**, Include **Emails**, then her email address (add a backup one if you like).
   - Login method: **One-time PIN**.
3. Open the application's **Overview** and copy the **Application Audience (AUD) tag**.
4. In `wrangler.toml`, set `ACCESS_TEAM` (the team name) and `ACCESS_AUD` (the tag), then `npm run deploy`.

From then on, `/admin` only opens after Cloudflare emails her a code, and the booking system double-checks Cloudflare's sign-in on every request. The password is no longer used.

## 6. Test before going live
Use Stripe **test mode** keys first:
1. Book a session on the website and pay with card `4242 4242 4242 4242`, any future date and any CVC. Check the confirmation email.
2. Check that the booking appears in Google Calendar and on `/admin`.
3. Press **Charge balance** and check the receipt email.
4. Buy a test gift card with a recipient email, then book as the recipient. Also try a small gift card (add one for $20 on the Gift cards tab) to see the rest paid in checkout.
5. Try **Download** on the admin page.

When it all works, swap in the **live** Stripe keys and webhook secret (`npx wrangler secret put …`) and redeploy.

---

## Melanie's day to day
- Open `https://<her-domain>/admin`, enter the code Cloudflare emails her, and bookmark it on her phone.
- **After each session:**
  - Deposit bookings: press **Charge balance**.
  - Membership, package and gift card bookings: press **Mark done**.
- **No-show:** press **No-show**. The deposit is kept, or the session is used.
- **Cancellations** (the client texts or emails her): press **Cancel**, then **Refund** or **No refund**. It suggests the right one based on the 48-hour rule.
- **"Said they have a … but it didn't match"**: press **Find their …** and pick it, or contact them.
- **Selling something in person** (cash or e-transfer): use **+ Add a package / membership / gift card** on that tab.
  - For gift cards, leave Code blank and one is created. Tick "Email the code" to send it.
- **Bookkeeping:** press **Download**, choose the dates, and send the file to her bookkeeper.

## Upkeep
There is nothing to maintain day to day. It was built to keep running untouched:
- The Stripe API version is pinned, so Stripe changes don't break it.
- The Google key doesn't expire.
- Everything stays within the free plans of Cloudflare and Resend.

If a calendar update, charge or scheduled email ever fails, Melanie gets an email telling her what happened.

Things that would need someone technical:
- Changing prices or services: edit `SERVICES` in `src/index.js` and the matching list in `public/index.html`, then deploy.
- Melanie changing her Google, Stripe or email account: redo the matching step above.

## Costs
- Cloudflare (website, booking system, sign-in), Resend, Google: $0 at her volume. Her domain name renews yearly as before.
- Stripe: 2.9% + $0.30 per card payment (Canada), plus 0.4% on Charge balance invoices (Stripe Invoicing).
- No monthly fee.
