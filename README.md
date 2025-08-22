An **AI-powered Notes & Summarizer** can show off mobile, AI, offline-first, and production chops in one app. Here’s a compact, production-grade blueprint you can build from today.

# App pitch

**Notium (working title)** — Capture text, voice, and images → get clean summaries, key points, tasks, and tags using AI. Works offline. Syncs across devices. Powerful search.

---

# Core features (MVP → Pro)

**MVP**

* Create notes: text, voice (auto-transcribe), image → text (OCR)
* AI actions per note: *Summarize, Extract bullets, Action items, Hashtags/Topics*
* Fast global search (full-text + semantic)
* Offline-first (local DB) with background sync
* Push notifications (reminders)
* Cloud backup & multi-device sync
* Minimal, polished UI (light/dark)

**Pro (unlock later)**

* Smart folders & auto-tags
* Cross-note insights (“What did I promise John this week?”)
* Audio highlights (timestamped bullets)
* Share as web link / PDF
* Calendar & email import
* On-device quick capture widget / shortcuts

---

# Tech stack (Expo-friendly)

* **App:** React Native (Expo), TypeScript, Expo Router
* **State:** Zustand or Jotai (simple, fast)
* **Offline DB:** **SQLite** (expo-sqlite/Drizzle or WatermelonDB). Use SQLite for reliability + sync queues.
* **Auth & backend:** **Supabase** (Auth, Postgres, Storage, Edge Functions)

  * Postgres extensions: `pg_trgm` (FTS), `pgvector` (semantic search)
* **AI:**

  * **Transcription:** Whisper API (server-side) or Deepgram (optional)
  * **Summaries/Tasks/Tags/Embeddings:** OpenAI or compatible provider via your server / Supabase Edge Functions
* **File storage:** Supabase Storage (audio, images)
* **Search:** Postgres FTS + `pgvector` for embeddings
* **Notifications:** Expo Notifications
* **CI/CD:** EAS Build + EAS Update (OTA)
* **Analytics & crash:** Sentry + PostHog (or Amplitude)

> Why Supabase? It gives you auth, Postgres, storage, Row-Level Security, and edge functions in one place, which pairs beautifully with Expo.

---

# High-level architecture

**App (offline-first)**

* Local SQLite tables: `notes`, `note_chunks`, `sync_outbox`, `attachments`
* Background tasks:

  * Queue outbound ops → sync when online
  * Upload new audio/images → get URLs
  * Call /embed & /ai endpoints → store results locally
* Push handling: schedule reminders, daily digest

**Backend**

* Supabase Auth (email/password, OTP, OAuth optionally)
* Postgres schema with RLS (user-scoped)
* Edge Functions (or Next.js API)

  * `/transcribe` (fetch file → Whisper → text)
  * `/ai/summarize` (note text → summary, bullets, tasks, hashtags)
  * `/embed` (note text → vector)
* Triggers to keep `updated_at`, `search_tsv`, and `embedding` in sync

---

# Data model (Postgres)

```sql
create table public.notes (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  title text,
  content text,              -- raw text
  summary text,              -- AI summary
  bullets text[],            -- AI bullets
  action_items text[],       -- AI tasks
  tags text[],               -- AI tags
  attachment_count int default 0,
  created_at timestamptz default now(),
  updated_at timestamptz default now(),
  search_tsv tsvector
);

create index notes_user_idx on public.notes(user_id);
create index notes_tsv_idx on public.notes using gin(search_tsv);

-- Semantic search
create extension if not exists vector;
alter table public.notes add column embedding vector(1536);  -- size depends on model
create index notes_embedding_idx on public.notes using ivfflat (embedding vector_cosine_ops);

-- Attachments (audio/image/pdf)
create table public.attachments (
  id uuid primary key default gen_random_uuid(),
  note_id uuid references public.notes(id) on delete cascade,
  user_id uuid not null references auth.users(id) on delete cascade,
  type text check (type in ('audio','image','pdf')),
  storage_path text not null,
  created_at timestamptz default now()
);

-- RLS
alter table public.notes enable row level security;
create policy "own notes" on public.notes
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);
alter table public.attachments enable row level security;
create policy "own attachments" on public.attachments
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);

-- FTS trigger (optional materialization)
create or replace function notes_tsv_update() returns trigger as $$
begin
  new.search_tsv :=
    setweight(to_tsvector('simple', coalesce(new.title,'')), 'A') ||
    setweight(to_tsvector('simple', coalesce(new.content,'')), 'B') ||
    setweight(to_tsvector('simple', array_to_string(new.tags, ' ')), 'C');
  return new;
end
$$ language plpgsql;

create trigger notes_tsv_trg before insert or update
on public.notes for each row execute function notes_tsv_update();
```

