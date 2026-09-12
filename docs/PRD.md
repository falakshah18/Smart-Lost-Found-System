# Product Requirements Document (PRD)

## Smart Lost & Found System 

## 1\. Overview

Lost items at Ahmedabad University currently sit at security desks, cafeteria counters, GIST area, or other locations with no shared digital record. Owners must physically check multiple places. Matching is slow and depends on memory.

This system makes recovery easier for the person who lost something while keeping the process extremely light for the person who found it.

**Core idea:**

- Finder takes one photo, selects a location, and submits instantly.  
- Free Computer Vision model processes tags and generates blurred thumbnails **asynchronously in the background** (never blocks the submit button).  
- Owner describes the item in normal language to a chatbot (after authenticating and providing consent).  
- System matches using category, colour, and a few simple clarifying questions only when needed.  
- Photo is shown **only** to the matched claimant, using a **backend-generated blurred thumbnail** during the matching phase, and a short-lived signed URL for the original.  
- Once confirmed, clear pickup instructions are given.  
- Valuable items always end with a short human check by admin/security.

Photos are never shown in public search results.

---

## 2\. Goals

- Make logging a found item almost effortless for guards/staff (Camera → Photo → Location → Submit).  
- Let students and faculty describe what they lost in everyday language and get a match quickly, with zero "empty thinking" states in the chat.  
- Show the photo only to the person who answered enough questions, using secure backend-generated thumbnails.  
- Give clear pickup location \+ what ID to bring.  
- For valuable items (phones, laptops, wallets, ID cards, jewellery), still run the chat questions so the owner knows the item exists, then direct them to admin/security for final handover.  
- Keep the whole experience short and polite for claimants.  
- Comply with India's DPDP Act 2023 for data privacy, including explicit consent and data deletion.

---

## 3\. Non-Goals (v1)

- Fine-grained identification (brand, model, serial) as a hard requirement.  
- Fully automated physical storage.  
- Multi-campus support.  
- Handling money, insurance, or complex disputes (admin decides).  
- Replacing existing physical lost-property points (cafeteria, GIST area, security desks, library, etc.).  
- Handling hazardous, illegal, or highly suspicious items (guards will use standard physical emergency protocols/radios for these; the app is strictly for standard lost property).

---

## 4\. Users

| Persona | Role | Main actions |
| :---- | :---- | :---- |
| Finder / Guard / Staff | Finds item, takes photo | Photo → select location → submit (AI tags async) |
| **Student Finder** | Finds an item | Logs it via the app (to get credit/recognition) **or** hands it to security and the security guard logs it on their behalf |
| Claimant (Student/Faculty) | Lost an item | Authenticate → give consent → describe in chat → answer few questions → see photo → get instructions |
| Admin / Campus Ops / Security | Oversees process | Reviews valuable items, unmatched cases, edge cases, and items needing manual AI tags |

---

## 5\. Core User Flows

### 5.1 Logging a Found Item (Finder Flow) — Extremely light & zero-wait

The guard/staff side is deliberately kept as light as possible. The absolute minimum path is: **Camera → Photo → Location → Submit.**

1. Open the web/app on phone.  
2. Camera opens with a brief overlay: *"Ensure no faces or sensitive documents are in the background."*  
3. Take one clear photo. **Client-side processing** runs automatically (target \< 500 KB):  
   - **HEIC → JPEG conversion**: photos in HEIC format (default on iPhones) are converted client-side (e.g. via `heic2any`) before compression, so the backend never has to handle HEIC.  
   - **All EXIF data (including GPS) is stripped** to protect the finder's privacy.  
   - **Optional auto-face blur** runs via `face-api.js` to protect bystanders.  
4. Select drop-off location from a short preset list (Main Security Desk, Cafeteria / GIST area, Library Desk, etc.).  
5. **Guard taps Submit immediately.** The item is saved with status \= `Found` and a `pending_ai_tags` flag. The guard is done in under 10 seconds.  
6. *(Optional)* Add a short note (5–10 words) e.g. "found under table in GIST café". This can be done after submit or skipped.  
7. *(Optional)* Toggle **"This looks valuable"** if the finder recognises a phone, laptop, wallet, ID card, or jewellery — see 5.1a below.  
8. **(Background Async)** A background job (Celery) triggers:  
   - Generates a heavily blurred/pixelated thumbnail.  
   - Runs the CV model to suggest category \+ dominant colour(s). If it fails, the item is flagged in the admin "Needs Tagging" queue.  
   - Runs soft duplicate detection. If a duplicate is found, a **non-blocking** toast is shown to the guard ("Possible duplicate logged"), and it is flagged for admin review.

