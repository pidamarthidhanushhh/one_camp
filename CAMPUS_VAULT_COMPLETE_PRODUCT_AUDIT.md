# Campus Vault — Complete Product, Technical & UX Audit

**Audit date:** 31 October 2026  
**Scope:** Current source tree and locally running application under `D:\Projects\campus`  
**Method:** Source inspection, local browser walkthrough, syntax checks, and automated tests  
**Change boundary:** This audit added only this report. No application source, configuration, or data files were changed.

> **Naming note:** The supplied audit brief calls the product “Campus Vault.” The application and package currently brand it as “Campus” (`Campus — your study space`, package name `campus-study-resources`). This report uses “Campus” for the implemented product and “Campus Vault” when referring to the requested product vision.

## 1. Executive Summary

Campus is a carefully styled academic-resource prototype with separate student and faculty/admin workspaces. Students can browse sample and locally uploaded materials, search/filter, save and rate resources, preview locally stored PDFs, join faculty Crates, and use Raze for AI-assisted study tasks. Faculty/admin demo controls provide an upload review queue, announcements, Crate management, and local dashboard counts.

The visual experience is more complete than the platform infrastructure. Sign-in is simulated in browser JavaScript; there is no database, real identity provider, server authorization, shared resource repository, or shared file storage. Resource records and most user-facing changes are browser-local. The Gemini endpoints are real server-side integrations, but AI access is not tied to an authenticated user and live provider behavior was not verified in this audit.

**Highest-priority findings**

1. **Do not treat the current login or admin role as security.** The role is checked client-side and stored in tab-scoped `sessionStorage`; the server and AI endpoints do not authenticate or authorize users.
2. **Verify and rotate the key if valid.** The `.env.example` inspected during this audit contained a non-placeholder-looking `GEMINI_API_KEY` value. Its validity was not tested and the value is intentionally omitted here. Treat it as exposed until verified; replace it with a harmless placeholder and revoke/rotate it if it is real.
3. **The core “shared campus library” is not yet shared.** Metadata lives in `localStorage`, PDFs in browser `IndexedDB`, and there is no database or object store. Different browsers/devices do not see the same resources, ratings, saves, Crates, or moderation state.
4. **Recommendations are deterministic ranking, not personalized AI recommendations.** Discovery promotes faculty-reviewed and highly scored records. It does not learn from a student’s profile or behavior.
5. **The project has no live institutional catalog.** Many of the 39 resources observed in Discover are clearly marked sample faculty records; Data Science and Mechanical department chips can be selected but have no matching seeded resources.
6. **The UI is a strong prototype foundation.** Desktop student and admin flows, dark theme, Raze, Crates, upload/review interactions, and empty/status states are present. The small test suite passed, but does not validate production auth, database/storage behavior, live Gemini, accessibility comprehensively, or browser/device compatibility.

**Current readiness:** Suitable for design/product discussion and local demonstration. Not suitable for real student data, actual institutional administration, or reliable cross-device resource sharing.

## 2. Project Overview

The product goal is a single campus-oriented place to discover and share course materials, learn from community/faculty quality signals, preview PDFs, and use an academic assistant. The current implementation is a browser-local demonstration centered on Matrusri Engineering College.

The root contains a small, unbundled web application:

| Path | Current responsibility |
|---|---|
| `index.html` | Login page, shared workspace shell, student/admin page markup, dialogs, notification center, Crates, Raze, and resource views |
| `app.js` | Client state, simulated login, view navigation, local persistence, resource operations, AI request orchestration, rendering, and event handlers |
| `styles.css` | Global visual system, responsive layout, theme-specific rules, and component styling |
| `server.mjs` | Node HTTP server, static-file allowlist, and Gemini-backed API endpoints |
| `pdf-preview.js` | Client PDF.js integration for rendering a selected local PDF |
| `server.test.mjs` | Node built-in test-runner tests with mocked Gemini responses |
| `package.json`, `package-lock.json` | Start/test scripts and installed dependency declarations |
| `README.md` | Local startup instructions, prototype limitations, and user-facing feature notes |
| `PROJECT_REPORT_FOR_GEMINI.md` | Earlier database-schema planning brief; portions are outdated relative to the current Raze implementation |
| `.env`, `.env.example`, `.gitignore` | Local environment/configuration files; secret values are not reproduced in this report |

No separate application packages, database migrations, deployment manifests, design-system package, or server-side user/resource modules were present in the inspected root. `node_modules` is installed. There is no `.git` metadata in the current workspace, so this audit cannot establish a clean baseline or attribute existing modifications to a particular change.

## 3. Technology Stack

| Concern | Actually used | Notes |
|---|---|---|
| Frontend | HTML, CSS, browser JavaScript (ES modules) | No React/Vue/Angular/Svelte application runtime or client-side bundler was identified. Pages are rendered from a large shared HTML document with DOM updates in `app.js`. |
| Backend | Node.js built-in `http`, `fs/promises`, `path`, and `url` modules | No Express/Fastify/Koa framework is used. `npm start` starts `server.mjs`. |
| AI provider | Google Gemini REST API; model ID `gemini-3.1-flash-lite` | Requests originate from `server.mjs`; the server reads `GEMINI_API_KEY`. Live provider calls were not made during this audit. |
| PDF viewer | `pdfjs-dist` 6.4.299 | Browser-side PDF.js renders pages to a canvas. |
| Browser persistence | `localStorage`, `sessionStorage`, and `IndexedDB` | Local to a browser origin/profile; not a shared database. |
| Database | None | No SQL/NoSQL client usage, schema, tables, or migrations found. There are no current database table names. |
| Identity provider | None | Firebase is declared as a dependency but no active application import/integration was identified. |
| Styling/fonts | Hand-authored CSS; Google Fonts `DM Sans` and `DM Serif Display` imported remotely | No styling framework or animation library was identified. |
| Testing | Node.js built-in `node:test` and `assert` | Tests start local HTTP servers and inject mocked Gemini responses. |
| Runtime tested | Node.js 24.15.0, npm 11.12.1 | README asks for Node.js 22.13 or newer. |

**Dependency interpretation:** `pdfjs-dist` is actively used by `pdf-preview.js`. `firebase` is in the manifest/lockfile but is not an implemented Firebase authentication, Firestore, or Storage integration. No build, lint, type-check, or CI script is defined in `package.json`.

## 4. System Architecture

```text
Browser
  ├─ index.html + app.js + styles.css
  ├─ localStorage: resource metadata, ratings, saves, Crates, announcements, theme
  ├─ sessionStorage: demo session, chat transcript, transient login feedback
  ├─ IndexedDB: uploaded PDF Blobs for local preview
  ├─ pdf-preview.js + PDF.js: canvas rendering of local PDF pages
  └─ same-origin POST /api/*
        └─ Node server.mjs
              └─ Gemini generateContent API
```

