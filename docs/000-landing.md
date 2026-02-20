# 000 — Landing Page

## Overview

Public landing page at `aigent.community`. Live throughout development. Value-oriented messaging for org leaders/CTOs/eng managers. i18n-ready, theme-aware, minimal.

## Context

Current `routes/index.tsx` renders boilerplate SaaS starter kit content (hero, features, footer). Must replace with aigent-specific landing. Existing components in `components/landing/` and `components/navigation/` will be gutted and rewritten.

No i18n setup exists yet. Must bootstrap `use-intl` infrastructure as part of this work.

## Goals

- Communicate aigent value prop clearly to decision-makers
- i18n-ready from day one (EN default, PL)
- Theme-aware (light/dark) with correct logo per theme
- Single CTA: sign up / get started
- SEO basics (meta, OG tags)

## Non-Goals

- Pricing page (later)
- Blog / changelog
- Animated demos or video
- FAQ section (later)

---

## Page Layout

Top-to-bottom, single-page scroll:

### 1. Navigation Bar

Rewrite `components/navigation/navigation-bar.tsx`.

| Element | Detail |
|---------|--------|
| Logo | theme-aware image (see Logo section) + "aigent" text |
| Nav links | none for now (single page) |
| Right side | ThemeToggle + locale switcher (EN/PL text buttons). "Sign In" hidden until auth phase |
| Behavior | sticky, transparent -> blur on scroll (keep existing pattern) |

### 2. Hero Section

Rewrite `components/landing/hero-section.tsx`.

**Messaging direction** (value, not features):

> Your agents work in the dark. You have no idea what they're building, whether it aligns with your vision, or if two agents just made contradictory decisions.
>
> aigent fixes that. Write your vision once. Every agent reads it. Every action is reported back.

Structure:
- Headline: problem-aware, short (e.g. "Your AI agents are flying blind")
- Subheadline: transformation statement (e.g. "Give every agent your vision. See everything they do.")
- Single CTA button: "Get Started" -> `/signup`
- Secondary: "See how it works" -> anchor link `#how-it-works` (smooth scroll)

No badges, no feature chips. Clean.

Background: keep existing gradient blob pattern, update colors to match brand.

### 3. How It Works

New component: `components/landing/how-it-works.tsx`.

3-step visual flow (horizontal on desktop, vertical on mobile):

1. **Write your vision** — "Define rules, decisions, architecture context. Your agents read it on every session."
2. **Agents report back** — "Every action logged automatically. See what changed, what was decided, what was built."
3. **Stay aligned** — "Dashboard shows team activity. Flag misalignment before it ships."

Each step: icon + title + 1-sentence description. Simple grid layout, no cards.

### 4. Value Props

New component: `components/landing/value-props.tsx`.

3 props, each ~2 sentences. Target: org leader pain points.

| Prop | Direction |
|------|-----------|
| Visibility | "Know what every agent is doing across your org. No more black-box AI." |
| Alignment | "One source of truth. Every agent reads your vision before writing a single line." |
| Knowledge sharing | "Decisions from one agent propagate to all. Org-wide learning, automatic." |

Layout: 3-column grid on desktop. Icon + title + description. Use lucide icons.

### 5. Social Proof / Trust

Hidden for now. No placeholder rendered. Add in later phase when we have testimonials/logos.

### 6. CTA Section

Repeat CTA before footer. "Ready to align your AI agents?" + "Get Started Free" button -> `/signup`.

### 7. Footer

Rewrite `components/landing/footer.tsx`.

- Brand name + tagline
- Links: Docs (placeholder), GitHub, X/Twitter
- Copyright
- Locale switcher (duplicate from nav for mobile accessibility)

---

## i18n Strategy

### Bootstrap

No i18n infra exists. Must create:

```
apps/user-application/src/
  i18n/
    core/
      shared.ts        # Locale type, SUPPORTED_LOCALES, DEFAULT_LOCALE
      client.ts        # rewrite functions for locale prefix
      server.ts        # middleware for locale detection
    messages/
      en.json          # English translations
      pl.json          # Polish translations
    query.ts           # server fn to load messages, queryOptions
```

### Translation Keys

Per i18n rules, nested JSON, dot notation:

```json
{
  "nav": {
    "signIn": "Sign In",
    "getStarted": "Get Started"
  },
  "hero": {
    "headline": "Your AI agents are flying blind",
    "subheadline": "Give every agent your vision. See everything they do.",
    "cta": "Get Started"
  },
  "howItWorks": {
    "title": "How it works",
    "step1": { "title": "Write your vision", "desc": "..." },
    "step2": { "title": "Agents report back", "desc": "..." },
    "step3": { "title": "Stay aligned", "desc": "..." }
  },
  "valueProps": {
    "visibility": { "title": "...", "desc": "..." },
    "alignment": { "title": "...", "desc": "..." },
    "knowledge": { "title": "...", "desc": "..." }
  },
  "cta": {
    "headline": "Ready to align your AI agents?",
    "button": "Get Started Free"
  },
  "footer": {
    "tagline": "Organizational context layer for AI agents",
    "copyright": "..."
  },
  "meta": {
    "title": "aigent — Align your AI agents with your vision",
    "description": "..."
  }
}
```

### Locale Strategy

Per i18n rules:
- Public pages: locale from URL path. No prefix = EN, `/pl/` = Polish
- Route rewrites handle prefix stripping
- `beforeLoad` resolves locale -> context -> `IntlProvider` wraps page
- Messages loaded via `useSuspenseQuery` with `staleTime: Infinity`

### Component Pattern

