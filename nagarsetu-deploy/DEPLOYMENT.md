# NagarSetu: Deployment Guide

Pick one path. If you only need the judges' demo, use **Path A**. If you want tickets, accounts and the AI to work for real, use **Path B** or **Path C**.

| Path | Best for | Cost | Data kept? | AI assignment |
|---|---|---|---|---|
| **A. Netlify (static demo)** | Hackathon judging, quick public link | Free | No, resets on reload | Yes, through a Netlify Function |
| **B. Render (full server)** | A live, persistent demo or pilot | Paid disk needed | Yes (SQLite on a disk) | Yes, built in |
| **C. Docker on your own machine or VPS** | Full control, local testing | Your own hosting | Yes (Docker volume) | Yes, built in |

Other static hosts (Vercel, GitHub Pages) also work for the demo. See "Other static hosts" at the end.

---

## Before you start

1. **Unzip** `nagarsetu-final.zip`. The folder is called `nagarsetu-deploy`.
2. **Never put secrets in files the browser downloads.** That means `config.js`, `index.html`, and anything in `css/` or `js/`. Keys go in environment variables only.
3. **Optional keys you may want:**
   - **Gemini API key** (AI staff assignment): create it in Google AI Studio.
   - **Google Maps JavaScript API key** (live map): create it in Google Cloud Console and restrict it to your website address and the Maps JavaScript API only. Leave it empty and the site uses its built-in map.
4. **Test locally first** (30 seconds, no install): run `python3 -m http.server 8080` inside the folder and open http://localhost:8080.

---

## Path A: Netlify (static demo)

The site runs entirely in the browser. Data resets on reload. Demo accounts only work in this mode.

1. Put the `nagarsetu-deploy` folder in a **GitHub repository** (recommended, because Netlify Functions deploy reliably this way).
2. In Netlify choose **Add new site > Import an existing project**, pick the repository, leave the build command empty and set the publish directory to `.` (already set in `netlify.toml`).
3. **Optional AI assignment:** go to **Site settings > Environment variables** and add:
   - `GEMINI_API_KEY` = your key
   - `GEMINI_MODEL` = `gemini-2.5-flash` (check this name in Google AI Studio; models get retired)
4. **Redeploy** the site after adding variables (Deploys > Trigger deploy).
5. **Optional live map:** put your Maps key in `config.js` as `GOOGLE_MAPS_API_KEY`. Browser Maps keys are public by design, so restrict the key to your Netlify address.

**Quick drag-and-drop alternative:** you can drag the folder into Netlify Drop for a fast static preview. Netlify Functions may not be included that way, so the AI step will quietly be skipped and the rule-based assignment is used. To be sure the function deploys, use GitHub or the Netlify CLI (`netlify deploy --prod`).

**Check it worked:** open the site, press **Try the demo**, report an issue as a resident, then sign in as admin (`admin` / `admin123`). With the key set, a small **AI** badge and reason appear next to the assigned staff name in the department console.

---

## Path B: Render (full server)

FastAPI serves the website and the API from one address. Tickets and accounts persist on a disk.

1. Put the folder in a **GitHub repository**.
2. In Render choose **New > Blueprint** and select the repository. It reads `render.yaml`. A persistent disk needs a paid instance type; that is why the blueprint uses the `starter` plan.
3. Fill in the environment variables Render asks for:

| Variable | Required | What it is |
|---|---|---|
| `SECRET_KEY` | Yes | Signs login tokens. Render generates it for you |
| `ADMIN_PASSWORD` | Yes | Password for the `admin` account |
| `STAFF_PASSWORD` | Yes | Password for the six staff accounts |
| `GEMINI_API_KEY` | Optional | Turns on AI staff assignment |
| `GEMINI_MODEL` | Optional | Defaults to `gemini-2.5-flash` |
| `GOOGLE_MAPS_API_KEY` | Optional | Turns on the live Google map |
| `NOTIFY_WEBHOOK_URL`, `NOTIFY_WEBHOOK_TOKEN` | Optional | Sends escalation alerts to Slack, n8n or an SMS/WhatsApp gateway |
| `CORS_ORIGINS` | Only if the front end is on a different address | Comma-separated list of allowed origins |

