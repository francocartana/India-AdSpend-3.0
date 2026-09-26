# India AdSpend (v3)

A Chrome extension (Manifest V3) that reads a screenshot of an Indian government advertisement
— via drag-and-drop or a right-click context menu — and returns a neutral, factual summary plus
a CCRGA compliance score out of 100, powered by Gemini's vision model.

## Setup (2 minutes)

1. Go to `chrome://extensions`, turn on **Developer mode**.
2. Click **Load unpacked**, select this folder.
3. Click the extension icon. On first open it'll ask for a Gemini API key — click "Get a free
   key from Google AI Studio," copy it, paste it in, and hit **Save and continue**. No file
   editing needed; the key lives in the browser's local extension storage, per-user, and is
   never bundled or committed.
4. Click **"⚙️ Change API key"** any time (bottom of the popup) to swap in a different key —
   useful if your daily free-tier quota runs out, since quota is per-key.

## How to use it

- **Drag and drop** a screenshot of a government ad onto the square, or **click the square** to
  pick a file.
- **Click-to-analyze mode:** click the extension icon, then click **"Activate: click any ad
  image on this page."** The popup closes, the page gets a crosshair cursor and an orange
  outline on whatever image you hover, and your next click on any image sends it straight to
  the checker (press Esc to cancel). This is the closest approximation of "click an ad and check
  it" that's reachable without a full ad-network integration.
- **Right-click any ad image** on a webpage → "Check this ad with India AdSpend" also still
  works, as a one-step alternative to activating click mode first.
- Either flow re-opens the popup automatically where Chrome's gesture policy allows it; if it
  doesn't, a small orange "1" badge appears on the toolbar icon — click it to see the result.

## What it does under the hood

One Gemini vision call does most of the work:
1. Decides whether the image is actually an Indian government ad at all.
2. Reads the ad's text (OCR) and semantically checks it against the **live directory** (see
   below), using it as ground truth when it matches.
3. Writes a ≤200-word strictly factual summary (no promotional adjectives).
4. Returns **five separate 0–20 sub-scores** against the CCRGA rubric — relevance, neutrality,
   accuracy, attribution, cost-consciousness — which the extension itself sums into the final
   /100. This is the "standardized scoring": the total is deterministic arithmetic done in code,
   not a single holistic number the model invents on its own, so every score is auditable and
   reproducible criterion-by-criterion.

If the image isn't recognized as a government ad, it shows exactly this message:
> Sorry, this input does not match with our records for Indian government advertisements.
> Think there's a mistake? Email feedback to utkarsh1486@gmail.com

### Self-growing directory

`campaigns.json` only seeds the directory on first run — after that, the live directory lives in
`chrome.storage.local`, not the static file, so it can actually grow:
- **New campaign, no match found** → Gemini proposes a keyword slug and a new entry is created
  from what it inferred off the ad.
- **Same campaign, different creative** → the analysis is appended to that entry's
  `creativesSeen` array (extracted text, summary, score, timestamp) instead of duplicating it, so
  the directory accumulates context on a campaign every time a new execution of it is checked.
- A **"Reset directory to shipped defaults"** link at the bottom of the popup wipes everything
  learned and reseeds from the bundled `campaigns.json`, for a clean state before your demo.

### Copy report for Gem

Every completed check has a **"📋 Copy report for Gem"** button that copies a plain-text version
of the full report (total + breakdown + facts + flags + extracted ad text) to the clipboard,
ready to paste into the companion Gem to keep discussing it — drafting a complaint, asking for
more context on a specific flag, etc.

### Quota-exhaustion detection

If Gemini returns a 429 that survives retries, or three separate analyses in a row fail for any
reason, the popup assumes the free-tier daily quota on that key is used up and shows a banner
with a one-click "switch to a different API key" link straight to the key-entry screen — rather
than a raw error the person has to interpret themselves.

## ⚠️ Important disclosure — read this before presenting

**The two reference documents named in the brief were never uploaded to me**
(`Common_Cause_vs_Union_Of_India_on_13_May_2015.PDF` and `National_Advertising_Policy.pdf`).
I built the CCRGA rubric from real, independently verified Supreme Court guidelines (see below),
but I have not read either PDF directly. Any flag that would depend on the National Advertising
Policy is deliberately labeled `[Not assessed — reference document not supplied]` rather than
guessed at — **do not present this as if it scores against a document it hasn't seen.**
If you want it to reference the actual PDF text, upload both files and I'll rebuild the rubric
from them directly, and this disclosure will no longer apply.

The CCRGA guidelines actually used (verified independently): ads should relate to government's
constitutional/legal obligations or citizens' rights (not party identity); avoid glorifying
individual political figures or the ruling party; not present old schemes as new; be clearly
attributed to a government body; and be cost-proportionate.

## Other known limitations (say these out loud too)

- The self-growing directory lives in `chrome.storage.local` — it's **per-browser-profile, not
  shared** across users or devices. In production this would be a shared backend database; say
  so if asked how this would scale to a real newsroom or public tool.
- Real Google Ad creatives often render inside sandboxed cross-origin iframes the extension
  can't reach — click-to-analyze and right-click capture work reliably on standard `<img>`-based
  display ads (and CSS `background-image` ad units), not all ad formats. Screenshot drop is the
  reliable fallback for anything else.
- After clicking "Activate," you may need to click once on the actual page first (to give it
  focus) before the crosshair cursor visually shows — the click listener itself is live
  immediately regardless.
- Gemini's score is a reasoned estimate against a fixed rubric, not a legal determination —
  standardized scoring means consistent *arithmetic*, not a substitute for legal review.

## Files

- `manifest.json` — MV3 config (background service worker, content script, context menu, broad
  host permission for fetching ad images cross-origin, branded icon set)
- `background.js` — right-click context menu + click-to-analyze message handling, image capture
- `content.js` — injected into every page; adds the click-to-analyze mode and hover highlight
- `popup.html` / `popup.js` — India AdSpend branded UI, first-run key onboarding, drop-zone,
  activation button, and the Gemini vision call
- `campaigns.json` — seed data for the directory (copied into `chrome.storage.local` on first run)
- `logo.png` — wordmark shown in the popup header (transparent background)
- `icons/icon16.png`, `icon32.png`, `icon48.png`, `icon128.png` — toolbar/store icon, generated
  from the brand package's shield icon

**No API key ships in this repo.** Each person who installs the extension pastes in their own
free Google AI Studio key on first launch — nothing to strip out before committing, nothing to
rotate if this repo goes public.

## Branding

Built from the India AdSpend brand package: navy (`#123C69`, Indian Trust), teal (`#00ACC1`,
Transparency), orange (`#F89A1D`, National Pride), green (`#0A8F53`, Growth/Fairness), and off-
white (`#F7F7F7`, Clarity) — used consistently for surfaces, primary/secondary/ghost buttons, and
text, with the wordmark and shield icon pulled directly from the brand package.

One deliberate exception: the **low-compliance score badge uses a soft red**, not one of the five
brand colors. A traffic-light red for "serious violation" is a near-universal usability
convention: it means something specific to the person reading it, and diluting it into brand
orange would make a bad score look like a promotional accent instead of a warning. Everything
else — buttons, borders, backgrounds, the mid/high score tones — stays inside the five-color
system. Worth a sentence in your presentation if a judge asks about brand consistency.
