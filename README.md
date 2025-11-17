# E-Notary Platform Experience & Logic Spec

## 1. Product Vision
Deliver a trustworthy, end-to-end digital notarization service for Indian users leveraging Aadhaar-based identity verification and live video with licensed notaries. The product must be mobile-friendly, accessible, and guide non-technical users through every requirement.

## 2. Primary Roles
- **Applicant / Initiator** – logged-in user who creates the case, uploads the document, and may also be a signer.
- **Additional Participants** – other signers or viewers invited by the initiator.
- **Notary (Lawyer)** – verifies identities, conducts the live session, captures audit artifacts, and digitally signs the document.

## 3. Cross-Journey UX Principles
1. Plain-language microcopy with contextual helper text.
2. Strong validation, inline feedback, and explicit error recovery.
3. Progress indicator (Steps 0–5) anchored on top of every step view.
4. Prominent privacy notes: "Your Aadhaar and documents are encrypted in transit and at rest."
5. All CTA labels are action-oriented (e.g., "Continue", "Send Invites", "Join Call").

---

## 4. User Journey & Screen Specs

### Step 0 – Pre-requisites Gate
**Screen Components**
- Card listing four checkbox items.
- Disabled `Continue` CTA until every checkbox is selected.
- Light info text: "Need help? Reach support via chat."

**Checkbox Copy**
1. My Aadhaar is linked to my mobile number and is active.
2. I am able to make an online payment.
3. I am using a mobile or laptop with camera enabled.
4. I have a stable internet connection.

**Logic**
- Persist selections locally (session storage) to avoid re-entry on refresh.
- On `Continue`, create draft application (status: Draft) and route to Step 1.

### Step 1 – Initiate Notary Application
Layout: Two-column desktop (Document + Participants). Single-column stacked on mobile.

#### 1A. Document Upload Panel
- Drag-and-drop zone with PDF icon and rules summary.
- Validation order: file type → size → page size (pre-check via PDF metadata) → signature detection (flag by scanning for embedded signature fields).
- Error state example: "Please upload an A4 PDF under 10 MB." / "Looks like this PDF already contains signatures. Upload an unsigned version."
- Display file name, size, page count once validated; include `Replace` button.

#### 1B. Participants Panel
- Pre-filled Participant Card for User 1 (read-only name/email/phone; editable toggle "Will sign this document?").
- Button `Add another participant` reveals dynamic form rows.
- Fields per participant: Full Name (text), Email (email type), Phone (+91 pattern), Toggle `Will sign?` (default ON).
- Inline validation: red helper text beneath invalid inputs; disable `Send Invitations` until at least one signer toggle true.
- Delete icon available for added participants except User 1.

#### 1C. Document Type Dropdown
Options:
1. CONTRACT / AGREEMENT / WILL / INSTRUMENT – EXECUTION, VERIFICATION, AUTHENTICATION
2. AFFIDAVIT OR ADMINISTERING OATH
3. NOTING INSTRUMENT NOT EXCEEDING INR 10,000/-

Selected value persists in application record and appears on Notary console and emails.

#### 1D. Primary CTA
- `Send Invitations` (desktop right-aligned, mobile sticky footer). When clicked:
  1. Validate entire form.
  2. Save application (status: Awaiting Verification).
  3. Trigger participant emails (Section 4.1).
  4. Surface confirmation modal summarizing next steps.

### Step 2 – Email Invitation & Scheduler
- Background job dispatches emails to all participants and notary pool.
- Email contents: Document title, type, initiator name, case ID, expected call window (auto-generated or selected slot), CTA `Join Notary Session` (one-time tokenized link).

### Step 3 – Aadhaar Verification Flow
**Screen States**
1. **Enter Aadhaar** – masked input (XXXX-XXXX-1234), note: "We never store your number." CTA `Send OTP`.
2. **OTP Entry** – 6-digit segmented input, timer (60s) and `Resend OTP` after expiry. Error toast for wrong OTP; after 3 failed attempts lock for 15 minutes.
3. **Success** – Display verified name (from Aadhaar KYC), timestamp, device fingerprint ID. CTA `Join Video Call` + fallback link emailed.

### Step 4 – Video Call & Live Notarization
**Layout**
- **Left column (60%)**: Document viewer (PDF canvas with page thumbnails). Scroll-synced for all participants; notary can "Spotlight" sections.
- **Right column (40%)**: Grid of authenticated participants with labels (Notary, Party 1, Party 2...). Display green badge "Verified via Aadhaar".
- Top info bar: Case ID, document type, timer of session.
- Bottom center: `End Session` (visible to Notary), `Raise Hand` (participants), chat icon for text clarifications.

**Notary Control Panel (floating drawer)**
- `Capture Audit Photo` – takes snapshot of current video grid, stores base64 image + metadata (timestamp, participants present).
- `Mark Identity Confirmed` checkboxes per participant (auto-marked after Aadhaar but modifiable).
- `Apply Digital Signature` button (enabled once: all signers verified, group photo captured, call duration >= 2 min).
- `Flag Issue` to pause session and mark application as Failed/Pending Review.

