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

Guests can't hit file URLs; every file moves through a token-gated Visualforce remote action as base64. **Measured in M0** (§5.1) as an anonymous guest on the real signing page, calling the package's real endpoints:

| Path                                              | Works up to        | Fails at | Binding limit                                                     |
| ------------------------------------------------- | ------------------ | -------- | ----------------------------------------------------------------- |
| Guest download (`getSourcePdfBase64`, `:2397`)    | **5.5 MB** / file  | 5.75 MB  | VF remoting **response** cap — "exceeded maximum of 15 MB"        |
| Guest upload (`saveCompositedSignedPdf`, `:2692`) | **2.95 MB** / call | 3.0 MB   | VF remoting **request** cap (~4 MiB of base64) — "Input too long" |

Apex heap was **not** the binding limit on either path (an earlier ~2–2.5 MB heap estimate was wrong). Consequences:

- **Download is fine per drawing** — the existing remote-action pattern covers 0.1–5 MB drawings (the real 4.69 MB drawing: 921 ms).
- **Upload is the tight direction.** A single combined signed PDF can't exceed 2.95 MB — one 4.69 MB drawing already breaks it, several certainly do. This drives D1.
- M2's per-drawing marked-up upload (up to ~5 MB) also exceeds the upload cap — see §7.3.

### 3.2 Other constraints

