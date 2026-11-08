# Backstage Phases 3 and 5 (Brain and Memory) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn the existing listen loop (system audio → cloud STT → one answer) into an interview copilot that detects questions, streams a one-line key-points answer and then a full styled answer, grounded in Akshay's profile and his Google prep pack, and keeps every session for a debrief.

**Architecture:** The brain lives in TypeScript under `src/lib/brain/` because the transcript pipeline still runs in the webview (Rust owns audio only after Phase 2); the spec's Rust `brain/` module moves with the audio in Phase 2. Native streaming LLM clients live in `src/lib/llm/` beside the existing curl-template path. Profile, sessions, utterances and answers are SQLite tables added as migration 3. The speech page gains an answer pane with two tiers, style chips and a latency chip.

**Tech Stack:** React 19 + TypeScript 5.8, vitest, tauri-plugin-sql (sqlite), `@tauri-apps/plugin-http` fetch for streaming, existing `tauri-plugin-global-shortcut` wiring (custom action ids are forwarded to the frontend as `custom-shortcut-triggered`).

**Spec:** `/Users/akshay/Documents/GitHub/OpenCluely/docs/superpowers/specs/2026-10-27-merged-copilot-design.md` (sections "What wins over Verve" 4, 5, 7, 8; Architecture `brain/`, `llm/`, `storage`; Phases 3 and 5; "Calls made after the builder check").

Phase 1 is done at `/Users/akshay/Documents/GitHub/backstage` (branch `main`, head `d0ed65b`). All work in this plan happens in that repo.

## Global Constraints