### Step 5 – Completion & Sign-off
After Notary hits `Apply Digital Signature`:
1. Backend runs e-sign workflow, attaches notary's DSC to the PDF.
2. Application status set to Completed; audit log appended (photo ID, call log, signature hash, IP addresses).
3. Success modal for notary; participants see `Notarization completed successfully` screen with `Download Document` and `Go to Dashboard` buttons.
4. Email dispatch includes secure link (expiring after 7 days) and instructions for safe storage.

---

## 5. Dashboard Experience
**Overview Cards**
- Total applications, Completed, Pending, Issues.
- CTA `Start New Notarization`.

**Filter Bar**
- Date range picker, Status dropdown, Document Type dropdown, Search by participant/document name.

**Applications Table / List**
Columns: Case ID, Document Name, Type, Last Updated, Status pill, Actions.
Status colors: Draft (grey), Awaiting Verification (yellow), Scheduled (blue), In Progress (purple), Completed (green), Failed/Cancelled (red).
Actions: `View Details`, `Download` (if completed), `Resume` (if Draft), `Cancel`.

**Detail Drawer / Page**
- Summary card: document info, timestamps, payment receipt.
- Participants section: list with verification state and join timestamps.
- Activity timeline: Draft created → Invites sent → Aadhaar verified → Call started/ended → Notary signed.
- Files section: original upload, notarized PDF, audit photo (restricted).

---

## 6. Validation & Error Handling Reference
| Scenario | Message | Resolution |
| --- | --- | --- |
| Unsupported file type | "Only PDF files are allowed." | Show `Upload new file` button. |
| File >10 MB | "Please upload a file under 10 MB." | Suggest compress tips link. |
| Non-A4 PDF | "Document must be A4 sized." | Provide template download. |
| Signed PDF detected | "Uploaded PDF already has signatures. Upload an unsigned version." | Explain why for legal compliance. |
| No signer selected | "At least one participant must sign." | Auto-toggle initiator or prompt selection. |
| Invalid email/phone | Inline error: "Enter a valid email" / "Enter 10-digit mobile." | Prevent form submit. |
| OTP incorrect | Toast "That OTP doesn’t match. Please try again." | Allow 3 attempts then cooldown. |
| OTP expired | "OTP expired. Resend a new OTP." | Auto-focus on new OTP field. |
| Participant joins before verification | Gate screen: "Complete Aadhaar verification to proceed." | Show `Start verification` button. |

---

## 7. Security & Compliance Notes
- Encrypt documents in storage (AES-256) and use signed URLs for downloads.
- Aadhaar data processed via UIDAI-compliant APIs; store only masked identifiers and verification tokens.
- Video session hosted on secure WebRTC infrastructure with end-to-end TLS.
- Maintain audit logs: authentication events, IPs, device info, call recording metadata, group photo hash.
- Regular vulnerability assessments and SOC 2 controls.

---

## 8. Technical Architecture Overview
1. **Frontend** – React/Next.js or similar SPA, responsive layout, integrates with WebRTC SDK.
2. **Backend** – Node.js/Express or NestJS service with PostgreSQL for relational data and S3-compatible storage for documents.
3. **Services**
   - File validation microservice (PDF parser + AWS Lambda).
   - Aadhaar OTP verification integration via UIDAI AUA/KUA partner.
   - Email/SMS notifications via SES + SMS gateway.
   - Digital signature service using licensed DSC hardware or eSign provider.
4. **State Machine** – Application statuses enforced via backend state machine to avoid illegal transitions.

---

## 9. Implementation Checklist
- [ ] Build Pre-requisites step with persistent state and gating logic.
- [ ] Implement PDF upload component with validations and progress UI.
- [ ] Create participants form with dynamic rows and validation.
- [ ] Wire document type dropdown to backend schema.
- [ ] Implement invitation email templates and scheduler.
- [ ] Build Aadhaar OTP flow with retry logic and telemetry.
- [ ] Integrate WebRTC call room with document viewer synchronization.
- [ ] Develop Notary control panel and audit capture endpoint.
- [ ] Implement digital signing backend workflow.
- [ ] Build dashboard list/detail views with filtering and file downloads.
- [ ] Add analytics + error monitoring.

---

## 10. Copy Guidelines Examples
- Success: "All set! Your identity is verified. Join the video call when you're ready."
- Warning: "Keep this window open. Leaving will pause your notarization session."
- Security reminder: "Only verified participants can enter the call."

---

## 11. Accessibility Considerations
- Minimum 4.5:1 contrast ratio on text.
- Keyboard-focus indicators on all actionable elements.
- Provide captions/transcript toggle for the call (via WebRTC API integration).
- Support screen readers with semantic headings and ARIA labels.

---

## 12. Future Enhancements
- Payment integration prior to scheduling to avoid no-shows.
- Calendar slot selection with notary availability sync.
- Multi-language support for Hindi and other regional languages.
- Automated reminders via SMS and WhatsApp.