`server.mjs` serves `/`, `/student`, `/admin`, the allowlisted frontend files, and two PDF.js module files. `/student` and `/admin` are generated from the same HTML by a string transformation that removes the login page from the returned document. Client-side session checks decide whether to retain a role-specific workspace or return to the landing page. These are navigation conveniences, not server-enforced access controls.

The four AI endpoints are:

| Method and path | Purpose |
|---|---|
| `POST /api/analyze-pdf` | Analyze an uploaded PDF and return editable resource metadata plus an AI score |
| `POST /api/chat` | Send Raze conversation turns and optional resource context to Gemini |
| `POST /api/generate-study-summary` | Generate a concise PDF-grounded summary or longer study guide |
| `POST /api/generate-study-tools` | Generate validated quiz, flashcard, or mock-test output |

There is **no resource CRUD API**, shared search API, database adapter, object-storage service, user service, or background job worker. Browser code is the authority for nearly all prototype state.

The request/response contract is also prototype-level and uses non-streaming JSON:

| Endpoint | Request shape (simplified) | Response shape (simplified) |
|---|---|---|
| `/api/analyze-pdf` | `{ pdfBase64 }` | `{ resourceTitle, subject, course, tags, topicsCovered, focusTopics, aiScore }` |
| `/api/chat` | `{ messages: [{ role, text }], context? }` | `{ reply }` |
| `/api/generate-study-summary` | `{ title, pdfBase64? , topics? , detailStyle? }` | `{ summary, topicsCovered }` |
| `/api/generate-study-tools` | `{ title, toolType, pdfBase64? , topics? }` | `{ title, items }` |

These are API implementation shapes, not a versioned OpenAPI contract. Input/output validation lives in `server.mjs`; client and server types are not generated from a shared schema.

## 5. Complete Feature Inventory

Statuses describe implementation evidence, not product intent. “Verified” means exercised by this audit’s browser/tests; it does not mean production-ready.

| Feature | Current implementation and user flow | Status | Evidence and limitations |
|---|---|---|---|
| Landing and account selection | Branded landing/login screen with student and faculty/admin role toggle, prefilled demo account fields, password reveal, and status messages | UI only / placeholder | `index.html` login form; `app.js` role selection and `signInDemo()`; browser login tested. No registration, password reset, email verification, or real credential validation service. |
| Student/admin workspaces | `/student` and `/admin` return distinct documents; client shows role-specific navigation and screens | Partially implemented | `server.mjs` `renderWorkspacePage()`; client session stored in `sessionStorage`. Browser student and admin redirects tested. URLs are not authorization boundaries. |
| Academic catalog | Static department/course/year/semester structures, including Matrusri-specific examples and incomplete mappings | Partially implemented | `app.js` `departmentCatalog`, `sectionsByDepartment`, `updateCurriculumFilters()`; sample data, not authoritative institutional data. |
| Discover resource list | Cards with title, course/subject, format, author, summary/score, faculty marker, and save action | Implemented but not fully verified | `app.js` `resourceMarkup()`/`renderResources()`; browser displayed 39 records, many explicitly labeled demo/sample. Most seeded items do not have original PDF files. |
| Search | Client-side substring search over resource title, subject/course, academic metadata, tags, and reviewer | Implemented and verified | `state.query` and `getVisibleResources()` in `app.js`; no server index, stemming, typo tolerance, or semantic search. |
| Filters and feed tabs | Existing curriculum filters plus Suggested/New uploads and department chips (All, CSE, Data Science, EEE, ECE, Mech, Civil) | Implemented but not fully verified | `index.html` and `app.js` `getVisibleResources()`/`matchesDepartmentFeed()`; filters do not alter the search field. Data Science and Mech have no matching sample catalog records in the observed dataset, so those selections are empty. |
| Resource ordering / recommendations | Faculty reviewed/score priority and provisional combined score ordering; newest mode sorts by date; “Suggested” is not personalized | Partially implemented | `overallScore()` and `getVisibleResources()` in `app.js`; browser-only sorting with fixed/sample scores and no collaborative/user-aware recommendation service. |
| Resource details and PDF preview | Detail dialog can display resource metadata, topic lists and AI summary; uploaded PDFs use PDF.js page/zoom controls | Partially implemented | `index.html`, `app.js`, and `pdf-preview.js`; only PDFs stored in that browser can be previewed. No shared download/view URL. Real upload-to-preview journey was not fully end-to-end tested in this audit. Eligible student ratings are local, not a shared community aggregate. |
| PDF upload and AI analysis | Upload PDF, receive title/subject/course/tags/topics/focus topics/score suggestions, edit metadata, then submit to local queue | Partially implemented | `POST /api/analyze-pdf`; browser saves Blob in IndexedDB and metadata in `localStorage`. PDF input max is 12 MiB. No external-link ingestion or shared storage. |
| AI quality score | Live analysis endpoint requests a mandatory score; seeded cards also contain illustrative scores | Partially implemented | Gemini output is validated server-side; uploaded resources get AI score, but no online reference checking, provenance view, or calibrated quality model. |
| Community ratings | Rating control and local rating state/aggregation for resources | Partially implemented | `app.js` resource-rating state and UI; no account-backed identity, anti-abuse, shared aggregation, or verified “studied this” signal. |
| Faculty reviews/scores | Sample faculty-reviewed catalog items sort first; admin review queue handles upload publication decisions | Partially implemented | Faculty seed scores are demo data. The moderation queue is not the same as a faculty content-quality rubric, and faculty identity is not verified. |
| My library | Save/unsave resources and show saved materials | Implemented and verified | `localStorage` saved-ID set; empty state observed in browser. Saves do not sync across devices or accounts. |
| My uploads | Local view for resources uploaded from the current browser | Partially implemented | Metadata/local PDF separation; no server ownership, permanent link, or cross-device management. When admin requests changes, the upload remains listed, but no student edit/resubmit path was identified. |
| Admin moderation | Pending upload queue with approve/request changes actions and local counters | Partially implemented | `admin-view` and local `reviewStatus`; browser admin dashboard has the UI. No server authorization or moderation audit trail. |
| Campus announcements/notifications | Admin can publish local announcements; student notification center can read/mark items; saved-resource updates can appear | Partially implemented | Browser-local announcement/update and read state; no push/email notifications, shared source, or actual institution-wide delivery. |
| Crates | Admin creates a Crate/code; student joins; local membership and class-scoped resource behaviors | Partially implemented | UI/browser test showed one sample Crate. Codes and membership only exist in local browser state; not a shared classroom. |
| Raze chat | Dedicated student Raze view with embedded chat; history kept for the current browser tab/session, and selected resource summary/topics can provide context | Implemented but not fully verified | `/api/chat` and `sessionStorage`; browser confirmed chat form and dedicated view. Chat receives selected-resource summary/topics, not the original PDF. No server-side history, streaming, user profile, or retrieval index. |
| Raze summaries and practice tools | On-demand quick summary/longer guide, quiz, flashcards, and mock test based on PDF context; interactive answer/card controls | Implemented but not fully verified | `/api/generate-study-summary` and `/api/generate-study-tools`; whole PDFs are sent to Gemini for these operations; tests mock provider outputs. Practice output is in-memory only. Not every tool was generated in a live browser session. |
| Theme | Persistent light/dark switch across login/workspaces/Raze; cross-tab storage handling | Implemented and verified | `campus.theme.v1`, `applyTheme()` in `app.js`, and theme styles in `styles.css`; manual browser theme switching tested in earlier work and current view. |
| Account profile management | Account menu offers demo identity, shortcuts, and sign-out confirmation | UI only / placeholder | No profile editing, account lifecycle, security settings, or institutional identity. |
| Real user management / institution administration | No user directory, college provisioning, department admin, role-management, or tenant controls | Planned but not implemented | No server user model or database. |
| External resources and web references | Not implemented as a searchable/importable first-class source | Planned but not implemented | No external-resource ingestion endpoint or online-source retrieval for scoring. |

