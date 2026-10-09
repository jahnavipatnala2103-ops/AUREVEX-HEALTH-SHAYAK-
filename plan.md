# HealthSahayak — Implementation Plan

## Product and scope

Build one responsive healthcare-journey prototype for AUREBECKS, centered on the requested hackathon demonstration:

> Upload or scan a clearly synthetic medical report → identify and extract its information → explain the report in plain language → let the user review and confirm → save it to the Health Vault → update the Health Profile and Journey → let the Copilot answer later using only relevant records and show its evidence.

The main navigation stays **Home, Copilot, Health Vault, Journey, Profile**. Medications, appointments, follow-ups, emergency and insurance details, family/caregiver access, privacy and account options live within the relevant destination rather than becoming extra top-level pages. Priority remains document intelligence, summary, unified profile, timeline, context-aware Copilot, evidence, safety, language support, FHIR-style structure, then voice. The secondary workflows remain integrated but do not crowd the core demonstration.

### Prototype boundary and honest defaults

- This is a hackathon prototype, not a clinical product. Use clearly synthetic sample people, doctors, appointments, prescriptions and reports; label them as demo data. Do not claim diagnoses, clinical validation, production security, patient outcomes, partnerships or live ABDM/ABHA integration.
- Before users submit a document for AI processing, disclose the Preview context and that its content is sent to the configured AI service; obtain explicit consent. Check content before any durable storage or document-row creation: inspect image/camera bytes directly, and render up to three PDF pages from the original bytes on the server in memory. Do not trust client-supplied PDF previews. Fail closed when rendering/classification is unavailable or inconclusive. Persist the original only after a positive health-document result; keep extracted data in review until user confirmation.
- Use the provided Manus AI service for actual extraction and Copilot generation when configured. Do not silently substitute canned responses for an AI outage or present simulations as AI; give a clear retry/error state. Keep seeded synthetic examples and a downloadable, clearly synthetic sample lab-report PDF available for a dependable hackathon walkthrough; the sample uses the same opt-in upload, AI extraction, review and confirmation flow.
- Authentication uses the user-requested six-digit email OTP flow with server-side Resend delivery, HMAC-only code storage, expiry, single use, attempt limits, generic account responses and the starter's secure application session. Manus OAuth remains a visibly labelled fallback, not a substitute for an undelivered email code. No real account asks for an email password.
- Do not publish the site publicly as part of this implementation. Publication is separate from Preview/checkpointing and was not requested.

## Implementation approach

### Existing foundation

Continue the initialized **web-db-user** starter rather than replacing it. It already contains React 19, TypeScript, Vite, Tailwind, Wouter, Express, tRPC, Drizzle/MySQL, OAuth/session helpers, an LLM client, file-storage helpers, migrations, tests and a Docker build. Server and database are already enabled. Keep the starter’s pinned pnpm toolchain and reuse its existing auth and LLM/service helpers. Keep external dependencies minimal; use installed Radix/shadcn components and lucide icons where appropriate.

### Application shell and routes

Use five real page routes and keep the remaining dialogs/panels inside those product destinations:

- `/` — Home: personalized greeting and profile actions, one prominent Copilot composer with Ask/Voice, four suggested questions, a restrained Today summary and a short Journey preview.
- `/copilot` — context-aware conversation, prior conversations, language picker and voice states, distinct sections for what is known/general/uncertain, and a source-backed expandable “Why am I seeing this?” panel.
- `/vault` — scan/upload actions, categories, search and sort; document detail, plain-language summary and the upload/review/confirm flow.
- `/journey` — chronological clinical-event timeline and its requested event filters and sort.
- `/profile` — editable personal and health details, medications, appointments, next-step tasks, emergency and insurance details, family/caregiver access, access/privacy controls, account and the explicitly labelled demo/mock ABHA field.

Create and maintain `public/manus-routes.json` for these actual page routes. Keep browser requests relative and preserve Preview iframe compatibility.

### Data and authorization

