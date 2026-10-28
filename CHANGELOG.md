# Changelog

## 3.3.0 (2026-10-28)

First public standalone WPForms release, cut from the universal tracker v3.3.0. The shared core and the WPForms handler match the universal build.

- `wpforms_form_success` and `wpforms_form_failed` events, plus `ec_form_interaction` and `ec_tracker_initialized`
- `ec_event_id` and `ec_detection_method` on every event
- Consent gate (Consent Mode v2, OneTrust, Cookiebot, `EC_FORM_CONSENT_CHECK`) with queue and replay
- PII redaction, optional SHA-256 hashing, always-redacted sensitive field names
- 5-minute dedupe ledger
- First and last touch attribution with click IDs
- E.164 phone normalization with `defaultCountryCode` and `countryCodeField`
- `strictSuccess` option, failure diagnostics, SPA route handling
- GTM paste-ready file with production defaults (`debug: false`, `rawFields: false`) and a raw `.js` whose defaults never override a site's own config
