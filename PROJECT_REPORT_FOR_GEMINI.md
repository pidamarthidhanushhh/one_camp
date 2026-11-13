# Campus Study-Resource Platform — Project Report for Database Schema Design

## Purpose

Campus is a student-focused platform for finding, sharing, and studying from academic resources in one place. The intended audience includes MBBS/medical, BTech/engineering, and other degree students. It should replace scattered notes, PDFs, old Drive links, and chat attachments with a searchable, community-rated resource library.

Use this report to propose a production-ready relational database schema. Distinguish current prototype behavior from requirements that still need implementation. Do not assume that sample data or UI previews are live production data.

## Current implementation

- Responsive desktop/mobile resource discovery dashboard with Discover, My Library, and My Uploads views.
- Client-side search across resource title, subject, course/topic, and tags.
- Default college context is Matrusri Engineering College. The discovery filter supports department (CSE/Civil/EEE/ECE), year (1–4), section, semester (1–6), and subject. CSE sections use the user-provided list; other department sections remain unconfirmed, with "All sections" available for shared resources.
- User-provided CSE subjects are seeded for all four years. Civil, EEE, and ECE have illustrative, clearly marked sample subjects. Mapping is Year 1 → Sem 1–2, Year 2 → Sem 3–4, Year 3 → Sem 5–6; Year 4 semester mappings remain pending college confirmation.
- Default discovery ordering prioritizes faculty-reviewed resources by faculty score, then overall/community score. Seed faculty ratings are explicitly demo values, not actual staff reviews.
- Upload metadata includes department/year/section/semester as well as the AI-inferred editable subject.
- Crates page demonstrates faculty/admin crate creation, generated join codes, student joining, class-scope filtering, and resource access. Example code: `MEC-CSE2`. Authentication and cross-device sharing are not implemented; crate records and memberships are browser-local prototype state.
- Resource cards and details display title, subject, course/topic, file format, author, tags, AI score, optional community rating, and optional faculty rating. A full-PDF quick summary is created only on request.
- Save/unsave resources to the student's library.
- Upload dialog currently accepts PDF files up to 12 MB.
- Upload sends PDF bytes to a same-origin Node.js endpoint, which calls Gemini server-side. The API key is read from the server environment and is never part of browser code.
- Gemini suggests resource title, specific subject, course/topic, tags, topics covered, focus topics, and an AI quality score during upload. The student can edit the suggestions before adding the resource; no summary is generated during this step.
- Clicking an uploaded resource opens a PDF.js-rendered canvas preview of the original PDF, with page navigation, zoom controls, and extracted topic/focus lists. A separate button requests a concise AI summary of the full PDF, retained in browser storage after generation. This avoids reliance on inconsistent native embedded PDF viewers.
- Uploaded PDFs are kept as browser-local Blobs in IndexedDB for previews on that device; resource metadata and any requested AI summary are in browser local storage. Neither is shared cloud storage.
- College taxonomy, mock curricula, crates, crate memberships, and uploaded files are local to the current browser origin. They need a server/database/storage backend before real classroom sharing.
- A clearly labeled Python Basics prototype resource and uploaded resources identified as Python materials provide the same on-demand summary and curated interactive quiz/flashcards, limited to Python Basics, Installation, Data types (int, float, bool, string, list), and Variables and Expressions. The quiz/cards are fixed demo content, not model-generated. Closing the preview does not delete uploaded PDFs; browser storage is origin-specific.
- Raze is a floating study-buddy UI. Its current quiz, flashcard, and chat answers are hardcoded examples, not a connected AI chat product.
- The project has a small Node.js HTTP server, PDF.js (`pdfjs-dist`) for browser-side PDF rendering, Firebase in the dependency manifest, and API/static-file validation tests. The PDF.js version requires Node.js 22.13 or newer (or Node.js 24+).

## Current prototype limitations (important)

- There is **no database** and no authentication/user system.
- Six discovery resources plus one Python Basics prototype resource are hardcoded in `app.js`; seed scores, review counts, faculty ratings, summaries, and authors are illustrative.
- Uploaded resource metadata and saved resource IDs are stored in browser `localStorage`, scoped to one browser. They are not shared across students or devices.
- The original PDF is sent to Gemini for analysis and then retained only in browser IndexedDB for local previews. There is no object-storage integration, permanent shared download/view URL, or upload history on a server.
- The current form requires an analyzed PDF before a resource can be submitted. Link/external-resource uploads are not implemented yet.
- There are no student/community rating or faculty-rating submission workflows, moderation tools, faculty verification, or aggregate-rating database calculations yet.
- AI scores for initial sample resources are hardcoded; live Gemini analysis is wired for uploaded PDFs. Current score prompt assesses topical coverage, clarity, concise organization, and repetition. It does not perform online-reference checking.
- Resource discovery sorting currently computes a normalized combined score client-side. It is not a database ranking or recommendation service.
- The assistant has no user/session persistence or live model-backed chat.

## Current resource fields

The browser's resource objects currently use:

| Field | Current meaning |
|---|---|
| `id` | Prototype string identifier |
| `title` | Resource title; AI suggestion can be edited |
| `subject` | Specific discipline/subject inferred from the PDF; editable free text |
| `course` | Course name/code or most specific topic/chapter; editable free text |
| `format` | Display label such as PDF and page count/review status |
| `summary` | Legacy/sample description; uploads leave this blank until a quick summary is requested |
| `quickSummary` | Optional on-demand AI summary of an uploaded PDF or the Python Basics demo topics |
| `tags` | Up to six AI-generated topical labels |
| `topicsCovered` | AI-extracted document topics, up to 20 entries |
| `focusTopics` | AI-prioritized study topics, up to 8 entries |
| `college` | Prototype college, currently Matrusri Engineering College |
| `department` | CSE, Civil, EEE, or ECE |
| `year` | Academic year 1–4 |
| `section` | Department section or `All sections` for shared sample material |
| `semester` | Semester 1–6; null while Year 4 semester mapping is unconfirmed |
| `facultyReviewed` / `facultyReviewer` | Prototype review state and reviewer label; seed reviews are illustrative only |
| `aiScore` | AI quality score from 0 to 100 |
| `rating` | Optional community average from 0 to 5; sample data only so far |
| `ratings` | Community-rating count; sample data only so far |
| `facultyScore` | Optional faculty score from 0 to 5; sample data only so far |
| `author` | Display name; currently hardcoded/sample, not linked to an account |
| `cover` | UI illustration category, not academic metadata |
| `isUploaded` | Local UI flag for a browser-only upload |

Database design should not store a calculated average/count as the only source of truth. Preserve individual rating submissions and derive or safely maintain aggregates.

## Score and discovery behavior to account for

The current UI uses a provisional combined score:

- AI quality: 60% weight, normalized from 0–100.
- Community rating: 25% weight, normalized from 0–5 to 0–100.
- Faculty rating: 15% weight, normalized from 0–5 to 0–100.
- If community or faculty ratings are absent, the weights of available scores are renormalized. AI quality is expected for a published, successfully analyzed upload.

The schema should preserve the independent score inputs, their optionality, who supplied them, timestamps, and AI analysis provenance/version. Recommend whether to calculate rankings at query time or maintain derived aggregates, and explain how aggregates are refreshed consistently.

## Schema proposal requested

Please propose a PostgreSQL-first relational schema for the production product, including:

1. **Users and roles:** students, faculty, administrators/moderators; faculty verification/status; created/updated/deactivated timestamps. Support a student belonging to a college/university and degree/program/cohort if appropriate.
2. **Institutions and academic catalog:** institution, program/degree, course, and flexible subjects/topics. MBBS and engineering course structures differ, so avoid a fixed subject enum. Support free-text AI suggestions initially, later normalized subject/course matching, synonyms, and institution-specific course codes.
3. **Resources:** uploader/owner, title, subject/course association or flexible text, description/AI summary, content type, lifecycle/moderation status, visibility, created/updated/published timestamps, source type (uploaded file or external link), optional canonical URL, and soft deletion/archive if appropriate.
4. **Files/object storage:** metadata only in the relational DB (object-storage key, original filename, MIME type, size, checksum, upload status, page count, storage timestamps). Do not store large PDF binaries directly in normal resource rows. Design for private storage and authorized downloads.
5. **AI analysis history:** one resource may have multiple analysis attempts/versions. Store provider/model, prompt/schema version, status, timestamps, suggested title/subject/course/summary/tags/topics-covered/focus-topics, quality dimensions or score breakdown if available, total AI score, error state, and which suggestions the student accepted/edited. Do not store API keys or secrets.
6. **Ratings:** individual community ratings with user/resource relationship, score, optional review text, timestamps, edit policy, uniqueness/anti-spam constraints, and moderation state. Separate optional faculty review/ratings, reviewer verification, rubric, and comments if faculty scoring differs from student ratings. Decide whether students may submit faculty feedback or only verified faculty may issue faculty scores; document the choice.
7. **Saves/library:** user-resource bookmarks with uniqueness and timestamps.
8. **Tags/search:** searchable title, summary, course/subject and extracted text; resource-tag association; indexes for common filters, quality/rating ordering, and full-text search. Support efficient faceted search and pagination.
9. **Auditing/moderation:** resource review status, reports/flags, takedown history, and audit metadata if appropriate.
10. **Optional future assistant data:** chat sessions/messages and links to resources or generated quizzes/flashcards only if justified. The current assistant is a UI mock, so keep these entities optional and separate from the core resource schema.

## Output format requested from Gemini

Provide:

- An entity-relationship overview and assumptions.
- PostgreSQL `CREATE TABLE` DDL with primary keys, foreign keys, checks, unique constraints, indexes, and timestamps.
- Example queries for resource search/ranking, a student's saved library, uploaded resources, rating aggregates, and optional faculty ratings.
- A short explanation of transaction/consistency strategy for rating averages and counts.
- A migration plan from the current browser prototype/sample data.
- Separate core MVP tables from optional later-phase tables.
- Clear notes on fields that should be enums versus flexible text or foreign keys.

## Security and privacy constraints

- Never put Gemini API credentials in the schema, browser, SQL, or generated report.
- Store PDFs in private object storage and keep only storage metadata/keys in the database.
- Define authorization so users can only manage their own uploads, saves, and ratings; verified faculty permissions should be explicit.
- Consider student privacy, institutional separation, rate limits, file retention/deletion, and auditability.

## Project files (for implementation context)

- `index.html` — dashboard and upload/chat UI.
- `styles.css` — responsive visual design.
- `app.js` — sample resources, client-side search/sort/saves, upload form and editable AI suggestion behavior.
- `server.mjs` — static app server and `/api/analyze-pdf` Gemini endpoint.
- `server.test.mjs` — API validation tests with mocked Gemini responses.
- `README.md` — local startup and configuration notes.
