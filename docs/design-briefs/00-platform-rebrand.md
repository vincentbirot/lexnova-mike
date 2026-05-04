# Brief 00 — Platform Rebrand: Mike → Lexnova AI

## Context

Lex Nova Partners has forked the open-source Mike legal AI platform for internal firm use. The product needs to feel like Lex Nova's own internal tool, not a third-party application. Every visible mention of "Mike" must be replaced with "Lexnova AI". The visual identity must shift from Mike's neutral chrysanthemum mark to Lex Nova's brand identity. Branding extends across the web app, login screens, error pages, transactional emails, browser tab titles, and source-code metadata.

This rebrand is a prerequisite to every other feature brief. Without it, lawyers using the platform will be confused by the dual identity. It is not a deep redesign — the underlying design system (colors, typography, layout) is preserved. Only the wordmark, logo, and product name change.

## Audience

All Lex Nova users: partners, associates, paralegals, and admin staff.

## Primary user stories

A lawyer opens the app for the first time and immediately recognizes it as Lex Nova's tool. The wordmark in the sidebar reads "Lexnova AI" in a typeface consistent with Lex Nova's existing brand assets. The browser tab title reads "Lexnova AI — [page]". The favicon shows the Lex Nova mark.

A lawyer reads an email from the platform (matter assignment notification, password reset, etc.) and sees Lex Nova branding in the header, signature, and unsubscribe footer.

A lawyer shares a screenshot externally (e.g., to a client or counsel) without realizing that the product is built on Mike. The screenshot shows only Lex Nova identity.

A new associate joins the firm and sees Lex Nova branding from sign-up through onboarding. There is no Mike branding anywhere.

## Visual identity inputs needed from Lex Nova

Before implementation, the Lex Nova partners must provide:

1. Primary logo SVG (full wordmark, "Lex Nova Partners" or just "Lexnova AI" — the partners decide whether to use the full firm name or the AI sub-brand)
2. Mark / icon SVG for use in compact contexts (sidebar collapsed, favicon, app icon, push notifications)
3. Brand color hex values, ideally one primary and one accent that work in both light and dark modes
4. Brand font family if different from Inter / EB Garamond — otherwise we keep the current pairing
5. Any tagline or descriptor (e.g., "Internal Legal AI Platform" or just "AI for Lex Nova Partners")

If any of the above is not yet defined, use these temporary defaults:

- Wordmark: "Lexnova AI" set in EB Garamond Bold 600, color `--foreground`
- Mark: stylized "LN" monogram in EB Garamond, white on `--foreground` background, rounded-md
- Primary brand color: keep the existing azure blue `rgb(0, 136, 255)` since it complements the legal-document neutral palette
- Tagline: "AI for Lex Nova Partners" (used in metadata description, not displayed in chrome)

## Screens that need updating

### Sidebar (top section)

Replace `MikeIcon` SVG and "Mike" wordmark with Lexnova AI mark and wordmark. Mark is 32x32px, wordmark sits to the right with 8px gap. When sidebar is collapsed, only the mark shows. The collapse toggle (`PanelLeft`) remains adjacent to the wordmark on the right. On hover of the wordmark, no underline (it is not a link). On click of the wordmark, navigate to `/assistant` (current behavior).

### Login screen (`/login`)

Currently shows "Mike" centered above the login card. Replace with Lexnova AI logo (wordmark) centered, 80px wide, 32px below the top of the card. Below the logo, a small tagline in `text-muted-foreground` 13px: "Internal Legal AI Platform". Login form below remains unchanged structurally.

### Sign-up screen (`/signup`)

Same logo treatment as login. Form remains unchanged.

### Page titles in browser tabs

Update Next.js metadata in `frontend/src/app/layout.tsx`:

- `title`: change from `"Mike - AI Legal Platform"` to `"Lexnova AI"`
- Each route's `title` becomes `"Lexnova AI — [page]"` (e.g., "Lexnova AI — Matters")
- `description`: `"Internal AI platform for Lex Nova Partners. Document analysis, contract review, and matter management."`

### Favicon and app icons

Replace these files with Lex Nova versions:

- `frontend/public/favicon.ico` (32x32 multi-resolution)
- `frontend/public/icon.svg` (vector, used by modern browsers)
- `frontend/public/apple-touch-icon.png` (180x180)
- Add `frontend/public/icon-192.png` and `icon-512.png` for PWA installs