Use Drizzle migrations and the enabled MySQL database for durable app records. Extend the existing `users` table only as needed; add user-owned profile, health-record, observation, medication, appointment/encounter, journey-event, follow-up-task, conversation/message and access-grant records. Represent linked concepts using simplified FHIR-style resource types (Patient, Observation, Medication/MedicationRequest, Encounter, DiagnosticReport, DocumentReference and Condition), without suggesting a production ABDM connection. Link each record to its owner, source document and relevant timeline event so a confirmed record is not entered twice.

Create a default patient profile for an authenticated user and seed only explicitly labelled synthetic example data. Every protected query and mutation derives its owner from the verified session, never from a client-supplied `userId`. A new identity cannot grant itself doctor/admin privileges. The trusted project-owner identity retains its pre-existing platform-admin role; the admin-only Profile panel can assign Patient/Individual, Family Member, Caregiver or Doctor account types to existing users without ever self-assigning admin. Caregiver/family grants are opt-in, category-scoped, editable and revocable; show which categories and permissions are active. Shared users see only expressly granted records, tasks and timeline events: an Appointments grant returns appointment records only, not doctor-consultation encounters; timeline events require the separate Timeline grant. Provide a simple doctor view over only expressly authorized patient records. Do not claim email invitations, physical deletion or security features that the starter/platform has not implemented. Keep document bytes behind owner-authorized server actions; do not expose project storage credentials or signed URLs to the browser. Limit accepted file types and sizes server-side. Explain the actual demo storage/AI boundary in Privacy & Access rather than implying enterprise-grade security.

### Document intelligence and health summaries

Support camera/photo input and PDF/JPG/PNG. Validate MIME type, size and ownership; provide visible upload, processing, document-type detection, extraction, review, edit and confirmation states. Call the supplied server-side Manus LLM helper for genuine document analysis; provide the selected image/PDF in the documented multimodal shape through a server-authorized upload/download path. Ask the model for only fields the source states, date, units, printed reference intervals and whether a value is outside that provided interval. Validate the structured response and retain source references. Missing or unreadable fields must remain unknown, not be inferred. Treat extraction as a draft requiring explicit user review.

Before processing, state that this is a non-clinical demo and ask the user to confirm sending the chosen document to AI. On “Confirm & Save”, persist the reviewed document and structured observations once, associate a source citation, update profile/vault and create the timeline event; then make the record available to authorized, relevant Copilot retrieval. An unconfirmed draft does not update health data or context. Each report view separates **Important Findings**, **What This Means** and **Source**. State only that a result is outside the range printed on that report; one value does not establish a diagnosis. Suggest professional follow-up when appropriate. Show recovery for upload/model errors rather than inventing extracted data.

### Context-aware Copilot, languages and voice

Implement server-side LLM calls through the starter helper. Retrieve only records relevant to the question and the user's current access grants. Give the model the relevant source/date snippets, not a full health profile by default. For answers clearly separate **What We Know**, **General Information** and **What Is Uncertain**, cite relevant report/consultation/prescription records, and surface them in “Why am I seeing this?”. Use guarded instructions: no prescribing, dosage changes, diagnosis with certainty, clinical impersonation or false certainty; if evidence is insufficient, say the records are not enough and direct the user to the source document or a healthcare professional. Do not claim that an answer is a clinician-approved recommendation. Persist conversations and allow a prior conversation to reopen.

Support English, Telugu and Hindi for important interface strings, report summaries and Copilot answers, using a clear language selector. Use browser speech recognition/synthesis where available for voice input/output and expose LISTENING, PROCESSING and SPEAKING states; provide typed-input and non-spoken fallbacks when a device/browser lacks voice APIs. Do not claim bilingual handwritten OCR.

### Supporting product functions

Keep the requested medication view an organizer of instructions recorded in a prescription, start date, source and prescriber when present; include “View Source Prescription”. Never generate a treatment plan or recommend starting/changing medication. Keep upcoming and previous appointments simple, filterable and connected to Journey. The signed-in patient can add an appointment with provider, date/time, status and optional notes; save its owner-scoped appointment record and Journey event together in one database transaction. Show user-controlled follow-up tasks and completion state. Put emergency details, insurance provider/policy/date/document metadata, account settings and editable health details within Profile. Represent shared access with a compact scoped-permissions control and revoke action. Any ABHA-like value is visibly “DEMO / MOCK ABHA ID”.

