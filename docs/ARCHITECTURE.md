# High-Level Architecture & Technical Design

## Smart Lost & Found System 

## 1\. Recommended Stack

| Layer | Choice | Reason |
| :---- | :---- | :---- |
| Frontend | React \+ Vite \+ Tailwind \+ shadcn/ui | Fast, mobile-friendly, widely used in recent campus projects |
| Backend | FastAPI (Python) \+ Celery/Background Tasks | Excellent for CV \+ LLM integration, async job processing |
| Message Broker | Redis (Upstash free tier or small managed instance) | Celery requires a broker to function — Redis also doubles as the rate-limiting/session cache |
| Database | PostgreSQL (Neon or Supabase free tier) | Permanent free tiers, reliable |
| Photo Storage | Cloudflare R2 or Supabase Storage | Free / zero-egress options; supports signed URLs |
| CV Model | YOLOv11n or YOLO26n (Ultralytics) \+ OpenCV | Best free speed/accuracy balance in 2026 for category detection; OpenCV for dominant colour \+ thumbnail blur |
| LLM / Chat | Google Gemini 1.5 Flash (Free Tier) \+ Groq Paid (fallback) | Gemini Free offers generous quota for pilot; Groq Paid (pay-as-you-go) as reliable fallback if Gemini limits hit |
| Auth | University SSO (OAuth/SAML) \+ Email Magic Link (Resend/AWS SES) | SSO removes friction for students; Magic Link is free, requires zero telecom/DLT verification, and is easier than SMS OTP |
| Notifications | University SMTP or free Resend/Brevo tier | One-time "Notify Me" only |
| Face Blur | face-api.js (client-side) | Free, runs in browser, protects bystanders before upload |

---

## 2\. System Architecture Overview

flowchart TB

    subgraph Client\["Client Layer"\]

        FinderApp\["Finder / Guard Web App\\n(Camera, EXIF strip, face-blur)"\]

        ClaimantApp\["Claimant Web App\\n(Auth, Chat UI, Teaser UI)"\]

        AdminApp\["Admin Dashboard"\]

    end

    subgraph Backend\["FastAPI Backend"\]

        API\["REST API Layer"\]

        Auth\["Auth Service\\n(SSO / Magic Link)"\]

        MatchEngine\["Deterministic Match Engine\\n(scoring rubric, thresholds,\\nrow-lock on Pending Claim)"\]

        AuditLog\["Audit Log Writer"\]

        SignedURL\["Signed URL Service"\]

    end

    subgraph Async\["Async Workers (Celery)"\]

        CVJob\["CV Job\\n(YOLO \+ OpenCV)"\]

        ThumbJob\["Thumbnail Blur Job"\]

        DupJob\["Soft Duplicate Detection"\]

        CronJob\["Cron: Retention Cleanup"\]

    end

    subgraph Broker\["Message Broker"\]

        Redis\[("Redis\\nCelery broker, rate limits, session cache")\]

    end

    subgraph External\["External Services"\]

        LLM\["Gemini Flash (primary)\\nGroq (fallback)\\nJSON-mode \+ schema validation"\]

        Mail\["Resend / SMTP"\]

    end

    subgraph Storage\["Data Layer"\]

        DB\[("PostgreSQL\\nItems, Users, Sessions, Audit, Consent Log")\]

        Blob\[("Cloudflare R2 / Supabase Storage\\nOriginals \+ Thumbnails")\]

    end

    FinderApp \--\>|"Photo \+ Location"| API

    ClaimantApp \--\>|"Authenticated chat"| API

    AdminApp \--\>|"Review / manage"| API

    API \--\> Auth

    API \--\> MatchEngine

    API \--\> AuditLog

    API \--\> SignedURL

    API \<--\> DB

    API \--\> Blob

    API \-.enqueue via Redis.-\> CVJob

    API \-.enqueue via Redis.-\> ThumbJob

    API \-.enqueue via Redis.-\> DupJob

    CVJob \-.-\> Redis

    ThumbJob \-.-\> Redis

    DupJob \-.-\> Redis

    CronJob \-.-\> Redis

    CVJob \--\> DB

    ThumbJob \--\> Blob

    DupJob \--\> DB

    CronJob \--\> DB

    CronJob \--\> Blob

    MatchEngine \<--\> LLM

    API \--\> Mail

    SignedURL \--\> Blob

**Guiding technical principles**

- AI only *suggests*. Final matching score, thresholds, and decisions stay in deterministic backend code, using the scoring rubric (category \+50, colour \+30, location \+10, keyword overlap \+10; ≥70 direct match, 40–69 clarifying question, \<40 no match — weights tunable via config).  
- LLM calls use structured/JSON-mode output and are validated against a backend schema before use; a failed validation or timeout falls back to a deterministic multiple-choice question, never raw/unvalidated LLM output.  
- The item row is locked the moment a claimant enters `Pending Claim`, so a second claimant can't simultaneously confirm the same item; the lock releases if the claim is rejected.  
- Always have a manual path if AI is slow or unavailable.  
- Photos stay private; EXIF data is stripped client-side; HEIC photos are converted to JPEG client-side before compression; optional face-blur applied; backend-generated thumbnails power the Teaser UI.  
- Claim conversation stays short, survives network drops, and works across devices for authenticated users. Chatbot must never return empty "thinking" states.  
- Guard/staff side stays extremely light (Camera → Photo → Location → Submit); submit is never blocked by AI.  
- Every AI call has a visible loading state and a hard timeout with graceful fallback.  
- Full audit log of claim decisions (retained for 1 year).  
- **Context-Aware & Versioned Prompts**: the LLM system prompt is explicitly configured to detect 'electronics/valuables' from the CV tags (via the `is_valuable` field) and automatically append the unlock/IMEI warning. System prompts are versioned and stored in the repository.  
- **DPDP Compliance**: consent is recorded in a `consent_log` table (`user_id`, `consent_text_version`, `timestamp`, `ip_address`, `user_agent`) rather than only shown on screen; the user data deletion endpoint purges chat history and unlinks PII on request.