The mark on these icons should be the LN monogram or whatever logo Lex Nova provides.

### Error pages

`frontend/src/app/error.tsx`, `not-found.tsx`, `global-error.tsx`: replace any "Mike" text with "Lexnova AI". Pattern remains: friendly heading, helpful message, primary button to return home.

### Empty states

Where Mike currently appears in placeholder copy ("Welcome to Mike", "Mike is preparing your document"), replace with "Lexnova AI" or simply remove the brand name from the copy. Brand presence in the chrome (sidebar, tabs) is sufficient — empty states can be brand-less.

### Toast and error messages

Search for "Mike" in toast / error / alert strings. Replace with "Lexnova AI" or remove. Example: "Mike is thinking..." becomes either "Thinking..." or "Lexnova AI is thinking...". Prefer the former for elegance.

### Transactional emails (when added)

Future-proof: when we add Resend or similar email provider, every email template must use Lex Nova branding. Reserve the "from" address as `noreply@lexnovapartners.com` once DNS is configured. Email templates should follow this structure:

- Header: Lex Nova mark + wordmark, neutral background
- Body: white card on neutral background, Inter font, 14px body
- Signature: "— The Lex Nova team" or similar firm-voice closing
- Footer: small print, unsubscribe link if marketing, "This email was sent to {{email}} as a notification from Lexnova AI" for transactional

### Source code metadata

Update without UI impact:

- `frontend/package.json`: change `name` to `lexnova-ai-frontend`, update `description`
- `backend/package.json`: change `name` to `lexnova-ai-backend`, update `description`
- README.md at repo root: rewrite first paragraph to identify the project as Lexnova AI, the firm's internal fork of Mike, with a link to the upstream source for transparency

### About / version dialog (future)

Out of scope for this brief but anticipated: when we add an admin-side "About" dialog showing version info, it should credit the upstream Mike project under AGPL as a footer line: "Built on Mike (open-source legal AI). Source available at github.com/willchen96/mike under AGPL-3.0." This is a license obligation, not just a courtesy.

## Component additions

None. The rebrand reuses existing components and only swaps SVG assets and copy strings.

A new component `LexnovaWordmark` should replace `MikeIcon`. Same prop interface (size, className) for drop-in replacement. The mark for the collapsed sidebar lives as `LexnovaMark`. Both should support the same hover and active states as `MikeIcon` does today.

## Edge cases

**Mid-conversation rebrand** (after deployment when this lands): existing logged-in users should see the new branding on next page load without a full sign-out. No data migration is needed because no user-facing data references the brand name.

**Cached HTML in browser**: users may see Mike branding briefly on first reload after deploy due to ServiceWorker or Next.js cache. Force-refresh recommendation in the deploy notes.

**Print stylesheets**: if any pages have print stylesheets (none today, but document creation features may add them), the Lexnova AI wordmark should appear as the document footer along with the page number.

**Right-to-left languages**: not in scope today, but the wordmark should be flippable. Use SVG and ensure `direction: rtl` doesn't break the logo orientation if Lex Nova ever serves Arabic-speaking clients.

## Visual references

Use the existing `/login` and `/signup` pages as the reference for screens that center a logo above a form card. Use the existing sidebar as the reference for the inline wordmark + mark pattern. The Mike chrysanthemum logo at top-left of sidebar is the exact spot where Lexnova AI wordmark + mark go.

## Acceptance criteria

The rebrand is complete when:

1. A grep for "Mike" across `frontend/src` and `backend/src` returns only license / upstream attribution strings (no UI strings)
2. A grep for "MikeIcon", "MikeProject", "mikeApi", etc. shows these renamed to `LexnovaIcon`, `LexnovaMatter`, `lexnovaApi` etc. (or component names preserved with only the visible label changed — partner's call on internal renames)
3. The login screen, sign-up screen, sidebar, browser tabs, and error pages all show Lexnova AI branding
4. Favicons and app icons are replaced
5. `package.json` files reflect the Lex Nova naming
6. README explains the fork relationship
7. AGPL upstream attribution is preserved in a clearly visible footer or about dialog

## Out of scope for this brief

- Email template designs (Brief separate, when Resend is wired up)
- Marketing site or external landing page (Lex Nova has its own existing website)
- Logo design from scratch (assumes Lex Nova provides finished assets)
- Third-language support beyond English and eventual Thai (no ar/zh/fr right now)
