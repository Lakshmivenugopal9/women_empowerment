# Building HerCare: A Privacy-First Women's Health Tracker

I built a small web app called **HerCare** — a women's health and cycle tracker that runs entirely in the browser with no backend, no account, and no data leaving the device.

This post walks through what I built, the decisions I made, and what I learned along the way.

---

## The Problem I Wanted to Solve

Most health tracking apps require you to create an account and sync your data to a server. For something as personal as menstrual health, that feels like an unnecessary trade-off. I wanted to build something that:

- Works offline, in any browser
- Stores data only on the user's own device
- Has zero dependencies and no build step
- Is simple enough to be a single file

The result is `HerCare.html` — one file, open it and it works.

---

## What the App Does

### Splash Screen
On first load, a full-screen branded screen shows the app name with a smooth animation. It auto-dismisses after 3 seconds. Tapping it dismisses it immediately.

### Daily Log Entry
Users can log how they feel each day:

- **Date** — defaults to today
- **Cycle status** — Period Day, Ovulation, PMS, or Normal
- **Flow intensity** — Light, Medium, or Heavy (only visible when Period Day is selected)
- **Symptoms** — checkboxes for Cramps, Headache, Bloating, Mood Swings, Fatigue, Acne, Back Pain
- **Mood** — emoji picker with five options
- **Notes** — optional free text

Submitting the form saves the entry to `localStorage`. If an entry already exists for that date, it gets overwritten.

### History
The history view reads from `localStorage` and displays the last 30 entries as cards, sorted by most recent first. Each card shows the date, cycle status, mood, symptoms, and flow intensity. A "Clear All Data" button triggers a confirmation modal before deleting anything.

### Health Reminders
Four static tip cards covering hydration, iron-rich foods, exercise, and when to see a doctor.

### Personalised Health Plan
Users select one or more health concerns from: PCOS, Endometriosis, Irregular Periods, PMS, Fatigue. The app generates a plan with dietary recommendations, exercise suggestions, lifestyle tips, and medical guidance. The plan is rule-based — a JavaScript object maps each concern to pre-written content. When multiple concerns are selected, the sections are merged with deduplication.

### Dark and Light Theme
A toggle switches between themes. The preference is saved to `localStorage` and applied on every load before anything renders, so there's no flash of the wrong theme.

---

## Technical Approach

### Single HTML file
The entire app — HTML structure, CSS, and JavaScript — lives in one `.html` file. No npm, no bundler, no framework. This was a deliberate constraint that kept the code focused and the app portable.

### CSS custom properties for theming
Both themes are defined as CSS variable sets. Switching themes is a single attribute change on the `<html>` element:

```css
:root, [data-theme="light"] {
  --bg: #FFF8F6;
  --surface: #FFFFFF;
  --primary: #E8637A;
  --text: #2D2D2D;
}

[data-theme="dark"] {
  --bg: #1A1520;
  --surface: #241D2B;
  --primary: #E8637A;
  --text: #F5EEF0;
}
```

The browser handles the rest instantly — no JavaScript needed to repaint individual elements.

### View routing without a framework
All five views are `<section>` elements in the DOM simultaneously. JavaScript toggles a CSS class to show the active one. A CSS animation handles the fade-in:

```css
.view { display: none; }
.view.on { display: block; animation: vin .25s ease both; }

@keyframes vin {
  from { opacity: 0; transform: translateY(8px); }
  to   { opacity: 1; transform: translateY(0); }
}
```

### Flow field with CSS height transition
The flow intensity field only appears when "Period Day" is selected. Rather than toggling `display`, I used a `max-height` transition for a smooth reveal:

```css
.flow-wrap { max-height: 0; overflow: hidden; transition: max-height .35s ease, opacity .25s ease; opacity: 0; }
.flow-wrap.open { max-height: 120px; opacity: 1; }
```

### localStorage data model
Each log entry is stored as a JSON object:

```js
{
  id: "1719123456789",
  date: "2025-07-14",
  status: "period",
  flow: "medium",
  symptoms: ["cramps", "fatigue"],
  mood: "sad",
  notes: "Rough day."
}
```

All entries are stored as a JSON array under the key `hercare_entries`. Reading and writing is wrapped in try/catch to handle parse errors and storage quota errors gracefully.

### Rule-based health plan
The health plan is a plain JavaScript object:

```js
const HP = {
  pcos: {
    diet: ['Low-GI foods: oats, legumes, leafy greens', ...],
    exercise: ['30 min moderate cardio 5× per week', ...],
    lifestyle: ['Consistent sleep schedule (7–9 hours)', ...],
    doctor: ['Annual ultrasound to monitor ovarian cysts', ...]
  },
  // endometriosis, irregular-periods, pms, fatigue...
};
```

When the user selects concerns and clicks "Generate My Plan", the app merges the relevant sections across all selected concerns and renders them as cards.

---

## Design Choices

**Color palette** — warm rose `#E8637A` as the primary, dusty mauve `#C084A0` as an accent, deep slate `#1A1520` for the dark background, and off-white cream `#FFF8F6` for light. The goal was cohesive and calm, not loud.

**Typography** — Playfair Display for headings (elegant, editorial feel), DM Sans for body text (clean, readable at small sizes). Both loaded from Google Fonts.

**Mobile-first layout** — bottom navigation bar on mobile, top navigation on desktop, controlled with a single `@media (min-width: 768px)` breakpoint. No JavaScript involved in the layout switch.

**Touch targets** — all interactive elements have a minimum size of 44×44px, which is the standard recommendation for comfortable touch interaction on mobile.

---

## What I Learned

**CSS `max-height` transitions** are a practical way to animate show/hide without JavaScript managing `display`. The key is setting a `max-height` value large enough to fit the content.

**Splash screen tap-to-dismiss needs a guard flag.** Without one, rapid taps can call the dismiss function multiple times and cause the animation to glitch. A simple boolean flag fixes it.

**Single-file constraints are useful.** Not being able to split code across files forced me to think more carefully about structure and keep things lean.

**`localStorage` is reliable for this use case.** JSON round-trips cleanly for the entry data model, and wrapping reads/writes in try/catch handles edge cases like corrupted data or full storage without crashing the app.

---

## Project Structure

```
HerCare.html
├── <head>        Google Fonts, meta tags
├── <style>       All CSS — theme tokens, layout, components, animations
├── <body>
│   ├── #splash   Full-screen intro overlay
│   ├── #app
│   │   ├── #tnav         Top navigation (desktop)
│   │   ├── #main
│   │   │   ├── #view-home
│   │   │   ├── #view-log
│   │   │   ├── #view-history
│   │   │   ├── #view-reminders
│   │   │   └── #view-health-plan
│   │   └── #bnav         Bottom navigation (mobile)
│   ├── #moverlay         Confirmation modal
│   └── #toast            Success notification
└── <script>      All JavaScript — wrapped in an IIFE
```

---

## Possible Next Steps

- Cycle length prediction based on logged history
- CSV export of entries
- PWA manifest for home screen installation
- Charts for visualising cycle patterns over time
- Web Notifications API for reminders

---

## Summary

HerCare is a straightforward project with a clear constraint: everything in one file, everything on the device. It covers the core use case well — daily logging, history review, health tips, and a personalised plan — without any infrastructure overhead.

The full source is a single HTML file. Open it in a browser and it works.
