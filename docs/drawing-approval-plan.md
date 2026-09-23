# Drawing Approval — Sign Against Existing Record PDFs — Plan

> **Branch:** `feat/drawing-pdf-merge` (off `main` @ v3.57.0, `62183ca`)
> **Status:** DRAFT / RFC — design pending maintainer agreement. Successor to closed PR #197 (`exp/document-markup`, closed as dormant 2026-08-10).
> **Related:** #405 (template-defined supplemental PDFs — same concept, generation only), #407 (guided page fails open to the server re-render), #373 / #404 (sender UI work in flight).
> **How to use this doc:** decisions in §4 are _proposed_ until the RFC is agreed. Work milestones in order; tick boxes and leave a one-line note on anything that changes.

---

## 1. Use case

A record carries one or more **existing PDFs** — typically engineering drawings (A3, often landscape, vector CAD exports or scans, **0.1–5 MB each, several per record**). We want to send them to an **external customer via the normal emailed signing link** so the customer can **approve (sign) or decline (with reason)**. A later milestone (M2) lets the customer **redline** the drawings.

Constraints from the use case:

- Signers are **guests** on the Portwood Site — every byte they see or return goes through token-gated Apex.
- The send must work from **Flow** as well as the Signature Sender LWC (the process will be automated).
- The approval must identify **exactly which drawing revision** was approved.

## 2. What exists today (v3.57.0)