## 6. Frontend Architecture

- `index.html` is a single large document containing landing/auth UI, a common workspace shell, all major workspace views, dialogs, and the Raze panel. It is not a separate set of route-level components.
- `app.js` is a large client-side module that combines data definitions, session simulation, persistence helpers, view switching, sorting/filtering, HTML-template rendering, modal behavior, file handling, and API orchestration. UI views are primarily switched using `hidden` attributes and `data-view` state.
- Navigation exposes three top-level server paths (`/`, `/student`, `/admin`); Discover, My library, My uploads, Raze, Crates, and Admin are view states within the workspace document. These view changes are not distinct shareable URL routes.
- Client state begins from hardcoded sample resources plus browser-stored uploads. `localStorage` stores resource metadata and feature state; `sessionStorage` stores demo session/chat state; `IndexedDB` stores PDF blobs.
- `styles.css` is a single large stylesheet with CSS custom properties, component rules, breakpoints, theme overrides, and Raze-specific theme/layout rules. Styling uses handcrafted CSS rather than a component library.
- Icons are predominantly inline SVG and text symbols. Small hover/transition effects are CSS-based; no animation package was identified.
- Resource card/detail/chat HTML is assembled in JavaScript. User-controlled text is generally passed through an HTML escaping helper before insertion; dynamic HTML also includes fixed markup and status content.
- There is no state-management library, router, typed API client, frontend build pipeline, code splitting, or generated design tokens.

**Communication flow:** frontend events update browser state; browser persistence helpers write local records; selected AI actions serialize document/context/messages to same-origin HTTP endpoints; server validates inputs and makes a Gemini request; returned JSON is validated and rendered by the browser.

## 7. Website Design Audit

### Visual identity and hierarchy

The visual language is recognizably academic/product-oriented rather than a default browser form: a muted green/ivory palette, serif display headings (`DM Serif Display`), compact sans-serif utility text (`DM Sans`), resource-card illustrations, and inline iconography. The top-level desktop layout separates a persistent sidebar from a large scrollable workspace. Dark mode replaces the pale palette with charcoal surfaces and theme-aware text.

The landing page introduces the product and places role selection/sign-in alongside the campus story. The workspace organizes high-frequency navigation at left, account/theme/notifications at top, and content in the main panel. Discover includes a descriptive hero, search, prominent resource-feed and department controls, then cards and a trust note.

### Strengths observed

- Clear student vs admin workspaces with distinct navigation and vocabulary.
- Discover supports both search-first and browsing-by-department behavior.
- Card-level scores, faculty markers, and save controls communicate resource signals quickly.
- Dark and light modes are available across major surfaces, including Raze.
- Empty states and success/error feedback exist for many local workflows.
- The dedicated Raze page has a focused chat layout rather than exposing the AI as a full-screen default overlay.
- Desktop screenshot inspection at 1266×912 showed the sidebar/workspace split, hero, search, filters, cards, and notification affordances in a coherent hierarchy.

### Weaknesses and polish opportunities

- The interface may imply a mature shared institutional service despite prototype notices being secondary to the product chrome. Important local-only/data-security boundaries should be more prominent before students upload real work.
- The card grid is dense and places a fixed trust note in a right rail; at narrower widths, feed and department controls need careful wrapping and horizontal-scroll affordance. Some labels are very small.
- “Faculty reviewed,” “Suggested,” AI score, community score, and rating count are visually adjacent but their provenance and calculation are not fully explained at point of use.
- Some content is sample data and still occupies the primary discovery experience, which weakens perceived relevance and trust.
- The new department chips show categories unsupported by the seed catalog, producing empty states that can look like a broken filter rather than an unpopulated program.
- The public demo credentials are presented directly in the landing UI. This is transparent for a prototype but unsuitable for production.
- The web fonts are fetched through a remote Google Fonts import; font fallback is present, but offline/strict privacy environments may render differently.

### Product character

Campus currently feels like a **modern academic-resource prototype with a polished SaaS-style dashboard shell**. It is more approachable and student-oriented than a conventional college portal, but it does not yet behave like a reliable campus platform because shared content, verified identities, and institutional administration are absent. The design has a distinctive softer editorial tone; production trust will depend more on accurate real data and clear permissions than on further decoration.

## 8. Page-by-Page UX Evaluation