### 5.1a Valuable-Item Classification

The claimant flow branches on whether an item is "valuable" (FR13 vs FR14), so this needs an explicit trigger rather than being left to judgment:

- Every item has an `is_valuable` boolean field.  
- It is **auto-set to `true`** when the CV model's category tag matches a hardcoded allowlist: `phone`, `laptop`, `wallet`, `id_card`, `jewellery`.  
- The **finder can manually toggle it** at submission time (useful before AI tags arrive, or if the CV model misses it).  
- **Admin can override it** at any time from the dashboard.  
- The allowlist is stored as backend config so it can be extended without a code change.

### 5.2 Claiming an Item (Claimant Flow) — Short, polite, authenticated

1. **Claimant must authenticate first** (University SSO or Email Magic Link). No unauthenticated guest access.  
2. **Explicit Consent**: Before the chat starts, the user must acknowledge: *"By continuing, you consent to storing this conversation for up to 7 days after resolution for matching and dispute purposes, in accordance with the DPDP Act."* This consent is recorded in a dedicated `consent_log` (see FR32) rather than only shown on screen, so it holds up as proof for a legal audit.  
3. Claimant describes the lost item in normal language.  
4. **Immediate Response**: The system must *never* return an empty "thinking" state. The first response must immediately be either a match, a short clarifying question, or a "no match found" message.  
5. **Match scoring (deterministic, see 5.2a):** the backend, not the LLM, decides whether a description is a direct match, needs a clarifying question, or is a no-match.  
6. **Adaptive questions (kept short):**  
   - Distinctive items → ask 1–2 clarifying questions only if needed.  
   - Generic items → minimal or no extra questions.  
   - Maximum 3 questions in total.  
7. **Teaser UI**: If multiple items match, the system shows a **backend-generated blurred/pixelated thumbnail** while asking clarifying questions.  
8. Once confidence is reasonable, the clear photo is shown privately to that claimant only (via a short-lived signed URL), together with the optional short note from the finder.  
9. Clear one-tap buttons appear:  
   - **"Yes, that's mine"**  
   - **"No, show me the next closest one"**  
10. On "Yes":  
- The item is **locked** (see 5.2b) so a second claimant cannot simultaneously confirm the same item.  
- **Normal items** → status moves to `Ready for Pickup` \+ clear pickup instructions.  
- **Valuable items** (`is_valuable = true`) → chatbot uses this message:  
    
  > "We found a matching item. Because this looks valuable, please go to \[specific desk\] with your university ID for a quick final check. The item is currently at \[location\]." For valuable electronics the chatbot also instructs the claimant to be ready to unlock the device or provide IMEI/Serial number at the desk. Status moves to `Pending Claim` until admin/security confirms.  
    
11. On "No" → show next best match or escalate.

**Conversation state** is saved in the backend database session tied to the authenticated `user_id` (allowing cross-device continuity). State must survive network drops.

**Every claim decision (score, answers, outcome) is written to an audit log.**

### 5.2a Match Scoring Rubric

FR9 needs an actual formula, not just "rank by category and colour":

| Signal | Points |
| :---- | :---- |
| Category exact match | \+50 |
| Colour match | \+30 |
| Location proximity | \+10 |
| Keyword overlap in description | \+10 |

**Thresholds:**

- **≥ 70** → treat as a direct match (skip straight to Teaser UI / photo reveal).  
- **40–69** → ask a clarifying question before revealing anything.  
- **\< 40** → treat as no match.

Weights and thresholds are stored as backend config so they can be tuned post-launch without a code change.

### 5.2b Concurrent Claim Handling

Two claimants could describe the same lost item at the same time. To prevent both from reaching "Yes, that's mine" on the same item:

- The item row is **locked** (optimistic locking, or a DB-level lock) the moment a claimant enters the `Pending Claim` state.  
- A second claimant who reaches the same item while it's locked sees: *"This item is currently being verified by another claimant. We'll notify you if it becomes available again."*  
- If the first claim is rejected by admin, the lock is released and the item returns to `Found`, available to others.

### 5.3 Item Status Lifecycle

Every item moves through a clear state machine:

Found ──────► Pending Claim ──────► Ready for Pickup ──────► Returned

  │                 │                     │

  │                 ├─► Disputed          ├─► Disputed

  │                 │

  │                 └─► Found (Reject)

  │

  └─► Lost by Staff

- `Found` — just logged, available for matching.  
- `Pending Claim` — a claimant has matched (especially valuable items waiting for admin). *Admin can reject a fraudulent claim, moving it back to `Found`.*  
- `Ready for Pickup` — confirmed and ready for the owner to collect.  
- `Returned` — successfully handed over.  
- `Disputed` — conflict or ownership issue requiring admin intervention.  
- `Lost by Staff` — admin marks this if the physical item goes missing internally.

### 5.4 "Notify Me" Saved Search

- Claimant saves a short description once.  
- When a matching item is logged → **exactly one** email is sent.  
- After that single notification the alert is automatically disabled.

### 5.5 Admin Flow & Physical Handover

- Dashboard of all items with current status.  
- **"Needs Tagging" Queue**: Clearly surfaces items where async AI tagging failed, allowing admin to manually add category/colour.  
- Review queue for valuable claims, disputed items, and soft duplicates.  
- Ability to approve/reject claims (rejecting moves item back to `Found`), merge duplicates, mark returned, or mark `Lost by Staff`.

**Physical Handover UI**: When a claimant arrives at the desk, the guard can:

- **Scan the student's physical ID card barcode/QR** using the phone camera, OR  
- Search by student ID/Name, OR  
- View a **"Pending Pickups for Today"** list filtered by location.

The system displays the item photo and a large **"Confirm Handover"** button. Clicking this moves the status to `Returned` and triggers a "Pickup Confirmed" email to the student.

### 5.6 Anonymous Public Stats

Simple public page showing only aggregate numbers (items returned, average recovery time). No personal data.

---

## 6\. Functional Requirements