## Implementation clarifications

- Copilot evidence uses the server's owner-scoped source registry for citation title/date/type, optionally links the corresponding authorized Vault document, and only carries cited conversation turns forward when they intersect the current relevant evidence set.
- Profile → Privacy & Access contains a concise six-step data-flow explanation to support the requested judge walkthrough.
- The Home avatar currently uses an initials placeholder; persistent profile-photo upload is not implemented. Some secondary Profile settings/access labels remain in English. These visible gaps are retained as open outcomes rather than represented as complete.

## Project structure

Reuse the starter folders, organizing product code within them instead of introducing an oversized new architecture:

- `client/src/App.tsx` — route table and global providers.
- `client/src/pages/` — Home, Copilot, Vault, Journey and Profile page composition.
- `client/src/components/` — shared application shell, navigation, status/notice components, journey item and record source components; use the existing `components/ui/` primitives.
- `client/src/contexts/` — app state limited to UI/session state; durable health data remains server-owned.
- `client/src/lib/` — typed formatting, language labels, upload/voice helpers and non-sensitive presentation utilities.
- `client/src/index.css` — palette, typography, responsive layout, focus states, reduced-motion behavior and component-level design tokens.
- `server/routers.ts` or focused `server/routers/` modules — typed authenticated procedures for profile, documents, records, journey, tasks, grants and Copilot, with owner/access checks at the boundary.
- `server/db.ts` — typed queries and ownership-scoped data access, extending the current Drizzle connection.
- `server/_core/` — reuse OAuth, session, service and LLM helpers; only add a small health-specific service layer if it genuinely clarifies logic.
- `server/` — document validation/AI extraction and focused domain logic; never place Manus API credentials in client code.
- `drizzle/schema.ts` and additive migration files — health resource types and indexes/ownership relationships.
- `client/public/manus-routes.json` — current page-route manifest.
- project-root `app.config.ts` — quoted, durable HTTPS `logoUrl` literal for the accepted HealthSahayak mark before checkpointing.

## Design direction

- **Design movement:** contemporary editorial healthcare information design with quiet, Swiss-inspired hierarchy—clear, humane and practical rather than futuristic or generic SaaS.
- **Core principles:** calm clarity; evidence before inference; user control at consequential save/share points; a connected journey visible without clutter.
- **Color philosophy:** soft white/sand-neutral canvases and restrained cool-gray structure communicate cleanliness and legibility. One recognizable deep healthcare teal signals HealthSahayak actions and navigation; use semantic warning/status colors only where they clarify meaning, never as decoration. Maintain accessible contrast.
- **Signature brand color:** HealthSahayak Deep Teal `#26766E`, reserved for primary actions, selection and key journey/source accents.
- **Layout paradigm:** persistent, narrow five-item desktop navigation beside a left-anchored working column; Home balances its primary Copilot area with one compact attention/Today rail. Keep journeys in a single readable vertical thread. Collapse to a direct, single-column flow with accessible bottom navigation on mobile. Avoid centered marketing-style grids and do not fill every blank area with cards.
- **Signature elements:** a connected path/marker motif echoing the healthcare journey; small, consistent “source + date” chips attached to explanations; a distinctive connected H/track logo mark.
- **Interaction philosophy:** one obvious next action per state. Show process states and provenance; let the user inspect/edit extracted facts before a single confirm action. Share, revoke, mark done and change language only in clear user-controlled actions. Keyboard operation, visible focus and readable touch targets are required.
- **Animation:** subtle short opacity/position changes for navigation, dialogs and upload progress only; no looping/pulsing decoration. Respect `prefers-reduced-motion`. A processing indicator must communicate state without implying medical precision.
- **Typography:** Manrope for short headings and Inter for body/interface text, with system fallbacks; use a legible type scale, moderate line length, clear numerical units and larger body text for the patient view.
- **Brand essence:** an AI-powered health-journey companion for individuals that connects records to their care context; **calm, thoughtful, accountable**.
- **Brand voice:** plain, measured and source-aware; never alarming or falsely certain. Example lines: “One result is outside the range printed on your report.” “Your records don’t show the cause. Bring this result to your healthcare professional.”
- **Wordmark & logo:** HealthSahayak wordmark paired with a compact connected-track “H” symbol: two clear vertical paths joined by a short central link, representing information becoming a continuous healthcare journey. Draw as a small custom SVG, not an unrelated stock image or a default-font-only logo.

