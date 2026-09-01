# SankofaGo — Architecture Notes

Deeper notes on the decisions behind the platform. Two codebases, one product: a Flutter app (release candidate) and a live Next.js website, sharing a Supabase backend and a written set of contracts.

## 1. The trust model: AI drafts, humans book

The core product claim — "An AI plans it. A real person books it." — is enforced in code, not just marketing:

- The pricing engine never outputs an exact price, only an estimate range (90–110% of a catalog midpoint, rounded outward to the nearest $100), revealed only after the traveler has chosen themes, route, group, timing, and tier.
- The booking endpoint (`src/app/api/bookings/route.ts`) validates with Zod, HTML-escapes everything, and emails a structured itinerary summary to the team via Resend. There is no auto-confirm path anywhere.
- Nana's server-side system prompt (in the `nana-reply` edge function) forbids claiming any price, availability, visa rule, or safety condition is confirmed, forbids inventing supplier names, and treats user text as data, never instructions. Input and output are length-capped; history is truncated.

## 2. Nana AI: an interface, two brains

```dart
abstract class NanaBrain {
  Future<String> reply(String message);
  Stream<String> generateItinerary({required String idea, List<String> destinations});
  List<String> get suggestions;
}
```

- `LocalNanaBrain` is a fully offline knowledge engine over a researched Ghana topic base (`nana_knowledge.dart`, ~1,200 lines), with per-topic rotation cursors so repeated questions get fresh phrasing, and a "tell me more" intent that deepens the last topic.
- `RemoteNanaBrain` wraps it: feature-flagged (`NANA_REMOTE_ENABLED`), one retry, bounded timeout, and silent fallback to the local brain on any failure — signed-out, unconfigured provider (503), timeout, or malformed reply. Itinerary generation and suggestion chips stay local by design: the staged-progress UX is deterministic and offline-safe.
- The provider key lives only in Supabase function secrets. Without it, `nana-reply` reports `configured:false` and the app behaves exactly like the offline build.

The website's chat is a separate Genkit surface on the same model (`gemini-2.5-flash`), configured in `src/ai/genkit.ts`.

## 3. Deterministic pricing, versioned catalog

`src/lib/pricing.ts` + `pricing-catalog.ts` + `pricing-constants.ts`:

- Free-text trip descriptions are parsed with plain regex heuristics (duration, theme keywords, tier signals) — cheap, testable, and predictable.
- Markups are constants per tier (classic 0.28, plus 0.35, luxe 0.50) with a 0.15 commission; the catalog is versioned and served from a server-side store so a quote can cite the catalog version it was priced against.
- The booking payload carries `pricing.catalogVersion`, so the human reviewing the request knows exactly what the traveler saw.

## 4. The parity contract

`Flutter/docs/pricing_interface_contract.md` defines the traveler-facing estimate: when the range appears, what it contains (group total + per-person), the deposit preview (25%, non-charging), and rounding rules. Both the web `/plan` builder and the Flutter plan feature implement this document, which is how two codebases in two languages present the same trip the same way. Contracts-as-docs (`api_contract_draft.md`, `frontend_edge_state_contract.md`, `backend_schema.md`) are the repo's coordination mechanism.

## 5. A database that refuses to regress

The Supabase schema (27 migrations) is organized in layers — identity, catalog, supply/provenance, planning, commerce, trip ops — and defends itself three ways:

1. **Inline acceptance guards.** Every migration ends with SQL assertions that abort deployment if a grant or policy regressed.
2. **Deny-by-default catalog.** Base tables have RLS with no policies; curated `*_public` SECURITY DEFINER views expose only public columns (live-proven in `supabase/tests/catalog_security.sql`).
3. **Server-authoritative commerce.** Writer functions compute totals server-side; a forged client price is ignored, quote acceptance is idempotent, and illegal booking-lifecycle transitions are rejected — all covered by live rollback-only proofs against the real database.

## 6. Honest-claims discipline

`docs/RELEASE_READINESS.md` is the release gate: every claim is classified repository-complete, externally-blocked, or remaining work, with cited evidence (test counts, migration parity checks, security-advisor runs). On the website, JSON-LD structured data is derived strictly from visible page text — no ratings, reviews, or user counts exist anywhere in schema because none exist in reality. Fabricated testimonials were removed once and are permanently banned.

## 7. AEO: built to be quoted correctly

Because the honest-claims rule limits what the site can say, the AEO system makes sure AI search quotes the right things: `llms.txt` and `llms-full.txt` route handlers built from a single `llms-content.ts` module, per-page FAQ blocks that feed FAQPage schema, an RSS feed, and a 16+ post blog on heritage travel. One content source, three machine-readable projections.

## 8. Lead pipeline: never lose a lead

`/api/leads` posts to a Google Sheets webhook first; if the webhook is unconfigured or fails, it falls back to a Resend email — the inbox is the pre-launch CRM, matching how bookings already work. A honeypot `company` field filters bots, and the notification email uses solid colors only because gradients and rgba break in Outlook and Gmail dark mode.
