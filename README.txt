WallSmart — Vercel deployment package

Pages:
- index.html       Home / landing page
- visualizer.html  AI Wall Paint Visualizer
- about.html       About page — the WallSmart story, principles, and a CTA
- login.html       Sign in / create account, with real email verification

Serverless API (used by login.html):
- api/signup.js        creates an account (bcrypt-hashed password) and emails a 6-digit code
- api/verify.js        checks the code, marks the account verified, issues a session token
- api/resend-code.js    re-sends the code (30s cooldown per account)
- api/login.js          checks email+password, blocks sign-in until the email is verified
- api/logout.js         invalidates a session token
- api/saves-list.js     returns the signed-in user's saved rooms (newest first)
- api/saves-create.js   saves a thumbnail + chosen colour to the signed-in user's account
- api/saves-delete.js   removes one saved room
- lib/helpers.js        shared Redis + email-sending + session-lookup helpers

Saved rooms: on visualizer.html, once you're signed in and have applied a
colour, a "Save to my account" button appears next to "Save preview". It
saves a small JPEG thumbnail (resized to ~480px wide) plus the chosen colour
to that account, and a "Your saved rooms" gallery underneath the tool shows
everything you've saved, with a delete (×) button on each card. Guests see
"Sign in to save it" instead of the save button — nothing is stored for
signed-out visitors. Each account keeps its 12 most recent saves; saving a
13th drops the oldest. This reuses the same Upstash database as accounts
(no extra setup) — see SETUP below.

This is no longer a pure static site — signup/login need a small key-value
store and an email-sending account. Both have working free tiers.

SETUP (one-time, ~10 minutes)
------------------------------
1) Create a Redis database at https://upstash.com (free tier is enough).
   After creating it, copy its REST URL and REST token.

2) Create an account at https://resend.com (free tier: 100 emails/day,
   3,000/month). Create an API key.

   Important: until you verify your own sending domain in Resend, you can
   only send email to the address you signed up to Resend with — any other
   recipient gets rejected. For a real multi-user site, add and verify a
   domain under Resend > Domains, then send from an address on that domain
   (e.g. "WallSmart <verify@yourdomain.com>").

3) In your Vercel project, go to Settings > Environment Variables and add:
     UPSTASH_REDIS_REST_URL     (from step 1)
     UPSTASH_REDIS_REST_TOKEN   (from step 1)
     RESEND_API_KEY             (from step 2)
     RESEND_FROM                optional — defaults to
                                 "WallSmart <onboarding@resend.dev>"
                                 (only deliverable to your own Resend
                                 account email until a domain is verified)

4) Deploy. Because this project now includes a package.json and /api
   functions, deploy it as a proper Vercel project (vercel CLI, or connect
   the folder as a Git repo and import it in Vercel) rather than a plain
   drag-and-drop of static files — Vercel needs to run `npm install` for
   the two dependencies (@upstash/redis, bcryptjs) and build the functions
   in /api. Vercel's "Deploy" drag-and-drop on vercel.com/new also works
   fine, since it still runs the standard build for a detected Node project.

Until the environment variables above are set, index.html, visualizer.html
and about.html will work exactly as before — only login.html's signup/login
(which call /api/*) need them.

Accounts/data: passwords are hashed with bcrypt and stored in Redis, not in
the browser. A session is a random token stored in Redis (30-day expiry);
the browser only keeps that opaque token plus the visitor's name/email in
localStorage for the nav UI. There's no password-reset flow yet — see the
"Forgot password?" note on the sign-in form.