4. Deploy. Render builds the Docker image and starts the app on port 8000.

**Check it worked:** open `https://YOUR-APP.onrender.com/api/health`. You want `"ok": true` and `"audit_chain_ok": true`. If the Gemini key is set you also see `"ai_assign": true`.

---

## Path C: Docker (your own machine or a VPS)

1. Copy the settings template and fill it in:

       cp backend/.env.example backend/.env

   Set `SECRET_KEY` (a long random text), `ADMIN_PASSWORD`, `STAFF_PASSWORD`, and optionally `GEMINI_API_KEY` and `GOOGLE_MAPS_API_KEY`.
2. Start it:

       docker compose up -d --build

3. Open http://localhost:8000 and check http://localhost:8000/api/health.
4. Data lives in the Docker volume `nagarsetu-data`. Back it up before upgrades.
5. **Going public:** put HTTPS in front of port 8000 with a reverse proxy such as Caddy or nginx. Voice input and live location only work on HTTPS (or localhost).

**Run without Docker (server mode):**

    cd backend
    python3 -m venv .venv && source .venv/bin/activate
    pip install -r requirements.txt
    cp .env.example .env     # edit it
    set -a; source .env; set +a
    uvicorn app.main:app --port 8000 --workers 1

Use **one worker only**. The database is SQLite and the deadline worker runs inside the app.

---

## Other static hosts

- **Vercel:** import the repository. `vercel.json` is already set. The AI Netlify Function does not run on Vercel, so the demo uses the rule-based assignment.
- **GitHub Pages:** `.github/workflows/pages.yml` is included. Same limit on the AI function.
- **nginx container:** `Dockerfile.static` serves the static demo.

To use a static front end with a separate backend, set in `config.js`: `USE_API: true` and `API_BASE_URL: "https://your-backend"`, and set `CORS_ORIGINS` on the backend to the front end's address.

---

## Verify after deploying

- [ ] The landing page loads and the button reads **Try the demo**
- [ ] The animated cards and background move
- [ ] A resident can report an issue, speak it with the mic button (Chrome or Edge, HTTPS), and add up to 3 photos
- [ ] Staff see the photos on the ticket; the admin console shows who was assigned
- [ ] Language switch works for English, Hindi and Punjabi
- [ ] Server mode only: `/api/health` shows `ok` and `audit_chain_ok` as true

## Troubleshooting

| Problem | Likely cause and fix |
|---|---|
| No "AI" badge appears | No `GEMINI_API_KEY`, wrong model name, the Netlify Function was not deployed, or the team has only one staff member (nothing to choose). The rule-based assignment still works |
| Mic button is greyed out or says blocked | Use Chrome or Edge on HTTPS and allow the microphone in the browser address bar |
| Map shows boxes instead of Google Maps | `GOOGLE_MAPS_API_KEY` is empty or restricted to a different address. The built-in map is a normal fallback |
| Fonts look plain | Google Fonts could not load (no internet or blocked) |
| Tickets disappear on reload | You are in demo mode. Use Path B or C for persistence |
| Login fails in server mode | Check `ADMIN_PASSWORD` and `STAFF_PASSWORD`. If unset, the server prints the generated passwords once in its first startup log |
| Browser errors about CORS | Front end and backend are on different addresses. Set `CORS_ORIGINS` on the backend |
| Data lost after a Render redeploy | The disk is not attached. Check the `/data` mount in `render.yaml` |

## Security checklist

- Do not commit `backend/.env` or any key.
- If a key is ever pasted into a chat, a file or a public repository, **delete it and create a new one**.
- Change the demo passwords before any real use. The demo accounts (`admin123`, `staff123`) are for demo mode only.
- When AI assignment is on, ticket text is sent to Google. Say so in your privacy notice.
- This is a hackathon prototype, not an official government service and not an emergency service.
