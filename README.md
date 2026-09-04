# Claude Connector Request - intake design

A proposal to replace the current email-only process for requesting a Claude Enterprise connector with a structured Jira Service Management (JSM) request form on the AI Help (AIHELP) service desk, so every request arrives complete and is easy to triage.

This is a review package. Nothing here has been changed in Jira or Confluence.

## What is in here

| File | What it is |
|---|---|
| `connector-mockup.html` | Interactive mockup. Open it in any browser (no install). Two tabs: the **Request form** users would fill in, and the **Agent queue** showing how the resulting tickets look to reviewers. Includes a "preview of the ticket this would open" and the same fields end to end. |
| `how-to-get-soc2.html` | Requestor-facing guide: how to find and obtain a vendor's SOC 2 report (there is no central repository; a requestor with a vendor login can retrieve one ITS cannot). Linked from the form's SOC 2 field. |
| `IMPLEMENTATION-SPEC.md` | The build plan: prerequisites and who does what, the form fields, the conditional logic, the security and configuration requirements, and the proposed change to the Confluence connector page. |
| `SECURITY-REVIEW.md` | A security review of the design (15 findings, ranked) plus an edge-case catalog to test before go-live. The spec references these findings by ID (F1, F2, ...). |

## How to view

1. Download or clone this folder.
2. Double-click `connector-mockup.html` to open the mockup in your browser.
3. Read `IMPLEMENTATION-SPEC.md` for the build plan, then `SECURITY-REVIEW.md` for the risk detail.

The mockup is for approving the fields and flow. The real form would be built in JSM, so the production portal will use Atlassian's portal styling rather than this exact look.

## Status

Draft for review. Sample queue data is fictional. Next step after sign-off: an AIHELP project admin builds the request type from the spec, and an org admin handles the queue-visibility and mail-handler items called out in the prerequisites.