## End-to-end build workflow

- After approval, maintain the project's full `TODO.md` outcome list. Implement the approved scope in the existing starter, add the required route manifest and durable brand metadata/mark, preserve the starter application-session helpers, use email OTP as the requested primary sign-in and keep Manus OAuth as a clear fallback. Keep the dev server on initialized port 3000. Add only additive, data-preserving Drizzle migrations and inspect the generated SQL before applying it to the shared managed database; do not write or overwrite user records with demo seeds on server start. Use registered host-managed TypeScript diagnostics. Run `pnpm check`, relevant tests and `pnpm build`, resolving actionable diagnostics. Start/retain Preview with `pnpm dev` and verify its HTTP readiness and actual JSON route manifest. Review source and inspect Preview if a concrete visual defect needs it. Commit intended changes to canonical Manus `main` and confirm the checkpoint. Do not enable auto-publish or submit public publication; provide Preview unless publication is separately requested.


## Approved addendum — guest welcome, onboarding, and reliability

- **First visit:** unauthenticated visitors see a plain-language “How it works” welcome and two clear choices: “Sign in or create your account” with email OTP (Manus OAuth only as a clearly labelled fallback), or “Explore the sample demo” without signing in. The public `/demo` route is outside primary navigation, read-only, visibly synthetic and never writes sample records to any account. Returning authenticated users proceed to their workspace.
- **First signed-in visit:** persist an onboarding-complete flag in the owner-scoped health profile. Before the Home page, show a concise one-time profile form for name, date of birth with age derived for display, blood group, allergies, emergency contact/name/phone/notes, and preferred language. Make optional health/emergency information optional, explain it can be edited later, and save only to the authenticated owner.
- **Customer-facing honesty:** remove “Hackathon prototype” from all site screens, footers and dialogs while keeping sample/test-data, AI-processing, and non-clinical limitations plainly visible. Never make simulated content look like a user's personal answer.
- **Voice:** use short MediaRecorder clips and the existing managed platform Speech-to-Text surface from the authenticated server, without storing audio. Ask permission only after a mic tap; show LISTENING → PROCESSING; place the transcript into the editable composer for explicit user submission. Fall back to native browser recognition or clearly available typed input when recording/transcription is unavailable. Keep browser spoken output optional.
- **Copilot failure behavior:** show the user's question immediately. If Copilot fails, retain the text and display a visible inline retry/error state; do not silently erase the turn or misrepresent canned content as AI.
- **Appointments:** address the user-reported failed appointment save with an additive database migration that includes the appointment record enum where an existing database lacks it, keep record plus Journey event in one transaction, preserve form values on errors and show success only after commit.
- **Project structure:** add a welcome/entry component, one public read-only demo page and an authenticated onboarding component; keep the established five authenticated destinations and scoped tRPC APIs.

- **Final audit hardening:** use a dedicated protected, write-once onboarding operation that validates required identity fields server-side; the general profile editor cannot forge or clear completion. Blood group, allergies and emergency details remain optional. Keep the guided synthetic tour read-only and remove account-seeding/sample-file entrypoints so sample values cannot enter personal records through the application UI.


## Final user-requested product addendum: sign-in, demo, and judge walkthrough

The previous platform-only sign-in entry is insufficient for the latest user direction. Keep it only as a visible, clearly secondary fallback; the primary real-account flow is email plus a one-time verification code. Configure actual email delivery using the server-only Resend REST API. Store an HMAC hash of each code, not the code; codes expire after 10 minutes, are single-use, have a five-attempt ceiling, and requests are rate-limited per normalized email. Do not return whether an email is registered. Never claim an email was sent when the provider is not configured: show a truthful inline explanation and leave the fallback available. Keep provider credentials only in project secrets, never browser code or logs. First verified login continues into the server-enforced profile onboarding already described above.

