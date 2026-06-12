# Portfolio Website: Build & Design Architecture Document

Owner: Aloka Seneviratne
Purpose of this document: complete specification of the portfolio website so that any developer or AI assistant can understand, rebuild, modify, or extend it without prior context.

---

## 1. Project overview

A personal portfolio website for an embedded systems / electronics engineer that automatically showcases all public GitHub repositories as project cards. No database, no backend, no build step. One static HTML file hosted free on GitHub Pages.

### Core requirements
1. Auto-fetch all public repos from GitHub and display them as project cards (live, on every page load).
2. Professional, distinctive visual design (not a generic template look).
3. Completely free: hosting, domain (subdomain), and data source.
4. Zero maintenance: new repos appear automatically without redeploying.
5. Owner can tweak content easily (all personal settings in one CONFIG block).

### Explicit constraints / owner preferences
- Plain HTML/CSS/JS, no frameworks, no build tooling (chosen so the owner can host on GitHub Pages with a simple file upload and tweak the code directly).
- No em dashes anywhere in copy.
- Hosted at `https://<username>.github.io` (free subdomain, no custom domain).

---

## 2. Architecture

### 2.1 High-level
```
Visitor browser
   |
   |-- loads index.html (static, served by GitHub Pages)
   |
   |-- JS fetch() at runtime --> GitHub REST API (public, no auth)
   |        GET https://api.github.com/users/{username}/repos?per_page=100&sort=updated
   |
   '-- renders project cards client-side from JSON response
```

- **Hosting:** GitHub Pages, special repo named `<username>.github.io`, serves `index.html` at the root URL.
- **Data source:** GitHub REST API v3, unauthenticated. Rate limit is 60 requests/hour per visitor IP, which is ample for a portfolio (1 request per page load).
- **State:** none persisted. Everything is fetched fresh per page load. No cookies, no localStorage (intentionally avoided).
- **File count:** exactly one file, `index.html`, containing all HTML, CSS, and JS inline. Fonts load from Google Fonts CDN.

### 2.2 Page structure (top to bottom)
1. **Sticky header** with brand wordmark ("ALOKA.SENEVIRATNE", the dot in copper) and mono uppercase nav links: Projects, About, Contact (anchor links).
2. **Hero**: mono eyebrow line ("REV 2026.06 / OULU, FINLAND"), large display name (h1), tagline paragraph (embedded systems, firmware, sensors, signal processing, M.Sc. at University of Oulu), two buttons: "View projects" (solid, anchor) and "GitHub profile" (outline, external). Behind it: a decorative SVG of PCB traces with 45-degree bends and via dots, low opacity, pointer-events none, aria-hidden.
3. **Projects section** (id="projects"): section header with mono reference label "[ U1 ]" + h2. Contains a status element (loading / error / empty messages) and a responsive card grid (auto-fill, minmax 300px).
4. **About section** (id="about", "[ U2 ]"): two columns. Left: three short bio paragraphs (hardware-focused background, thesis on synthetic IMU time-series for Freezing of Gait detection, BLE mesh sensor work, MATLAB breath detection, prior consulting and math lecturing). Right: the signature element, a skills table styled as a PCB **Bill of Materials** with columns Ref / Component / Notes (rows: FW1 C/C++, MC1 ESP32/Arduino, DSP1 MATLAB/Python, NET1 IoT protocols, HDL1 SystemVerilog/VHDL, DOC1 Technical writing).
5. **Contact section** (id="contact", "[ J1 ]"): short availability line (open to embedded/electronics roles across Europe) and a row of button-styled links generated from CONFIG (Email, GitHub, LinkedIn).
6. **Footer**: copyright line and "Projects pulled live from the GitHub API".

### 2.3 JavaScript design
All script lives in one inline `<script>` at the end of `<body>`. No modules, no dependencies.

**CONFIG object** (the only thing the owner edits):
```js
const CONFIG = {
  githubUsername: "YOUR_GITHUB_USERNAME",  // required
  links: [ { label, url }, ... ],          // contact buttons
  excludeRepos: [],                        // repo names to hide
  hideForks: true,
  sortBy: "updated",                       // or "stars"
};
```