| Capability                                 | State                                                                                                                                                                                                                                                                                                                                                                             |
| ------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Sign a bare, pre-existing PDF              | **Removed in v3.18, deliberately.** `createRequestFromContentVersion` throws (`DocGenSignatureSenderController.cls:882-895`); `DocGenSignaturePdfFlowAction` ignores `contentVersionId`. This plan does **not** revive it — a signature-tagged template is always required.                                                                                                       |
| Generate a template + append record PDFs   | **Runner only**, "Combine with existing PDFs on this record" (`docGenRunner.js` `_generateMergedPdf` ~`:1291-1320`). Template always first; download or save; not Flow; not signing.                                                                                                                                                                                              |
| Include record PDFs in a signature request | **Not possible.** Every send path renders the signing document from a template (`createGuidedPdfSignatureRequest`, `DocGenSignatureSenderController.cls:1449`; viewing PDF `Blob.toPdf` at `:1606`, stored as `Source_Document_Id__c` at `:1619`).                                                                                                                                |
| Signature placement                        | Text anchors — `@@SIG-n@@` sentinels found in the PDF.js text layer on **any page** (`DocGenSignaturePdf.page` `locateAnchors` `:1125`, `hitToPdfRect` `:1203`). Placements store no page number. **Extra pages don't break placement.**                                                                                                                                          |
| Finalize (normal path)                     | Client-side pdf-lib composite onto the stored source bytes (`compositeAndFinalize` `:2150`) → `saveCompositedSignedPdf` (`DocGenSignatureController.cls:2692`) stores it as-is. Final hash = SHA-256 of the uploaded bytes.                                                                                                                                                       |
| Finalize (fallback)                        | If the signer has no session marks or the source didn't load, the page calls `saveSignature` (`DocGenSignaturePdf.page` ~`:3173`) → `TemplateSignaturePdfQueueable` **re-renders from the template** (`DocGenSignatureService.cls:1486`, `:1791`). Anything not from the template is dropped. See #407.                                                                           |
| Decline                                    | `declineSignature` (`DocGenSignatureController.cls:1826`) — reason text, status Declined, **no document produced**. Can be hidden org-wide (`Signature_Hide_Decline__c`) or per template (`Hide_Signer_Decline__c`); enforced server-side (#367).                                                                                                                                 |
| Signer form fields                         | text / number / date / checkbox / picklist, written back to the record at completion (`DocGenFieldWritebackService`). Future vehicle for richer outcomes (e.g. "approved as noted").                                                                                                                                                                                              |
| PDF libraries                              | `pdflib` static resource (pdf-lib 1.17.1) — already loaded by the signing page (`DocGenSignaturePdf.page:23`). `pdfjs4` (PDF.js 4.7.76) for rendering. The Runner's `docGenPdfMerger.js` is a regex byte-scanner (no object streams, no inherited `/Rotate`/`/CropBox`) — **not suitable for CAD/scanner PDFs**. `DocGenPdfMerger.cls` is Portwood-output-only by its own header. |

## 3. Constraints that shape the design

### 3.1 Size — the dominant constraint

Guests can't hit file URLs; every file moves through a synchronous Apex call (6 MB heap). Estimates below are **unmeasured** — M0 measures them.

| Path                                                                | Mechanism                     | Est. heap per call | Est. practical ceiling |
| ------------------------------------------------------------------- | ----------------------------- | ------------------ | ---------------------- |
| Guest download (`getSourcePdfBase64`, `:2397`)                      | whole file → base64, one call | ~2.3× file         | ~2–2.5 MB per file     |
| Guest upload (`saveCompositedSignedPdf`, `:2692`)                   | whole file ← base64, one call | ~3.3× file         | ~1.5–1.8 MB per file   |
| Internal fetch (`DocGenController.getContentVersionBase64`, `:710`) | Aura, base64                  | 4 MB response cap  | ~3 MB per file         |
| Async Apex (Queueable)                                              | —                             | 12 MB heap         | ~5 MB per file (est.)  |

Apex cannot read part of a `ContentVersion` — `VersionData` always loads whole — so "just chunk the existing endpoint" doesn't lower the peak. With several 5 MB drawings per request, **a single combined signed PDF (10–20 MB) cannot be uploaded by a guest through any Apex path.** This drives decisions D1 and D3.

### 3.2 Other constraints

- **No browser at send time for Flow.** Apex can't merge third-party PDFs, so the merge can't happen at send (which is what #197 did — in the sender's browser). It has to happen in the **signer's** browser.
- **Fallback re-render** (§2) silently loses non-template pages; the page currently fails open (#407).
- **Integrity:** nothing hashes the viewing PDF at send today. An approval must bind the exact drawing revisions.
- **Decline can be hidden** (#367) — an approval request without Decline is only half a sign-off.
- **Managed package rules** (`.claude/skills/managed-package-rules`): new Flow inputs must be `global`; API names are forever; guest `SYSTEM_MODE` reads must be token-keyed; verify in a namespaced org.

## 4. Proposed design decisions (pending RFC)

| #   | Decision                      | Proposal                                                                                                                                                                                                                                                                                                                                                                           |
| --- | ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| D1  | Signed output                 | **Approval record**: the template pages + signatures + certificate, where the certificate **lists every attached document** (title, version, SHA-256, page range as viewed). The drawings stay on the record untouched. Upload stays small regardless of drawing count/size. A single combined "approved pack" is an optional later extension, only viable under the size ceiling. |
| D2  | Where the merge happens       | **At view time, in the signer's browser**, with pdf-lib `copyPages` (already loaded on the page). The merged document is for **display**; signatures are stamped onto the **template-only** bytes (anchors mapped back via the page provenance map).                                                                                                                               |
| D3  | Transport                     | **Per file**, never whole-pack. Mechanism chosen in M0 (§5): T2 chunked remote actions or T3 guest Apex REST. Send-time size cap set to the measured ceiling.                                                                                                                                                                                                                      |
| D4  | Identity of what was approved | Pin each attachment by **ContentVersion Id** (`VersionData` is immutable per version) + **SHA-256** computed server-side. Browser re-checks the hash before display (defence in depth).                                                                                                                                                                                            |
| D5  | Entry points                  | Signature Sender LWC **and** the Flow action "Portwood: Create Signature Request" take the same thing: an ordered list of file Ids + position (before/after template).                                                                                                                                                                                                             |
| D6  | Failure mode                  | **Fail closed.** If any attachment can't be fetched, verified or merged, signing is blocked with an actionable message. Both certificate builders (client and server) list the attachments, so even the fallback path produces a record that binds them.                                                                                                                           |
| D7  | Scope                         | Single-template requests only in M1. No markup, no packets, no generation-only supplemental PDFs (that's #405 — share the data model and naming with it).                                                                                                                                                                                                                          |

### 4.1 Data model (proposed — naming to agree with #405)

New child object, working name `DocGen_Signature_Attachment__c` (master-detail → `DocGen_Signature_Request__c`):

| Field                    | Type      | Notes                                 |
| ------------------------ | --------- | ------------------------------------- |
| `Content_Version_Id__c`  | Text(18)  | Pinned version                        |
| `Content_Document_Id__c` | Text(18)  | For display / linking                 |
| `Title__c`               | Text(255) | Snapshot of file title at send        |
| `Version_Number__c`      | Text(20)  | Snapshot of `VersionNumber`           |
| `File_Size__c`           | Number    | Bytes                                 |
| `Sha256__c`              | Text(64)  | Server-computed                       |
| `Position__c`            | Picklist  | `Before` / `After` the template pages |
| `Sort_Order__c`          | Number    |                                       |

A child object (rather than a JSON field on the request) gives FLS, queryability, per-attachment status, and room for M2 (per-drawing markup output) without reshaping.

### 4.2 Flow of a request

1. **Send** (LWC or Flow) → `createGuidedPdfSignatureRequest` (options/overload TBD) validates each file in `USER_MODE`: sender can read it, it's a PDF, it's **linked to the related record**, size ≤ cap. Resolves ContentDocument → latest ContentVersion and pins it, computes SHA-256, inserts attachment rows. Everything else (template render, placements, signers, emails, expiry, verification) is unchanged — single-phase send, no Draft orphans.
2. **Signer opens the link** → init response includes an attachment **manifest** (index, title, version, size, position, sha256 — no record Ids). The page fetches the template viewing PDF (existing) and each attachment by **token + index** (server resolves the CV from the request — never client-supplied), verifies SHA-256, merges with pdf-lib, builds the provenance map, renders, locates anchors (on template pages) as today.
3. **Approve** → `compositeAndFinalize` stamps onto the template-only bytes; `addCertificatePage` lists the attachments; `saveCompositedSignedPdf` stores the approval record on the related record (unchanged).
4. **Decline** → existing `declineSignature` with reason (unchanged).

## 5. M0 — Spike (do first; ~1 day)

Goal: replace the estimates in §3.1 with measurements and de-risk pdf-lib on real drawings.

- [ ] **Samples:** real drawings — CAD vector export + scan; ~0.1 / 1 / 3 / 5 MB; A3 landscape; one using `/Rotate`; one permissions-restricted (encrypted) if the CAD export does that.
- [ ] **Merge:** in the VF signing-page context, pdf-lib `copyPages` of template viewing PDF + samples. Check: rotation/crop preserved; template `@@SIG-n@@` anchors still found by PDF.js; behaviour on encrypted input (pdf-lib can't decrypt — must fail cleanly); time + memory on desktop and a mid-range phone.
- [ ] **Viewer:** PDF.js with 10+ A3 pages — the page currently renders and keeps every page canvas (`DocGenSignaturePdf.page` ~`:963-988`); decide whether lazy rendering is needed.
- [ ] **Transport ceilings** in a **namespaced** scratch org with a Site, files 1–6 MB:
    - T1 — existing base64 `@RemoteAction` (the `getSourcePdfBase64` shape).
    - T2 — chunked: a Queueable splits each attachment into ≤ ~1.5 MB internal chunk files at send; guest fetches chunk _k_ by token + index; browser concatenates. Measure the split's own ceiling in async.
    - T3 — guest `@RestResource` on the Site streaming raw bytes (`responseBody = VersionData`, ~1× heap). New surface type for the package (none exist today) → security review + maintainer call.
- [ ] **Output:** a table of measured limits → pick T2 or T3, set the send-time cap, confirm D1.

## 6. M1 — Attach record PDFs to a signature request (one PR, after RFC agreement)

### Backend

- [ ] `DocGen_Signature_Attachment__c` + fields + permission sets (Admin/User as appropriate; **no guest object access** — token-keyed `SYSTEM_MODE` only).
- [ ] Send: attachments on the canonical `createGuidedPdfSignatureRequest` (options DTO vs another overload — agree in RFC); validation per §4.2; SHA-256 (sync under cap, else async).
- [ ] Flow: `DocGenSignatureFlowAction.Request` gains `global` `@InvocableVariable`s `attachedDocumentIds` (`List<String>`, CV or CD Ids) and `attachedDocumentPosition` (`Before` default). Verify in a namespaced org.
- [ ] Guest: manifest on the init response; attachment fetch endpoint(s) per M0 (T2/T3), gated like `getSourcePdfBase64` — token format, `assertSignerReadable`, expiry, terminal state, PIN-verified when required.
- [ ] Server-path certificate (`DocGenSignatureService` verification block) lists attachments (D6).
- [ ] Coordinate the fail-closed guard with #407.

### Signing page (`DocGenSignaturePdf.page`)

- [ ] Fetch attachments (parallel), verify SHA-256 (WebCrypto), merge with pdf-lib in manifest order, build provenance map `{mergedPage → attachmentIndex | 'template', sourcePage, rotation, cropBox}`.
- [ ] Anchor hits mapped to template-only page indices; `compositeAndFinalize` stamps onto template-only bytes.
- [ ] `addCertificatePage` lists attachments (title, version, SHA-256, page range as viewed).
- [ ] Fail closed on any attachment error (no silent fallback).
- [ ] Lazy page rendering if M0 says so; simple "Drawing 1 of N" navigation (nice-to-have).

### Sender LWC (`docGenSignatureSender`)

- [ ] Record-PDF picker (reuse `DocGenController.getRecordPdfs`, `:5663`; add size), order + before/after, size warnings. Keep the change small — #373 is reworking this component.
- [ ] Warn when Decline is hidden on the template/org.

### Tests & gates

- [ ] Apex: send validation (no access, not a PDF, not linked to record, over cap, deleted file); guest IDOR (token A cannot read request B's attachments), index out of range, expired, declined/terminal, PIN not verified; Flow action; certificate content.
- [ ] Content-correctness (signing has none today — see repo `CLAUDE.md` "Subsystem caution"): an end-to-end check that the stored approval PDF's certificate lists every attachment with the right SHA-256, and contains no unresolved `@@SIG-` token (ties to #407).
- [ ] Adversarial guest-security review of the new endpoint(s).
- [ ] `npm run qa`; namespaced pre-flight org (`RunLocalTests`, no perm set assigned); prettier; Code Analyzer.
- [ ] UserGuide section + CHANGELOG entry.

## 7. Guardrails so M1 doesn't box in M2 (markup / redline)

1. **pdf-lib only** for merging and flattening — never the regex merger. M2 needs `/Rotate`/`/CropBox` handling and vector drawing.
2. **Provenance map is first-class.** M2 marks are keyed to _(attachment, source page)_ in unrotated page space, not to merged-page indices.
3. **Per-file transport both ways.** M2 returns a marked-up copy **per drawing**; M1's transport choice must also work for upload of a file of roughly drawing size.
4. **Vector markup, not raster.** Raster overlays at A3 resolution inflate files past the per-file ceiling (the limit #197 hit).
5. **Pinned CV + SHA-256** lets a marked-up return prove which revision was marked.
6. **Decline produces no document today.** M2's "decline with markup" needs one — don't bake in "decline = no output".
7. `hitToPdfRect` ignores `/Rotate` and `/CropBox` — harmless for M1 (anchors are on template pages), must be fixed for M2.
8. Markup must not write audit rows shaped like signer audits — the certificate maps audits by `Signer__c` and completion back-fills every audit row's hash.

## 8. Open questions for the maintainer

1. **Output (D1):** approval record binding attachments by hash — or do you want a single combined signed PDF (only viable when the total is under the ceiling)?
2. **Transport (D3):** chunked remote actions (T2) vs a guest Apex REST endpoint (T3)?
3. **Data model:** child object vs JSON field; one model shared with #405's supplemental PDFs? API naming (frozen forever).
4. **API shape:** options DTO for `createGuidedPdfSignatureRequest` vs another overload?
5. **Linkage rule:** require attachments to be linked to the related record (proposed: yes)?
6. **Decline:** block or warn when Decline is hidden on an attachment-bearing request?
7. **#407:** land fail-closed separately first, or as part of M1?

## 9. Out of scope for M1

Markup/redline (M2, separate PR + design); richer outcomes such as "approved as noted" (possible later via a signer picklist form field); packets with attachments; combined-pack output; generation-only supplemental PDFs for Runner/Flow (#405).