- **No browser at send time for Flow.** Apex can't merge third-party PDFs, so the merge can't happen at send (which is what #197 did — in the sender's browser). It has to happen in the **signer's** browser.
- **Fallback re-render** (§2) silently loses non-template pages; the page currently fails open (#407).
- **Integrity:** nothing hashes the viewing PDF at send today. An approval must bind the exact drawing revisions.
- **Decline can be hidden** (#367) — an approval request without Decline is only half a sign-off.
- **Managed package rules** (`.claude/skills/managed-package-rules`): new Flow inputs must be `global`; API names are forever; guest `SYSTEM_MODE` reads must be token-keyed; verify in a namespaced org.

## 4. Proposed design decisions (pending RFC)

| #   | Decision                      | Proposal                                                                                                                                                                                                                                                                                                                                                                                 |
| --- | ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| D1  | Signed output                 | **Approval record**: the template pages + signatures + certificate, where the certificate **lists every attached document** (title, version, SHA-256, page range as viewed). The drawings stay on the record untouched. Upload stays small regardless of drawing count/size. A single combined "approved pack" is an optional later extension, only viable under the 2.95 MB upload cap. |
| D2  | Where the merge happens       | **At view time, in the signer's browser**, with pdf-lib `copyPages` (already loaded on the page). The merged document is for **display**; signatures are stamped onto the **template-only** bytes (anchors mapped back via the page provenance map).                                                                                                                                     |
| D3  | Transport                     | **Per file**, never whole-pack, using the **existing token-gated remote-action pattern** (`getSourcePdfBase64` shape) — measured to 5.5 MB per file, so no chunking or REST in M1. Send-time cap **5 MB per attachment**.                                                                                                                                                                |
| D4  | Identity of what was approved | Pin each attachment by **ContentVersion Id** (`VersionData` is immutable per version) + **SHA-256** computed server-side. Browser re-checks the hash before display (defence in depth).                                                                                                                                                                                                  |
| D5  | Entry points                  | Signature Sender LWC **and** the Flow action "Portwood: Create Signature Request" take the same thing: an ordered list of file Ids + position (before/after template).                                                                                                                                                                                                                   |
| D6  | Failure mode                  | **Fail closed.** If any attachment can't be fetched, verified or merged, signing is blocked with an actionable message. Both certificate builders (client and server) list the attachments, so even the fallback path produces a record that binds them.                                                                                                                                 |
| D7  | Scope                         | Single-template requests only in M1. No markup, no packets, no generation-only supplemental PDFs (that's #405 — share the data model and naming with it).                                                                                                                                                                                                                                |

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

- [ ] **Samples:** real drawings — CAD vector export + scan; ~0.1 / 1 / 3 / 5 MB; A3 landscape; one using `/Rotate`; one permissions-restricted (encrypted) if the CAD export does that. _So far: one real AutoCAD 2021 plot (4.69 MB, 6× A3 landscape, no `/Rotate`, unencrypted, no object streams, 88 images) — pdf-lib `copyPages` + save OK in 59 ms (Node). More samples, incl. a scan, pending._
- [ ] **Merge:** in the VF signing-page context, pdf-lib `copyPages` of template viewing PDF + samples. Check: rotation/crop preserved; template `@@SIG-n@@` anchors still found by PDF.js; behaviour on encrypted input (pdf-lib can't decrypt — must fail cleanly); time + memory on desktop and a mid-range phone.
- [ ] **Viewer:** PDF.js with 10+ A3 pages — the page currently renders and keeps every page canvas (`DocGenSignaturePdf.page` ~`:963-988`); decide whether lazy rendering is needed.
- [x] **Transport ceilings** for T1 (existing base64 `@RemoteAction`) — results in §5.1. T1 suffices for M1 downloads.
- [ ] Deferred to M2 (upload of marked-up drawings > 2.95 MB): T2 — chunked remote actions; T3 — guest `@RestResource` on the Site streaming raw bytes (new surface type for the package → security review + maintainer call).
- [ ] Sender-side SHA-256 at send: confirm hashing several ~5 MB files in one synchronous send stays within heap (or hash per file in a Queueable).
- [x] **Output:** measured limits (§5.1) → T1 for M1, send-time cap 5 MB per attachment, D1 confirmed.

### 5.1 M0 results — transport (2026-09-23)

**Method.** Scratch org built by `scripts/qa/setup-org.sh` conventions (`--no-namespace`; a `portwoodglobal`-namespaced org needs the project Dev Hub — remoting and heap limits are namespace-independent), full `force-app` deploy, a classic Salesforce Site serving `DocGenSignaturePdf` with `DocGen_Guest_Signature` on its guest user. Test files: valid PDFs padded with random (incompressible) bytes to exact sizes, plus the real 4.69 MB drawing, attached to an Account. One `DocGen_Signature_Request__c` + `DocGen_Signer__c` per file with `Source_Document_Id__c` = that file. From an anonymous browser session on the Site, the real `getSourcePdfBase64` and `saveCompositedSignedPdf` were invoked via `Visualforce.remoting.Manager.invokeAction` (`buffer:false`, 120 s timeout).

| File size (MB)     | Download `getSourcePdfBase64` | Upload `saveCompositedSignedPdf` |
| ------------------ | ----------------------------- | -------------------------------- |
| 1                  | OK — 412 ms                   | OK — 3,309 ms                    |
| 2                  | OK — 517 ms                   | OK — 2,515 ms                    |
| 2.5                | OK — 612 ms                   | OK — 2,067 ms                    |
| 2.75 / 2.85 / 2.95 | —                             | OK — 1,862 / 1,654 / 1,615 ms    |
| 3                  | OK — 762 ms                   | **fail**                         |
| 3.5 / 4            | OK — 718 / 718 ms             | **fail**                         |
| 4.69 (real CAD)    | OK — 921 ms                   | —                                |
| 5                  | OK — 923 ms                   | **fail**                         |
| 5.25 / 5.5         | OK — 1,777 / 1,030 ms         | **fail** (5.5)                   |
| 5.75 / 6           | **fail**                      | **fail** (6)                     |
| 7 / 8              | —                             | **fail**                         |

- Download failures: `Remoting response size exceeded maximum of 15 MB.`
- Upload failures: `Input too long. [1, 149]` — rejected by the platform before Apex runs (signer/request left untouched). 2.95 MB = 4,124,400 base64 chars passed; 3.0 MB = 4,194,304 chars failed.
- Every successful upload was verified server-side: a full-size file on the related record and the request `Signed`.

## 6. M1 — Attach record PDFs to a signature request (one PR, after RFC agreement)

### Backend

- [ ] `DocGen_Signature_Attachment__c` + fields + permission sets (Admin/User as appropriate; **no guest object access** — token-keyed `SYSTEM_MODE` only).
- [ ] Send: attachments on the canonical `createGuidedPdfSignatureRequest` (options DTO vs another overload — agree in RFC); validation per §4.2; SHA-256 (sync under cap, else async).
- [ ] Flow: `DocGenSignatureFlowAction.Request` gains `global` `@InvocableVariable`s `attachedDocumentIds` (`List<String>`, CV or CD Ids) and `attachedDocumentPosition` (`Before` default). Verify in a namespaced org.
- [ ] Guest: manifest on the init response; a token + index attachment fetch remote action with the same shape and gates as `getSourcePdfBase64` (measured to 5.5 MB) — token format, `assertSignerReadable`, expiry, terminal state, PIN-verified when required.
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
3. **Upload is M2's hard limit.** M2 returns a marked-up copy **per drawing** (up to ~5 MB), which exceeds the measured 2.95 MB remote-action request cap. M2 needs chunked upload (T2) or a guest REST endpoint (T3) — spike both at the start of M2. M1 must not assume drawings ever travel back in one call.
4. **Vector markup, not raster.** Raster overlays at A3 resolution inflate files past the per-file ceiling (the limit #197 hit).
5. **Pinned CV + SHA-256** lets a marked-up return prove which revision was marked.
6. **Decline produces no document today.** M2's "decline with markup" needs one — don't bake in "decline = no output".
7. `hitToPdfRect` ignores `/Rotate` and `/CropBox` — harmless for M1 (anchors are on template pages), must be fixed for M2.
8. Markup must not write audit rows shaped like signer audits — the certificate maps audits by `Signer__c` and completion back-fills every audit row's hash.

## 8. Open questions for the maintainer

1. **Output (D1):** approval record binding attachments by hash — or do you want a single combined signed PDF (only viable when the total is under the 2.95 MB upload cap)?
2. **Transport — M2 only:** for uploading marked-up drawings above the 2.95 MB request cap, chunked remote actions (T2) or a guest Apex REST endpoint (T3)? (M1 needs neither — §5.1.)
3. **Data model:** child object vs JSON field; one model shared with #405's supplemental PDFs? API naming (frozen forever).
4. **API shape:** options DTO for `createGuidedPdfSignatureRequest` vs another overload?
5. **Linkage rule:** require attachments to be linked to the related record (proposed: yes)?
6. **Decline:** block or warn when Decline is hidden on an attachment-bearing request?
7. **#407:** land fail-closed separately first, or as part of M1?

## 9. Out of scope for M1

Markup/redline (M2, separate PR + design); richer outcomes such as "approved as noted" (possible later via a signer picklist form field); packets with attachments; combined-pack output; generation-only supplemental PDFs for Runner/Flow (#405).