**Functions:**
- `LANG_COLORS`: lookup map of ~20 common GitHub language colors (C #555555, C++ #f34b7d, Python #3572A5, MATLAB #e16737, SystemVerilog #DAE1C2, VHDL #adb2cb, etc.), fallback muted green for unknown languages.
- `timeAgo(iso)`: relative time string (today / Nd ago / Nmo ago / Ny ago) from `pushed_at`.
- `el(tag, cls, text)`: tiny DOM helper. All rendering uses createElement/textContent, never innerHTML with API data (XSS safety; the only innerHTML use is a static instructional message).
- `renderCard(repo, index)`: builds one card: absolute-positioned mono designator "U{n}" top-right (revealed in copper on hover), repo name as external link, description (fallback "No description yet."), up to 5 topics as tags, meta row with language dot + name, star count if > 0, spacer, relative updated time.
- `loadRepos()`: async. Guard clause: if username is empty or still the placeholder, show a friendly setup-instruction error pointing to the CONFIG block. Fetches the API, handles 404 (user not found), 403 (rate limit), other non-OK statuses with specific messages. Filters out archived repos, forks (if hideForks), and excludeRepos. Sorts by stars or pushed_at. Empty result shows an encouraging empty state. On success hides status, fills and reveals the grid.
- `applyConfig()`: points the hero GitHub button at the user profile and renders contact link buttons (mailto links don't get target=_blank; external ones get rel="noopener").

### 2.4 Motion layer (added in rev 2)
All motion is vanilla JS + CSS, no libraries. Every effect is disabled by prefers-reduced-motion, with CSS fallbacks ensuring content is never stuck invisible.
- **Hero entrance:** eyebrow, h1, tagline and buttons rise in with a staggered `rise` keyframe animation (0.05s to 0.35s delays, `both` fill mode so reduced motion shows them at rest state).
- **PCB trace draw-in:** `animateTraces()` measures each hero SVG path with getTotalLength, sets stroke-dasharray/offset inline, forces a reflow, then transitions offset to 0 with staggered delays (0.18s apart, 1.3s duration). Via dots fade in after the traces (from 1.3s). Inline styles only, so without JS or with reduced motion the static SVG shows fully drawn.
- **Scroll reveals:** `.reveal` class (opacity 0, translateY 18px) transitions to `.in` when an IntersectionObserver (threshold 0.12, rootMargin -40px bottom) sees the element. `reveal(node, delay)` sets a per-element `--d` transition-delay. Applied to section heads, about columns, contact elements, and each project card with a `(i % 6) * 0.07s` stagger. If the observer is unavailable or reduced motion is set, elements are never given the reveal class and stay visible. A reduced-motion media query also forces `.reveal` to opacity 1 as a second safety net.
- **Micro-interactions:** nav links get a copper underline sweeping in from the left (`::after` scaleX); buttons lift 1px on hover; cards gain a soft drop shadow on hover alongside the existing lift and copper border.
- **Head additions:** theme-color meta (#0d3528), Open Graph title/description/type, inline SVG data-URI favicon (soldermask square, copper trace and via).
- **Mobile header fix:** below 720px the brand drops to 11px with tighter letter-spacing and nav gaps shrink, preventing brand/nav collision at 390px width.

### 2.5 Error and empty states (required behaviors)
- Placeholder username -> instructional message, not a raw error.
- 404 -> "GitHub user X was not found. Check the spelling in CONFIG."
- 403 -> "GitHub API rate limit reached. Try again in a few minutes."
- No repos after filtering -> "No public repositories to show yet. Push a project to GitHub and refresh."

---

## 3. Design system

### 3.1 Concept
The visual identity is grounded in the owner's world: printed circuit boards. Deep soldermask green background, copper trace accent, off-white "silkscreen" typography, mono reference designators (U1, U2, J1) labeling sections and cards, faint PCB trace artwork in the hero, and a skills table presented as a Bill of Materials. The BOM table is the single signature element; everything else stays quiet and disciplined.

Avoid these generic AI-design defaults: cream background + serif + terracotta; near-black + acid green; broadsheet hairline-rule newspaper look.

### 3.2 Color tokens (CSS custom properties)
Palette: matte board black with copper accent (rev 3, chosen over the original PCB green). Copper and pad-gold accents carried over unchanged; the green base and trace tones were replaced with tinted near-blacks.
| Token | Hex | Role |
|---|---|---|
| --soldermask | #121214 | page background, matte board black |
| --panel | #19191d | card and table-header background |
| --panel-edge | #2b2b33 | borders, dividers |
| --silkscreen | #ece9e2 | primary text, off-white |
| --muted | #9a9aa4 | secondary text |
| --copper | #d99a5b | accent: eyebrows, refs, buttons, hover borders |
| --pad-gold | #ecc784 | links, highlights, button hover |
| --trace | #232329 | faint trace lines, dormant card refs |

Note: the hero SVG trace stroke (#232329), via fills (#2b2b33), theme-color meta (#121214), favicon background (#121214), and the JS unknown-language fallback dot (#9a9aa4) all track these tokens and must be changed together if the palette changes again. To re-theme, swap the eight :root values plus those five hard-coded spots.

### 3.3 Typography
| Role | Face | Usage |
|---|---|---|
| Display | Chakra Petch (500/600/700) | h1, h2, h3 card titles. Squared, technical character |
| Body | IBM Plex Sans (400/500/600) | paragraphs, table cells |
| Utility | IBM Plex Mono (400/500) | nav, eyebrows, refs, buttons, meta rows, tags, footer. Uppercase with generous letter-spacing (0.08 to 0.18em) |

Loaded via Google Fonts with preconnect. h1 sizes with clamp(2.4rem, 6vw, 4.2rem).

### 3.4 Components
- **Buttons (.btn):** mono uppercase, 1px copper border, transparent; hover fills copper with soldermask text. Solid variant inverts by default.
- **Cards (.card):** panel background, 1px panel-edge border, square corners (no border-radius anywhere), hover lifts 2px and turns border copper.
- **Topic tags (.topic):** tiny mono, gold text, bordered.
- **BOM table:** collapsed borders, mono uppercase caption and headers, copper mono ref column.
- **Status box:** dashed border, centered mono text; error variant in gold/copper.

### 3.5 Quality floor
- Responsive: grid auto-fills; below 720px the About columns stack, hero padding shrinks, nav gaps tighten.
- Accessibility: visible :focus-visible outlines (gold, offset 3px), semantic landmarks (header/main/section/article/footer), aria-label on nav, aria-hidden on decorative SVG, scope attributes on table headers.
- prefers-reduced-motion: disables smooth scroll, all transitions and animations.
- Security: API data rendered via textContent only; external links get rel="noopener".

---

## 4. Deployment (GitHub Pages, free)

1. Edit CONFIG in index.html: set `githubUsername`, update `links` (email, GitHub, LinkedIn).
2. Create a **public** GitHub repo named exactly `<username>.github.io`.
3. Upload index.html (web UI: Add file > Upload files, or git push to main).
4. Site is live at `https://<username>.github.io` within a couple of minutes. GitHub Pages auto-serves index.html from the root; no settings changes usually needed (if not live, enable Pages in repo Settings > Pages > Deploy from branch > main).

New repos appear on the site automatically on the next page load. No redeploy ever needed for project updates.

### Content tips for good-looking cards
- Give each repo a one-line description (becomes card text).
- Add topics to repos (gear icon next to About on the repo page); up to 5 show as tags.
- Hide unwanted repos via `excludeRepos` in CONFIG instead of making them private.

---

## 5. Testing approach (how rev 2 was verified)
Playwright (Python, Chromium) against a local http.server, with the GitHub API mocked via route interception (the sandbox IP is rate limited, and mocking makes tests deterministic). Verified: setup-instruction state with placeholder username, 5 of 6 fixture repos rendered (fork filtered), newest-first sort, topic tags and language dots, card opacity 1 after reveal animation, trace dashoffset reaching 0, mobile 390px layout with no header overlap (bounding box check), and full visibility under emulated prefers-reduced-motion. Zero console or page errors.
Note when testing a patched copy: replace only the CONFIG value `githubUsername: "YOUR_GITHUB_USERNAME"`, not every occurrence of the placeholder string, because the guard clause compares against that same literal.

## 6. Known limitations and accepted trade-offs
- Unauthenticated API: 60 req/hour per visitor IP. Fine for a portfolio; would need a token + proxy only at high traffic (deliberately out of scope).
- Language colors: static subset map, not the full GitHub linguist set; unknown languages get a neutral dot.
- One language per repo (GitHub's primary language field); per-repo language breakdown would need an extra API call per repo (skipped to stay within rate limits).
- README content is not pulled into cards (could be added via the readme endpoint, but descriptions + topics were judged sufficient).

## 7. Possible future enhancements (not built)
- Pinned-repos-first ordering via GitHub GraphQL (requires a token, so likely via a small serverless function).
- Per-project detail pages rendering the repo README (marked.js + readme endpoint).
- Dark/light toggle (currently dark-only by design).
- Custom domain (~10 to 15 EUR/year): add CNAME file + DNS A records to GitHub Pages IPs.
- Simple caching of the API response in memory per session.

## 8. Writing style rules for any future edits
- No em dashes in any copy or code comments. Use commas, periods, or rephrase.
- Copy is plain, specific, active voice. Buttons say exactly what they do ("View projects", not "Explore").
- Errors explain what went wrong and how to fix it, never vague.
- Keep the register professional and human; avoid AI-sounding filler phrases.