| Page/view | Purpose and interactions | Strengths | Friction, missing behavior, and status |
|---|---|---|---|
| Landing/login (`/`) | Explain Campus, choose Student or Faculty/Admin, sign in, reveal password, switch theme | Clear role choice, friendly copy, prefilled prototype flow, responsive layout | No registration, recovery, institutional SSO, verified email, or genuine session. Login contents are publicly discoverable in source/UI and must not be used for real credentials. At 680px, the card measured 8px beyond the viewport while the page clipped overflow; investigate the tablet-width layout. |
| Student workspace (`/student`) | Discover resources and navigate My library, My uploads, Raze, Crates | Desktop workspace and search layout are coherent; search, feeds, department chips, and curriculum filters coexist | State/data belong to browser storage. “Suggested” is deterministic sorting, not user-aware. A real user can see mostly sample content until real shared repository exists. |
| Discover | Search, filter, sort, save, open detail, browse Suggested/New uploads and departments | Richest implemented user journey; 39 records rendered in local browser during audit | Resource set is largely sample; no server search/pagination; several department choices are unpopulated; no evidence ranking is experimentally validated. |
| Resource detail/PDF preview | Inspect details/topics/summary, preview original PDF, rate or save | PDF.js canvas with page/zoom controls is a stronger preview approach than relying on native embedded viewers | Only local uploaded PDFs are available. Seeded resources often have no source PDF. Actual upload-to-view flow was not fully validated in this audit. No access control, download authorization, citation/source metadata, or extracted full-text search. |
| My library | Return to saved resources | Simple, familiar, clear empty state | Empty in a fresh browser; no folders, notes, cross-device sync, or filters specific to a library. |
| My uploads | Track files submitted by current browser | Role-specific view and upload status | Not a durable owner dashboard: records/files are browser-local and can be lost when browser data is removed. No upload version history or storage quota UI. |
| Raze AI | Chat, discover relevant campus resources, select recent PDFs, request study guides/practice | Dedicated workspace, recent-PDF history, document-context distinction, compact AI widget elsewhere | Live provider was not tested; no streaming, server-side thread persistence, citations/page references, robust retrieval, or model usage visibility. Context is only as good as the supplied PDF/summary/topics. |
| Crates | Join a faculty crate with code or create/manage one as admin | Familiar class-code concept; sample code/demo content gives the UI an understandable path | In-browser-only; no actual shared classroom, roster, memberships, course enforcement, permissions, or synchronised crate contents. |
| Admin dashboard (`/admin`) | Review queue, publish announcements, inspect counts, manage Crates | Visible queue and status, local overview metrics and announcements; admin workspace tested | Every action is browser-side; no administrative audit log, verified faculty permissions, server-side moderation, user/college management, or shared updates. Dashboard counts include demo content. |
| Account menu/notifications | Navigate shortcuts, mark notices read, sign out | Custom sign-out confirmation, accessible labels on major controls, read/unread states | Profile is not editable; notices do not reach all users because they are local data. |
| AI tools | Summaries, quizzes, flashcards, mock tests | Implemented in Raze/resource context rather than a separate overstuffed AI menu | No document-independent general academic tool beyond demo restrictions; generated answer quality and grounding not verified live. |

There are no separate profile, institution-setup, course-management, assignments, grades, or faculty-verification pages in the inspected app.

## 9. Student Journey Analysis

1. **Discovering Campus:** A polished landing message explains the study-resource proposition and displays a role-select sign-in card. It is clear that the current sign-in is a demo, but the product still resembles a real login.
2. **Signing in:** User selects student and uses prefilled browser-side demo credentials. Redirect to `/student` occurs. There is no registration, password reset, or institution verification.
3. **Setting academic context:** No onboarding asks for college/program/year. The main filters default to the Matrusri demo taxonomy. A student must open College filters and select academic fields; those filters are unrelated to the department chips added beneath Suggested/New uploads.
4. **Finding materials:** Text search and filters operate locally across currently available metadata. Top-rated/faculty-reviewed items are ordered first. The New uploads feed is limited to approved uploads, and department chips narrow the same feed. There is no server search, pagination, personalized ranking, or zero-result recommendation.
5. **Reading documents:** The intended path is to open a resource detail and preview a local PDF. This works only if a PDF Blob exists in the same browser’s IndexedDB; sample listings may not have one.
6. **Using Raze:** A student can switch to a dedicated AI view, chat, search campus resources, choose PDFs recently previewed, and create summaries/practice content. Chat history is session-scoped, not a persistent account record.
7. **Saving/rating:** Saves and ratings are local. They do not represent verified usage or community consensus across students.
8. **Contributing:** Student uploads a PDF, waits for AI analysis, edits suggested metadata, chooses an academic context/sharing label, and submits for local admin review. The file stays in browser storage, so this does not contribute to a cross-device community repository.
9. **Returning:** Local saved resources and recent PDFs help the same browser continue. A different device/browser profile loses access to this prototype’s data.

**Largest student friction:** the interface promises a unified campus library while the actual materials are seeded/local. Real shared content and trustworthy sign-in are prerequisites to delivering the core promise.

## 10. Teacher and Administrator Journey Analysis

### Teacher / contributor

The uploader contributes through the student upload dialog, supplies PDF metadata and sharing terms, receives AI-generated metadata suggestions, and waits for admin approval. The UI supports editing suggestions but does not establish a faculty account, verified author identity, source citation, permanent uploaded file, contribution history shared across devices, or a mechanism to respond to reviewer feedback beyond local status.

### College / department administrator

The faculty/admin demo sees the admin dashboard and Crates, can review local pending uploads, approve/request changes, publish announcements, and inspect counts. These interactions are suitable for demonstrating a concept but not for actual governance.

Missing for a real institution:

- Invite/provision/disable users and verify faculty status.
- Define roles and scope (institution, department, course, crate).
- Set authoritative academic structure and manage academic-year transitions.
- Enforce tenant and role boundaries on the server.
- Review history, moderator identity, reasons, appeals, and audit trail.
- Publish announcements to all eligible signed-in accounts.
- Manage shared file retention, access, versioning, and takedown.

**Largest admin friction:** dashboards report browser-local sample counts, not a live operational queue. Admin actions can be bypassed by manipulating client state or directly calling the public AI API.

## 11. AI Ecosystem Audit

### Current implementation

- Provider: Google Gemini REST `generateContent`, model ID `gemini-3.1-flash-lite`.
- Key: read by Node server from `GEMINI_API_KEY` and sent to Gemini in an API header, not a URL.
- Frontend calls the same-origin analysis/chat/summary/tool endpoints.
- Prompt and validation logic are implemented in `server.mjs`; the response is expected to contain structured JSON for resource analysis and study tools.
- PDF analysis asks for an editable title, subject/course, tags, covered/focus topics, and a 0–100 AI score. Summary and study-tool generation happen on demand.
- Chat receives recent conversation turns and optional selected-resource context. The app persists a bounded recent transcript in `sessionStorage`.
- More specifically, the client retains up to 30 chat messages and sends at most the latest 16 per request. A selected resource contributes its title/course, up to 5,000 characters of summary, and up to 20 topics. The server distinguishes this metadata context from the full PDF.
- No Firebase AI SDK is active. No vector database, embeddings, retrieval index, tool-calling agent loop, external research workflow, or web-source evaluation was identified.
- Responses are not streamed. The UI waits for the endpoint response; there is no token-by-token generation.
- No request/user-level AI usage accounting, quotas, token billing dashboard, or server-side conversation persistence exists.
- For analysis, summaries, and practice generation, the client reads and base64-encodes the complete PDF and the server includes it as one inline PDF part in a Gemini request. There is no chunking, page retrieval, embedding, or RAG store. Raze chat itself does not attach or retrieve the PDF; it can receive the resource summary/topics generated separately.
- Persistence differs by output: analysis suggestions are transient until upload submission; summaries/guides can be stored with local resource metadata; chat is tab-session state; practice output is in-memory and not saved as a reusable study set.
- AI generated content can be inaccurate. The app includes cautionary language but does not provide citation/page-span traceability for every generated claim.

