# Engineering notes

The implementation as of August 2026, described without exposing application source or operational identifiers. Figures come from the codebase's own build dossier.

## Architecture

A React 19 single-page application in TypeScript, built with Vite, deployed on Firebase Hosting. Firestore is the only datastore; Firebase Auth handles client and staff accounts; there are no Cloud Functions, because the project runs on the free tier by design. Every route is lazy-loaded behind `React.lazy` and a Suspense boundary, so the Firebase SDK — the single largest dependency at roughly 110 KB gzipped — lands in its own chunk instead of blocking first paint. The main bundle is about 92 KB gzipped. 47 modules, roughly 9,300 lines.

## 1. Security rules as the authorization layer

The clinic needed booking, lead capture and a CMS, but no server. Firebase's free tier has no Cloud Functions, so there was nowhere to put server-side logic — and rather than add a host and a bill, the trust boundary moved into Firestore rules.

Public collections accept **creates only**, and only when the payload carries the required keys with the expected types; reads, updates and deletes require the admin claim. Appointments add a second read path so an authenticated client can see the documents matching their own email and nothing else. Nine collections, each with its own rule block.

Transactional email — the one thing that genuinely wants a server — goes through a third-party form API, fire-and-forget, so a failed notification can never block the Firestore write that is the actual source of truth.

**Tradeoff.** Rules are a validation layer, not a business-logic layer: they can refuse a malformed booking but cannot, say, check a calendar for a double-booking. That is accepted because staff review every booking by hand anyway. The rules are also the only enforcement of the model, so a rule regression would be a data exposure rather than a broken page; the mitigation is that the rule set is small and reviewed with every collection change.

## 2. Making a single-page app rank

Client-rendered routes are a known SEO liability, and this site lives on local search. Three mechanisms address it. Eight high-value routes are prerendered at build time by running the app in headless Chrome and capturing the resulting HTML. Every page drives its own title, description, canonical URL, Open Graph, Twitter Card and JSON-LD through a custom hook that restores the previous values on unmount, so metadata never leaks between routes. The sitemap is written by a build script that imports the same data module the UI renders from — services, team and articles cannot drift out of sync with what is advertised to crawlers, because there is only one list.

## 3. Consent that actually gates something

The clinic runs paid TikTok campaigns, so the pixel is load-bearing — but under GDPR and ePrivacy it must not fire before consent. Consent lives in a versioned four-category store (essential, functional, analytics, third-party); bumping the version forces a re-prompt if the categories ever change. The pixel injects itself only once the analytics category is granted, and subscribes to a custom event so it activates the moment a visitor changes their mind — no reload. The Google Maps iframe on the contact page is gated the same way and renders a static address card with map, Waze and Moovit links when third-party cookies are declined.

## 4. The content audit

The clinic reported that service photos "sometimes didn't match." They matched almost nothing. Content had been migrated from a hosted CMS that named every image with an opaque hash, so pairings had been made by filename, effectively at random. Hashing the image directory showed why: **29 device photographs were only 16 unique files**, the rest byte-identical duplicates under different names. A biofeedback description sat over a photo of a different device; four of five blog posts carried unrelated equipment; a "generic" fallback pool was quietly serving specific medical equipment as decoration on any article without a cover.

The fix was not a find-and-replace. I opened every unique image, identified the equipment in each, and replaced the hash references with a named, commented map — so a wrong pairing is now visible on sight in code review instead of hidden behind a hash. The service list was consolidated from sixteen ad-hoc entries to the six categories and fifteen sessions the clinic actually books, verified against their e-commerce site, with a 301 for every retired URL: **20 redirects, zero new 404s.**

This is the story I would tell in an interview, because it is not a coding problem. It required noticing that a vague complaint pointed at something systemic, finding the mechanism, fixing the root cause rather than the symptoms, and leaving the codebase in a state where the same class of bug is visible next time.

## What this design does not do

- No automated tests. The project predates the verify-gate discipline of the later ones; changes are checked by hand before deploy.
- No server-side rendering. Prerendering covers eight routes; the rest render client-side and rely on Googlebot executing JavaScript.
- No real-time booking conflict detection, for the reason given under the rules tradeoff.