**Local SQLite (mirror subset)**

* `notes_local` (same fields + `dirty`, `deleted`, `last_synced_at`)
* `outbox` (id, type: create/update/delete, payload json, retry\_count)
* `attachments_local` (uri, mime, status: pending/uploaded)

---

# Sync strategy (reliable & simple)

1. **Every change = enqueue to `outbox`** (create/update/delete).
2. Background job (AppForeground + Periodic):

   * If online & authed:

     * Upload pending attachments to Supabase Storage (get public URL or signed URL).
     * Flush `outbox` to backend endpoints (batched).
     * Pull server changes using `updated_after` cursor.
3. **Conflict rule:** Last-write-wins with `updated_at` (you can later add CRDT if needed).
4. **Embeddings:** Recompute server-side on create/update; send back vector-ready data or only results.

---

# API endpoints (Edge Functions / Next.js API)

* `POST /ai/summarize` → `{summary, bullets[], action_items[], tags[]}`
* `POST /ai/embed` → `{vector: number[]}`
* `POST /transcribe` (file URL) → `{text}`
* `GET /notes?updated_after=…` → delta sync
* `POST /notes/batch` → create/update/delete in one call (idempotent)

> Secure these with Supabase JWT; verify `user_id` from token server-side.

---

# App folder structure (Expo + TS)

```
app/
  _layout.tsx
  index.tsx                 // home (list + search)
  note/[id].tsx             // editor, AI actions
  capture.tsx               // voice/image capture
  settings.tsx
components/
  NoteCard.tsx
  AIActionBar.tsx
  EmptyState.tsx
  SearchBar.tsx
lib/
  supabase.ts
  ai.ts                     // client callers
  db/
    schema.ts               // drizzle/watermelon models
    sqlite.ts
    sync.ts                 // outbox + pull/push
  store/
    useNotes.ts             // Zustand
    useSession.ts
utils/
  text.ts
  dates.ts
assets/
  lottie/
```

---

# Key UX flows

**Capture**

* Quick-add FAB → Text | Voice | Image
* Voice: record (Expo-AV) → save file → upload → `/transcribe` → insert text → auto-run `/ai/summarize`

**Editor**

* Markdown-ish editor (react-native-markdown-display)
* AI Action bar: Summarize • Bullets • Action Items • Improve Writing • Hashtags
* One-tap copy of bullets/tasks
* Tag chips (editable)

**Search**

* Search bar → local FTS over SQLite; if online, enrich with semantic results (server).
* “Ask your notes” mini-chat (later): RAG over your notes using embeddings.

**Notifications**

* Per-note reminder
* Daily digest at 8pm: “3 new notes, 5 open action items”

---

# Example: AI action client code (sketch)

```ts
// lib/ai.ts
import { supabase } from './supabase';

export async function aiSummarize(noteId: string, content: string) {
  const { data, error } = await supabase.functions.invoke('ai-summarize', {
    body: { noteId, content }
  });
  if (error) throw error;
  return data as {
    summary: string;
    bullets: string[];
    action_items: string[];
    tags: string[];
  };
}
```

**Edge Function (pseudo, Deno runtime)**

