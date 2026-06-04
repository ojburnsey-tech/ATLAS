# Vulcan Quanta — CLAUDE.md

AI-powered quantity surveying SaaS for UK builders and contractors. Produces priced Bills of Quantities (BoQ) from uploaded architectural drawings in under 2 minutes.

---

## Repository layout

```
/
├── Vulcan Quanta.html          # Main entry point — loads JSX files via <script type="text/babel">
├── Vulcan Costing.html         # Alternate entry (same architecture, different TWEAK_DEFAULTS)
├── Vulcan Quanta Standalone.html  # Fully self-contained bundle (all JS inlined)
├── tweaks-panel.jsx            # Design-tool panel + all reusable form controls
├── vq-shared.jsx               # Shared layout: Header, Footer, AnnouncementBar, BoQMockup, ToastContainer
├── vq-pages.jsx                # All page components (one file, section-delimited)
├── logo.png                    # Brand mark (opaque)
├── logo-transparent.png        # Brand mark (transparent bg — used everywhere in UI)
└── uploads/                    # Sample assets (reference PDF, screenshots)
```

---

## Tech stack

| Concern | Choice |
|---|---|
| UI framework | React 18.3.1 (UMD build, loaded from unpkg) |
| JSX transform | Babel standalone 7.29.0 (in-browser, no build step) |
| Styling | Plain CSS in `<style>` block inside HTML; CSS custom properties throughout |
| Fonts | DM Sans (headings/display) · Inter (body) — Google Fonts |
| Routing | Custom — single `page` state string, `go()` prop drilled to all pages |
| State | React `useState` only; no Redux, Zustand, or Context |
| Build toolchain | None — open the HTML file directly in a browser |

**There is no npm, no bundler, no compile step.** JSX is compiled on first load by Babel standalone. This is intentional for rapid prototyping.

---

## How the app is wired together

### Symbol sharing across files

Because each JSX file is evaluated independently by Babel, they cannot use ES module imports. Components are published to `window` at the end of every file:

```js
// vq-shared.jsx
Object.assign(window, { AnnouncementBar, Header, Footer, BoQMockup, ToastContainer });

// vq-pages.jsx
Object.assign(window, { LandingPage, ResultsPage, DashboardPage, ... });

// tweaks-panel.jsx
Object.assign(window, { useTweaks, TweaksPanel, TweakSlider, ... });
```

The HTML entry point then destructures from `window` before rendering:

```js
const { Header, Footer, ... } = window;
const { LandingPage, ResultsPage, ... } = window;
```

**Load order matters** — the HTML `<script>` tags must appear in dependency order: `tweaks-panel.jsx` → `vq-shared.jsx` → `vq-pages.jsx` → inline app script.

### Routing

Navigation is a plain `page` string in `App` state:

```js
const [page, setPage] = useState('landing');
const go = useCallback(p => { setPage(p); window.scrollTo({ top: 0, behavior: 'instant' }); }, []);
```

`go` is passed as a prop to every page component. No URL changes occur — this is a single-page prototype without browser history integration.

**Available page keys:**

| Key | Component | Chrome (Header/Footer) |
|---|---|---|
| `landing` | `LandingPage` | Yes |
| `pricing` | `PricingPage` | Yes |
| `upload` | `UploadPage` | Yes |
| `signin` | `SignInPage` | No |
| `signup` | `SignUpPage` | No |
| `dashboard` | `DashboardPage` | No |
| `results` | `ResultsPage` | No |
| `settings` | `SettingsPage` | No |

`showChrome` boolean gates Header/Footer:
```js
const showChrome = !['signin','signup','dashboard','results','settings'].includes(page);
```

### Toast notifications

A global `toast(msg, type)` function is prop-drilled to every page. Types: `'info'`, `'success'`, `'error'`. Toasts auto-dismiss after 3.8 seconds.

```js
toast('PDF export coming soon.', 'info');
toast('Changes saved.', 'success');
toast('Please fill in all fields.', 'error');
```

---

## Design system

### CSS tokens (`:root`)

```css
/* Neutral scale (Tailwind slate) */
--c-950  #0F172A    --c-900  #111827    --c-800  #1E293B
--c-700  #374151    --c-600  #475569    --c-500  #64748B
--c-400  #94A3B8    --c-300  #CBD5E1    --c-200  #E2E8F0
--c-100  #F1F5F9    --c-50   #F8FAFC    --white  #FAFAFA

/* Semantic */
--amber  #F97316    --amber-d  #EA6C0A
--blue   #2563EB    --blue-d   #1D4ED8
--green  #10B981
--red    #EF4444

/* Layout */
--max-w  1120px     --px  80px (→ 32px @ 1024px → 20px @ 768px)
--font-d  'DM Sans'    --font-b  'Inter'
--ease   cubic-bezier(0.4, 0, 0.2, 1)
--t      200ms var(--ease)
```

