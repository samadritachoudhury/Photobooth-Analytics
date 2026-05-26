# Sammy's Little Booth ♡

> *take four · keep forever*

a browser-based photo booth with the whole point being, to instrument a real consumer product and study how people actually use it.

---

## what it does

you open the booth, take 4 photos, pick a filter, add a caption, download your strip.
that's the product. clean, fast, runs entirely in the browser.

underneath that there's a full session analytics pipeline quietly doing its job of
tracking every meaningful moment from page load to download, writing it all to
Supabase in real time.

---

## features

**the booth itself**
- 4-shot photo strip — takes 4 captures and lays them into a downloadable strip
- 9 filters — douyin dreamy, 2000s sony, warm skin, cherry blossom, milky white, neon night, retro film, yoda, original
- 3 strip styles — classic, film, polaroid
- custom caption printed on the strip
- self-timer — none / 3s / 5s
- download at 2× resolution or copy to clipboard
- works on mobile and desktop

**the analytics layer**
- every session gets a unique UUID on page load
- full conversion funnel tracked: page visit → booth opened → photos taken → strip completed → downloaded → shared
- filter preference tracked at capture time (not click time — so it reflects what you actually shot with, not what you browsed)
- A/B test running on download button copy: "save your strip" vs "keep memory" — testing whether emotional wording improves conversion
- AI interest survey shown post-completion: yes / maybe / not now — stored as behavioral signal
- device type detection: mobile / tablet / desktop
- session duration, retake count, share count, download count — all saved per session
- `beforeunload` fallback so partial sessions still get recorded if you close the tab
- serialised save queue — no concurrent upsert race conditions

**the dashboard (on-page)**
- live visit count, completion status, download count, share count
- top filter by actual usage
- average time per photo
- exportable portfolio summary

---

## tech stack

| layer | what |
|---|---|
| frontend | vanilla HTML, CSS, JavaScript — no framework |
| camera | `getUserMedia` API |
| image processing | Canvas API — filters, tints, film grain, vignette all applied on canvas |
| analytics storage | Supabase (PostgreSQL) via JS SDK |
| local analytics | `localStorage` — persists filter usage and visit history in-browser |
| fonts | Playfair Display + IM Fell English via Google Fonts |

---

## supabase table — `user_sessions`

every completed or partial session writes one row. upserts on `session_id` so
multiple saves within a session update the same row rather than creating duplicates.

```
session_id       uuid / text   unique — generated with crypto.randomUUID()
device_type      text          mobile / tablet / desktop
filter_selected  text          active filter at time of save
completion_time  int           seconds from booth open to strip complete
retakes          int           how many times "start over" was clicked
downloads        int           download button clicks this session
shares           int           share button clicks this session
photos_taken     int           number of shots taken (0–4)
completed        bool          true if all 4 photos were taken
booth_opened     bool          true if camera permission was granted
ai_interest      text          yes / maybe / not now / NULL if ignored
ab_variant       text          A or B — assigned once per session on load
timestamp        timestamptz   ISO timestamp of last save
```

`NULL` in `ai_interest` means the popup was never interacted with — not that the
user declined. active decline is stored as `"Not now"`. that distinction matters
for behavioral analysis.

---

## conversion funnel

```
page visit
    ↓
booth opened  (booth_opened = true)
    ↓
photos taken  (photos_taken = 1 → 4)
    ↓
strip complete  (completed = true)
    ↓
downloaded  (downloads ≥ 1)
    ↓
shared  (shares ≥ 1)
```

drop-off between any two stages is where the interesting stuff lives.

---

## A/B test

**experiment:** does emotionally-worded copy on the download button increase conversion?

| variant | button text |
|---|---|
| A | ♡ save your strip ♡ |
| B | ♡ keep memory ♡ |

variant is randomly assigned on page load, persisted for the session, and stored
in Supabase alongside every other session metric. to analyse: filter by `ab_variant`
and compare `downloads / completed` conversion rates between the two groups.

---

## running it locally

it's a single HTML file. no build step or dependencies to install.

```bash
git clone https://github.com/yourusername/sammys-little-booth
cd sammys-little-booth
# open index.html in your browser
# or use live server in VS Code for a cleaner dev experience
```

camera access requires HTTPS or localhost — it won't work opened as a plain
`file://` URL in most browsers. VS Code Live Server handles this automatically.

---

## privacy

- no personal data is collected — no names, no emails, no photos stored anywhere
- photos exist only in the browser's memory during the session
- the downloaded image goes directly to your device
- analytics data is behavioural only: device type, filter choice, timing, button clicks
- all analytics are client-side first, then written to Supabase
- no cookies, no tracking pixels, no third-party analytics scripts
- architecture mirrors privacy-first tools like Plausible — data you own, nothing you don't

---

## project context

this was built for my Business Analytics + FinTech portfolio to demonstrate
product instrumentation on a real consumer product rather than a toy dataset.

the photobooth is the product. the analytics layer is the actual work.

the conversion funnel, A/B test, behavioral segmentation, and session architecture
are the same patterns used by growth teams at consumer tech companies —
just running in a single HTML file instead of a microservices stack.

---

*made with way too much attention to detail ♡*