```ts
// supabase/functions/ai-summarize/index.ts
import { createClient } from 'https://esm.sh/@supabase/supabase-js';
import OpenAI from 'https://esm.sh/openai';

export async function serve(req: Request) {
  const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!, Deno.env.get('SUPABASE_ANON_KEY')!,
    { global: { headers: { Authorization: req.headers.get('Authorization')! } } }
  );

  const { content, noteId } = await req.json();
  const openai = new OpenAI({ apiKey: Deno.env.get('OPENAI_API_KEY')! });

  const prompt = `
  You are a ruthless note cleaner. Return JSON with:
  - summary (<=120 words)
  - bullets (5 crisp bullets)
  - action_items (imperative voice)
  - tags (3-6 lowercase hashtags)
  Text:
  """${content}"""
  `;

  const resp = await openai.chat.completions.create({
    model: "gpt-4o-mini",
    messages: [{ role: "user", content: prompt }],
    response_format: { type: "json_object" }
  });

  const parsed = JSON.parse(resp.choices[0].message.content!);
  // Optionally update DB here
  return new Response(JSON.stringify(parsed), { headers: { 'content-type': 'application/json' }});
}
```

---

# Example: SQLite outbox pattern (client)

```ts
// lib/db/sync.ts
export async function enqueue(type: 'create'|'update'|'delete', payload: any) {
  await db.run('insert into outbox(type,payload,created_at) values (?,?,strftime("%s","now"))',
    [type, JSON.stringify(payload)]);
}

export async function flushOutbox(token: string) {
  const rows = await db.getAll('select * from outbox order by created_at asc limit 50');
  if (!rows.length) return;

  const res = await fetch(`${API_URL}/notes/batch`, {
    method: 'POST',
    headers: { 'Authorization': `Bearer ${token}`, 'Content-Type': 'application/json' },
    body: JSON.stringify({ ops: rows.map(r => ({ id: r.id, ...JSON.parse(r.payload) })) })
  });

  if (res.ok) {
    const okIds = rows.map(r => r.id);
    await db.run(`delete from outbox where id in (${okIds.map(()=>'?').join(',')})`, okIds);
  }
}
```

---

# Prompt kit (stable outputs)

* **Summarize:** “Summarize the note in <=120 words. Prefer decisions, dates, owners, risks.”
* **Bullets:** “Extract 5 crisp bullets; remove filler; no intro text.”
* **Action items:** “List imperative tasks with owners and due dates if present. Output plain text list.”
* **Tags:** “Propose 3–6 lowercase hashtags; prefer existing tags in the note.”

---

# Security & privacy

* Enforce RLS on all tables (only `user_id = auth.uid()`).
* Sign URLs for private attachments; short TTL.
* Do not send PII in push payloads (only ids).
* At-rest encryption (Supabase default for disks); in-transit TLS.
* Add “Local-only” toggle per note (never leaves device).

---

# Performance & QA

* Virtualized lists (FlashList/RecyclerListView).
* Debounced saves; optimistic UI.
* Profiler & memory checks on long note lists.
* Unit tests (Vitest) for stores/utils; Detox for e2e flows (capture → AI → save → search).

---

# CI/CD & release

* EAS Build for dev/prod channels.
* EAS Update for safe OTA (guard native module changes).
* Sentry for crashes; PostHog for funnels (capture rate, AI success, sync errors).

---

# Monetization (optional)

* Free: core notes + summaries.
* Pro: unlimited transcription, semantic search, daily digests, share links.
  Use **Expo In-App Purchases** or Stripe Customer Portal via web.

---

# 1-week MVP checklist (scope, not dates)

* [ ] Auth + basic note CRUD (local & sync)
* [ ] Voice capture → transcribe → insert text
* [ ] AI summarize/bullets/tags via Edge Function
* [ ] Search (local FTS) + list UI
* [ ] EAS build + OTA + Sentry
* [ ] Nice theming + icon + splash

---