```tsx
import { useTranslations } from "use-intl"

export function HeroSection() {
  const t = useTranslations("hero")
  return (
    <h1 className="text-foreground">{t("headline")}</h1>
  )
}
```

---

## Theme-Aware Components

### CSS Variables

All components use theme-aware classes per `ui.md` rules:
- `text-foreground`, `text-muted-foreground`, `bg-background`, `bg-card`, `border-border`
- Never hardcode `text-white`, `bg-black`, etc.

### Logo Handling

Two logo files:
- `/aigent_white_bg_round.png` — dark logo on transparent/white bg -> use in **light** theme
- `/aigent_black_bg_round.png` — light logo on transparent/black bg -> use in **dark** theme

Implementation:

```tsx
import { useTheme } from "@/components/theme/theme-provider"

export function Logo({ className }: { className?: string }) {
  const { resolvedTheme } = useTheme()

  // Show both, hide one via CSS to avoid flash on theme change
  return (
    <div className={className}>
      <img
        src="/aigent_white_bg_round.png"
        alt="aigent"
        className="block dark:hidden h-8 w-8"
      />
      <img
        src="/aigent_black_bg_round.png"
        alt="aigent"
        className="hidden dark:block h-8 w-8"
      />
    </div>
  )
}
```

CSS-based show/hide (via `dark:` variant) prevents flash. No JS theme check needed for rendering.

---

## Route Placement

Landing page stays at `routes/index.tsx` (path `/`). This is already the case.

The route file orchestrates sections:

```tsx
export const Route = createFileRoute("/")({
  component: LandingPage,
})

function LandingPage() {
  return (
    <div className="min-h-screen bg-background">
      <NavigationBar />
      <main>
        <HeroSection />
        <HowItWorks />
        <ValueProps />
        <CtaSection />
      </main>
      <Footer />
    </div>
  )
}
```

### SEO

Update `__root.tsx` meta:

```ts
...seo({
  title: "aigent — Align your AI agents with your vision",
  description: "Shared context layer for dev teams using AI coding agents. Write your vision, every agent reads it, every action reported back.",
})
```

Add OG image later (Phase 5).

---

## CTA Strategy

Single funnel: all CTAs go to `/signup`.

| Location | Text | Variant |
|----------|------|---------|
| Hero | "Get Started" | `default` (primary) |
| Hero secondary | "See how it works" | `outline`, anchor `#how-it-works` smooth scroll |
| Bottom CTA | "Get Started Free" | `default` (primary), larger |
| Nav | "Sign In" | hidden until auth phase |

No pricing, no plan selection on landing. Sign up is email/password (Better Auth).

---

## File Changes Summary

### New Files

| File | Purpose |
|------|---------|
| `src/i18n/core/shared.ts` | locale types, constants |
| `src/i18n/core/client.ts` | rewrite functions |
| `src/i18n/core/server.ts` | locale middleware |
| `src/i18n/messages/en.json` | English strings |
| `src/i18n/messages/pl.json` | Polish strings |
| `src/i18n/query.ts` | server fn + query options for messages |
| `src/components/landing/how-it-works.tsx` | how it works section |
| `src/components/landing/value-props.tsx` | value prop section |
| `src/components/landing/cta-section.tsx` | bottom CTA section |
| `src/components/landing/logo.tsx` | theme-aware logo component |

### Modified Files

| File | Change |
|------|--------|
| `src/routes/index.tsx` | replace with new landing layout |
| `src/routes/__root.tsx` | update SEO meta, add IntlProvider wiring |
| `src/components/landing/hero-section.tsx` | rewrite with aigent messaging |
| `src/components/landing/features-section.tsx` | delete (replaced by how-it-works + value-props) |
| `src/components/landing/footer.tsx` | rewrite with aigent branding |
| `src/components/landing/index.ts` | update exports |
| `src/components/navigation/navigation-bar.tsx` | update branding, add locale switcher, remove boilerplate links |

---

## Implementation Checklist

1. [ ] Bootstrap i18n infrastructure (`i18n/core/*`, `i18n/messages/*`, `i18n/query.ts`)
2. [ ] Wire IntlProvider in `__root.tsx` (locale detection via `beforeLoad`, messages via suspense query)
3. [ ] Create `Logo` component with CSS-based theme switching
4. [ ] Rewrite `NavigationBar` — aigent branding, logo, locale switcher, sign in
5. [ ] Rewrite `HeroSection` — value messaging, single CTA
6. [ ] Create `HowItWorks` component — 3-step flow
7. [ ] Create `ValueProps` component — 3 value props
8. [ ] Create `CtaSection` component — bottom CTA
9. [ ] Rewrite `Footer` — aigent branding
10. [ ] Delete `FeaturesSection`
11. [ ] Update `routes/index.tsx` — compose new sections
12. [ ] Update `__root.tsx` SEO meta
13. [ ] Write all EN translation strings
14. [ ] Write all PL translation strings
15. [ ] ~~Add locale route rewrite rules~~ — deferred, i18n infra only for now
16. [ ] Test light/dark theme — logo swap, all sections
17. [ ] Test EN/PL switching
18. [ ] Deploy to staging, verify at aigent.community

---

## Resolved Questions

1. **Logo files** — solid background, CSS show/hide works as-is
2. **Locale switcher** — just EN/PL text buttons
3. **"Sign In" in nav** — hide for now, show when auth phase ships
4. **Secondary CTA** — anchor link (smooth scroll to `#how-it-works`)
5. **OG image** — defer to later phase
6. **`/pl/` locale routing** — defer, prepare i18n infra only, add route rewrites later
7. **Copy direction** — iterate on headline/subheadline during implementation

---