`--amber` is overridable at runtime by the TweaksPanel accent toggle (amber ↔ blue). Never hardcode `#F97316` in new CSS — use `var(--amber)`.

### Typography classes

| Class | Font | Size | Weight |
|---|---|---|---|
| `.display-xl` | DM Sans | 76px | 800 |
| `.display-lg` | DM Sans | 48px | 700 |
| `.display-md` | DM Sans | 32px | 700 |

### Button system

Base: `.btn` — then chain modifiers:

```
Colour:   .btn-amber  .btn-blue  .btn-outline  .btn-ghost  .btn-ghost-dim  .btn-nav-pill
Shape:    .btn-pill   (border-radius: 100px)
Size:     .btn-sm     .btn-lg
```

All buttons inherit `font-family: var(--font-b)` and use `cursor: pointer`.

### Layout

- `.inner` — centred container, `max-width: 1120px`, horizontal padding `var(--px)`
- `.section` — standard marketing section, `padding: 96px 0`
- Section backgrounds: `.section-white`, `.section-light`, `.section-dark`, `.section-darker`
- Dashboard: `.dash-wrap` (flex) → `.dash-side` (224px sidebar) + `.dash-main` (flex: 1)

### Responsive breakpoints

| Breakpoint | `--px` | Notable changes |
|---|---|---|
| ≤ 1024px | 32px | Hero goes single column; sidebar hidden; 2-col feature grid |
| ≤ 768px | 20px | Nav/header-right hidden, hamburger shown; single column everywhere |

---

## TweaksPanel

`tweaks-panel.jsx` ships a floating design-tool panel for live prototype iteration. It is not end-user UI — it is only shown when the host activates edit mode.

### useTweaks hook

```js
const TWEAK_DEFAULTS = /*EDITMODE-BEGIN*/{
  "accent": "amber",
  "headline": "Price any job in 2 minutes, not 8 hours.",
  "cta": "Start free"
}/*EDITMODE-END*/;

function App() {
  const [t, setTweak] = useTweaks(TWEAK_DEFAULTS);
  // t.accent, t.headline, t.cta
  // setTweak('key', value)  OR  setTweak({ key: value, ... })
}
```

The `/*EDITMODE-BEGIN*/` / `/*EDITMODE-END*/` delimiters allow the host tool to rewrite the JSON block on disk when tweaks are saved.

### Available controls

| Component | Use for |
|---|---|
| `TweakSlider` | Numeric range (font size, spacing, opacity) |
| `TweakToggle` | Boolean flags (dark mode, feature flags) |
| `TweakRadio` | 2–3 short options; auto-falls-back to `TweakSelect` past ~10–16 chars/option |
| `TweakSelect` | Many/long options |
| `TweakText` | Free-text copy edits |
| `TweakNumber` | Precise numeric with scrub-drag support |
| `TweakColor` | Curated colour/palette swatches (3–4 options recommended) |
| `TweakButton` | Trigger actions from the panel |
| `TweakSection` | Visual separator / label |
| `TweakRow` | Layout wrapper (use inside custom controls) |

### Host postMessage protocol

The panel uses `window.postMessage` to communicate with the host iframe container:

| Direction | Message type | Meaning |
|---|---|---|
| → parent | `__edit_mode_available` | Panel mounted and ready |
| ← parent | `__activate_edit_mode` | Show the panel |
| ← parent | `__deactivate_edit_mode` | Hide the panel |
| → parent | `__edit_mode_set_keys` | Tweak value changed; host rewrites EDITMODE block |
| → parent | `__edit_mode_dismissed` | User closed the panel |

A `tweakchange` CustomEvent is also dispatched on `window` for in-page listeners (e.g. deck thumbnails).

---

## Page components reference

### LandingPage
Props: `{ go, tweaks, toast }`

Sections in order: Hero → Logo strip → How it works (3 steps) → ROI stats → Features grid (6 cards) → Trust section → Pricing cards → Early access CTA → FAQ accordion → Security strip.

`tweaks.headline` and `tweaks.cta` are live-editable via TweaksPanel. `renderHeadline()` bolds `"2 minutes"` and `"8 hours"` in amber.

### ResultsPage
Props: `{ go, toast }`

Interactive BoQ table. Quantities are editable inline (`<input>` in table cells). Contingency percentage is also editable. Grand total recalculates live. PDF/Excel/Share buttons show toast placeholders (not yet implemented). Uses BCIS Q2 2026 hardcoded rates.

Flagged rows (items needing review) show an amber left-border and a `⚠ Verify` chip.

### UploadPage
Props: `{ go, toast }`

Drag-and-drop PDF upload zone. On file select/drop, runs a simulated progress animation (`uploading` → `processing` → `done`) and auto-navigates to `results`. Accepts `.pdf` only, up to 50 MB per copy.