- Repo `/Users/akshay/Documents/GitHub/backstage`, branch `main`. Commands: `npm test` (vitest, runs `scripts/copy-vad-assets.mjs` first), `npx tsc --noEmit`, `cd src-tauri && cargo test`.
- No network call the user did not configure. The guard test `tests/no-phone-home.test.ts` stays green; no new CDN or hosted endpoint anywhere under `src/`.
- API keys only via `src/lib/storage/secrets.ts` (`getSecret("ai", providerId, "api_key")`), never in localStorage, SQLite or logs. Native providers use ids `native-gemini`, `native-anthropic`, `native-openai`, `native-ollama`.
- Spec latency target: first token of the answer under one second after the interviewer stops talking, measured and shown in the UI. In this phase "stops talking" is the arrival of the Rust `speech-detected` event (the end-of-utterance from Pluely's VAD); STT time counts against the budget.
- Generation queue semantics (spec "Calls made after the builder check"): one job in flight; a two-tier answer is one job; a new non-duplicate question cancels and replaces the running job and the partial answer is kept marked `interrupted`; dedupe window 20 s; nothing queues behind.
- Question-detection modes (spec win 4): `questions`, `every_pause`, `manual`, `transcript_only`.
- Style presets (spec win 5): `concise`, `star`, `technical`, `code`. Fast tier is one line of key points, at most 25 words.
- Shortcuts (spec): Cmd+Enter answer now, Cmd+1..4 style presets, Cmd+[ / Cmd+] shorten/lengthen. Existing Pluely ids stay; new ids are `answer_now`, `style_concise`, `style_star`, `style_technical`, `style_code`, `answer_shorter`, `answer_longer`.
- Profile fields (spec win 7): resume, job description, STAR stories, custom instructions. One profile row (`id = 1`).
- First profile is seeded from `/Users/akshay/Documents/GitHub/ai-job-search/documents/applications/google_software_engineer_iii_aiml_genai_google_cloud_ai/interview_prep_phone_screen.md`. That file stays where it is; it is imported through the UI, never copied into the repo.
- Phase 3 done: the "they ask, overlay answers" loop hits the latency target on a real call. Phase 5 done: a saved STAR story is used verbatim in an answer and a debrief is generated from a finished session.

## Review Focus

1. The interviewer says "Okay, so, um, tell me about a time you disagreed with a teammate" and STT returns it as two segments. The second segment alone must still classify as a question and the first must not fire a generation. (Test in Task 4.)
2. The fast tier answers in 400 ms and the full tier is still streaming when the interviewer asks a follow-up. The follow-up must cancel the full stream, keep the partial text marked interrupted in the session, and start a new job; the fast tier for the new question must not wait for the old stream to drain. (Test in Task 5 and Task 7.)
3. The primary model returns HTTP 429 mid-stream after sending nothing. The fallback chain must move to the next model without surfacing an error line into the answer pane. (Test in Task 6.)
4. The profile contains a 2,000-word prep pack. The system prompt must stay under a fixed budget and the STAR stories must survive truncation ahead of the likely-questions list, because the stories are what the answers quote. (Test in Task 2.)
5. A session is open when the app is quit. On next launch the session must appear in the list with an `ended_at` filled from the last utterance, and a debrief must still be generatable. (Test in Task 10.)

---

### Task 1: SQLite migration 3 and the profile and session data layer

**Files:**
- Create: `src-tauri/src/db/migrations/profile-sessions.sql`
- Modify: `src-tauri/src/db/main.rs` (add `Migration { version: 3, description: "create_profile_and_session_tables", kind: Up }`)
- Create: `src/lib/database/profile.action.ts`, `src/lib/database/session.action.ts`
- Modify: `src/lib/database/index.ts` (export both)
- Create: `src/types/brain.ts` (exported from `src/types/index.ts`)
- Test: `tests/migration-3.test.ts`

**Interfaces:**
- Produces, in `src/types/brain.ts`:
  ```ts
  export type Speaker = "them" | "me";
  export type AnswerTier = "fast" | "full";
  export type AnswerStyle = "concise" | "star" | "technical" | "code";
  export type AnswerLength = "shorter" | "normal" | "longer";
  export type BrainMode = "questions" | "every_pause" | "manual" | "transcript_only";
  export interface Profile { resume: string; jobDescription: string; stories: string; instructions: string; updatedAt: number }
  export interface Session { id: string; title: string; startedAt: number; endedAt: number | null; debrief: string | null }
  export interface Utterance { id: string; sessionId: string; speaker: Speaker; text: string; t0: number; t1: number }
  export interface Answer { id: string; sessionId: string; utteranceId: string; tier: AnswerTier; style: AnswerStyle; model: string; text: string; latencyMs: number | null; interrupted: boolean; createdAt: number }
  ```
- Produces, in `profile.action.ts`: `getProfile(): Promise<Profile>` (empty strings when no row), `saveProfile(p: Omit<Profile,"updatedAt">): Promise<Profile>` (upsert `id = 1`).
- Produces, in `session.action.ts`: `startSession(title: string): Promise<Session>`, `endSession(id: string, endedAt?: number): Promise<void>`, `addUtterance(u: Omit<Utterance,"id">): Promise<Utterance>`, `addAnswer(a: Omit<Answer,"id"|"createdAt">): Promise<Answer>`, `updateAnswer(id: string, patch: Partial<Pick<Answer,"text"|"latencyMs"|"interrupted">>): Promise<void>`, `listSessions(): Promise<Session[]>` (newest first), `getSessionTranscript(id: string): Promise<{ session: Session; utterances: Utterance[]; answers: Answer[] }>`, `saveDebrief(id: string, debrief: string): Promise<void>`, `closeStaleSessions(): Promise<number>` (sets `ended_at` = max utterance `t1` for sessions with `ended_at IS NULL`; returns count).
- Consumes: `getDatabase()` from `src/lib/database/config.ts`, same pattern as `chat-history.action.ts`.

- [ ] **Step 1: Write the failing test**

```ts
// tests/migration-3.test.ts
import { readFileSync } from "node:fs";
import { describe, it, expect } from "vitest";
const sql = readFileSync("src-tauri/src/db/migrations/profile-sessions.sql", "utf8");
const main = readFileSync("src-tauri/src/db/main.rs", "utf8");
describe("migration 3", () => {
  it("is registered as version 3", () => {
    expect(main).toMatch(/version:\s*3/);
    expect(main).toContain('include_str!("migrations/profile-sessions.sql")');
  });
  it.each([
    ["profile", ["id INTEGER PRIMARY KEY CHECK (id = 1)", "resume TEXT", "job_description TEXT", "stories TEXT", "instructions TEXT", "updated_at INTEGER"]],
    ["sessions", ["id TEXT PRIMARY KEY", "title TEXT", "started_at INTEGER", "ended_at INTEGER", "debrief TEXT"]],
    ["utterances", ["session_id TEXT NOT NULL", "speaker TEXT NOT NULL CHECK(speaker IN ('them', 'me'))", "text TEXT NOT NULL", "t0 INTEGER NOT NULL", "t1 INTEGER NOT NULL"]],
    ["answers", ["utterance_id TEXT NOT NULL", "tier TEXT NOT NULL CHECK(tier IN ('fast', 'full'))", "style TEXT NOT NULL", "model TEXT NOT NULL", "latency_ms INTEGER", "interrupted INTEGER NOT NULL DEFAULT 0"]],
  ])("creates %s with the spec columns", (table, columns) => {
    expect(sql).toContain(`CREATE TABLE IF NOT EXISTS ${table}`);
    for (const c of columns) expect(sql).toContain(c);
  });
  it("indexes utterances and answers by session", () => {
    expect(sql).toContain("idx_utterances_session");
    expect(sql).toContain("idx_answers_session");
  });
});
```

- [ ] **Step 2: Run to verify it fails** — `npm test -- migration-3` → FAIL, file not found.

- [ ] **Step 3: Write the SQL and register it**

Tables exactly as the test spells the columns; `utterances.session_id` and `answers.session_id`/`answers.utterance_id` are foreign keys with `ON DELETE CASCADE`; `answers.created_at INTEGER NOT NULL`.

- [ ] **Step 4: Write `src/types/brain.ts`, `profile.action.ts`, `session.action.ts`** with the signatures in Interfaces. Ids are `${Date.now()}-${Math.random().toString(36).slice(2, 9)}` like chat history.

- [ ] **Step 5: Verify** — `npm test` PASS; `npx tsc --noEmit` clean; `cd src-tauri && cargo test` 3 passed.

- [ ] **Step 6: Commit** — `git add -A && git commit -m "feat(db): profile, sessions, utterances and answers tables with data layer"`

---

### Task 2: Prep-pack importer and prompt builder

**Files:**
- Create: `src/lib/brain/prep-pack-importer.ts`, `src/lib/brain/prompt-builder.ts`, `src/lib/brain/styles.ts`, `src/lib/brain/index.ts`
- Test: `tests/prep-pack-importer.test.ts`, `tests/prompt-builder.test.ts`, `tests/fixtures/prep-pack.md` (a synthetic 12-line file with the same headings as the real pack: `# Title`, `## 1. The 4-week plan`, `## 2. Likely questions (priority order)`, `## 3. STAR answers`, `## 4. Tough questions, customized`, `## 7. Consistency brief (what the interviewers read)`, `## 8. Questions to ask`; invented content, no real facts)

**Interfaces:**
- Produces: `importPrepPack(markdown: string): Omit<Profile, "updatedAt">` mapping headings by number, case-insensitive: section 7 → `resume`, section 3 → `stories`, sections 2 + 4 + 8 (concatenated with their headings) → `instructions`, `jobDescription` ← the line after the `#` title if it starts with "Role:" or "Target:" else `""`. Unknown headings are ignored. A file with none of the numbered headings returns the whole text as `instructions`.
- Produces: `STYLE_PRESETS: Record<AnswerStyle, { label: string; instruction: string }>` and `LENGTH_INSTRUCTION: Record<AnswerLength, string>` in `styles.ts`. Exact copy:
  - concise: "Answer in 3 to 5 short sentences a person can say aloud. No headers, no bullets."
  - star: "Answer as a STAR story: Situation, Task, Action, Result, one short paragraph each, using the candidate's own stories when one fits."
  - technical: "Answer as a senior engineer would in an interview: name the approach, the trade-off, and the complexity. Short paragraphs."
  - code: "Answer with a short explanation followed by one code block in the requested language. No prose after the code."
  - shorter: "Halve the length of a normal answer." normal: "" longer: "Add one more concrete detail or example than a normal answer."
- Produces: `buildSystemPrompt(input: { profile: Profile; style: AnswerStyle; length: AnswerLength; tier: AnswerTier; language: string; budgetChars?: number }): string` in `prompt-builder.ts`. Default `budgetChars` 12000. Order of blocks: role line, style/length instruction (fast tier instead gets exactly "Reply with one line of key points, at most 25 words, no preamble."), `## Candidate facts` (resume), `## Target role` (jobDescription), `## Candidate stories` (stories), `## Candidate instructions` (instructions), language line. Truncation when over budget drops from the end of `instructions` first, then `resume`, then `jobDescription`; `stories` is never truncated.
- Consumes: `Profile`, `AnswerStyle`, `AnswerLength`, `AnswerTier` from Task 1.

- [ ] **Step 1: Write the failing tests**

```ts
// tests/prep-pack-importer.test.ts
import { readFileSync } from "node:fs";
import { describe, it, expect } from "vitest";
import { importPrepPack } from "../src/lib/brain/prep-pack-importer";
const md = readFileSync("tests/fixtures/prep-pack.md", "utf8");
describe("importPrepPack", () => {
  it("maps the consistency brief to resume facts", () => {
    expect(importPrepPack(md).resume).toContain("Consistency brief");
  });
  it("maps STAR answers to stories", () => {
    expect(importPrepPack(md).stories).toContain("STAR answers");
    expect(importPrepPack(md).stories).not.toContain("4-week plan");
  });
  it("maps likely, tough and ask-back questions to instructions", () => {
    const p = importPrepPack(md);
    expect(p.instructions).toContain("Likely questions");
    expect(p.instructions).toContain("Tough questions");
    expect(p.instructions).toContain("Questions to ask");
  });
  it("falls back to instructions for an unstructured file", () => {
    expect(importPrepPack("just some notes").instructions).toBe("just some notes");
  });
});
```

```ts
// tests/prompt-builder.test.ts
import { describe, it, expect } from "vitest";
import { buildSystemPrompt } from "../src/lib/brain/prompt-builder";
const profile = { resume: "R".repeat(100), jobDescription: "J".repeat(100), stories: "S".repeat(100), instructions: "I".repeat(100), updatedAt: 0 };
describe("buildSystemPrompt", () => {
  it("includes every profile block and the style instruction", () => {
    const p = buildSystemPrompt({ profile, style: "star", length: "normal", tier: "full", language: "english" });
    for (const h of ["## Candidate facts", "## Target role", "## Candidate stories", "## Candidate instructions"]) expect(p).toContain(h);
    expect(p).toContain("Situation, Task, Action, Result");
  });
  it("fast tier asks for one line of at most 25 words", () => {
    expect(buildSystemPrompt({ profile, style: "concise", length: "normal", tier: "fast", language: "english" })).toContain("at most 25 words");
  });
  it("truncates instructions first and never the stories", () => {
    const big = { ...profile, instructions: "I".repeat(5000), stories: "S".repeat(3000) };
    const p = buildSystemPrompt({ profile: big, style: "concise", length: "normal", tier: "full", language: "english", budgetChars: 4000 });
    expect(p.length).toBeLessThanOrEqual(4000);
    expect(p).toContain("S".repeat(3000));
  });
  it("applies the length modifier", () => {
    expect(buildSystemPrompt({ profile, style: "concise", length: "shorter", tier: "full", language: "english" })).toContain("Halve the length");
  });
});
```

- [ ] **Step 2: Run to verify they fail** — `npm test -- prep-pack prompt-builder` → FAIL, modules not found.
- [ ] **Step 3: Write the fixture, then implement the three modules** per Interfaces.
- [ ] **Step 4: Verify** — `npm test` PASS; `npx tsc --noEmit` clean.
- [ ] **Step 5: Commit** — `git commit -am "feat(brain): prep-pack importer, style presets and budgeted prompt builder"`

---

### Task 3: Profile page

**Files:**
- Create: `src/pages/profile/index.tsx`
- Modify: `src/routes/index.tsx` (route `/profile`), `src/hooks/useMenuItems.tsx` (menu entry "Profile", icon `UserIcon` from lucide, placed after Dashboard)

**Interfaces:**
- Consumes: `getProfile`, `saveProfile` (Task 1); `importPrepPack` (Task 2).
- Produces: a page with four textareas (Resume facts, Target role, STAR stories, Instructions), a Save button, an "Import markdown" button backed by a hidden `<input type="file" accept=".md,.txt">` read with `FileReader`, and a "Replace / Append" choice applied to the import (append joins with a blank line). Saves show a toast via the existing `sonner` usage if present, else inline "Saved".

- [ ] **Step 1: Build the page**, following `src/pages/responses/index.tsx` for `PageLayout` and `Header` usage.
- [ ] **Step 2: Verify** — `npx tsc --noEmit` clean; `npm run tauri dev`, open the dashboard (Cmd+Shift+D), Profile, import the Google prep pack file, Save, quit, relaunch, confirm the four fields persist.
  Expected: Stories field starts with "## 3. STAR answers".
- [ ] **Step 3: Commit** — `git add -A && git commit -m "feat(profile): editor page with markdown import"`

---

### Task 4: Question detector

**Files:**
- Create: `src/lib/brain/question-detector.ts`
- Test: `tests/question-detector.test.ts`

**Interfaces:**
- Produces:
  ```ts
  export type UtteranceKind = "question" | "backchannel" | "statement";
  export interface Classification { kind: UtteranceKind; confidence: number; reason: string }
  export function classifyUtterance(text: string): Classification;
  export function normalizeForDedupe(text: string): string; // lowercase, strip punctuation and fillers (um, uh, so, okay, like), collapse spaces
  export function isDuplicate(text: string, recent: { text: string; at: number }[], now: number, windowMs?: number): boolean; // windowMs default 20000; duplicate when token Jaccard >= 0.8 with any recent entry inside the window
  export function shouldGenerate(mode: BrainMode, c: Classification, duplicate: boolean): boolean;
  ```
- Rules for `classifyUtterance`, in order: fewer than 3 words after filler stripping → `backchannel` (0.9) if the stripped text is in the backchannel list (`yeah`, `okay`, `right`, `got it`, `uh huh`, `mm hmm`, `sure`, `i see`, `makes sense`, `great`, `cool`, `thanks`, `thank you`) else `statement` (0.6); ends with `?` → `question` (0.95); starts with a wh-word or auxiliary inversion (`what|why|how|when|where|which|who|can you|could you|would you|do you|did you|have you|are you|is there|tell me|walk me through|describe|explain|give me an example|talk about`) → `question` (0.85); contains an embedded prompt (`i'd like to hear`, `i want to know`, `curious`) → `question` (0.7); otherwise `statement` (0.6).
- `shouldGenerate`: `questions` → kind is `question` and not duplicate; `every_pause` → kind is not `backchannel` and not duplicate; `manual` and `transcript_only` → false.

- [ ] **Step 1: Write the failing test**

```ts
// tests/question-detector.test.ts
import { describe, it, expect } from "vitest";
import { classifyUtterance, isDuplicate, shouldGenerate, normalizeForDedupe } from "../src/lib/brain/question-detector";
describe("classifyUtterance", () => {
  it.each([
    ["Tell me about a time you disagreed with a teammate", "question"],
    ["How would you design a rate limiter?", "question"],
    ["Okay, so, um,", "backchannel"],
    ["uh huh", "backchannel"],
    ["Great, that makes sense.", "backchannel"],
    ["We use Kubernetes for all our batch jobs.", "statement"],
    ["I'd like to hear about your eval harness", "question"],
  ])("%s → %s", (text, kind) => {
    expect(classifyUtterance(text).kind).toBe(kind);
  });
  it("classifies the second half of a split question on its own", () => {
    expect(classifyUtterance("tell me about a time you disagreed with a teammate").kind).toBe("question");
    expect(classifyUtterance("Okay, so, um,").kind).toBe("backchannel");
  });
});
describe("isDuplicate", () => {
  const recent = [{ text: "Tell me about a time you disagreed with a teammate", at: 1000 }];
  it("flags a repeat inside 20 s", () => {
    expect(isDuplicate("Um, tell me about a time you disagreed with a teammate?", recent, 15000)).toBe(true);
  });
  it("lets the same question through after the window", () => {
    expect(isDuplicate("Tell me about a time you disagreed with a teammate", recent, 30000)).toBe(false);
  });
  it("does not flag a different question", () => {
    expect(isDuplicate("How would you design a rate limiter", recent, 2000)).toBe(false);
  });
  it("normalizes fillers and punctuation", () => {
    expect(normalizeForDedupe("Um, so, Tell me about it?")).toBe("tell me about it");
  });
});
describe("shouldGenerate", () => {
  const q = classifyUtterance("How would you design a rate limiter?");
  const s = classifyUtterance("We use Kubernetes for all our batch jobs.");
  it("questions mode fires on questions only", () => {
    expect(shouldGenerate("questions", q, false)).toBe(true);
    expect(shouldGenerate("questions", s, false)).toBe(false);
  });
  it("every_pause fires on statements too, never on duplicates", () => {
    expect(shouldGenerate("every_pause", s, false)).toBe(true);
    expect(shouldGenerate("every_pause", q, true)).toBe(false);
  });
  it("manual and transcript_only never fire", () => {
    expect(shouldGenerate("manual", q, false)).toBe(false);
    expect(shouldGenerate("transcript_only", q, false)).toBe(false);
  });
});
```

- [ ] **Step 2: Run to verify it fails** — `npm test -- question-detector` → FAIL, module not found.
- [ ] **Step 3: Implement** per the rules above.
- [ ] **Step 4: Verify** — `npm test` PASS.
- [ ] **Step 5: Commit** — `git add -A && git commit -m "feat(brain): heuristic question detector with dedupe and mode gate"`

---

### Task 5: Generation queue

**Files:**
- Create: `src/lib/brain/generation-queue.ts`
- Test: `tests/generation-queue.test.ts`

**Interfaces:**
- Produces:
  ```ts
  export interface GenerationJob { id: string; question: string; run: (signal: AbortSignal) => Promise<void> }
  export interface QueueEvents { onStart?: (job: GenerationJob) => void; onDone?: (job: GenerationJob) => void; onInterrupted?: (job: GenerationJob) => void; onRejectedDuplicate?: (question: string) => void }
  export class GenerationQueue {
    constructor(events?: QueueEvents, dedupeWindowMs = 20000, now: () => number = Date.now);
    submit(job: GenerationJob): "started" | "replaced" | "duplicate"; // replaced = a running job was aborted first
    cancel(): void;
    get current(): GenerationJob | null;
  }
  ```
- Semantics: `submit` aborts the current job's signal (firing `onInterrupted` for it), records the question in the dedupe list (using `normalizeForDedupe` and `isDuplicate` from Task 4), and starts the new job immediately without awaiting the old `run` promise. `onDone` fires only when `run` resolves and the job is still current. A rejected `run` that was aborted is not an error.

- [ ] **Step 1: Write the failing test**

```ts
// tests/generation-queue.test.ts
import { describe, it, expect, vi } from "vitest";
import { GenerationQueue } from "../src/lib/brain/generation-queue";
function job(id: string, question: string, log: string[]) {
  return { id, question, run: (signal: AbortSignal) => new Promise<void>((resolve) => {
    log.push(`start ${id}`);
    const t = setTimeout(() => { log.push(`done ${id}`); resolve(); }, 50);
    signal.addEventListener("abort", () => { clearTimeout(t); log.push(`abort ${id}`); resolve(); });
  }) };
}
describe("GenerationQueue", () => {
  it("runs one job and reports done", async () => {
    const log: string[] = []; const done = vi.fn();
    const q = new GenerationQueue({ onDone: done });
    expect(q.submit(job("a", "How would you design a rate limiter?", log))).toBe("started");
    await new Promise((r) => setTimeout(r, 80));
    expect(log).toEqual(["start a", "done a"]); expect(done).toHaveBeenCalledOnce();
  });
  it("a new question aborts the running job and starts at once", async () => {
    const log: string[] = []; const interrupted = vi.fn();
    const q = new GenerationQueue({ onInterrupted: interrupted });
    q.submit(job("a", "How would you design a rate limiter?", log));
    expect(q.submit(job("b", "Tell me about your eval harness", log))).toBe("replaced");
    expect(log).toEqual(["start a", "abort a", "start b"]);
    expect(interrupted).toHaveBeenCalledWith(expect.objectContaining({ id: "a" }));
    expect(q.current?.id).toBe("b");
  });
  it("rejects a duplicate inside the window and keeps the running job", () => {
    let now = 0; const log: string[] = [];
    const q = new GenerationQueue({}, 20000, () => now);
    q.submit(job("a", "How would you design a rate limiter?", log));
    now = 5000;
    expect(q.submit(job("b", "um how would you design a rate limiter", log))).toBe("duplicate");
    expect(q.current?.id).toBe("a");
  });
  it("accepts the same question after the window", () => {
    let now = 0; const log: string[] = [];
    const q = new GenerationQueue({}, 20000, () => now);
    q.submit(job("a", "How would you design a rate limiter?", log));
    now = 25000;
    expect(q.submit(job("b", "How would you design a rate limiter?", log))).toBe("replaced");
  });
});
```

- [ ] **Step 2: Run to verify it fails** — `npm test -- generation-queue` → FAIL.
- [ ] **Step 3: Implement** per Interfaces.
- [ ] **Step 4: Verify** — `npm test` PASS.
- [ ] **Step 5: Commit** — `git add -A && git commit -m "feat(brain): single-flight generation queue with cancel-and-replace and dedupe"`

---

### Task 6: Native streaming LLM clients and fallback chain

**Files:**
- Create: `src/lib/llm/types.ts`, `src/lib/llm/sse.ts`, `src/lib/llm/gemini.ts`, `src/lib/llm/anthropic.ts`, `src/lib/llm/openai.ts`, `src/lib/llm/fallback.ts`, `src/lib/llm/index.ts`
- Test: `tests/llm-sse.test.ts`, `tests/llm-clients.test.ts`, `tests/llm-fallback.test.ts`

**Interfaces:**
- Produces, in `types.ts`:
  ```ts
  export interface LlmMessage { role: "user" | "assistant"; content: string }
  export interface LlmRequest { system: string; messages: LlmMessage[]; maxTokens: number; temperature?: number; signal?: AbortSignal }
  export interface ModelTarget { providerId: "native-gemini" | "native-anthropic" | "native-openai" | "native-ollama"; model: string; baseUrl?: string; thinking?: boolean } // thinking false sends Gemini thinkingBudget 0
  export interface LlmClient { stream(target: ModelTarget, apiKey: string, req: LlmRequest, fetchImpl?: typeof fetch): AsyncIterable<string> }
  export class LlmHttpError extends Error { constructor(public status: number, message: string) }
  ```
- Produces, in `sse.ts`: `async function* sseEvents(body: ReadableStream<Uint8Array>, signal?: AbortSignal): AsyncIterable<{ event?: string; data: string }>` splitting on blank lines, joining multi-line `data:`; `[DONE]` is yielded as data and the caller stops.
- Produces: `geminiClient`, `anthropicClient`, `openaiClient: LlmClient`. Endpoints: Gemini `POST {baseUrl ?? "https://generativelanguage.googleapis.com"}/v1beta/models/{model}:streamGenerateContent?alt=sse` with header `x-goog-api-key`, body `{ systemInstruction: {parts:[{text}]}, contents: [...], generationConfig: { maxOutputTokens, temperature, thinkingConfig: { thinkingBudget: 0 } when thinking === false } }`, text at `candidates[0].content.parts[*].text`; Anthropic `POST {baseUrl ?? "https://api.anthropic.com"}/v1/messages` with `x-api-key`, `anthropic-version: 2023-06-01`, `stream: true`, text from `content_block_delta` events' `delta.text`; OpenAI-compatible `POST {baseUrl ?? "https://api.openai.com"}/v1/chat/completions` with `Authorization: Bearer`, `stream: true`, text at `choices[0].delta.content`; `native-ollama` uses `openaiClient` with default `baseUrl` `http://localhost:11434` and an empty key. Non-2xx responses throw `LlmHttpError(status, bodyText)` before yielding anything. `fetchImpl` defaults to `@tauri-apps/plugin-http` `fetch`.
- Produces, in `fallback.ts`: `async function* streamWithFallback(chain: ModelTarget[], req: LlmRequest, deps: { getKey: (providerId: string) => Promise<string>; clients?: Record<string, LlmClient>; onModelUsed?: (target: ModelTarget) => void }): AsyncIterable<string>` which tries targets in order, moves to the next on `LlmHttpError` with status 429 or 503 or on a network error before the first chunk, and rethrows anything else or any error after the first chunk; `onModelUsed` fires with the target that produced the first chunk.
- Consumes: `getSecret` for keys (passed in as `getKey` so tests need no mock).

- [ ] **Step 1: Write the failing tests**

```ts
// tests/llm-sse.test.ts
import { describe, it, expect } from "vitest";
import { sseEvents } from "../src/lib/llm/sse";
function body(text: string) { return new Blob([text]).stream() as ReadableStream<Uint8Array>; }
describe("sseEvents", () => {
  it("splits events and joins multi-line data", async () => {
    const out = [];
    for await (const e of sseEvents(body("event: ping\ndata: a\ndata: b\n\ndata: [DONE]\n\n"))) out.push(e);
    expect(out).toEqual([{ event: "ping", data: "a\nb" }, { data: "[DONE]" }]);
  });
  it("handles a chunk boundary inside a line", async () => {
    const chunks = ["data: {\"x\":", "1}\n\n"];
    const stream = new ReadableStream<Uint8Array>({ start(c) { for (const s of chunks) c.enqueue(new TextEncoder().encode(s)); c.close(); } });
    const out = []; for await (const e of sseEvents(stream)) out.push(e.data);
    expect(out).toEqual(['{"x":1}']);
  });
});
```

```ts
// tests/llm-clients.test.ts
import { describe, it, expect, vi } from "vitest";
import { geminiClient, anthropicClient, openaiClient } from "../src/lib/llm";
import { LlmHttpError } from "../src/lib/llm/types";
const req = { system: "sys", messages: [{ role: "user" as const, content: "hi" }], maxTokens: 50 };
function fakeFetch(status: number, sse: string, captured: any) {
  return vi.fn(async (url: string, init: any) => {
    captured.url = url; captured.init = init;
    return new Response(sse, { status, headers: { "content-type": "text/event-stream" } });
  }) as unknown as typeof fetch;
}
async function collect(it: AsyncIterable<string>) { const out = []; for await (const c of it) out.push(c); return out.join(""); }
describe("gemini", () => {
  it("streams text parts and sends thinkingBudget 0 when thinking is off", async () => {
    const cap: any = {};
    const f = fakeFetch(200, 'data: {"candidates":[{"content":{"parts":[{"text":"Hel"}]}}]}\n\ndata: {"candidates":[{"content":{"parts":[{"text":"lo"}]}}]}\n\n', cap);
    const text = await collect(geminiClient.stream({ providerId: "native-gemini", model: "gemini-2.5-flash", thinking: false }, "K", req, f));
    expect(text).toBe("Hello");
    expect(cap.url).toContain("/v1beta/models/gemini-2.5-flash:streamGenerateContent?alt=sse");
    expect(cap.init.headers["x-goog-api-key"]).toBe("K");
    expect(JSON.parse(cap.init.body).generationConfig.thinkingConfig.thinkingBudget).toBe(0);
  });
  it("throws LlmHttpError on 429 before yielding", async () => {
    const f = fakeFetch(429, "quota", {});
    await expect(collect(geminiClient.stream({ providerId: "native-gemini", model: "m" }, "K", req, f))).rejects.toBeInstanceOf(LlmHttpError);
  });
});
describe("anthropic", () => {
  it("streams content_block_delta text", async () => {
    const cap: any = {};
    const f = fakeFetch(200, 'event: content_block_delta\ndata: {"type":"content_block_delta","delta":{"type":"text_delta","text":"Hi"}}\n\nevent: message_stop\ndata: {"type":"message_stop"}\n\n', cap);
    expect(await collect(anthropicClient.stream({ providerId: "native-anthropic", model: "claude-sonnet-5-5" }, "K", req, f))).toBe("Hi");
    expect(cap.init.headers["x-api-key"]).toBe("K");
    expect(cap.init.headers["anthropic-version"]).toBe("2023-06-01");
    expect(JSON.parse(cap.init.body).system).toBe("sys");
  });
});
describe("openai-compatible", () => {
  it("streams choices delta content and stops at [DONE]", async () => {
    const cap: any = {};
    const f = fakeFetch(200, 'data: {"choices":[{"delta":{"content":"Yo"}}]}\n\ndata: [DONE]\n\n', cap);
    expect(await collect(openaiClient.stream({ providerId: "native-openai", model: "gpt-5" }, "K", req, f))).toBe("Yo");
    expect(cap.init.headers["Authorization"]).toBe("Bearer K");
  });
  it("ollama defaults to localhost:11434 with no auth header", async () => {
    const cap: any = {};
    const f = fakeFetch(200, "data: [DONE]\n\n", cap);
    await collect(openaiClient.stream({ providerId: "native-ollama", model: "llama3" }, "", req, f));
    expect(cap.url).toBe("http://localhost:11434/v1/chat/completions");
    expect(cap.init.headers["Authorization"]).toBeUndefined();
  });
});
```

```ts
// tests/llm-fallback.test.ts
import { describe, it, expect, vi } from "vitest";
import { streamWithFallback } from "../src/lib/llm/fallback";
import { LlmHttpError } from "../src/lib/llm/types";
const req = { system: "", messages: [], maxTokens: 10 };
const client = (behaviour: () => AsyncIterable<string>) => ({ stream: behaviour });
async function* ok(text: string) { yield text; }
async function* fail(status: number): AsyncIterable<string> { throw new LlmHttpError(status, "x"); }
async function collect(it: AsyncIterable<string>) { const out = []; for await (const c of it) out.push(c); return out.join(""); }
describe("streamWithFallback", () => {
  const chain = [{ providerId: "native-gemini" as const, model: "a" }, { providerId: "native-openai" as const, model: "b" }];
  it("skips a 429 and a 503 to the next model", async () => {
    const used = vi.fn();
    const text = await collect(streamWithFallback(chain, req, { getKey: async () => "k", onModelUsed: used,
      clients: { "native-gemini": client(() => fail(429)), "native-openai": client(() => ok("B")) } }));
    expect(text).toBe("B"); expect(used).toHaveBeenCalledWith(chain[1]);
  });
  it("rethrows a 401 without trying the next model", async () => {
    const next = vi.fn(() => ok("B"));
    await expect(collect(streamWithFallback(chain, req, { getKey: async () => "k",
      clients: { "native-gemini": client(() => fail(401)), "native-openai": client(next) } }))).rejects.toBeInstanceOf(LlmHttpError);
    expect(next).not.toHaveBeenCalled();
  });
  it("surfaces an error after the first chunk instead of switching", async () => {
    async function* partial(): AsyncIterable<string> { yield "A"; throw new Error("cut"); }
    await expect(collect(streamWithFallback(chain, req, { getKey: async () => "k",
      clients: { "native-gemini": client(partial), "native-openai": client(() => ok("B")) } }))).rejects.toThrow("cut");
  });
});
```

- [ ] **Step 2: Run to verify they fail** — `npm test -- llm` → FAIL, modules not found.
- [ ] **Step 3: Implement** per Interfaces. `index.ts` exports the three clients, `streamWithFallback`, and `clientFor(providerId): LlmClient`.
- [ ] **Step 4: Verify** — `npm test` PASS; `npx tsc --noEmit` clean; `tests/no-phone-home.test.ts` still green (the provider hosts are user-configured endpoints, not phone-home; they appear only in `src/lib/llm/`).
- [ ] **Step 5: Commit** — `git add -A && git commit -m "feat(llm): native Gemini, Anthropic and OpenAI-compatible streaming clients with 429/503 fallback"`

---

### Task 7: Two-tier answer orchestrator with latency timestamps

**Files:**
- Create: `src/lib/brain/answer-orchestrator.ts`
- Test: `tests/answer-orchestrator.test.ts`

**Interfaces:**
- Produces:
  ```ts
  export type AnswerEvent =
    | { type: "fast"; text: string; latencyMs: number; model: string }
    | { type: "full_chunk"; chunk: string }
    | { type: "full_first_token"; latencyMs: number; model: string }
    | { type: "full_done"; text: string }
    | { type: "interrupted"; partial: string }
    | { type: "error"; message: string };
  export interface AnswerInput { question: string; history: LlmMessage[]; profile: Profile; style: AnswerStyle; length: AnswerLength; language: string; stoppedAt: number; fastChain: ModelTarget[]; fullChain: ModelTarget[] }
  export interface AnswerDeps { stream: typeof streamWithFallback; getKey: (providerId: string) => Promise<string>; now?: () => number; clients?: Record<string, LlmClient> }
  export async function* answerQuestion(input: AnswerInput, signal: AbortSignal, deps: AnswerDeps): AsyncIterable<AnswerEvent>;
  ```
- Semantics: fast and full requests start concurrently (`Promise.race`-free: both streams are consumed; fast is collected to one string and emitted as one `fast` event with `latencyMs = now() - stoppedAt` at its first chunk; full emits `full_first_token` at its first chunk then `full_chunk`s). Fast uses `buildSystemPrompt({tier:"fast"})` with `maxTokens` 60; full uses `tier:"full"` with `maxTokens` 700 (`concise`/`shorter`: 350; `longer`: 1000). When `signal` aborts, yield `interrupted` with the full text so far and stop. A fast-tier failure is swallowed (the full answer still streams); a full-tier failure yields `error`.
- Consumes: `buildSystemPrompt` (Task 2), `streamWithFallback`, `ModelTarget`, `LlmMessage`, `LlmClient` (Task 6).

- [ ] **Step 1: Write the failing test**

```ts
// tests/answer-orchestrator.test.ts
import { describe, it, expect } from "vitest";
import { answerQuestion } from "../src/lib/brain/answer-orchestrator";
const profile = { resume: "", jobDescription: "", stories: "STORY-9 retry storm", instructions: "", updatedAt: 0 };
const base = { question: "Tell me about a production incident", history: [], profile, style: "star" as const, length: "normal" as const, language: "english", stoppedAt: 1000,
  fastChain: [{ providerId: "native-gemini" as const, model: "fast" }], fullChain: [{ providerId: "native-gemini" as const, model: "full" }] };
function streamFor(map: Record<string, (req: any) => AsyncIterable<string>>) {
  return (chain: any[], req: any) => map[chain[0].model](req);
}
async function* words(ws: string[], delay = 0) { for (const w of ws) { if (delay) await new Promise((r) => setTimeout(r, delay)); yield w; } }
async function collect(it: AsyncIterable<any>) { const out = []; for await (const e of it) out.push(e); return out; }
describe("answerQuestion", () => {
  it("emits the fast line with its latency, then full chunks, then done", async () => {
    let t = 1000; const now = () => (t += 100);
    const events = await collect(answerQuestion(base, new AbortController().signal, { getKey: async () => "k", now,
      stream: streamFor({ fast: () => words(["Key ", "points"]), full: () => words(["Full ", "answer"]) }) as any }));
    const fast = events.find((e) => e.type === "fast");
    expect(fast).toMatchObject({ text: "Key points", model: "fast" }); expect(fast.latencyMs).toBeGreaterThan(0);
    expect(events.filter((e) => e.type === "full_chunk").map((e) => e.chunk)).toEqual(["Full ", "answer"]);
    expect(events.at(-1)).toEqual({ type: "full_done", text: "Full answer" });
  });
  it("sends the STAR instruction and the stories in both system prompts", async () => {
    const systems: string[] = [];
    await collect(answerQuestion(base, new AbortController().signal, { getKey: async () => "k",
      stream: ((chain: any[], req: any) => { systems.push(req.system); return words(["x"]); }) as any }));
    expect(systems).toHaveLength(2);
    for (const s of systems) expect(s).toContain("STORY-9 retry storm");
    expect(systems.some((s) => s.includes("at most 25 words"))).toBe(true);
    expect(systems.some((s) => s.includes("Situation, Task, Action, Result"))).toBe(true);
  });
  it("yields interrupted with the partial text when aborted mid-stream", async () => {
    const ac = new AbortController();
    const it = answerQuestion(base, ac.signal, { getKey: async () => "k",
      stream: streamFor({ fast: () => words(["k"]), full: () => words(["A", "B", "C"], 30) }) as any });
    const events: any[] = [];
    for await (const e of it) { events.push(e); if (e.type === "full_chunk" && e.chunk === "A") ac.abort(); }
    expect(events.at(-1)).toEqual({ type: "interrupted", partial: "A" });
  });
  it("still streams the full answer when the fast tier fails", async () => {
    async function* boom(): AsyncIterable<string> { throw new Error("fast down"); }
    const events = await collect(answerQuestion(base, new AbortController().signal, { getKey: async () => "k",
      stream: streamFor({ fast: boom, full: () => words(["ok"]) }) as any }));
    expect(events.some((e) => e.type === "error")).toBe(false);
    expect(events.at(-1)).toEqual({ type: "full_done", text: "ok" });
  });
});
```

- [ ] **Step 2: Run to verify it fails** — `npm test -- answer-orchestrator` → FAIL.
- [ ] **Step 3: Implement** per Interfaces.
- [ ] **Step 4: Verify** — `npm test` PASS; `npx tsc --noEmit` clean.
- [ ] **Step 5: Commit** — `git add -A && git commit -m "feat(brain): two-tier answer orchestrator with latency timestamps"`

---

### Task 8: Brain settings and the Models page

**Files:**
- Create: `src/lib/storage/brain-settings.storage.ts`
- Create: `src/pages/models/index.tsx`
- Modify: `src/routes/index.tsx` (route `/models`), `src/hooks/useMenuItems.tsx` (entry "Models and brain", icon `BrainIcon`, after Profile)
- Test: `tests/brain-settings.test.ts`

**Interfaces:**
- Produces:
  ```ts
  export interface BrainSettings { mode: BrainMode; style: AnswerStyle; length: AnswerLength; fastChain: ModelTarget[]; fullChain: ModelTarget[]; useNativeModels: boolean; keys: Record<string, "" | "<keychain>"> }
  export const DEFAULT_BRAIN_SETTINGS: BrainSettings; // mode "questions", style "concise", length "normal", useNativeModels false, fastChain [{native-gemini, gemini-2.5-flash-lite, thinking:false}], fullChain [{native-gemini, gemini-2.5-flash}, {native-anthropic, claude-sonnet-5-5}]
  export function getBrainSettings(storage?: Pick<Storage,"getItem">): BrainSettings; // localStorage key "brain_settings"; merges over defaults; malformed → defaults
  export function setBrainSettings(patch: Partial<BrainSettings>, storage?: Pick<Storage,"getItem"|"setItem">): BrainSettings;
  export function cycleLength(current: AnswerLength, dir: "shorter" | "longer"): AnswerLength; // clamps at the ends
  ```
- The Models page: a `useNativeModels` switch ("Use native model clients; off = the curl provider selected in Dev space"), one `SecretField` per native provider (`kind="ai"`, `providerId` as in Global Constraints, `variable="api_key"`; the marker is kept in `brain_settings.keys[providerId]` only as `"<keychain>"` or `""`), an editable list for each chain (provider select, model text, thinking toggle for Gemini), mode radio (four modes with one-line descriptions), style and length selects.
- Consumes: `SecretField` (Phase 1), `ModelTarget` (Task 6), `BrainMode`/`AnswerStyle`/`AnswerLength` (Task 1).

- [ ] **Step 1: Write the failing test**

```ts
// tests/brain-settings.test.ts
import { describe, it, expect } from "vitest";
import { getBrainSettings, setBrainSettings, cycleLength, DEFAULT_BRAIN_SETTINGS } from "../src/lib/storage/brain-settings.storage";
function mem(init: Record<string,string> = {}) { const m = new Map(Object.entries(init)); return { getItem: (k: string) => m.get(k) ?? null, setItem: (k: string, v: string) => { m.set(k, v); } }; }
describe("brain settings", () => {
  it("returns defaults when nothing is stored or the value is malformed", () => {
    expect(getBrainSettings(mem())).toEqual(DEFAULT_BRAIN_SETTINGS);
    expect(getBrainSettings(mem({ brain_settings: "nope" }))).toEqual(DEFAULT_BRAIN_SETTINGS);
  });
  it("merges a patch over the stored value", () => {
    const s = mem(); setBrainSettings({ mode: "manual" }, s);
    expect(getBrainSettings(s).mode).toBe("manual");
    expect(getBrainSettings(s).style).toBe("concise");
  });
  it("cycles length and clamps", () => {
    expect(cycleLength("normal", "shorter")).toBe("shorter");
    expect(cycleLength("shorter", "shorter")).toBe("shorter");
    expect(cycleLength("normal", "longer")).toBe("longer");
  });
  it("never stores a key value, only the marker", () => {
    const s = mem(); setBrainSettings({ keys: { "native-gemini": "<keychain>" } }, s);
    expect(s.getItem("brain_settings")).not.toContain("AIza");
  });
});
```

- [ ] **Step 2: Run to verify it fails** — `npm test -- brain-settings` → FAIL.
- [ ] **Step 3: Implement the storage module, then the page.**
- [ ] **Step 4: Verify** — `npm test` PASS; `npx tsc --noEmit` clean; `npm run tauri dev`: enter a Gemini key on the Models page, quit, `security find-generic-password -s co.climbtogether.backstage -a ai/native-gemini/api_key -w` prints it; `grep -r AIza ~/Library/WebKit/co.climbtogether.backstage` finds nothing.
- [ ] **Step 5: Commit** — `git add -A && git commit -m "feat(settings): brain mode, style, model chains and native provider keys"`

---

### Task 9: Wire the brain into the listen loop; answer pane, latency chip, shortcuts

**Files:**
- Create: `src/hooks/useBrain.ts`
- Create: `src/pages/app/components/speech/AnswerPane.tsx`
- Modify: `src/hooks/useSystemAudio.ts:224-300` (the `speech-detected` handler: after `setLastTranscription`, call `brain.onUtterance(transcription, Date.now())` instead of `processWithAI` when `useNativeModels` is on or mode is not `manual`; keep `processWithAI` for the curl path when `useNativeModels` is off so Dev-space providers still work)
- Modify: `src/pages/app/components/speech/index.tsx` (render `AnswerPane` in place of `ResultsSection` when a brain answer exists; keep `ResultsSection` for history)
- Modify: `src/config/shortcuts.ts` (append the seven actions from Global Constraints with macOS keys `cmd+enter`, `cmd+1`..`cmd+4`, `cmd+[`, `cmd+]`; windows/linux use `ctrl`), `src/hooks/useGlobalShortcuts.ts:225` (dispatch the new ids to `useBrain` via a `window` CustomEvent `backstage:brain-action` with `detail.action`)
- Test: `tests/shortcuts-config.test.ts`

**Interfaces:**
- Produces, `useBrain()` (called inside `useSystemAudio`, which returns it as `brain` so the speech page and `AnswerPane` read one instance) returning `{ fast: { text: string; latencyMs: number; model: string } | null; full: string; fullLatencyMs: number | null; status: "idle" | "thinking" | "streaming" | "interrupted" | "error"; error: string | null; style: AnswerStyle; length: AnswerLength; mode: BrainMode; setStyle; setLength; setMode; onUtterance(text: string, stoppedAt: number): Promise<void>; answerNow(): void; sessionId: string | null }`. `onUtterance` persists the utterance (speaker `them`), classifies, gates by mode and dedupe, and submits a `GenerationJob` whose `run` consumes `answerQuestion` and persists `fast` and `full` answers (with `latencyMs`, `model`, `interrupted`). `answerNow` submits the most recent utterance regardless of mode. A session is started lazily on the first utterance and `closeStaleSessions()` runs once on mount.
- `AnswerPane` props: the `useBrain()` return. Renders the fast line in bold at the top with a chip `⚡ {latencyMs} ms · {model}`, the full answer in `Markdown` below with its own chip when `fullLatencyMs` is set, four style chips (Cmd+1..4 hints) and the length indicator, and an "Interrupted" badge for that status.
- Consumes: Tasks 1, 2, 4, 5, 6, 7, 8.
- Not carried over from the spec's `llm/` bullet: per-request stream ids. The native clients are per-call async generators consumed by one `useBrain` instance, so no event bus can mix chunks; the curl path keeps Pluely's behaviour unchanged.

- [ ] **Step 1: Write the failing test**

```ts
// tests/shortcuts-config.test.ts
import { describe, it, expect } from "vitest";
import { DEFAULT_SHORTCUT_ACTIONS } from "../src/config/shortcuts";
describe("brain shortcuts", () => {
  it.each([
    ["answer_now", "cmd+enter"], ["style_concise", "cmd+1"], ["style_star", "cmd+2"], ["style_technical", "cmd+3"], ["style_code", "cmd+4"], ["answer_shorter", "cmd+["], ["answer_longer", "cmd+]"],
  ])("%s defaults to %s on macOS", (id, key) => {
    expect(DEFAULT_SHORTCUT_ACTIONS.find((a) => a.id === id)?.defaultKey.macos).toBe(key);
  });
  it("does not collide with existing defaults", () => {
    const keys = DEFAULT_SHORTCUT_ACTIONS.map((a) => a.defaultKey.macos);
    expect(new Set(keys).size).toBe(keys.length);
  });
});
```

- [ ] **Step 2: Run to verify it fails** — `npm test -- shortcuts-config` → FAIL.
- [ ] **Step 3: Add the shortcut actions, then `useBrain`, `AnswerPane`, and the three modifications.** Confirm in `src-tauri/src/shortcuts.rs` that `cmd+[`/`cmd+]` parse through `validate_shortcut_key`; if the Rust parser rejects them, use `cmd+shift+,` and `cmd+shift+.` and update the test and Global Constraints note in the commit message.
- [ ] **Step 4: Verify** — `npm test` PASS; `npx tsc --noEmit` clean. Manual: `npm run tauri dev`, Models page: native on, Gemini key set, mode `questions`; start listening (Cmd+Shift+L or the mic button); play a recorded mock interview through the speakers; the fast line appears with a chip, then the full answer streams; Cmd+2 switches the next answer to STAR; Cmd+Enter answers the last statement in `manual` mode.
  Expected: at least one fast chip under 1000 ms on a short question.
- [ ] **Step 5: Commit** — `git add -A && git commit -m "feat(brain): wire detector, queue and two-tier answers into listen mode with latency chip and shortcuts"`

---

### Task 10: Session list and debrief

**Files:**
- Create: `src/lib/brain/debrief.ts`
- Create: `src/pages/sessions/index.tsx`, `src/pages/sessions/view.tsx`
- Modify: `src/routes/index.tsx` (`/sessions`, `/sessions/:id`), `src/hooks/useMenuItems.tsx` (entry "Sessions", icon `HistoryIcon`, after Chats)
- Test: `tests/debrief.test.ts`, `tests/session-actions.test.ts`

**Interfaces:**
- Produces, `debrief.ts`: `buildDebriefPrompt(t: { session: Session; utterances: Utterance[]; answers: Answer[] }): { system: string; user: string }` where `user` is the transcript as `[mm:ss] THEM: …` / `[mm:ss] ME: …` lines (times relative to `startedAt`) with each full answer indented under its question as `   → answer (fast 420 ms, full 1300 ms, interrupted)` when applicable, and `system` asks for: what was asked, what the candidate said versus what was suggested, three things to do better, questions to prepare, in Markdown with those four headings. `async function* generateDebrief(sessionId: string, deps: { stream: typeof streamWithFallback; getKey; chain: ModelTarget[] }): AsyncIterable<string>` streams and then `saveDebrief`.
- Produces, `session-actions.test.ts`: tests `closeStaleSessions` SQL shape and the transcript ordering by mocking `getDatabase` with `vi.mock("@/lib/database/config")` returning an object whose `select`/`execute` record SQL and return canned rows.
- Pages: list (title, started, duration, "Debrief ready" badge) and view (transcript with speaker labels, answers under questions, Generate/Regenerate debrief button streaming into a `Markdown` block).

- [ ] **Step 1: Write the failing tests**

```ts
// tests/debrief.test.ts
import { describe, it, expect } from "vitest";
import { buildDebriefPrompt } from "../src/lib/brain/debrief";
const session = { id: "s", title: "Google screen", startedAt: 60_000, endedAt: 600_000, debrief: null };
const utterances = [
  { id: "u1", sessionId: "s", speaker: "them" as const, text: "Tell me about a production incident", t0: 65_000, t1: 68_000 },
  { id: "u2", sessionId: "s", speaker: "me" as const, text: "Sure, the retry storm", t0: 69_000, t1: 75_000 },
];
const answers = [
  { id: "a1", sessionId: "s", utteranceId: "u1", tier: "fast" as const, style: "star" as const, model: "m", text: "Retry storm, 8-attempt cap", latencyMs: 420, interrupted: false, createdAt: 0 },
  { id: "a2", sessionId: "s", utteranceId: "u1", tier: "full" as const, style: "star" as const, model: "m", text: "S: At Climb Together…", latencyMs: 1300, interrupted: true, createdAt: 0 },
];
describe("buildDebriefPrompt", () => {
  it("renders a timed, labelled transcript with answers under their questions", () => {
    const { user } = buildDebriefPrompt({ session, utterances, answers });
    expect(user).toContain("[00:05] THEM: Tell me about a production incident");
    expect(user).toContain("[00:09] ME: Sure, the retry storm");
    expect(user).toContain("→ S: At Climb Together… (fast 420 ms, full 1300 ms, interrupted)");
  });
  it("asks for the four debrief sections", () => {
    const { system } = buildDebriefPrompt({ session, utterances, answers });
    for (const h of ["What was asked", "What you said versus what was suggested", "Three things to do better", "Questions to prepare"]) expect(system).toContain(h);
  });
});
```

```ts
// tests/session-actions.test.ts
import { describe, it, expect, vi } from "vitest";
const calls: { sql: string; args: any[] }[] = [];
vi.mock("@/lib/database/config", () => ({ getDatabase: async () => ({
  execute: async (sql: string, args: any[] = []) => { calls.push({ sql, args }); return { rowsAffected: 2 }; },
  select: async (sql: string, args: any[] = []) => { calls.push({ sql, args }); return []; },
}) }));
import { closeStaleSessions, getSessionTranscript } from "../src/lib/database/session.action";
describe("session actions", () => {
  it("closeStaleSessions fills ended_at from the last utterance and returns the count", async () => {
    expect(await closeStaleSessions()).toBe(2);
    const sql = calls.at(-1)!.sql.replace(/\s+/g, " ");
    expect(sql).toMatch(/UPDATE sessions SET ended_at = \(SELECT MAX\(t1\) FROM utterances WHERE utterances.session_id = sessions.id\) WHERE ended_at IS NULL/);
  });
  it("getSessionTranscript orders utterances by t0 and answers by created_at", async () => {
    await getSessionTranscript("s").catch(() => {});
    const sqls = calls.map((c) => c.sql.replace(/\s+/g, " "));
    expect(sqls.some((s) => /FROM utterances WHERE session_id = \$1 ORDER BY t0 ASC/.test(s))).toBe(true);
    expect(sqls.some((s) => /FROM answers WHERE session_id = \$1 ORDER BY created_at ASC/.test(s))).toBe(true);
  });
});
```

- [ ] **Step 2: Run to verify they fail** — `npm test -- debrief session-actions` → FAIL.
- [ ] **Step 3: Implement `debrief.ts`, adjust `session.action.ts` SQL to match, build the two pages.**
- [ ] **Step 4: Verify** — `npm test` PASS; `npx tsc --noEmit` clean. Manual: after the Task 9 mock interview, quit the app without stopping the session, relaunch, Sessions shows it with a duration, open it, Generate debrief streams four headings.
- [ ] **Step 5: Commit** — `git add -A && git commit -m "feat(memory): session list, transcript view and streamed debrief"`

---

### Task 11: Phase done check with the Google prep pack

**Files:**
- Modify: `README.md` ("Verification" section: add the Phase 3 and 5 lines with the date and numbers)

- [ ] **Step 1: Build signed** — `npm run tauri build -- --bundles app,dmg 2>&1 | tail -3` → `Finished 2 bundles`; `scripts/verify-signature.sh src-tauri/target/release/bundle/macos/Backstage.app && echo SIGNED`.
- [ ] **Step 2: Idle network still clean** — `scripts/verify-idle-network.sh src-tauri/target/release/bundle/macos/Backstage.app 60 && echo QUIET`.
- [ ] **Step 3: Profile seeded** — Profile page, import `interview_prep_phone_screen.md`, Save. In a session, ask aloud (through speakers from a recording) "Tell me about a time you handled a production incident" with style STAR.
  Expected: the full answer contains the retry-storm story's "8-attempt cap" and "24-hour cooldown" wording from the pack (spec Phase 5 done: a saved STAR story used verbatim).
- [ ] **Step 4: Latency** — over five short questions, record the fast chip values.
  Expected: median under 1000 ms (spec Phase 3 done). If not, note the STT share (time from `speech-detected` to transcript) in the README line; Phase 2's local Whisper is the planned fix for that share.
- [ ] **Step 5: Debrief** — stop listening, Sessions, open, Generate debrief.
  Expected: four headings, and "Questions to prepare" lists at least one question from the pack's section 2.
- [ ] **Step 6: Commit** — `git add README.md && git commit -m "docs: Phase 3 and 5 verification"`
