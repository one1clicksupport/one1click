# 1click website: setup guide

Plain HTML, CSS and JavaScript. No build step. Hosted on Netlify from GitHub.
The chatbot runs as a Netlify Function on the same site, so you do not need Cloudflare.

## Folder map
- `public/` the website (this is what Netlify publishes)
  - `site.config.js` every editable value (links, emails, hero text, speeds, chatbot switch)
  - `reviews.json`, `portfolio.json` the two scrolling rows
  - `index.html`, `styles.css`, `main.js`, `chatbot.js`, `chat.css`
- `netlify/functions/chat.mjs` the chatbot API (served at `/api/chat`)
- `worker/worker.js` the chatbot brain (Gemini, key rotation, calendar tools). Used by the Netlify Function.
- `apps-script/Code.gs` Google calendar, Google Sheet and Gmail for bookings and leads
- `scripts/` tests and the optional portfolio photo tools
- `netlify.toml` tells Netlify to publish `public` and find the function

## 1. Put it on GitHub (5 minutes)
Easiest, no commands:
1. Create a free account at github.com. Click New repository. Name it `1click-site`. Choose Private. Create.
2. Click "uploading an existing file". Unzip the project, drag everything inside the folder (not the folder itself) into the page. Commit changes.

With commands (if you have Git installed), run these inside the project folder:
```
git init
git add .
git commit -m "1click site"
git branch -M main
git remote add origin https://github.com/YOURNAME/1click-site.git
git push -u origin main
```

## 2. Deploy on Netlify (5 minutes)
1. At app.netlify.com choose Add new site, Import an existing project, GitHub. Pick `1click-site`.
2. Build command: leave empty. Publish directory: `public` (it is already set by `netlify.toml`). Deploy.
3. Open the Netlify address. Check the page. Every time you push to GitHub, Netlify redeploys by itself.
4. Optional: Site configuration, Change site name, to get `1click.netlify.app` style addresses.

## 3. Make the chatbot live
The chatbot works in mock mode (scripted replies) until you do this. Do the Google part first.

### A. Apps Script (calendar, Sheet, email)
1. Create a Google Sheet named `1click leads`.
2. In the Sheet: Extensions, Apps Script. Delete the sample code. Paste all of `apps-script/Code.gs`.
3. Project Settings (gear), Script properties, Add property. Name `SHARED_TOKEN`, value a long random string. Save it somewhere, you need it in step C.
4. Pick the function `authorize` and press Run. Accept the permissions. If Google warns "unverified app", choose Advanced, then continue. It is your own script.
5. Deploy, New deployment, type Web app. Execute as: Me. Who has access: Anyone. Deploy.
6. Copy the Web app URL (ends with `/exec`).
7. If you change the code later: Deploy, Manage deployments, pencil icon, Version: New version.

### B. Two Gemini keys
1. Go to aistudio.google.com. Create a key in one Google project.
2. Create a second key in a different Google project (so each key has its own quota). Keep both.

### C. Add settings in Netlify
Site configuration, Environment variables, Add a variable. Add these:
| Name | Value |
|---|---|
| `GEMINI_KEY_1` | first key |
| `GEMINI_KEY_2` | second key |
| `APPS_SCRIPT_URL` | the `/exec` URL from step A |
| `APPS_SCRIPT_TOKEN` | the same value as `SHARED_TOKEN` |
| `GEMINI_MODEL` | `gemini-3.1-flash-lite` (check the current Flash Lite name in AI Studio if replies stop) |
| `BOOKING_LINK` | `https://calendar.app.google/sT8JPnZzvAiFJUCD8` |

Mark the keys and token as secret. Then Deploys, Trigger deploy, so the function picks them up.

### D. Switch the site to live mode
In `public/site.config.js`:
- `chatbotMockMode: false`
- `appsScriptUrl: "the /exec URL"` (this also gives you a backup copy of every form lead in the Sheet)
Commit and push. Netlify redeploys in about a minute.

### E. Test (3 minutes)
Open the site, chat "book a call", finish the flow. Check: the Sheet got a row, the calendar has the event, and both inboxes got the email. Also send one test through the booking form.

## 4. Other settings you should check
- Google Calendar: your appointment schedule should be 30 minutes, every day, 12:00 to 23:30 Dhaka time, visible to anyone with the link. The same hours are in `site.config.js` and at the top of `Code.gs`. Keep the three in sync.
- Formspree free plan: 50 submissions per month. After that, submissions are held without an email. The Sheet backup (`alsoLogToSheet`) catches them. Upgrade Formspree if you get close.
- Gemini free tier: small daily limits, and Google may use free tier content to improve its products. If both keys run out, the bot says so lightly and shows the Pick a time button.
- Booking in a pop up window: paste the embed address from Google Calendar (Appointment schedule, Share, Embed) into `booking.embedUrl`. Without it, booking opens in a new tab.

## 5. Photos on the portfolio cards (optional, 5 minutes)
Cards show large white icons on purple right now (zero extra weight). To add real photos:
1. Get a free key at pexels.com/api.
2. Install Node.js, open a terminal in the project folder, run `npm i`.
3. Run `PEXELS_API_KEY=your_key npm run photos` (Windows PowerShell: `$env:PEXELS_API_KEY="your_key"; npm run photos`).
4. It saves one small WebP per industry (about 15 to 25 KB each) in `public/assets/industry` and fills `portfolio.json`. Commit and push.
Prefer real homepage screenshots instead? `npx playwright install chromium`, then `npm run shots`.
A photo that fails to load falls back to the icon, so nothing breaks.

## 6. Your own domain later
- Easiest: keep hosting on Netlify. Netlify, Domain management, Add a domain. If the domain is at Hostinger, change its nameservers to Netlify's, or add the DNS records Netlify shows. Nothing else changes.
- If you want the site files on Hostinger hosting: upload the contents of `public/` to `public_html`. The chatbot function stays on Netlify, so set `chatApiUrl` in `site.config.js` to `https://YOURSITE.netlify.app/api/chat` and add the variable `ALLOWED_ORIGIN` = `https://yourdomain.com` in Netlify.
- Cloudflare is not needed. If you ever want it, `worker/worker.js` runs as a Cloudflare Worker unchanged (`worker/wrangler.toml` is ready).

## 7. Keeping it fast
- First load JavaScript is about 9 KB gzipped. The chatbot code loads only after the page is idle.
- Images load lazily. Reviews and portfolio data load only when you scroll near them.
- Animations use transform only and pause when off screen.
- Run `npm run check` after edits. It checks for dashes in copy, config keys, the chatbot brain and key rotation.

## Troubleshooting
- Chatbot says "Oops, my wires crossed": open Netlify, Logs, Functions, `chat`. Usually a missing variable or you did not redeploy after adding it.
- Old logo or layout showing: hard refresh (Ctrl+Shift+R), or open in a private window.
- Booking times look off: check the hours match in the three places listed above.
