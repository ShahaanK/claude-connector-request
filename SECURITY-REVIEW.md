# Security Review and Edge-Case Catalog: "Request a Claude Connector"

Design and QA review only. No mockups were changed, no Jira touched, nothing pushed.

**Artifacts reviewed**

- Mockup: `connector-mockup.html` (single-file tabbed mockup: request Form + AIHELP agent Queue, both client-side JS).
- Requirements: the current Confluence "Requesting a Claude Connector" page.

**Planned v1 implementation being assessed:** a JSM request type "Request a Claude Connector" on the AIHELP service desk (`service_desk`), built with Atlassian Forms/ProForma. Structured intake only, no custom workflow. Confluence page links to the JSM customer portal. Users authenticate via SU SSO through the portal. SOC 2 reports uploaded as attachments. A data-classification field captures Confidential (FERPA/HIPAA/PII/financial) involvement. Email fallback to aihelp@syr.edu. A custom web-form-to-Jira-REST backend was considered and rejected.

## How to read the ranking

Findings are ranked by their impact on **deployed v1 (JSM/ProForma)**, not by the order of the task brief and not by their severity as a bug in the mockup considered as a standalone web page. The mockup contains genuine client-side vulnerabilities, but most of them die with the mockup because Atlassian renders the real queue. Each finding carries an **Applies to** value so the distinction is explicit:

- `JSM v1` - carries into the deployed system.
- `Mockup + derived viewer` - a real vulnerability in the mockup, and in any custom queue viewer built from this code, but not in the JSM-rendered queue.
- `Mockup only` - a bug in the mockup as an artifact, no deployed-system analogue.
- `Process` - a people/procedure gap, not code.

---

# Deliverable 1 - Security findings (most severe first)

## Summary of findings by severity

| Severity | Count |
|---|---|
| High | 5 |
| Medium | 6 |
| Low | 4 |
| Total | 15 |

---

## F1. Confidential free-text and data-classification exposed to the entire AIHELP agent group (High) - JSM v1

**Where:** JSM v1 (agent-group scope + queue visibility). Modeled in the mockup queue at lines 386-408, which surfaces `use` (free-text use case), `requester`, `dept`, and a red-styled `Data class` column including "Confidential" (line 393).

**Failure scenario.** This intake is, by design, where users declare that FERPA/HIPAA/PII/financial data may flow through a connector. The free-text "Business use case" field invites descriptions that themselves contain Confidential specifics (for example "summarize student conduct records for..."). In JSM, every issue in a service desk is readable by every **agent** on that service desk. JSM agent licensing is frequently granted to a broad ITS population, so "who can see the queue" can silently expand to far more people than the small AI review team. The result is a standing, searchable store of Confidential-tagged descriptions readable by a group nobody scoped for this purpose. The mockup makes the exposure concrete by rendering requester name, department, and the Confidential tag together in one table.

**Mitigation.**
- Confirm and tightly scope the **AIHELP agent group** to the named AI review team only. Treat this as a checkable config item, not an assumption. If ITS agents at large hold AIHELP agent licenses, this finding is live on day one.
- Add data minimization **at the point of entry**: a form-level warning telling requesters not to paste actual Confidential data (records, identifiers, sample content) into the use-case field, only to describe the integration.
- Consider issue-security-level or a separate restricted request type for Confidential/Restricted submissions so those tickets are not visible to the full agent pool.

---

## F2. SOC 2 reports stored as ticket attachments: NDA exposure, broad visibility, indefinite retention (High) - JSM v1 / Process

**Where:** JSM v1 attachment handling and retention. In the mockup the SOC 2 status select is lines 191-205; the conditional upload (a text field standing in for the real file upload) is lines 200-204.

**Failure scenario.** SOC 2 Type II reports are typically NDA-bound vendor documents. Uploaded as Jira attachments, they inherit ticket visibility (reporter plus every AIHELP agent, per F1) and Jira's default attachment retention (effectively indefinite). This creates a contractual exposure (redistribution of an NDA document beyond its permitted audience) layered on top of the security exposure. Attachments also persist after the ticket is closed and after the requester leaves SU.

**Mitigation.**
- Default the intake to a **link to the vendor trust portal** rather than a file upload. The source requirements doc itself notes vendors "publish this on a trust page," so the link path is the common case. Make file upload the exception path, used only when no trust portal exists.
- If files are accepted, define a **retention rule** (delete attachment N days after decision) and restrict attachment download to the scoped agent group (ties to F1).
- Record where the NDA report lives rather than storing a second copy in Jira where practical.

