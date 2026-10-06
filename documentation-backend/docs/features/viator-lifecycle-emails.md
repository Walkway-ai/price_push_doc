# Viator lifecycle emails (ENG-2498, ENG-2499, ENG-2777)

The [Walkway <> Viator lifecycle plan](https://docs.google.com/document/d/1T8sonOXhZjCdQc2jTBP8mjKLtc0y9DaIDxzDbzXb5jQ/edit)
defines nine emails for Viator partners who arrive through the Supplier Center
link-out. Three backend modules carry them, all on the `lifecycle` consent
stream (never the marketing checkbox), all checked against `email_hard_suppressions`
at send time, all **off by default** behind one env flag each.

| # | Email | Module | Trigger | Stops when |
| --- | --- | --- | --- | --- |
| 1 | Onboarding started, not completed | `src/viator-resume-email` | 24 h after the last onboarding activity, state A–D | state E |
| 2 | Onboarding still incomplete | `src/viator-lifecycle-email` (`E2`) | 3 days after #1, still A–D | state E |
| 3 | Price recommendations are ready | `src/recommendation-ready-email` | a recommendation exists, ≥ 24 h after onboarding completed | one send per partner |
| 4 | Recommendations not explored | `E4` | 2 days after #3, list never loaded since onboarding completed | `recommendations.list_viewed` |
| 5 | Product tour incomplete | `E5` | 2 days after first login, tour flag still true, a recommendation exists | tour flag flips |
| 6A/B/C | Inactive user | `E6A` `E6B` `E6C` | 5 / 8 / 14 days since last login, recommendation exists | a login |
| 7 | Viator direct pricing early access | `E7` | 2 distinct days viewing recommendations, or 14 days after onboarding | waitlist joined |

Every email sends at most once per partner; a partner is on one track at a time
(onboarding → recommendation → inactive, or engagement); no two lifecycle emails
within 24 h. The decision for #2–#7 is `decideLifecycleEmail()` in
`src/viator-lifecycle-email/viator-lifecycle-decision.ts`, a pure function over
signals read at send time, so every stop condition is unit-tested.

## Signals, and where they come from

- **Onboarding state**: the resume ladder A–E (`ViatorResumeStateService`), built
  from artifacts (channel syncs, active product, comp set, price band), shared by
  all three modules so "finished" means one thing.
- **Recommendations viewed**: `feature_usage_events` `recommendations.list_viewed`,
  emitted by the two v2 list routes since ENG-2777. Before that nothing recorded a
  view.
- **Logins**: `User.lastActive` (touched on every login and handoff) and the
  `auth.login` / `viator_partner.entry` events (90-day retention).
- **Tour**: `User.mustShowRecommendationsTour` (ENG-2459).
- **Recommendation**: the #3 payload (`RecommendationReadyPayloadService`), so #4 and
  #6 name the same product and quote the same prices as #3.
- **Waitlist**: `User.viatorWaitlistJoined` (ENG-2346).

## CTAs and tracking

Buttons land on the supplier portal (a partner has no Walkway session until they
come back through it), tagged `utm_source=viator_lifecycle_email&utm_content=<key>`
(`recommendation_ready_email` for #3). #7 is the exception: a signed, sessionless
link `GET /api/viator-lifecycle-email/waitlist/:token` joins the waitlist
idempotently and redirects to the app with `?viatorWaitlist=joined|already|invalid`.
Each send emits `lifecycle_email.sent`.

## Operating it

Admin routes, tokenless like the rest of `api/admin/*`:
`GET /api/admin/viator-lifecycle-email/stats`, `preview/:userId?emailKey=`,
`sends/:userId`, `POST dry-run?limit=`, `POST test-send` (+ `x-test-token`).
Same shape under `viator-resume-email` and `recommendation-ready-email`.

Rollout, one email at a time: `POST dry-run` to read who would get what and why,
`test-send` to an internal address, then `VIATOR_LIFECYCLE_EMAIL_<KEY>_ENABLED=true`.
Timing knobs: `VIATOR_LIFECYCLE_E2_DAYS`, `_E4_DAYS`, `_E5_DAYS`,
`VIATOR_LIFECYCLE_INACTIVE_DAYS` (`5,8,14`), `_E7_DAYS`, `_E7_MIN_VIEW_DAYS`,
`VIATOR_LIFECYCLE_MIN_HOURS_BETWEEN`, `VIATOR_LIFECYCLE_CTA_URL`.

Database: `prisma/manual/eng-2777-viator-lifecycle-email-sends.sql` creates
`viator_lifecycle_email_sends` and an index on `feature_usage_events
(user_id, feature, occurred_at desc)`.

## Open points from the plan

- **#6 timing**: plan table 7 / 14, copy table 5 / 8 / 14 (default), a comment calls
  it excessive. One env var.
- **After #2**: a partner still stuck gets nothing more, since #6 needs
  recommendations and those need state E.
- **Intro email** with product visuals: no trigger yet, not built.
- **Templates**: one generic template (resume shell + plan copy + price card) until
  the frontend task delivers per-email designs. The #1 and #3 template headlines
  still read the old subjects; the inbox reads the plan's.
- **Estimated revenue impact** in #4 and competitive-set prices in #6B: the plan's
  copy mentions them, the payload exposes current / recommended / delta / date
  only. Needs a cheap read of expected revenue and competitor prices per slot.
