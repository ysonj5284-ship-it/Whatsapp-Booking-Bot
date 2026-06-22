# WhatsApp salon/spa booking bot

Multi-tenant booking bot. One deployment serves many salons - each salon is
just an entry in `src/config/clients.json`.

## How it works

Customer messages a salon's WhatsApp number -> Meta sends the message to
our `/webhook` endpoint -> we identify which salon by `phone_number_id` ->
run the booking conversation -> check/book the salon's Google Calendar ->
confirm on WhatsApp. A scheduler sends appointment reminders and post-visit
review requests automatically.

---

## Part 1 - Get WhatsApp Cloud API access (do this once per salon... almost)

You actually only need to do the developer/app setup ONCE. Adding a new
salon later is just adding their own WhatsApp number under the same Meta app.

1. Go to https://business.facebook.com and create a Meta Business Account
   if you don't have one (use your agency's details).
2. Go to https://developers.facebook.com/apps -> Create App -> select
   "Business" type.
3. Inside the app, add the "WhatsApp" product.
4. Under WhatsApp > API Setup, you'll see a **test phone number** Meta gives
   you for free - good enough to build and test with your own phone before
   the salon's real number is live.
5. Copy the **Temporary access token** shown there into your `.env` file as
   `WHATSAPP_TOKEN`. (This expires in 24 hours - for production you generate
   a **permanent token** under System Users in Business Settings; the
   test token is just for development.)
6. Note the **Phone Number ID** shown on that same page - this is what goes
   into `clients.json` as the key for this salon.

### Connecting a real salon's number (when you're ready to go live)
The salon needs to give up their number for WhatsApp Business API exclusively
(it stops working as a normal personal WhatsApp). Under API Setup, click
"Add phone number" and follow Meta's verification flow (SMS/call code).
Each new number gets its own Phone Number ID - that becomes a new entry in
`clients.json`.

### Setting up the webhook in Meta
1. Deploy this code first (see Part 3) so you have a live URL.
2. In the Meta app, go to WhatsApp > Configuration > Webhook.
3. Callback URL: `https://your-deployed-url.com/webhook`
4. Verify token: anything you want, but it must match `WEBHOOK_VERIFY_TOKEN`
   in your `.env` exactly.
5. Subscribe to the `messages` field.

---

## Part 2 - Set up Google Calendar (per salon)

1. Go to https://console.cloud.google.com, create a project (or reuse one
   across all salons - one project is fine).
2. Enable the "Google Calendar API" under APIs & Services > Library.
3. Create a **Service Account** under APIs & Services > Credentials.
   Download its JSON key file - save it as `google-service-account.json`
   in this project's root folder.
4. The service account has its own email, looks like
   `something@your-project.iam.gserviceaccount.com`.
5. For EACH salon: create a Google Calendar for them (or use an existing
   one), then **share that calendar** with the service account email,
   giving it "Make changes to events" permission.
6. Copy that calendar's ID (Calendar Settings > Integrate calendar >
   Calendar ID, looks like `xxxxx@group.calendar.google.com`) into
   `clients.json` for that salon.

---

## Part 3 - Deploy

Recommended: **Render** (render.com) - free tier to start, ~$7/month
"Starter" tier once you have a real paying client (free tier sleeps after
15 min idle, causing slow first replies - fine for testing, not for
production).

1. Push this code to a GitHub repo.
2. On Render: New > Web Service > connect your repo.
3. Build command: `npm install`
4. Start command: `npm start`
5. Add environment variables (from `.env.example`) in Render's dashboard -
   do NOT commit your real `.env` file to GitHub.
6. Upload `google-service-account.json` as a "Secret File" in Render
   (under Environment), not as a regular repo file.
7. Once deployed, copy the live URL and use it as your webhook URL in
   Meta (see Part 1).

---

## Part 4 - Add a new salon client

Open `src/config/clients.json` and add a new entry, keyed by that salon's
WhatsApp Phone Number ID:

```json
"123456789012345": {
  "businessName": "New Salon Name",
  "city": "Jaipur",
  "timezone": "Asia/Kolkata",
  "googleCalendarId": "abc123@group.calendar.google.com",
  "businessHours": {
    "open": "10:00",
    "close": "20:00",
    "slotDurationMinutes": 30,
    "closedDays": ["Monday"]
  },
  "services": [
    { "id": "haircut", "name": "Haircut & styling", "price": 600, "durationMinutes": 45 }
  ],
  "messages": {
    "greeting": "Hi! Welcome to {businessName} 🌸\nHow can I help you today?",
    "reviewRequest": "Thanks for visiting {businessName} today! ⭐ Mind leaving us a quick Google review?",
    "googleReviewLink": "https://g.page/r/your-link/review"
  }
}
```

Redeploy (or if using Render with auto-deploy on git push, just push the
change) and the new salon is live - no code changes needed.

---

## Local testing without a real Meta account

You can test the conversation logic without any real WhatsApp/Google
credentials connected - the server will run and log clearly when it tries
to reach WhatsApp or Google Calendar, without crashing. This lets you
verify the booking flow logic before doing the real API setup.

```
npm install
cp .env.example .env
npm start
```

Then POST a fake webhook payload to `http://localhost:3000/webhook` to
simulate an incoming message (see Meta's webhook payload format in their
docs, or ask me for a test payload).

---

## File guide

- `src/server.js` - Express app, webhook routes
- `src/config/clients.json` - all salon configs (the multi-tenant heart)
- `src/config/clientStore.js` - loads/reads client configs
- `src/handlers/webhookParser.js` - turns Meta's payload into plain fields
- `src/handlers/conversationHandler.js` - the booking conversation logic/state machine
- `src/handlers/scheduler.js` - reminder + review request cron jobs
- `src/services/whatsappClient.js` - sends messages via WhatsApp Cloud API
- `src/services/calendarService.js` - Google Calendar slot checking + booking
- `src/services/db.js` - simple file-based storage for conversation state and bookings

## Known limitations to be aware of

- Conversation state and bookings are stored in a local `data.json` file
  (via lowdb). This is fine for a handful of salons, but if you scale to
  many clients or need multiple server instances, migrate to a real
  database (Postgres/MongoDB) - the `db.js` interface is small, so this
  swap is contained to one file.
- WhatsApp's free tier Cloud API gives 1,000 free "business-initiated"
  conversations per month, then charges per conversation - check current
  Meta pricing as you scale.
- The list UI used for services/time slots maxes out at 10 rows per Meta's
  limits - if a salon offers more than 10 slots in a day, only the first
  10 are shown (already handled in the code with `.slice(0, 10)`).