### Safeguards and limitations

`server.mjs` bounds request sizes and validates the expected output shapes. Chat turns, summary requests, study tools, and metadata output have explicit limits/validation. The `%PDF-` prefix check is only a lightweight header check, not structural PDF parsing. Tests exercise validation and mock upstream responses; they do not prove model grounding or quality. Oversized JSON handling destroys the request once the cap is crossed, so the intended JSON 413 response may not reach the caller; this path was not covered by tests.

Persistence behavior is not uniform: the longer-guide flow checks whether the upload metadata was saved and reports failure, while the regular summary path ignores the persistence helper’s return value (`app.js:1195–1234, 1815–1845`). A summary may therefore appear successful even if browser persistence failed.

Raze is best described as a **model-backed academic chat/study-tool interface with limited document context**, not a full personal AI agent. It does not independently search a broad campus corpus using retrieval, retain long-term user memory, cite page passages, plan across tools, or autonomously manage a study program.

### Product opportunities

Reuse existing server plumbing for stronger document summaries, review prompts, and study-set generation. A real document-grounded assistant needs a clear content pipeline, reliable text/page extraction, privacy/retention policy, citations to page spans, and evaluation tests. Add retrieval only when shared resource scale and measured query quality justify it; do not introduce a vector database solely to resemble other AI products.

## 12. Raze AI Assessment

| Capability | Current evidence | Gap |
|---|---|---|
| Conversational academic assistant | `/api/chat` sends bounded message history to Gemini; Raze page and dock share chat UI | No streaming, persistent account threads, profile memory, model selection, or usage controls |
| Resource search | Client searches the local available resource list and offers matching results | No server index, permissions-aware corpus retrieval, or external online research |
| PDF context | Uploaded PDF summaries/topics can be used as study context; PDF-grounded summary/tools have dedicated endpoints | No page-level citations, verified retrieval, OCR/scan handling guarantees, or source quote inspection |
| Quick review / long guide | Server endpoint supports summary and longer guide modes | Output quality not live-evaluated; no storage/ownership model beyond current local resource state |
| Quiz / flashcards / mock test | Structured output is validated and tests mock expected output counts; interactions exist | No attempt analytics, mastery tracking, adaptive difficulty, evidence citations, or persisted study-set library |
| Recently viewed PDFs | Browser-local recent PDF list resumes files actually previewed | Browser-only and limited; not synchronized or server-retained |

**Recommendation:** Keep Raze deliberately focused on “find a campus resource and study from it.” Prioritize page citations, transparent selected-resource context, saved study sets, and reliable document processing before broad autonomous-agent behavior.

## 13. Backend and Database Audit

### Backend

- Minimal custom Node HTTP server in `server.mjs`.
- Four POST AI routes; GET/HEAD static serving.
- Static serving uses an allowlist for application assets and the two PDF.js modules. Tests confirm environment and project files are not served through the static route.
- JSON APIs return structured status/error responses; missing AI configuration returns 503; upstream/model issues are logged and surfaced as generic errors.
- In-memory rate limiting permits eight requests per IP address per 60-second window. This is process-local and not a durable/distributed quota.
- The JSON request body cap is approximately 1.4 times the 12 MiB PDF limit. If a body exceeds it, the server destroys the request; the client may not receive the intended structured 413 response.
- Origin checks compare the `Origin` host to request `Host`; a missing Origin is allowed. This is not authentication.
- No persistent job queue, async analysis state machine, database connection, external object-store client, or deployment server adapter was identified.

### Database and persistence

There is no database schema, database engine, table, migration, or transaction layer. Current persistence is:

| Data | Current location | Consequence |
|---|---|---|
| Seed resources/catalog | Hard-coded JavaScript objects | Static, illustrative, application-redeploy changes |
| Upload metadata/review status | Browser `localStorage` | Single browser origin/profile; user-editable |
| Saved IDs, rating state, Crates, announcements, notification read state, theme | Browser `localStorage` | Not shared between accounts/devices |
| Demo role/email, recent chat history, login feedback | Browser `sessionStorage` | Tab/session scoped; not server authenticated |
| Original uploaded PDFs | Browser `IndexedDB` Blobs | Local preview only; browser storage can be cleared/quota-limited |
| AI API key | Server environment | Must be managed as a secret; do not place a real value in the sample template |

No production relational schema should be inferred from `PROJECT_REPORT_FOR_GEMINI.md`; that file is a database-planning prompt, not an implemented schema. It also predates the current live Raze wiring, so its statements that Raze is hardcoded are stale.

### Data model direction (recommendation)

For production, start with institution, user/membership/role, academic catalog, resource, object metadata, AI-analysis history, community rating, verified faculty review, bookmark, Crate/membership, announcement, moderation/report, and audit-event entities. Store PDFs in private object storage and keep only keys/checksums/metadata in the relational database. Make tenant and resource authorization explicit in every data query. Keep AI output/version history separate from canonical user-accepted metadata.

## 14. Security and Permissions

This is a product/security risk summary, not a penetration test.

