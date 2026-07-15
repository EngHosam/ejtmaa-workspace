---
name: website-link-href-audit
description: Audits website cross-page navigation — verifies internal links use the Link component from @my-ssr/web-core with a typed Href<keyof MPagesRoutes> object (not a raw string path), that unimplemented pages pass href undefined, that external is used only for external URLs, and that same-page section nav is scroll-based not route-based. Use when reviewing Link usage, finding raw <a> internal links, wiring new footer/header nav targets, or debugging a link that full-reloads.
---

# Website Link / Href Audit

Audits navigation defects on `website/`:

1. Raw `<a>` / `As={"a"}` used for an internal route (bypasses client-side `nav.push` → full reload).
2. `Link`/`push` given a raw string path instead of a typed `Href<keyof MPagesRoutes>` object.
3. A fake `identify` passed for an unimplemented page instead of `href: undefined`.
4. `external` used for an internal route.
5. `Link`/`push` used for a same-page section jump (should be `scrollIntoView`).

## Background (why these break)

- `Link` (`@my-ssr/web-core` `src/components/Link.tsx`) renders an `<a href>` and intercepts click → `getNav().push(to)`. A raw `<a href="/login">` does a full reload.
- `Href<Ident> = { identify: Ident, replace?, sub? } & ParamsQuery<Ident>`. The object form is type-checked against `MPagesRoutes` (`routes.ts`); a string path is not.
- `useNav()` (CSR-only) returns `BrowserNavigator`; `push({ identify: "X" })` for route nav. Must not be called during render/SSR.
- In-page section nav is scroll-based (`scrollToLandingSection`), not route-based.

See `.cursor/rules/website-link-href-navigation.mdc`.

## Audit steps

1. **Find link candidates.** Grep:
   - `As=\{?[\"']a[\"']\}?` and `<a ` — any raw anchor.
   - `href=\{?[\"']\/` — string paths starting with `/`.
   - `from \"@my-ssr/web-core\"` imports — confirm `Link` is imported where internal nav exists.
   - `useNav` / `\.push\(` — programmatic nav call sites.
2. **For each internal nav target**, confirm:
   - It uses `Link` (or `useNav().push` for button-triggered nav), not a raw `<a>`.
   - `Link`/`push` receives a typed object `{ identify: "X" }` (`Href<keyof MPagesRoutes>`), not a string path.
   - The `identify` value is a member of `MPagesRoutes` (`routes.ts`); `yarn type-check` rejects unknown identifies.
3. **Unimplemented pages.** If a nav target's page is not yet in the `routes` map + `MPagesRoutes`, the caller MUST pass `href: undefined` (Link renders without destination). Flag any guessed/fake `identify` or string path.
4. **External links.** `external` (or a plain `<a>`) is valid ONLY for external URLs (different origin). Flag `external` on an internal route.
5. **Same-page section jumps.** Landing section nav must use `scrollToLandingSection`/`scrollToLandingTop` (`landing-layout/drawer.ts`), not `Link`/`push`. Flag route-based section nav.
6. **Utils + Link combo.** Where `<Text As={Link}>` / `<Box As={Link}>` is used, confirm `Link` props (`href`/`to`/`external`/`replace`/`redirect`/`query`/`state`) flow through `extra`, and `textDecoration`/`buttonReset` follow W29.
7. **Verify** with `yarn type-check` (rejects bad `identify` and wrong `Href` shape).

## Report format

```
<file>:<line> — raw <a> for internal route <target> → use <Text As={Link} extra={{href: {identify: "X"}}}>
<file>:<line> — Link/push given string path "<path>" → use typed Href {identify: "X"}
<file>:<line> — fake identify "X" for unimplemented page → pass href: undefined until wired in routes.ts + MPagesRoutes
<file>:<line> — external used for internal route → remove external (or use plain internal Link)
<file>:<line> — Link/push used for same-page section jump → use scrollToLandingSection
```

## Related

- `.cursor/rules/website-link-href-navigation.mdc`
- `.cursor/rules/website-route-registry-contract.mdc`
- `docs/platforms/website/route-registry-contract.md`
- `docs/invariants/website.md` W54, W55