---

## F3. Email fallback (aihelp@syr.edu) bypasses the structured gate and the identity controls (High) - JSM v1 / Process

**Where:** JSM v1 mail handler and the process. Referenced in the mockup at line 228 and the requirements doc (the current process is email-only).

**Failure scenario.** The email channel is the weakest link precisely because it is unstructured. A request arriving by email:
- Skips every required field (no SOC 2 status, no data classification, no contract confirmation captured), so the "requests are held until SOC 2 is provided" gate is silently bypassed at intake.
- Sets the **reporter from the `From` header**, which is spoofable, so the SSO-established identity that the portal provides is lost.
- If the AIHELP mail handler accepts **external senders** and **auto-creates customer accounts** for unknown addresses (a common JSM default), the queue is reachable by anyone on the internet, enabling spam, phishing lures aimed at reviewers, and email-to-ticket field injection via crafted subject/body.
- Users may attach SOC 2 or Confidential material to plaintext email, a less controlled path than the portal.

**Mitigation.**
- Verify the mail handler's sender policy: restrict to authenticated SU senders, disable auto-creation of customer accounts for unknown external addresses, or disable email create entirely and use the address only for questions.
- Auto-reply to email submissions with a link to the portal request and treat email tickets as "unverified reporter / incomplete intake" requiring the requester to resubmit via the portal for anything touching Confidential data.
- Do not let an email-created ticket satisfy the SOC 2 gate without agent confirmation.

---

## F4. Stored XSS in the queue via unescaped innerHTML across every ticket field (High) - Mockup + derived viewer

**Where:** Mockup queue rendering. The `kv()` helper builds HTML by string concatenation (line 377). Row HTML is concatenated from `t.key`, `t.connector`, `t.dept`, `t.requester`, `t.soc2`, `t.data`, `t.status`, `t.created` and assigned via `innerHTML` (lines 388-395). The detail row concatenates `t.use`, `t.users`, `t.contract`, and more into `innerHTML` (lines 397-405).

**Failure scenario.** In a real build these values are attacker-controlled: any SU user can put `<img src=x onerror=alert(document.cookie)>` (or a session-stealing payload) into the connector name or use-case field. When a reviewer opens the queue, the script runs in the reviewer's session (stored XSS). The form path proves the fix is known: the form preview uses `dd.textContent=r[1]` (line 358) and is safe. The queue never uses that pattern; there is no `escapeHtml` helper anywhere in the file.

**Scope honesty.** In deployed v1 the queue is rendered by Atlassian, which escapes field values, so this specific cluster does **not** carry into JSM. It is a real, High-severity vulnerability in the mockup as a code artifact and in any custom queue viewer someone builds by copying this code. F5 is the one member of this cluster that does reach v1.

**Mitigation.** Apply the pattern already used at line 358: route all dynamic values through `textContent` (or an `escapeHtml` helper) instead of `innerHTML` string concatenation. Do not let this rendering pattern be copied into any production viewer.

---

## F5. Vendor docs URL rendered as a link with no scheme validation; form validation gives false assurance (High) - Mockup + derived viewer, with a checkable JSM v1 variant

**Where:** Mockup, line 401: `kv('Vendor docs URL','<a href="'+t.docs+'" target="_blank" rel="noopener">'+t.docs+'</a>')`. Form validation at line 335: `/^https?:\/\//i`.

**Failure scenario (mockup / derived viewer).** The docs URL becomes an `href` taken directly from user input with no sanitization. Two chained problems:
1. **Attribute breakout.** The form regex `/^https?:\/\//i` correctly rejects `javascript:`, which makes the residual risk non-obvious. But `https://example.com" onmouseover="alert(1)` **passes** that regex and then breaks out of the `href` attribute at line 401, injecting an event handler. `https://` alone (empty host) also passes the regex.
2. **Scheme risk if validation is absent.** The queue applies no validation at all to `t.docs`; a viewer built from this code that renders a stored URL without the form-side check would also be exposed to `javascript:` and `data:` scheme links.

**Checked-and-clean (reported as negatives, not manufactured findings).**
- `rel="noopener"` **is present** at line 401, so reverse-tabnabbing via `window.opener` is already mitigated. (`rel="noreferrer"` is not set, but `noopener` is the security-relevant one.)
- **Open redirect: not applicable.** There is no redirect endpoint; the link is a direct external anchor. Nothing here bounces through an SU-controlled URL.