| ID | Requirement |
| :---- | :---- |
| FR1 | Accept photo upload with client-side compression (\< 500 KB). |
| FR2 | Run free/open CV model and generate blurred thumbnails **asynchronously in the background**; guard submit is never blocked. |
| FR3 | Allow manual override of category/colour by finder or admin. |
| FR4 | Optional short free-text note from finder (shown only to matched claimant). |
| FR5 | Soft duplicate detection: warn (non-blocking) or flag when same category \+ colour \+ location appears within 48 hours. |
| FR6 | Auto-register item with photo, timestamp, location, status, optional note; AI tags added when available. |
| FR7 | Chatbot accepts free-text description (only after authentication and explicit consent). |
| FR8 | Parse description into structured attributes **using LLM structured/JSON-mode output, validated against a backend schema before use**; if validation fails, fall back to a direct multiple-choice clarifying question instead of relying on the LLM parse. |
| FR9 | Rank items by match score, using the scoring rubric in 5.2a (category \+ colour primary, with location and keyword overlap as secondary signals; weights/thresholds tunable via backend config). |
| FR10 | Ask clarifying questions only when needed (max 3). |
| FR11 | Show photo privately only after sufficient confidence. |
| FR12 | One-tap buttons: "Yes, that's mine" and "No, show me the next closest one". |
| FR13 | On confirmation of normal items → status `Ready for Pickup` \+ clear instructions. |
| FR14 | On confirmation of valuable items → exact message directing claimant to the correct desk \+ status `Pending Claim`. |
| FR15 | "Notify Me" saved search: send only one notification, then automatically disable the alert. |
| FR16 | Explicit status lifecycle including `Disputed`, `Lost by Staff`, and `Found (Reject)`. |
| FR17 | Full audit log of every claim decision (score \+ answers \+ outcome). |
| FR18 | Admin dashboard for all items \+ "Needs Tagging" queue \+ review of valuable/unmatched/disputed cases \+ merge soft duplicates \+ manual AI tag fallback. |
| FR19 | Anonymous public stats page. |
| FR20 | Photos never publicly browsable. |
| FR21 | Guard/staff logging remains extremely light — Camera → Photo → Location → Submit; submit is never blocked by AI. |
| FR22 | Client-side image processing must strip all EXIF data (including GPS) before upload to protect finder privacy. |
| FR23 | For valuable electronics, the chatbot must instruct the claimant to be ready to unlock the device or provide IMEI/Serial at the desk. |
| FR24 | Implement a "Teaser" UI using **backend-generated blurred/pixelated thumbnails**; reveal the clear photo (via short-lived signed URL) only upon correct identification. |
| FR25 | Conversation state is saved in the backend database session tied to authenticated `user_id` (cross-device continuity). State must survive network drops. |
| FR26 | Physical Handover UI: Guard can scan student ID barcode/QR, search by ID/Name, or view "Pending Pickups for Today" list, then click "Confirm Handover" to move status to `Returned` and send a confirmation email. |
| FR27 | Client-side optional auto-face blur via `face-api.js` (or similar) to protect bystanders in uploaded photos. |
| FR28 | System must support a "Delete My Data" request from the user profile, purging their chat history and unlinking their PII, in accordance with DPDP Act 2023\. |
| FR29 | Admin can reject a claim in `Pending Claim` state, transitioning the item back to `Found` so others can claim it. |
| FR30 | All signed photo URLs must expire within a short window (e.g. 5–15 minutes) and the backend must verify the claim token before issuing them. |
| FR31 | Every item has an `is_valuable` boolean, auto-set from a backend-configured CV-tag allowlist (`phone`, `laptop`, `wallet`, `id_card`, `jewellery`), manually toggleable by the finder at submission and overridable by admin. |
| FR32 | Explicit consent is recorded in a `consent_log` table (`user_id`, `consent_text_version`, `timestamp`, `ip_address`, `user_agent`) — not just shown on screen — so it stands as proof for a legal audit. |
| FR33 | When a claimant reaches `Pending Claim` on an item, the item row is locked so a second claimant cannot simultaneously confirm the same item; they instead see a "currently being verified" message. |
| FR34 | Client-side upload pipeline must convert HEIC photos (default iPhone format) to JPEG before compression, so the backend never receives HEIC. |

---

## 7\. Non-Functional Requirements

- Claimant conversation must feel short and polite; **no empty "thinking" states** in chat responses.  
- Finder logging must stay under \~10 seconds even on weak Wi-Fi (AI runs async).  
- Privacy: photos visible only to the matched claimant and admins. EXIF/GPS data is stripped; optional face-blur applied.  
- **Data Retention & Compliance**:  
  - **Chat logs** are hard-deleted **7 days** after an item reaches `Returned` (unless flagged for a `Disputed` case).  
  - **Photos** are hard-deleted from cloud storage **30 days** after closure.  
  - **Audit logs** (scores, decisions, state transitions) are retained for **1 year** for dispute resolution, then anonymized/deleted.  
  - System must support a "Delete My Data" request in accordance with India's DPDP Act 2023\.  
- **Rate Limiting & Abuse Prevention**: Chatbot inputs are limited to 10 messages per minute per user. Photo uploads are limited to 20 per hour per Guard ID.  
- Prefer free / open models and reliable free managed APIs (with paid fallback if needed for pilot stability).  
- Support English primarily; detect and reply in Hindi/Gujarati if the free model handles it well.  
- System remains usable even if AI services are temporarily down (async tagging \+ admin manual fallback).  
- Conversation state survives temporary network loss and works across devices for authenticated users.  
- Every claim decision is auditable.  
- **Zero-wait submission**: Guard must never wait for AI processing to submit a found item.  
- **LLM reliability**: every LLM call must produce schema-validated structured output (JSON mode \+ backend Pydantic validation); if validation fails or the call times out, the system falls back to a deterministic multiple-choice question rather than surfacing raw or malformed LLM output.

---

## 8\. Success Metrics