| Finding | Evidence | Severity / action |
|---|---|---|
| Demo authentication is not authentication | Credentials and role selection are in browser JavaScript; session role/email are held in `sessionStorage` | **Critical before real data.** Integrate server-backed identity and enforce permissions on every resource/admin/API operation. |
| Admin authorization is client-side only | `/admin` server response is a rendered application shell; there is no server identity/role check | **Critical before real data.** Do not expose moderation or tenant data on a client-only trust boundary. |
| AI endpoints are unauthenticated | No server-side session/user/ownership check; anyone who can reach the endpoint can attempt Gemini calls | **High.** Add auth, per-user quotas, abuse controls, and authorization before any public deployment. |
| Key-like value in `.env.example` | The inspected environment template had a non-placeholder-looking key string; validity unknown, omitted here | **Treat as potentially exposed.** Verify privately; revoke/rotate if valid, replace with a safe placeholder, and confirm no real key is committed or shared. |
| Local browser data is not access-controlled | User-editable localStorage/IndexedDB holds metadata, memberships, ratings, and files | **High for production.** Never use this as an institutional record or access-control source. |
| Uploaded PDF privacy/retention not defined | PDF is sent to Gemini and retained in local IndexedDB; no production retention/deletion policy or consent surface was verified | **High product/privacy decision.** Document provider processing, retention, access, and delete behavior before real student uploads. |
| IP rate limit is weak protection | In-memory 8 requests/IP/60s and Origin/Host check; absent Origin allowed | **Medium/High.** Add account-level quotas, distributed limits, monitoring, and robust deployment controls. |
| PDF validation is shallow | Prefix check, size limit, output validation; no full parser/content safety pipeline was established | **Medium.** Add hardened file inspection, malware scanning, safe private storage, and abuse limits as appropriate. |
| No general hardening evidence | No CSP/security-header policy or TLS/deployment configuration was present in the inspected project files | **Needs environment review.** Configure transport and browser security headers at deployment; do not infer them from local HTTP behavior. |

The server does mitigate some exposure by allowlisting static paths, avoiding API keys in the query string, limiting request size, validating model output, and not returning raw upstream details in generic errors. These are useful foundations but do not replace identity, authorization, privacy, and operational controls.

## 15. Performance and Reliability

### Verified

- `npm test` completed with **18 passing, 0 failing** tests on Node 24.15.0. Test output included an expected simulated Raze upstream-capacity warning while the mocked test passed.
- `node --check app.js`, `node --check server.mjs`, and `node --check pdf-preview.js` passed.
- Local desktop browser session rendered the student workspace with 39 resource cards and the admin dashboard. These are local development observations, not production load measurements.
- The test suite uses mocked Gemini responses and does not call the live model.

### Potential concerns requiring measurement

- `app.js`, `styles.css`, and `index.html` are large, monolithic files (approximately 130.5 KB, 125.2 KB, and 36.8 KB respectively, measured from the local tree). No minification, compression, code-splitting, or asset pipeline is defined.
- The app renders/filter-sorts client-side arrays and cards; there is no pagination or virtualization. This is acceptable for the current few dozen seed records but should be benchmarked with institutional-scale catalogs.
- PDF bytes are sent in base64 JSON request bodies, adding encoding/request-memory overhead. The server request cap is sized for a maximum 12 MiB PDF; concurrent large requests may pressure memory.
- PDF rendering is canvas-based; large/scanned PDFs may use substantial browser memory. No real large-file performance benchmark was run.
- Remote font import may delay or alter typography on cold/offline connections.
- Gemini quota, latency, model availability, cost, retry policy, and timeout behavior were not measured with live calls.
- There is no production logging/observability, health endpoint, durable rate-limit state, or background processing.

**Not measured:** Lighthouse/Core Web Vitals, bundle transfer size, API latency, Gemini latency/cost, concurrent uploads, large-document rendering, or performance on low-end/mobile devices.

## 16. Accessibility and Responsiveness

### Positive foundations

- Labels and accessible names are used on many form controls and buttons.
- Semantic sections, headings, dialogs, and live status regions are present.
- Theme switches expose `aria-pressed`; menus/notifications expose expanded state; keyboard Escape handling exists for selected overlays.
- CSS includes visible `:focus-visible` outlines.
- Browser inspection at 390px showed the admin workspace document width equal to viewport width and a two-column stat grid. Desktop browser inspection used 1266×912.
- Theme state is reflected in the document and color-scheme metadata.

### Gaps and areas to verify

- Active navigation is represented visually by a class; code inspection did not establish `aria-current`/equivalent current-page state on navigation controls.
- The upload file input is hidden inside a label, and may not provide a keyboard-focusable control in all browser/assistive-technology combinations; verify and improve.
- Department chips scroll horizontally on narrow screens; ensure keyboard users can reach offscreen chips and that the scroll affordance is discoverable.
- The upload file input is marked `hidden` within the drop-zone label; there is no separate visible keyboard-focusable file-picker button in that markup. Drag-and-drop exists, but does not replace keyboard access.
- Resource cards use clickable `<article>` surfaces. The title and save controls are buttons, but the whole card surface itself is not a keyboard control.
- Upload and detail native dialogs do not show the same explicit `aria-labelledby` association present on the sign-out dialog. Confirm they receive a reliable accessible name in supported browsers.
- The stylesheet’s custom `:focus-visible` selector covers buttons, inputs, and selects but not textareas. Active navigation is toggled by CSS class without an equivalent `aria-current`/pressed state.
- The `/` search shortcut excludes focused inputs and textareas but does not explicitly exclude `contenteditable`; Escape closes selected menus and the floating assistant, but the Raze-embedded chat is not closed through the same Escape path. A close button exists.
- Text and controls include compact typography (some labels around 7–10px in desktop CSS). Conduct WCAG contrast and text-resize testing rather than judging by color alone.
- The login grid is clipped at 680px: the login card right edge measured 688px in a 680px viewport while the page clipped overflow. Audit tablet widths around the login breakpoint.
- The audit did not run a screen reader, keyboard-only end-to-end exercise, automated accessibility scanner, or full responsive matrix. A single 390px admin measurement does not establish complete mobile accessibility.
- Focus behavior after opening/closing all native dialogs and the custom account menu should be validated; only a subset of Escape/focus flows was source-inspected.
- No reduced-motion audit was performed. CSS uses short transition effects; honor `prefers-reduced-motion` consistently if animation use grows.

## 17. Competitive Benchmarking

This is a capability benchmark, not a claim that the products were tested side-by-side during this audit.

| Reference | Capability pattern worth learning from | Campus today | Campus-specific opportunity / avoid |
|---|---|---|---|
| Google Drive | Reliable shared file access, permissions, ownership, cross-device sync | PDF storage is per-browser | Build authorized shared object storage. Do not recreate general-purpose file management beyond study resources. |
| Notion | Flexible knowledge organization and user-created structure | Academic resources are cataloged into fixed demo fields and cards | Let students organize saved study collections once shared identity exists. Avoid turning the product into a general notes editor without user evidence. |
| Moodle / Canvas | Course rosters, institutional role boundaries, course-scoped administration | Crates demonstrate class groupings but are local and unverified | Adopt explicit course/institution scope and roles if colleges require them. Assignments/grades are not necessary for a resource-library MVP. |
| Perplexity | Visible sources and answer provenance | Raze can use local PDF context but has no general citation layer | Add page citations and source excerpts to grounded answers. Do not claim web research until it is genuinely implemented. |
| ChatGPT | Flexible conversation, persistent history, streaming interactions | Raze has bounded session history and model-backed APIs | Add durable history/streaming only after privacy and identity decisions. Differentiate through campus resources and traceable document answers, not generic chat breadth. |