---

## 3\. Teaser UI & Photo Security Implementation

sequenceDiagram

    participant Guard as Finder/Guard

    participant API as FastAPI Backend

    participant Celery as Celery Worker

    participant Blob as Object Storage

    participant LLM as Gemini/Groq

    participant Claimant as Claimant App

    Guard-\>\>API: Submit photo \+ location (EXIF stripped, HEIC converted)

    API-\>\>Blob: Store original photo

    API--\>\>Guard: 200 OK (status \= Found) \[instant\]

    API-\>\>Celery: Enqueue thumbnail \+ CV tagging job

    Celery-\>\>Blob: Generate blurred/pixelated thumbnail

    Celery-\>\>API: Update item with tags \+ thumbnail\_url \+ is\_valuable

    Claimant-\>\>API: Describe item (post-auth, post-consent)

    API-\>\>LLM: Extract structured fields (JSON mode)

    LLM--\>\>API: Structured response

    API-\>\>API: Validate against schema (fallback to MCQ on failure)

    API-\>\>API: Deterministic match scoring (rubric)

    API--\>\>Claimant: thumbnail\_url (Teaser UI) \+ clarifying question(s)

    Claimant-\>\>API: "Yes, that's mine"

    API-\>\>API: Lock item row (enter Pending Claim)

    API-\>\>API: Verify active claim token

    API-\>\>Blob: Generate short-lived signed URL (5–15 min)

    API--\>\>Claimant: original\_url (signed) \+ pickup instructions

    API-\>\>API: Write decision to audit log

- The backend Celery job generates a heavily blurred/pixelated thumbnail (via OpenCV or Sharp) at upload time and stores it alongside the original.  
- The frontend requests the `thumbnail_url` during the Teaser phase.  
- Only after the backend verifies the claim does it issue a **short-lived signed URL** (e.g. 5 minutes) for the `original_url`.  
- The backend must verify the active claim token before generating the signed URL. Never rely on frontend CSS for privacy.

---

## 4\. Data Flow Summary

1. Finder uploads compressed \+ EXIF-stripped (HEIC converted, optionally face-blurred) photo → item saved immediately as `Found` → Celery job (via Redis broker) runs async: generates blurred thumbnail \+ runs YOLO for tags \+ sets `is_valuable` from the CV-tag allowlist (admin can tag/override manually if AI fails).  
2. Claimant authenticates → gives consent (recorded in `consent_log`) → describes item → LLM (via Gemini, falling back to Groq Paid) extracts structured fields in JSON mode → backend validates the schema (falls back to a direct question on failure) → backend scores deterministically using the rubric → optional short questions with Teaser UI (backend thumbnails) → on "Yes," item row is locked → private photo shown (via short-lived signed URL) → decision written to audit log → status updated.  
3. At the desk the guard scans student ID / searches / views pending list → uses Physical Handover UI to confirm return and trigger the confirmation email.  
4. Cron job hard-deletes chat logs 7 days after closure, hard-deletes photos 30 days after closure, and retains audit logs for 1 year.

---

## 5\. Item Status State Machine

stateDiagram-v2

    \[\*\] \--\> Found

    Found \--\> PendingClaim: Claimant matches

    Found \--\> LostByStaff: Physical item goes missing

    PendingClaim \--\> Disputed: Ownership conflict

    PendingClaim \--\> Found: Admin rejects (Found (Reject))

    PendingClaim \--\> ReadyForPickup: Admin/security confirms

    ReadyForPickup \--\> Returned: Guard confirms handover

    ReadyForPickup \--\> Disputed: Ownership conflict

    Returned \--\> \[\*\]

    LostByStaff \--\> \[\*\]

    Disputed \--\> \[\*\]

---

## 6\. Reliability & Compliance Notes

- **Rate limiting**: chatbot inputs capped at 10 messages/minute/user; photo uploads capped at 20/hour/Guard ID. Redis backs the rate-limit counters.  
- **Celery broker**: Redis (Upstash free tier or small managed instance) is required for Celery to function, and doubles as the rate-limit/session cache.  
- **Match scoring**: deterministic rubric (category \+50, colour \+30, location \+10, keyword overlap \+10; ≥70 direct match, 40–69 clarifying question, \<40 no match), weights/thresholds stored as tunable backend config — never left to the LLM to decide.  
- **LLM output validation**: all LLM calls use JSON/structured-output mode; responses are validated against a backend schema (e.g. Pydantic) before use. On validation failure or timeout, the flow falls back to a deterministic multiple-choice question.  
- **Concurrent claims**: the item row is locked when a claimant enters `Pending Claim`; a second claimant is told the item is being verified and is notified if it's released back to `Found`.  
- **Retention & deletion cron**: chat logs (7 days post-closure), photos (30 days post-closure), audit logs (1 year, then anonymized).  
- **DPDP Act 2023**: explicit consent recorded in a `consent_log` (user, consent text version, timestamp, IP, user agent) before chat starts; "Delete My Data" endpoint purges chat history and unlinks PII; DPA terms with third-party AI vendors reviewed by Campus Legal.  
- **Image pipeline**: HEIC photos (default on iPhones) are converted to JPEG client-side before compression, so the backend only ever handles JPEG/PNG/WebP.  
- **Degraded-mode operation**: if Gemini/Groq or the CV model is unavailable, the system falls back to admin manual tagging and does not block core flows.