### DashboardPage
Props: `{ go, toast }`

Sidebar layout. Currently shows an empty state with CTA to upload. Sidebar links: Projects (active), New project, Settings, Sign out.

### SettingsPage
Props: `{ go, toast }`

Four tabs: Account (personal details, password, danger zone), Branding (company identity, logo upload, brand colours), Rates (regional settings, trade rate overrides), Billing (current plan, payment method, invoice history).

### SignUpPage
Props: `{ go, toast, plan }`

`plan` prop accepts `'free'`, `'pro'` (default), or `'studio'` to pre-select a plan. Validates name/email/password (8+ chars minimum) before simulating account creation and navigating to `upload`.

### SignInPage
Props: `{ go, toast }`

Email + password form. Simulates auth and navigates to `dashboard`. "Forgot password?" triggers a toast.

### PricingPage
Props: `{ go, toast }`

Full pricing page with plan cards + detailed feature comparison table.

---

## Shared components

### Header
- Sticky, `z-index: 200`, dark with backdrop blur
- Nav links: How it works → `landing`, Pricing → `pricing`, Docs (soon, shows toast)
- Buttons: Sign in → `signin`, Start free → `signup`
- Hamburger menu for mobile (≤ 768px)

### Footer
- 4-column grid: brand description, Product links, Company links, Legal links
- Company details: Vulcan Quanta Ltd · 12 St Ann's Square · Manchester, M2 7HG · Co. No. 14987234 · VAT 456 789 012

### BoQMockup
Static decorative component used in the hero section. Shows a sample BoQ table with trade sections, flagged items, and grand total.

### ToastContainer
Fixed, bottom-centred, `z-index: 9999`. Renders active toasts from `App` state. Toasts animate in via `toastIn` keyframe.

---

## Scroll reveal

On every page change, `App` runs an `IntersectionObserver` effect over these selectors:

```
.feat-card, .step, .pricing-card, .trust-pt, .acc-item,
.sec-item, .roi-grid > div, .logos-row, .boq-mockup,
.empty-state, .scard
```

Elements start with `opacity: 0; transform: translateY(22px)` (`.will-reveal`) and transition to visible (`.appeared`) when they enter the viewport. Siblings get staggered `transitionDelay` up to 320ms.

Hero elements animate in via CSS `@keyframes heroIn` with staggered delays — no JS involved.

---

## Product context

- **Market**: UK builders, contractors, and QS (quantity surveying) practices
- **Core value prop**: Replaces 6–8 hours of manual BoQ measurement per job
- **Rates**: BCIS (Building Cost Information Service) Q2 2026 — NRM2-aware structure
- **Accuracy claim**: 94% on standard projects
- **Data handling**: GDPR-compliant, UK-hosted, drawings deleted after 30 days, never used for model training
- **Status**: Private beta as of June 2026

### Pricing tiers

| Plan | Price | Key limits |
|---|---|---|
| Free | £0 | 2 projects/month, watermarked output, PDF only |
| Pro | £39/month | Unlimited projects, custom branding, PDF + Excel |
| Studio | £99/month | Everything in Pro + 5 team seats, white-label, custom rates |

---

## Working conventions

### Adding a new page

1. Add the component to `vq-pages.jsx` and include it in the `Object.assign(window, {...})` at the bottom
2. Destructure it in the HTML entry point's `const { ... } = window;` block
3. Add it to the `pages` map in `App`
4. Add it to the `showChrome` exclusion list if it should have no Header/Footer
5. Add a `TweakSelect` option for the "Jump to page" control in TweaksPanel if useful for development

### Adding a new tweak

1. Add the key and default value to `TWEAK_DEFAULTS` (inside the `/*EDITMODE-BEGIN*/` block)
2. Add a corresponding `Tweak*` control inside `<TweaksPanel>` in `App`
3. Consume `t.yourKey` in the relevant component

### CSS conventions

- Use `var(--token)` — never hardcode hex values for colours in CSS
- Use `var(--t)` for `transition` shorthand
- Use `var(--ease)` for custom easing
- Section backgrounds must use one of the four `.section-*` classes or match their values
- New marketing sections go between existing `<section>` elements in `LandingPage`; always add `className="section section-*"` and wrap content in `<div className="inner">`

### Mock data

All data is hardcoded. There is no API. The upload flow fakes a progress bar with `setInterval`. The sign-in/sign-up flows fake network delay with `setTimeout`. PDF/Excel export shows a toast saying "coming soon."

Do not add real fetch calls or API keys without first establishing a backend strategy.

---

## Files NOT to edit

- `Vulcan Quanta Standalone.html` — this is an auto-generated self-contained bundle. Edit the source files (`vq-*.jsx`) and re-bundle; do not hand-edit the standalone.
- `logo.png` / `logo-transparent.png` — brand assets, replace only with approved brand files.
- `uploads/` — sample reference material only.