**Differentiation opportunity:** a trusted, institution-aware library where students can see why a resource ranks well, who reviewed it, what course it matches, and which PDF passages support AI study aids. That requires verified provenance and shared data more than a larger feature count.

## 18. Product Improvement Opportunities

| Improvement | Problem solved / beneficiaries | Expected outcome | Required work and dependencies | Complexity |
|---|---|---|---|---|
| Server-backed identity and role/tenant authorization | Students, faculty, administrators lack secure identity and data boundaries | Trustworthy sign-in and permission enforcement | Identity provider, server session/JWT policy, institution membership, authorization checks on every API/data operation | High |
| Shared resource database and private file storage | Uploads/Crates/saves exist only in the uploading browser | Students can access approved content across devices | Relational DB, object storage, signed/authorized file delivery, migrations, retention and backup policy | High |
| Authoritative institution/academic catalog | Sample course mapping and empty department filters reduce discovery trust | Accurate department/year/semester/subject facets | College confirmation and admin-managed catalog; migration from static constants | Medium |
| Explainable discovery/ranking | “Suggested” obscures deterministic weights and has no personalization | Students understand why an item ranks and can choose rating/newest/faculty signals | Server-backed ratings/reviews, documented ranking policy, metrics and feedback loop | Medium |
| Verified ratings and faculty rubric | Current scores are sample/local and easily manipulated | Credible community/faculty quality signal | Identity, review eligibility, verified faculty, moderation and audit trail; anti-abuse constraints | Medium/High |
| Document lifecycle and source provenance | Local-only PDFs, missing original documents on seed cards, weak ownership/versioning | Durable resource library with clear author, source, license, version, status and delete behavior | Resource/file schema, private storage, ownership, checksum/versioning, takedown flow | High |
| Grounded Raze answers with citations | AI output has limited context and no page-level evidence | Students can verify answers in the PDF and detect unsupported claims | Text/page extraction, page references, retrieval/evaluation, UI citations, privacy/retention policy | Medium/High |
| Upload reliability and feedback | Upload depends on AI analysis before metadata can be saved; no shared processing status | Clear recoverable upload journey and transparent moderation | Durable upload pipeline, idempotency, job states, error/retry flow, quotas | Medium/High |
| Admin auditability and announcement delivery | Admin actions only affect local browser state | Accountable moderation and actual campus communications | Server-side role policy, audit events, shared announcement service and notification delivery | High |
| Accessibility/responsive remediation | Tablet login clipping, compact text, unverified keyboard flows | More reliable access for keyboard, screen-reader, and small-screen users | WCAG review, responsive layout refinements, file-picker/ARIA fixes, assistive-technology testing | Medium |
| Student study collections and saved context | Basic saves do not support structured return-to-study workflows | More useful, organized library | Shared identity and resource persistence first; then collections/notes | Medium |

Avoid adding features that duplicate a full LMS until a college explicitly requests course operations, assignments, or grades. Campus’s strongest product scope is a trusted resource repository plus document-grounded study assistance.

## 19. Prioritized Development Roadmap

| Priority / phase | Current problem | Proposed solution | Expected outcome | Dependencies | Complexity |
|---|---|---|---|---|---|
| **P0 — Immediate risk containment** | `.env.example` may contain a real-looking credential | Privately verify it; revoke/rotate if valid; replace with a placeholder; scan distribution history before sharing | Prevent accidental API credential exposure | Secret owner/provider access | Low if invalid; urgent if valid |
| **P0 — Before pilot** | Demo login and client-side roles are not access control | Introduce server-backed identity and authorization; enforce role and institution access on all routes | Protect student uploads, admin actions, and AI quota | Identity and role decisions | High |
| **P0 — Before pilot** | Browser-local metadata/files are not shared or durable | Add production DB and private object storage with ownership/access checks, retention, backups, and migrations | Deliver the core one-stop shared library promise | Identity/tenant model, catalog definitions | High |
| **P1 — Core discovery** | Sample catalog and empty department options do not represent a complete campus | Confirm academic data; implement admin-managed catalog; show clear empty/unsupported states | Relevant results and trustworthy filters | College data owner | Medium |
| **P1 — Trust and quality** | Sample/optional ratings lack verified source and score explanation | Implement verified community/faculty review model and explain AI/community/faculty signals | Credible top-resource ranking | Identity, resource persistence, moderation policy | Medium/High |
| **P1 — Contribution workflow** | Upload, review, preview, and ownership are browser-local | Persist upload lifecycle, make retries safe, store file privately, support reviewer reasons and user-visible status | Reliable resource submission and moderation | DB/object storage/auth | High |
| **P1 — Document-aware Raze** | AI answers lack traceable PDF evidence and live quality evaluation | Return page citations/excerpts; test against representative course PDFs and known answers | More verifiable study help | PDF extraction, selected-resource API, evaluation set | Medium/High |
| **P2 — Library quality** | Search and sort are local substring/basic ranking | Add indexed full-text/faceted search, pagination, canonical URLs, and ranking explainability | Faster finding at larger scale | Shared resource DB and real catalog | Medium |
| **P2 — Operational governance** | No audit trail, data lifecycle management, or shared notification delivery | Record admin actions and provide controlled takedown, retention, and announcements | Accountable college operations | Identity, DB, policy decisions | Medium/High |
| **P2 — Accessibility/responsiveness** | 680px login clipping and incomplete accessibility verification | Test keyboard, screen reader, contrast, zoom, touch targets, and tablet breakpoints; remediate | More inclusive and robust product | Design/dev QA | Medium |
| **P3 — Personalization** | Current suggestions are global deterministic sorts | Add opt-in course profile and behavior-aware recommendations only after sufficient trusted data exists | More relevant recommendations | Verified identity, stable usage data, privacy consent | Medium/High |
| **P3 — Learning analytics** | No persisted practice progress | Offer user-controlled quiz history/mastery insights if students want it | Better follow-up study planning | Persistent identity/attempt model, privacy decisions | Medium |

**Suggested phase gates:** (1) secret and privacy decisions; (2) secure identity + persistence; (3) trustworthy content lifecycle/catalog; (4) measured search/ranking and cited AI; (5) optional personalization. Do not start with advanced AI-agent features while the underlying resource repository is local-only.

## 20. Existing Technical Debt