- % of found items logged without tag correction.  
- % of claims resolved through chatbot without admin help.  
- Average time from found → returned.  
- Usage of "Notify Me" (and conversion from the single notification).  
- Claimant and finder satisfaction.  
- False-positive rate on confirmed matches.  
- Number of soft duplicates flagged and successfully merged.  
- Number of items that enter the `Disputed`, `Lost by Staff`, or `Found (Reject)` state.  
- Average guard submission time (target: \< 10 seconds).  
- Number of "Delete My Data" requests processed (compliance metric).

---

## 9\. Assumptions & Constraints

- Physical items continue to be stored at existing campus points.  
- Guards will use the system only if it is faster than a paper register.  
- University SSO is available (or can be integrated); otherwise Email Magic Link remains the fallback (no SMS OTP due to DLT/telecom verification complexity).  
- University ID is normally sufficient; valuable items get an extra human check at the desk (including unlock/IMEI request for electronics).  
- Initial CV accuracy will not be perfect — that is why photo confirmation \+ short questions \+ audit log exist.  
- Gemini Free Tier quota is sufficient for pilot; Groq Paid is available as fallback if limits are hit.  
- **Campus Legal has reviewed and approved the Data Processing Agreement (DPA) language for any third-party AI services (Gemini/Groq) used in the system.**

---

## 10\. Open Questions

- Exact list of current drop-off locations that already hold lost property.  
- Official university policy on how long unclaimed items are kept before disposal/donation.  
- Designated admin / Campus Operations contact for valuable and disputed items.  
- Exact University SSO provider details (OAuth/SAML endpoints).  
- Whether Hindi/Gujarati support is required in the first version.  
- Legal review of DPDP Act 2023 compliance wording for the "Delete My Data" flow and explicit consent text.  
- **Finalize Data Processing Agreement (DPA) terms with University Legal for third-party AI APIs.**

---

## 11\. Suggested Phasing

| Phase | Scope | Notes |
| :---- | :---- | :---- |
| Phase 1 | Portal \+ photo upload (compression \+ EXIF strip \+ optional face-blur) \+ manual tags \+ location \+ basic search \+ "Notify Me" \+ status lifecycle (including Disputed, Lost by Staff, Reject) \+ admin view ("Needs Tagging" queue) \+ Physical Handover UI (with scan/search/pending list) \+ public stats \+ audit log skeleton \+ Email Magic Link auth | Proves workflow, no AI required |
| Phase 2 | Async Celery jobs for CV model (YOLOv11n/YOLO26n \+ OpenCV) \+ backend-generated blurred thumbnails \+ soft duplicate detection (non-blocking) | Faster for finders; zero-wait submit |
| Phase 3 | Chatbot (Gemini Free \+ Groq Paid fallback) \+ matching \+ Teaser UI (backend thumbnails \+ short-lived signed URLs) \+ private photo \+ one-tap Yes/No \+ adaptive questions \+ valuable-item handling (including unlock/IMEI instruction) \+ full audit log \+ cross-device conversation state \+ explicit consent flow | Core claim experience |
| Phase 4 | Cron jobs for 7-day chat log deletion, 30-day photo deletion, 1-year audit log retention \+ "Delete My Data" DPDP compliance endpoint \+ small polish | Ready for wider pilot |

---

## 12\. Why This Version Works 

- **Zero-wait for guards**: Camera → Photo → Location → Submit is instant; AI tagging and thumbnail generation run async in the background. Guards finish in under 10 seconds.  
- **Privacy & Compliance First**: EXIF stripped, optional face-blur, backend-generated thumbnails (no CSS tricks), short-lived signed URLs for original photos, explicit DPDP consent, and clear data deletion policies (7 days for chat, 30 for photos, 1 year for audit).  
- **Short and polite for claimants**: Authenticated chat with zero "empty thinking" states, Teaser UI, network resilience, and cross-device continuity.  
- **Valuable items handled realistically**: Owner learns the item exists and where it is, then goes to the desk ready with unlock/IMEI if needed.  
- **Student Finders** can log items themselves for recognition or hand them to security.  
- **University SSO \+ Email Magic Link** removes login friction (no SMS OTP / DLT headaches).  
- **Explicit, robust status machine**: Includes `Disputed`, `Lost by Staff`, and `Found (Reject)` (allowing admins to put fraudulent claims back in the pool).

