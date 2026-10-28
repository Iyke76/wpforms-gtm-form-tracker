# WPForms GTM Form Tracker

Confirmation-based form tracking for WPForms in Google Tag Manager.

This standalone tracker pushes dataLayer events when WPForms submissions actually succeed or fail. It is built for GTM, GA4, server-side GTM, Meta CAPI and Google Enhanced Conversions workflows where button-click tracking is too noisy.

Docs and payload examples: **[iyrexmedia.com/gtm-form-listeners](https://iyrexmedia.com/gtm-form-listeners/)**

## Why this exists

WPForms submits by AJAX on some forms and by full page reload on others, and many sites redirect to a thank-you page. Click triggers count validation errors, and thank-you triggers count refreshes. This listener handles all three WPForms outcomes and keys fields by their visible labels instead of `wpforms[fields][n]`.

## Files

| File | Use case |
| --- | --- |
| `wpforms-form-tracker.gtm.html` | GTM Custom HTML paste-ready version. Includes the `<script>` wrapper and a production config block at the top. |
| `wpforms-form-tracker.js` | Raw JavaScript for self-hosting. Its config block only applies when `window.EC_FORM_CONFIG` is not already set. Do not wrap it in another `<script>` tag when loading by URL. |

## Events

```text
wpforms_form_success
wpforms_form_failed
ec_form_interaction
ec_tracker_initialized
```

### Success event

`wpforms_form_success` fires on the first of these:

- jQuery `wpformsAjaxSubmitSuccess` (AJAX forms)
- `.wpforms-confirmation-container` rendering
- A non-AJAX postback confirmation page, matched to data stored at submit
- Form removed or page unload after submit (inferred, disable with `strictSuccess: true`)

### Failure event

`wpforms_form_failed` fires on `wpformsAjaxSubmitFailed` or when WPForms renders `label.wpforms-error`, `.wpforms-error-container` or `.wpforms-has-error` after the visitor has interacted with the form.

### Diagnostic events

`ec_form_interaction` fires once per form after the first real input. It carries metadata only, no PII.

`ec_tracker_initialized` fires once per page load so you can confirm the script loaded before a test submission.

## Payload example

```js
{
  event: "wpforms_form_success",
  ec_event_id: "uuid-v4",
  ec_detection_method: "confirmation_event",
  ec_tracker_version: "3.3.0",
  ec_name: "Jane Doe",
  ec_first_name: "Jane",
  ec_last_name: "Doe",
  ec_email: "jane@example.com",
  ec_phone: "+15551234567",
  form_url: "https://example.com/contact",
  form_cta: "Submit",
  form_platform: "wpforms",
  form_position: "top",
  form_index: 1,
  form_id: "12",
  ec_submit_attempt: 1,
  ec_time_to_submit_s: 42,
  ec_fast_submit: false,
  ec_first_source: "google",
  ec_first_medium: "cpc",
  ec_last_source: "google",
  ec_last_medium: "cpc",
  ec_click_id: "Cj0K...",
  ec_click_id_type: "gclid"
}
```

With `rawFields: true`, `ec_raw_fields` is added with every captured field and its `_bucket` classification. With `hashPii: true`, `ec_email_sha256` and `ec_phone_sha256` are added.

When PII consent is denied, PII keys are removed and `ec_pii_redacted: true` is added.

## Quick start in GTM

1. Open Google Tag Manager and create a **Custom HTML** tag.
2. Paste the full contents of `wpforms-form-tracker.gtm.html`.
3. Trigger it on **All Pages (Page View)**, or **Consent Initialization** if your consent setup needs the tracker early.
4. Create a **Custom Event** trigger with regex `^wpforms_form_(success|failed)$`, or event name `wpforms_form_success` for leads only.
5. Create Data Layer Variables for the keys you need (`form_id`, `ec_detection_method`, `ec_event_id`, `ec_email`, `ec_phone`, ...).
6. Attach your GA4, Google Ads, Meta or server-side GTM tags to the trigger.
7. Preview, submit a real test form, confirm one `wpforms_form_success`, then publish.

Retire any old click or thank-you trigger for the same form, otherwise each lead is counted twice.

## Platform notes

- Field keys come from the visible `.wpforms-field-label` text (plus sub-labels such as First and Last), not `wpforms[fields][1]`.
- The honeypot (`.wpforms-field-hp`, `wpforms[hp]`) and HTML fields are excluded.
- Storage is namespaced (`ec_wpforms_form_data`), so it can run next to other standalones, but not next to the universal tracker.

## Configuration

Edit the config block at the top of the GTM file, or define `window.EC_FORM_CONFIG` before the raw `.js` loads.

| Option | Default | What it does |
| --- | --- | --- |
| `debug` | `false` | `[EC]` console logs. Testing only |
| `hashPii` | `false` | Add SHA-256 hashes of email and phone (HTTPS only) |
| `rawFields` | `false` | Include `ec_raw_fields`. Keep it out of GA4 |
| `dedupe` | `true` | Suppress a repeat success for the same form and person within 5 minutes |
| `strictSuccess` | `false` | Drop inferred successes (`pagehide`, `form_removed`) |
| `defaultCountryCode` | `"1"` | Phone fallback when the form has no country field |
| `countryCodeField` | `""` | Exact field name holding an ISO2 or dial code |
| `redactFields` | `[]` | Extra field names (strings or RegExps) to strip |
| `extraThankYouPatterns` | `[]` | Extra thank-you URL substrings |
| `queueOnDenied` | `true` | Queue events while consent is denied and replay them on grant |
| `submitWindowMs` | `15000` | How long after submit a success signal is accepted |
| `ttlMs` | `600000` | Freshness window for thank-you recall |

Password, card, SSN, IBAN and similar field names are always redacted.

## Consent

Auto-detected: Google Consent Mode v2, OneTrust, Cookiebot.

For anything else, define this before the tracker loads:

```html
<script>
window.EC_FORM_CONSENT_CHECK = function () {
  return { analytics: true, pii: false };
};
</script>
```

| Consent state | Behavior |
| --- | --- |
| `analytics: false` | Nothing fires and nothing is stored. Events queue and replay on grant |
| `pii: false` | Events fire without PII keys and with `ec_pii_redacted: true` |

If no CMP or override is found, the tracker assumes full consent. Match that to your own legal requirements.

## PII warning

`ec_name`, `ec_email`, `ec_phone`, the address keys and `ec_raw_fields` are for Enhanced Conversions, Meta CAPI and first-party CRM workflows. **Do not send them to GA4.** GA4's terms prohibit PII in event parameters. Send `form_id`, `form_platform` and `ec_detection_method` to GA4 instead.

## Detection methods

Every success and failure event includes `ec_detection_method`.

Success:

```text
confirmation_event
confirmation_class
stored_recall
form_removed
pagehide
```

Failure:

```text
error_class
```

## Debugging

```js
window._ecFormTrackers.wpforms.status()
```

Returns version, consent state, forms attached, the last event and the active config.

## Known limitations

- Without jQuery, AJAX success relies on the confirmation container render.
- Cross-domain redirects cannot use same-domain recall.
- Inferred successes (`form_removed`, `pagehide`) are on by default. Set `strictSuccess: true` to drop them.
- Bot submissions that pass the platform's own validation still count as success. Use `ec_fast_submit` to filter.
- Phone normalization is best-effort.

## Version

Current version: `3.3.0`. See `CHANGELOG.md`.

## License

MIT. See `LICENSE`.

## Author

Iyke Abel  
LinkedIn: https://www.linkedin.com/in/iykeabel/  
Email: abeliyke05@gmail.com