- Large monolithic `app.js`, `index.html`, and `styles.css`; UI/data/API/persistence concerns are tightly coupled.
- Duplicate/stale documentation: `PROJECT_REPORT_FOR_GEMINI.md` describes an earlier state and says Raze is a hardcoded mock, whereas the current source has Gemini-backed chat and study endpoints. It should be marked historical or updated before being used as engineering context.
- Firebase is a declared but unused dependency, creating ambiguity about intended architecture.
- Hardcoded catalog/sample resources, scores, reviewers, and demonstration credentials coexist with live upload/AI behaviors.
- UI strings, role checks, route navigation, resource schemas, and browser persistence are not centrally typed/validated.
- Local state versions are keyed in strings, but there is no migration strategy for browser data schema changes.
- No CI workflow, build/lint/typecheck scripts, production deployment configuration, or observability contract was found.
- No shared API contract documentation or generated client exists; frontend/server must stay manually synchronized.
- No audit log or model prompt/version history exists for institutional quality review.

## 21. Known Limitations and Unverified Areas

### Known from implementation

- No database, migrations, object storage, real authentication, server-backed roles, or cross-device persistence.
- Gemini calls require server configuration and live service availability.
- Seed curriculum/faculty reviews/resource scores are illustrative; taxonomy needs institutional confirmation.
- Resource recommendations and search are local and deterministic.
- AI API tests use mocks. The model was not called during this audit.
- `.env.example` showed a key-like non-placeholder value in inspection; validity is unknown and the value is deliberately not copied into this report.
- Current workspace has no `.git` metadata; no clean/diff baseline could be established.

### Not verified in this audit

- Real Gemini response quality, source grounding, latency, pricing, quota behavior, or provider data-retention terms.
- Real PDF upload/analysis/IndexedDB preview lifecycle with large, malformed, encrypted, or scanned files.
- Production hosting/TLS/reverse proxy, network binding, deployment secrets, backup/restore, or operational monitoring.
- Assistive-technology and screen-reader behavior; comprehensive keyboard-only testing; formal WCAG contrast.
- Full mobile/tablet cross-browser behavior. One 390px admin layout and 680px login layout were measured; the latter clipped by 8px.
- Whether any institutional catalog/user data exists outside this inspected project directory.

### Validation performed

- `npm test`: **18 tests passed, 0 failed**. Tests use Node’s built-in runner and mocked Gemini behavior.
- `node --check app.js`, `node --check server.mjs`, `node --check pdf-preview.js`: passed.
- Local browser walkthrough: landing/login, student Discover, Raze, Crates, admin dashboard, theme, and desktop screenshots; student/admin sign-in transitions observed.
- Browser viewport inspection: student desktop at 1266×912; admin at 390×844; login at widths 390, 680, 720, 780, and 1024. At 680px the login card extends beyond the viewport and is clipped.
- No live Gemini, external web research, load testing, Lighthouse run, or production security testing was performed.

## 22. Questions Requiring Product Decisions

1. Is access public, restricted to a verified college email, or restricted to institution-invited accounts?
2. Is the first launch single-college (Matrusri) or multi-institution? Who owns/updates the canonical academic catalog?
3. Which resource types are in scope: PDF only, external URLs, notes, question papers, presentations, or faculty course packs?
4. What sharing and copyright/license rules must apply? Who handles infringement claims and takedowns?
5. Who can issue a faculty score, what rubric is used, and how should community, faculty, and AI signals combine?
6. Should students be able to rate only after opening a resource or confirming they used it? How are abuse and brigading handled?
7. What exactly does the mandatory AI score measure, how should it be calibrated, and should repetition/completeness/clarity appear as separate dimensions?
8. May uploaded PDFs be sent to Gemini? What consent, deletion, retention, and provider processing terms are acceptable?
9. Should Raze conversations be saved to student accounts, remain ephemeral, or be user-controlled? Should generated practice sets persist?
10. What file-size, retention, and access policy is required for approved and rejected uploads?
11. Is Campus meant to replace only scattered resource links, or eventually provide LMS features such as assignments/grades? Keep those scopes separate unless evidence supports expansion.
12. What minimum accessibility target and supported browser/device matrix should release meet?

## 23. Final Product Assessment

Campus has a strong visual/product concept and an unusually broad prototype for a small codebase: discovery, PDF preview, moderation, Crates, notifications, theme switching, and model-backed study assistance are all represented. The best current product direction is a **trusted campus resource library with transparent quality signals and document-grounded study tools**.

The main risk is the gap between the polished UI and the local-only implementation. A student can upload a PDF and see an AI-derived score, but classmates cannot reliably retrieve it from their own devices; an admin can approve a material, but the approval is not enforced by a server; a rating exists, but it is not a campus-wide verified rating. This is a prototype boundary, not a small polish issue.

Build the durable trust layer first: verify/rotate the possible template credential, decide identity and institutional scope, add server-enforced authorization, persist resources and metadata, store files privately, and define moderation/licensing policy. Then validate search/ranking with real faculty-approved data. Add citations and retrieval quality to Raze after the resource corpus is dependable. That sequence best supports the stated student value without overbuilding a generic LMS or agent platform.

## GPT HANDOFF — PRODUCT DISCUSSION CONTEXT

Campus is a student-centered academic resource-library prototype for finding, sharing, rating, and studying campus materials. The intended users are students, faculty/contributors, and college administrators. Its product philosophy is one trusted place for course resources plus a focused academic assistant, rather than a general LMS.

The current system is a plain HTML/CSS/JavaScript client served by a small Node HTTP server. The browser stores resource metadata, scores, saved items, Crates, announcements, theme, and uploaded PDFs locally (`localStorage`, `sessionStorage`, `IndexedDB`). There is no database or shared file store. The declared Firebase dependency is not an active integration. Four Node endpoints call Gemini for PDF analysis, Raze chat, summaries, and practice tools; the API tests use mocked responses.

Implemented experiences include student/admin demo sign-in and separate workspaces, search and academic filters, suggested/new feeds, resource cards/details, local PDF.js preview, save/rate controls, local upload/moderation, Crates, announcements/notifications, light/dark themes, and Raze with PDF-grounded study aids. Incomplete foundations include real identity and authorization, shared storage, verified catalog data, durable moderation, cross-user ratings, accessible end-to-end PDF lifecycle, and cited/quality-evaluated AI answers.

The design is an editorial, green-accented student workspace with serif headings and light/dark themes. Its biggest UX issue is that the polished shared-library promise is not met by local-only data and sample content; some department filters are empty. Its biggest technical risk is client-only roles plus unauthenticated AI APIs. Its most important next decisions concern institution access, catalog ownership, copyright/retention, rating policy, and whether Raze conversations persist. The audit ran 18 passing tests and syntax checks, inspected main desktop/student/admin flows, and did not call live Gemini or verify a production deployment.