At the opening screen, preserve the clear two-path choice: **Email sign-in / create account** or **Demo sign-in**. Add a concise five-step workflow board visible before either choice: Upload a document → Review extracted findings → Confirm what to save → See the Health Vault/Profile/Journey update → Ask the source-linked Copilot. Add a public judge brief (not a main-navigation item) that succinctly covers the motivating fragmented-record problem, product differentiation, technical stack, cloud/database deployment approach, healthcare value, safety boundaries, scalability potential, and non-integrated capabilities honestly.

Demo sign-in accepts the visibly disclosed sample-only credentials `demo@healthsayhak.test` / `HealthSahayak-demo`. This is a front-end gate to synthetic read-only walkthrough content, not account authentication. It must not write any sample values into patient tables or request real medical information; clearly label the records as synthetic and show where different sample document types appear in the Vault. Real users must never be asked to use the demo password for an account.

Appointment reliability: preserve user-entered appointment fields on every failure, show a plain-language inline error and retry action, and show success only after both the appointment and linked Journey event commit. The live owner-scoped Drizzle transaction was smoke-verified by accepting both inserts and the returned record ID inside a rollback-only transaction; the development server uses `tsx watch` and has reloaded the latest backend. Keep diagnostic logs restricted to error codes, never health values or free-text notes.

Release truth: email OTP delivery is not production-ready until a Resend API key and verified sender address are configured through the protected project-secret workflow. The present stable Preview is not represented as a public production launch.


## Latest user direction: email OTP, explicit demo login, and judge materials

This latest request supersedes the earlier provider-unspecified default for authentication. Real personal accounts must have a plainly visible email sign-in/create-account option that sends a six-digit one-time code; no password is requested for real accounts. Use Resend over the documented server API, with an HTTPS-only app session cookie matching the starter OAuth session contract. Apply per-email cooldown/hourly limits, one active code, 10-minute expiry, single use, a five-attempt cap, HMAC-hashed code storage, generic account responses, and no PII/code in logs. Resend credentials and sender address remain server-side. Keep the verified Manus login visible as a true fallback, not as a substitute for an OTP that was never delivered.

The separate demo is a visibly read-only front-end demonstration with the sample-only login `demo@healthsayhak.test` / `HealthSahayak-demo`; its public credentials do not create a real identity or store demo data. After the demo login, show the synthetic sample document shelf, report, review/confirm steps, source-grounded example answer, and connected journey. Extend the first welcome screen's flow board to: upload → review AI-found details → user confirms → Vault/Profile/Journey update → Copilot answers with source evidence. Provide a brief public “For judges” page with concise, accurate answers about motivation, problem value, differentiation, actual frontend/backend/database/AI stack, managed deployment status, safety, scalability potential and non-live ABDM/clinical claims.

The appointment failure screenshot is addressed using the existing table schema, code/watch-reloaded backend, a data-preserving enum migration and a rollback-only live DB transaction that accepted both the appointment and linked event inserts plus returned ID; preserve entered details and show only honest success/error after transaction commit.


## Latest user-requested refinements

- Enforce a health-only Copilot boundary on the server before invoking the model. Off-topic requests (including “Who is Virat Kohli?”) receive a short localized redirect to health questions; they never receive biography or general trivia. Health answers continue to use relevant authorized records, safety constraints and verified citations. Use prior health context only for tightly scoped pronoun follow-ups.
- For image/camera uploads, classify the source image bytes in memory before any managed-storage or database write. For PDFs, render up to three pages from the original server-received bytes to in-memory JPEGs for the health-only classification; never trust client-supplied previews. Fail closed if rendering/classification is unavailable or unclear. Only after a positive check may the server store the original and extract it for user review. Reject unrelated files with a localized message before they enter storage, the Vault, Profile, Journey or Copilot.
- Add an owner-checked Vault removal action. Removal hides the source and deletes linked health records, Journey events and cited Copilot turns from the app's active data; future Copilot retrieval cannot use it. The managed runtime storage API has no physical delete endpoint, so the confirmation must clearly disclose that uploaded bytes remain in project storage in this preview; do not promise permanent erasure.