**JSM v1 variant (checkable, not a claim about Atlassian's sanitizer).** In v1 the risk reduces to a ProForma field-type decision. A URL-typed field auto-renders as a clickable link in the agent view; a short-text field renders as inert text. Verify (a) which field type the docs URL uses, (b) whether the agent view auto-links it, and (c) whether the rendered link preserves the submitted scheme. Prefer a short-text display-only field, or a URL field with an allowlist regex (`^https://` plus a host check), so a reviewer is never one click from an attacker-chosen scheme.

**Mitigation.** Escape the text and validate the scheme with an allowlist (`https:` only) before constructing any link; in JSM, choose the field type deliberately per the check above.

---

## F6. Formatting / markup injection when field values flow into the Jira summary, description, and labels (Medium) - JSM v1

**Where:** JSM v1 field mapping. Modeled in the mockup at line 350 (`'Connector request: '+val('cname')` becomes the summary) and lines 342-356 (fields mapped into ticket rows).

**Failure scenario.**
- **Summary.** Connector name flows verbatim into the summary. Newlines and control characters in the name can break summary rendering and downstream automation/notifications that parse the summary. Overlong names (no length cap, see E-B1) truncate unpredictably.
- **Description / rich fields.** If ProForma writes answers into a wiki-markup-rendered description (older Jira config) rather than ADF, a user can inject `{code}`, `{color}`, panel macros, or `[link|javascript:...]` wiki syntax. With ADF the risk is much lower because ADF is structured, not re-parsed from text; confirm the rendering path.
- **Labels.** If connector name is auto-mapped to a label, spaces and special characters split or corrupt the label, polluting label-based filters and automation.
- **Control / bidi characters.** Right-to-left override and zero-width characters in any field can disguise a value shown to a reviewer (for example making a hostile URL read as a benign one).

**Mitigation.** Prefer ADF-rendered fields over wiki-markup description. Strip control and bidi characters and cap length on the summary-bound field. Do not derive labels directly from free text; use a controlled connector picklist where possible. Sanitize at field-mapping time, not just at display.

---

## F7. Portal open to unauthenticated or overly broad submission; raise-on-behalf-of weakens attestation (Medium) - JSM v1

**Where:** JSM v1 portal permissions. The mockup form has no auth (by nature) and shows the reporter as "(the signed-in SU user)" at line 348.

**Failure scenario.** JSM customer portals can be configured "anyone can raise" (anonymous) or "any logged-in customer," and can auto-create customer accounts. If misconfigured, unauthenticated or non-SU parties can file connector requests, defeating the SSO assumption. Separately, **raise-on-behalf-of** lets an agent set an arbitrary reporter; even though SSO establishes portal identity, the audit trail of *who actually attested to the declared data classification* is weakened when the reporter is set by someone else.

**Mitigation.** Restrict the portal to authenticated SU SSO users only; disable anonymous raise and external auto-provisioning. Keep raise-on-behalf-of restricted and, when used, record the acting agent in a field so the attestation trail is intact.

---

## F8. No rate limiting or abuse controls on submission (Medium) - JSM v1 / Process

**Where:** JSM v1 intake. Not represented in the mockup (the mockup only previews).

**Failure scenario.** With the portal open to all SU users and no throttle, a compromised SU account or a bot can flood the queue with hundreds of requests (each potentially carrying malicious payloads per F4/F6), burying legitimate requests and exhausting reviewer attention. The email channel (F3) is an even easier flood vector.

**Mitigation.** Add JSM automation to detect burst submissions from one reporter, deduplicate on connector name (ties to E-W1), and rate-limit or quarantine. Monitor email-created tickets separately.

---

## F9. Malicious file upload masquerading as the "SOC 2 report" (Medium) - JSM v1

**Where:** JSM v1 attachment path. Mockup upload stand-in at lines 200-204.

**Failure scenario.** The one attachment this process actively solicits is a document from an external party. A requester (or a spoofed email sender, F3) can upload malware, a macro-laden Office document, a zip bomb, a password-protected archive that evades scanning, or an HTML/SVG file containing script. Jira has historically served certain attachment types (SVG, HTML) inline; a reviewer previewing such a file in-browser could trigger script in the Jira origin. A reviewer downloading and opening a malicious "SOC 2.pdf" is a straightforward endpoint-compromise path.

**Mitigation.** Confirm AV scanning on Jira attachments (Atlassian Cloud scans, but verify for this instance). Enforce type and size limits; prefer accepting only PDF and only via the trust-portal-link path (F2). Ensure attachments are served with `Content-Disposition: attachment` (not inline) so SVG/HTML cannot execute in the Jira origin. Advise reviewers to open vendor documents in a sandboxed viewer.

---

## F10. Requester identity spoofing via the email channel (Medium) - JSM v1

**Where:** JSM v1 mail handler. See F3.

**Failure scenario.** Distinct from F3's structural bypass: the specific integrity issue is that an email-created ticket takes its reporter from a forgeable `From` header. An attacker can file a request that appears to come from a trusted department head, lending false weight to a risky connector request during review.

**Mitigation.** As F3: prefer portal-only intake for anything consequential; flag email-created tickets as unverified reporter; require portal (SSO) confirmation before a connector is approved.

---

## F11. Rejected custom-backend option: risks that justify avoiding it (Medium) - Process (context)

**Where:** The rejected web-form-to-Jira-REST design. Included briefly per the brief.

**Why it was rightly avoided.** A custom backend would introduce, all of which JSM/ProForma avoids:
- **Service-account token at rest.** A stored Jira API token with ticket-create (and likely read) scope becomes a high-value secret needing secure storage and rotation; if leaked it enables queue-wide spam and data exfiltration.
- **CSRF.** A custom form POST endpoint must implement CSRF protection or attackers can forge submissions from a victim's authenticated browser.
- **Unauthenticated proxy risk.** The backend effectively becomes a create-ticket proxy; without its own auth it is an open abuse vector.
- **All input validation and injection defense move to you** (F4/F5/F6) instead of inheriting Atlassian's handling.
- **SSRF.** If the backend ever fetches the submitted docs URL (for validation or preview), it becomes an SSRF surface into SU's network.

**Conclusion.** Staying on JSM/ProForma removes the token-storage, CSRF, and SSRF surfaces entirely and inherits Atlassian's authn and attachment scanning. The finding is informational: it validates the chosen path.

---

## F12. `socClass()` fall-through renders unknown SOC 2 values as the most alarming state (Low) - Mockup + derived viewer

**Where:** Mockup, line 366: `function socClass(v){ return v==="Attached"?"soc-attached":(v==="Can obtain"?"soc-can":"soc-none"); }`. Consumed at line 392.

**Failure scenario.** Any value that is not exactly "Attached" or "Can obtain" falls through to `soc-none` (red, "vendor lacks one" styling). The sample data already exercises this: AIHELP-4152 has `soc2:"Unsure"` (line 375), a value the form's SOC 2 select (lines 193-198) cannot even produce, and it renders red. A future added status option would silently render to a reviewer as the worst case, biasing an approval decision. This is a mis-render with decision consequences, not a cosmetic issue.

**Mitigation.** Map states explicitly and render an unrecognized value neutrally (grey) with the literal text, never as "vendor lacks one." Keep the form's option set and the queue's known states in sync.

---

## F13. `soc2file` accepts a free-text link: second unvalidated-URL and potential SSRF surface (Low) - Mockup + JSM v1 consideration

**Where:** Mockup, line 203: `<input type="text" id="soc2file" placeholder="filename.pdf or a link to the vendor trust page" />`.

**Failure scenario.** This field explicitly accepts "a link," so it is a second user-supplied URL alongside the docs URL, with the same scheme/breakout risk as F5 if ever rendered as a link, and no validation at all here. If anything server-side ever fetches this link (for example to auto-verify a trust page), it becomes an SSRF surface.

**Mitigation.** Validate as `https:`-only, escape on display, and never server-side-fetch it without an allowlist. In JSM, make the SOC 2 evidence field a URL type with the same allowlist regex as F5.

---

## F14. No length limits on any input; oversized submissions (Low) - Mockup + JSM v1

**Where:** Mockup, all text inputs and the `usecase` textarea (line 169); no `maxlength` anywhere. Carries to JSM if field limits are not set.

**Failure scenario.** A user can paste megabytes into the use-case textarea or connector name. In the mockup this bloats the preview and (via F4) amplifies any payload. In JSM, unbounded free text stresses summary generation (F6), notifications, and search indexing.

**Mitigation.** Set `maxlength` on the mockup and ProForma character limits in v1 (for example connector name 100, use case 2000).

---

## F15. Client-side-only validation (Low, expected for a mockup) - Mockup

**Where:** Mockup, the whole `submit` handler (lines 329-360); `novalidate` on the form (line 147).

**Failure scenario.** All required-field and URL checks are client-side and trivially bypassed. This is acceptable and expected for a mockup, and in v1 ProForma enforces required fields server-side. Noted only so it is not mistaken for a control. The one carry-over lesson is that the URL regex is weak (F5) and should not be treated as sanitization anywhere.

**Mitigation.** Rely on ProForma server-side required/validation in v1; do not port the mockup's regex as a security control.

---

# Deliverable 2 - Edge-case catalog

Each row lists the case, expected handling, and (where the mockup mishandles it) the file line. File is `connector-mockup.html` unless noted.

## Input validation

| Case | Expected handling | Mockup status |
|---|---|---|
| Required field left blank | Block submit, focus first invalid, announce error | Handled (lines 329-340), but errors not announced to assistive tech (see Accessibility) |
| Docs URL = `https://` with no host | Reject | Mishandled: passes `/^https?:\/\//i` (line 335) |
| Docs URL with `javascript:`/`data:` scheme | Reject | Form rejects (line 335); queue does not validate (line 401) |
| Docs URL with trailing `" onmouseover=...` | Reject / escape | Passes form regex, breaks out on render (F5, line 401) |
| Whitespace-only connector name / use case | Reject | Handled via `.trim()` (line 332) |
| SOC 2 status left as default "Select one..." | Reject | Handled (line 339, empty value) |
| Contract radio unselected | Reject | Handled (lines 337-338) |

## Boundary and size limits

| Case | Expected handling | Mockup status |
|---|---|---|
| Connector name of 5,000 chars | Cap length, truncate summary safely | No `maxlength` (line 153); breaks summary (F14/F6) |
| Use case of several MB pasted | Reject or cap | No limit (line 169) |
| Extremely long docs URL | Cap / validate | No limit |
| Many rapid submissions | Rate-limit / dedupe | N/A in mockup; F8 in v1 |

## Malicious input

| Case | Expected handling | Mockup status |
|---|---|---|
| `<img src=x onerror=...>` in any field | Escape on render | Mishandled in queue (F4, lines 388-405) |
| Attribute-breakout payload in docs URL | Escape + scheme allowlist | Mishandled (F5, line 401) |
| Wiki-markup macros in use case | ADF field or strip | v1 field-mapping concern (F6) |
| Control / zero-width / RTL-override chars | Strip before store and display | Not handled (F6) |
| Homoglyph / lookalike host in docs URL | Surface real host to reviewer | Not handled; reviewer sees raw text |

## Data semantics

| Case | Expected handling | Mockup status |
|---|---|---|
| SOC 2 = "Vendor lacks one" | Warn inline; per policy near-certainly not approvable; offer local-MCP alternative from the source doc; do not silently create a ticket destined to be declined | Mishandled: submits with no warning (lines 197, 329-360) |
| Data class = "Confidential" or "Restricted" + no SOC 2 | Hard deflect at the form with the policy and the local-MCP alternative | Mishandled: "Restricted" is a plain option (line 214), no gate |
| Contract = "Unsure" | Accept, flag for follow-up | Acceptable |
| SOC 2 = "Unsure" | Form cannot produce this value (lines 193-198) yet queue uses it (line 375) and renders it red | Inconsistency + mis-render (F12) |
| SOC 2 = "Attached" then changed to another status | Clear stale attachment reference | Preview reads current value correctly (line 354), but stale `soc2file` value persists in DOM (see Mockup bugs) |

**Highest-value data-semantics edge case:** the form allows Restricted (line 214) and allows submitting with SOC 2 = "Vendor lacks one" (line 197) with no warning, even though the source requirements make Confidential/Restricted-plus-no-SOC-2 near-certainly non-approvable. Expected handling is inline deflection at the form (show the policy, offer the Claude Desktop local-MCP alternative the source doc describes) rather than creating a ticket destined to be declined. This is simultaneously an edge case, a process gap, and a ProForma-expressible mitigation (conditional logic on the two fields).

## Workflow and process

| Case | Expected handling | Mockup status |
|---|---|---|
| Two users request the same connector (Box twice) | Detect duplicate, link tickets | No dedupe; N/A in mockup, F8 in v1 |
| Connector already enabled org-wide (M365, Atlassian) | Deflect before ticket: show the already-enabled list | Not handled; nothing stops a request for an enabled connector |
| Connector already requested and pending | Surface the existing ticket | Not handled |
| Requester leaves SU mid-review | Reassign ownership; require a current SU sponsor | Not handled (process); attachments and ticket orphan |
| Approved connector's sponsor later leaves SU | Periodic re-attestation of ownership | Process gap |
| Request declined, user resubmits unchanged | Detect and reference prior decision | Not handled |

## Internationalization and encoding

| Case | Expected handling | Mockup status |
|---|---|---|
| Non-ASCII connector name (accents, CJK) | Store and display UTF-8 correctly | Page is UTF-8 (line 4); untested in summary mapping |
| Emoji in free text | Preserve, do not break summary | Untested |
| RTL script or bidi-override in fields | Neutralize bidi controls (F6) | Not handled |
| Very long single-token input (no spaces) | Wrap / cap | No cap (F14) |

## Attachments

| Case | Expected handling | Mockup status |
|---|---|---|
| Wrong file uploaded (not a SOC 2) | Reviewer verifies; no automated trust | Process |
| Malware / macro doc / SVG-with-script as SOC 2 | AV scan; serve as attachment not inline; type/size limit | v1 concern (F9); mockup upload is a text stand-in (lines 200-204) |
| Password-protected archive evading scan | Reject or hold | v1 (F9) |
| Zip bomb / oversized file | Size limit | v1 (F9/F14) |
| Link to a trust page requiring auth | Reviewer cannot open; request accessible copy | Process; `soc2file` accepts a link (line 203, F13) |
| Expired or wrong-vendor SOC 2 report | Reviewer validates scope and date | Process |

## Accessibility

| Case | Expected handling | Mockup status |
|---|---|---|
| Validation errors announced to screen readers | `aria-invalid` + `aria-describedby` linking input to `.err` | Mishandled: only a `.invalid` class is toggled (line 332); errors are visual-only (lines 71-73) |
| Required fields conveyed non-visually | `required` / `aria-required` | Only a visual asterisk (line 151); no ARIA |
| Expandable ticket rows usable by keyboard | Focusable, `role`/`aria-expanded`, Enter/Space handler | Mishandled: click-only handler, row not focusable, no `aria-expanded` (line 406) |
| Tablist roving focus | `tabindex="-1"` on unselected tabs | Partial: arrow keys work (lines 305-311) but no roving tabindex |
| Live preview announced | `aria-live` region | Handled (line 231) |
| Decorative caret hidden from AT | `aria-hidden` | Caret is decorative text (line 388); acceptable but not marked |

## Mockup-specific bugs

| Bug | Detail | Line(s) |
|---|---|---|
| Reset does not restore conditional/preview state | `type="reset"` fires no `change` event, so the SOC 2 upload block stays visible, the stale `soc2file` value persists in the DOM, `.invalid` classes remain, and the preview stays displayed after a reset | 223 (button); 322-324 (conditional); 357-359 (preview) |
| `socClass()` fall-through renders unknown values red | Any value other than "Attached"/"Can obtain" renders as "vendor lacks one" red styling; `Unsure` sample data triggers it | 366, 392, 375 |
| Form option set diverges from queue data | SOC 2 select cannot produce "Unsure", but the queue sample data uses it | 193-198 vs 375 |
| Queue XSS via innerHTML | All ticket fields concatenated into `innerHTML` | 377, 388-395, 397-405 |
| Docs URL rendered as unvalidated link | `href` from user input, no scheme allowlist | 401 |
| URL regex accepts empty host | `https://` alone passes | 335 |
| "Submit request" only previews | Button label implies submission; mitigated by the mock-flag banner (line 118) | 222, 359 |

---

# Highest-value items to act on

1. **Scope the AIHELP agent group and add a data-minimization warning** (F1) - the single highest-stakes item, since this system is itself about handling Confidential data.
2. **Default SOC 2 evidence to a trust-portal link, not a file upload; set retention** (F2) - removes NDA and malware exposure at once (F9).
3. **Lock down the email fallback** (F3/F10) - restrict senders, disable external auto-provisioning, treat email tickets as unverified/incomplete.
4. **Deflect non-approvable requests at the form** (highest-value edge case) - Confidential/Restricted + no SOC 2 should show policy and the local-MCP alternative rather than create a doomed ticket.
5. **Do not port the mockup's rendering pattern** - if any custom viewer is ever built, escape all values (apply the line-358 `textContent` pattern) and allowlist the docs URL scheme (F4/F5).
