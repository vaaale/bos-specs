# Implementation Tasks: Memory App Redesign

**Feature**: `memory-app-redesign`  
**Branch**: `bos/023-memory-app`  
**Generated**: 2026-07-05

---

## Task Breakdown

### Phase 0: Prerequisites & Verification

#### Task 0.1: Verify Spec 021 Implementation
- [ ] Check `/Documents/Memory/Episodes/` directory exists with sample files
- [ ] Verify `/Documents/Memory/Topics/` directory exists with topic files
- [ ] Confirm `/Documents/Memory/.watermarks.json` exists
- [ ] Validate scheduler jobs in `/Documents/System/scheduler-jobs.json` for `system:memory.fast-loop` and `system:memory.slow-loop`
- **Output**: Report on which components are implemented vs. missing

#### Task 0.2: Backend API Audit
- [ ] Test `GET /api/memory?target=user` returns USER.md entries
- [ ] Test `GET /api/memory?target=memory` returns MEMORY.md entries
- [ ] Test `POST /api/memory` with add/replace/remove actions
- [ ] Test `GET /api/memory/search?q=test` returns search results
- [ ] Test `POST /api/memory/consolidate` triggers slow loop
- [ ] Test `POST /api/assistant/reflect` with conversationId triggers fast loop
- [ ] Test `GET /api/config?namespace=memoryLoops` returns configuration
- **Output**: List of working endpoints vs. missing/broken

#### Task 0.3: Identify Backend Gaps
- [ ] Document missing episode endpoints (list, get, delete)
- [ ] Document missing topic management endpoints
- [ ] Identify required changes to `src/lib/agent/memory/` services
- [ ] Create prioritized list of backend tasks
- **Output**: Gap analysis document with implementation priorities

---

### Phase 1: Backend API Completion

#### Task 1.1: Episode Service Implementation
**File**: `src/lib/agent/memory/episodes.ts` (create if missing)

```typescript
export interface EpisodeMeta {
  filename: string;
  conversationId: string;
  createdAt: string;
  updatedAt: string;
  watermark: string;
  skillsUsed: string[];
  status: 'pending' | 'consolidated';
  skillCandidates?: string[];
  turnCount: number;
}

export interface Episode extends EpisodeMeta {
  title: string;
  sections: {
    taskOutcome: string;
    whatWorked: string;
    whatFailed: string;
    correctionsReceived: string;
    durableLessonCandidates: string;
    profileSuggestions: string;
  };
}

export async function listEpisodes(): Promise<{ pending: EpisodeMeta[]; consolidated: EpisodeMeta[] }>;
export async function getEpisode(filename: string): Promise<Episode | null>;
export async function deleteEpisode(filename: string): Promise<void>;
export async function archiveEpisode(filename: string): Promise<void>;
```

**Requirements**:
- Scan `/Documents/Memory/Episodes/` for `.md` files
- Parse frontmatter for metadata
- Parse markdown sections for content
- Atomic reads with error handling
- Handle `.Archive/` subdirectory

#### Task 1.2: Episode API Routes
**Files**: 
- `src/pages/api/memory/episodes.ts` (GET - list)
- `src/pages/api/memory/episodes/[filename].ts` (GET - single, DELETE)

```typescript
// GET /api/memory/episodes
export async function GET(req: Request) {
  const { pending, consolidated } = await listEpisodes();
  return Response.json({ pending, consolidated });
}

// GET /api/memory/episodes/[filename]
export async function GET(req: Request, { params }: { params: { filename: string } }) {
  const episode = await getEpisode(params.filename);
  if (!episode) return Response.json({ error: 'Not found' }, { status: 404 });
  return Response.json(episode);
}

// DELETE /api/memory/episodes/[filename]
export async function DELETE(req: Request, { params }: { params: { filename: string } }) {
  await deleteEpisode(params.filename);
  return Response.json({ success: true });
}
```

**Requirements**:
- Validate filename format (prevent path traversal)
- Return proper error codes (404, 500)
- Log delete operations

#### Task 1.3: Topic Service Extensions
**File**: `src/lib/agent/memory/topics.ts` (extend existing)

```typescript
export async function addTopicEntry(slug: string, content: string): Promise<number>;
export async function replaceTopicEntry(slug: string, id: number, content: string): Promise<void>;
export async function deleteTopicEntry(slug: string, id: number): Promise<void>;
export async function getTopicBudget(slug: string): Promise<{ used: number; max: number }>;
```

**Requirements**:
- Enforce 4000 char budget per topic
- Incremental operations only (no full rewrites)
- Atomic writes with temp file + rename
- Entry ID generation (auto-increment or timestamp-based)

#### Task 1.4: Topic API Extensions
**File**: `src/pages/api/memory.ts` (extend existing POST/DELETE handlers)

```typescript
// POST /api/memory with target: "topic"
if (target === 'topic') {
  if (action === 'add') {
    const newId = await addTopicEntry(topic, content);
    return Response.json({ success: true, newId });
  }
  if (action === 'replace') {
    await replaceTopicEntry(topic, id, content);
    return Response.json({ success: true });
  }
}

// DELETE /api/memory?target=topic&topic=<slug>&id=<number>
if (target === 'topic') {
  await deleteTopicEntry(topic, id);
  return Response.json({ success: true });
}
```

#### Task 1.5: Configuration Validation
**File**: `src/lib/config/registry.ts` (verify memoryLoops namespace)

```typescript
export const configNamespaces = {
  // ... existing namespaces
  memoryLoops: {
    title: 'Memory Loops',
    fields: [
      { key: 'fastLoop.enabled', type: 'boolean' },
      { key: 'fastLoop.tickIntervalSec', type: 'number', min: 60, max: 600 },
      { key: 'fastLoop.idleThresholdSec', type: 'number', min: 60, max: 1800 },
      { key: 'fastLoop.turnCap', type: 'number', min: 10, max: 100 },
      { key: 'fastLoop.minNewTurns', type: 'number', min: 2, max: 20 },
      { key: 'slowLoop.enabled', type: 'boolean' },
      { key: 'slowLoop.intervalSec', type: 'number', min: 1800, max: 7200 },
      { key: 'slowLoop.batchSize', type: 'number', min: 1, max: 50 },
      { key: 'episodeArchiveAgeDays', type: 'number', min: 7, max: 90 },
      { key: 'topicBudget', type: 'number', min: 2000, max: 10000 },
    ],
  },
};
```

**Requirements**:
- Add validation rules (min/max values)
- Ensure defaults match spec (120s, 300s, 40 turns, etc.)

#### Task 1.6: Run History Endpoint
**File**: `src/pages/api/logs.ts` (extend or create)

```typescript
// GET /api/logs?category=memory.loops&limit=20
export async function GET(req: Request) {
  const url = new URL(req.url);
  const category = url.searchParams.get('category');
  const limit = parseInt(url.searchParams.get('limit') || '20');
  
  if (category !== 'memory.loops') {
    return Response.json({ error: 'Invalid category' }, { status: 400 });
  }
  
  const logs = await queryLogs({ category, limit });
  return Response.json({
    history: logs.map(log => ({
      loopType: log.data.loopType, // 'fast' | 'slow'
      timestamp: log.timestamp,
      stats: [
        { label: 'Processed', value: `${log.data.processed} conversations` },
        { label: 'Created', value: `${log.data.episodesCreated}` },
        { label: 'Updated', value: `${log.data.episodesUpdated}` },
        { label: 'Archived', value: `${log.data.archived}` },
        { label: 'Refusals', value: `${log.data.refusals}` },
      ],
    })),
  });
}
```

**Requirements**:
- Query central logging system
- Aggregate loop execution data
- Return formatted history entries

#### Task 1.7: Backend Testing
- [ ] Write unit tests for episode service functions
- [ ] Write unit tests for topic service extensions
- [ ] Write integration tests for all new API routes
- [ ] Test edge cases (missing files, invalid IDs, budget overflows)
- [ ] Run `npx tsc --noEmit` and fix errors
- [ ] Run `npm run lint` and fix warnings

---

### Phase 2: App Structure & Navigation

#### Task 2.1: Update App Manifest
**File**: `src/apps/memory/manifest.ts` (or create `src/apps/memory-app/manifest.ts`)

```typescript
import type { AppManifest } from "@/os/types";

const manifest: AppManifest = {
  id: "memory",
  name: "Memory",
  icon: "Brain", // Ensure this exists in src/components/desktop/icons.tsx
  defaultWidth: 1200,
  defaultHeight: 800,
  order: 40,
  singleton: true,
  builtin: true,
};

export default manifest;
```

**Requirements**:
- Verify "Brain" icon exists in `src/components/desktop/icons.tsx`
- If not, add it: `import { Brain } from "lucide-react"; export const ICONS = { ..., Brain };`

#### Task 2.2: Create Main App Component
**File**: `src/apps/memory/index.tsx`

```tsx
"use client";

import { useState, useEffect } from "react";
import type { AppProps } from "@/components/apps/types";
import { useOSStore } from "@/store/os-provider";
import { Brain, FileText, BookOpen, Settings, Search, Clock, CheckCircle, Folder } from "lucide-react";

export default function MemoryApp({ windowId }: AppProps) {
  const [activeTab, setActiveTab] = useState<"profile" | "episodes" | "topics" | "loops" | "search">("profile");
  const setTitle = useOSStore((s) => s.setTitle);
  
  // Stats state (will be populated from API)
  const [stats, setStats] = useState({ pending: 0, consolidated: 0, topics: 0 });

  useEffect(() => {
    setTitle(windowId, "Memory");
    
    // Load stats
    const loadStats = async () => {
      const [episodesRes, topicsRes] = await Promise.all([
        fetch("/api/memory/episodes"),
        fetch("/api/memory/topics")
      ]);
      const episodes = await episodesRes.json();
      const topics = await topicsRes.json();
      setStats({
        pending: episodes.pending.length,
        consolidated: episodes.consolidated.length,
        topics: topics.length,
      });
    };
    loadStats();
  }, [windowId, setTitle]);

  const tabs = [
    { id: "profile" as const, label: "Profile & Notes", icon: Brain },
    { id: "episodes" as const, label: "Episodes", icon: FileText },
    { id: "topics" as const, label: "Topics", icon: BookOpen },
    { id: "loops" as const, label: "Memory Loops", icon: Settings },
    { id: "search" as const, label: "Search", icon: Search },
  ];

  return (
    <div className="flex h-full flex-col bg-[#15171e] text-white">
      {/* Header */}
      <header className="flex items-center justify-between border-b border-white/10 bg-white/[0.02] px-4 py-3">
        <h1 className="flex items-center gap-2 text-base font-semibold">
          <Brain className="h-6 w-6 text-violet-300" />
          Memory
        </h1>
        <div className="flex gap-2">
          <div className="rounded border-l-2 border-amber-400 bg-white/5 px-2 py-1 text-[10px]">
            <Clock className="mr-0.5 inline h-3 w-3" />
            {stats.pending} Pending
          </div>
          <div className="rounded border-l-2 border-emerald-400 bg-white/5 px-2 py-1 text-[10px]">
            <CheckCircle className="mr-0.5 inline h-3 w-3" />
            {stats.consolidated} Consolidated
          </div>
          <div className="rounded border-l-2 border-white/10 bg-white/5 px-2 py-1 text-[10px]">
            <Folder className="mr-0.5 inline h-3 w-3" />
            {stats.topics} Topics
          </div>
        </div>
      </header>

      {/* Tabs */}
      <nav className="flex gap-1 border-b border-white/10 bg-white/[0.02] px-4 py-2">
        {tabs.map((tab) => (
          <button
            key={tab.id}
            onClick={() => setActiveTab(tab.id)}
            className={`flex items-center gap-1 rounded px-3 py-1.5 text-xs font-medium transition-colors ${
              activeTab === tab.id ? "bg-white/15 text-white" : "text-white/70 hover:bg-white/10"
            }`}
          >
            <tab.icon className="h-3.5 w-3.5" />
            {tab.label}
          </button>
        ))}
      </nav>

      {/* Tab Content */}
      <div className="min-h-0 flex-1 overflow-auto p-4">
        {activeTab === "profile" && <ProfileTab />}
        {activeTab === "episodes" && <EpisodesTab />}
        {activeTab === "topics" && <TopicsTab />}
        {activeTab === "loops" && <LoopsTab />}
        {activeTab === "search" && <SearchTab />}
      </div>
    </div>
  );
}

// Import tab components (create in next tasks)
import ProfileTab from "./components/ProfileTab";
import EpisodesTab from "./components/EpisodesTab";
import TopicsTab from "./components/TopicsTab";
import LoopsTab from "./components/LoopsTab";
import SearchTab from "./components/SearchTab";
```

#### Task 2.3: Create Tab Component Stubs
**Files**:
- `src/apps/memory/components/ProfileTab.tsx` (stub)
- `src/apps/memory/components/EpisodesTab.tsx` (stub)
- `src/apps/memory/components/TopicsTab.tsx` (stub)
- `src/apps/memory/components/LoopsTab.tsx` (stub)
- `src/apps/memory/components/SearchTab.tsx` (stub)

Each stub should:
- Export default function component
- Return a placeholder div with "TODO: Implement {TabName}"
- Use proper TypeScript typing

**Requirements**:
- Ensure all imports resolve
- App compiles without errors
- All tabs render when clicked

#### Task 2.4: Verify App Launches
- [ ] Start dev server: `npm run dev`
- [ ] Launch Memory app from dock
- [ ] Verify all 5 tabs are visible and clickable
- [ ] Confirm active tab state changes correctly
- [ ] Check browser console for errors

---

### Phase 3: Profile & Notes Tab

#### Task 3.1: Implement Two-Pane Layout
**File**: `src/apps/memory/components/ProfileTab.tsx`

```tsx
"use client";

import { useState, useEffect } from "react";
import { Plus, Info, Trash2 } from "lucide-react";

export default function ProfileTab() {
  const [userEntries, setUserEntries] = useState<string[]>([]);
  const [memoryEntries, setMemoryEntries] = useState<string[]>([]);
  const [loading, setLoading] = useState(true);
  const [userBudget, setUserBudget] = useState({ used: 0, max: 1200 });
  const [memoryBudget, setMemoryBudget] = useState({ used: 0, max: 2000 });

  useEffect(() => {
    const load = async () => {
      try {
        const [userRes, memoryRes] = await Promise.all([
          fetch("/api/memory?target=user"),
          fetch("/api/memory?target=memory")
        ]);
        const user = await userRes.json();
        const memory = await memoryRes.json();
        setUserEntries(user.entries);
        setMemoryEntries(memory.entries);
        setUserBudget({ used: user.charCount, max: 1200 });
        setMemoryBudget({ used: memory.charCount, max: 2000 });
      } catch (err) {
        console.error("Failed to load memory:", err);
      } finally {
        setLoading(false);
      }
    };
    load();
  }, []);

  if (loading) {
    return (
      <div className="flex items-center justify-center py-12">
        <div className="h-6 w-6 animate-spin rounded-full border-2 border-white/20 border-t-violet-300" />
      </div>
    );
  }

  return (
    <div className="grid grid-cols-[320px_1fr] gap-3">
      {/* User Profile Pane */}
      <div className="flex flex-col rounded-lg border border-white/10 bg-white/[0.02]">
        <div className="flex items-center justify-between border-b border-white/10 px-3 py-2">
          <h2 className="text-xs font-semibold uppercase tracking-wide text-white/60">User Profile</h2>
          <button className="rounded bg-white/10 p-1 hover:bg-white/20">
            <Plus className="h-3.5 w-3.5" />
          </button>
        </div>
        <div className="min-h-0 flex-1 overflow-auto p-3">
          {userEntries.map((entry, idx) => (
            <div key={idx} className="group mb-2 rounded-lg bg-white/[0.03] p-2 hover:bg-white/5">
              <div className="flex items-start justify-between">
                <p className="text-xs leading-relaxed">{entry}</p>
                <button className="opacity-0 group-hover:opacity-100 hover:text-red-300">
                  <Trash2 className="h-3.5 w-3.5" />
                </button>
              </div>
            </div>
          ))}
          {userEntries.length === 0 && (
            <div className="py-8 text-center text-xs text-white/40">No entries yet</div>
          )}
        </div>
        <div className="border-t border-white/10 px-3 py-2">
          <div className="text-[10px] text-white/60">{userBudget.used.toLocaleString()} / {userBudget.max.toLocaleString()} chars</div>
          <div className="mt-1 h-1 w-full rounded bg-white/10">
            <div 
              className={`h-full rounded ${getBudgetColor(userBudget.used, userBudget.max)}`} 
              style={{ width: `${(userBudget.used / userBudget.max) * 100}%` }} 
            />
          </div>
        </div>
      </div>

      {/* Agent Notes Pane */}
      <div className="flex flex-col rounded-lg border border-white/10 bg-white/[0.02]">
        <div className="flex items-center justify-between border-b border-white/10 px-3 py-2">
          <h2 className="text-xs font-semibold uppercase tracking-wide text-white/60">Agent Notes</h2>
          <button className="rounded bg-white/10 p-1 hover:bg-white/20">
            <Plus className="h-3.5 w-3.5" />
          </button>
        </div>
        <div className="min-h-0 flex-1 overflow-auto p-3">
          {memoryEntries.map((entry, idx) => (
            <div key={idx} className="group mb-2 rounded-lg bg-white/[0.03] p-2 hover:bg-white/5">
              <div className="flex items-start justify-between">
                <p className="text-xs leading-relaxed">{entry}</p>
                <button className="opacity-0 group-hover:opacity-100 hover:text-red-300">
                  <Trash2 className="h-3.5 w-3.5" />
                </button>
              </div>
            </div>
          ))}
          {memoryEntries.length === 0 && (
            <div className="py-8 text-center text-xs text-white/40">No entries yet</div>
          )}
        </div>
        <div className="border-t border-white/10 px-3 py-2">
          <div className="text-[10px] text-white/60">{memoryBudget.used.toLocaleString()} / {memoryBudget.max.toLocaleString()} chars</div>
          <div className="mt-1 h-1 w-full rounded bg-white/10">
            <div 
              className={`h-full rounded ${getBudgetColor(memoryBudget.used, memoryBudget.max)}`} 
              style={{ width: `${(memoryBudget.used / memoryBudget.max) * 100}%` }} 
            />
          </div>
        </div>
        <div className="mt-2 rounded-lg border border-violet-400/20 bg-violet-400/10 p-2 text-xs text-white/90">
          <Info className="mr-1 inline h-3.5 w-3.5" />
          Changes take effect in your <strong>next conversation</strong>.
        </div>
      </div>
    </div>
  );
}

function getBudgetColor(used: number, max: number) {
  const percent = (used / max) * 100;
  if (percent > 80) return "bg-red-400";
  if (percent > 50) return "bg-amber-400";
  return "bg-emerald-400";
}
```

#### Task 3.2: Add Entry CRUD Operations
- [ ] Implement "Add" button with modal or inline input
- [ ] Connect to `POST /api/memory` with `target: "user"` or `target: "memory"`
- [ ] Implement delete with confirmation dialog
- [ ] Handle errors (budget overflow, validation failures)

#### Task 3.3: Test Profile Tab
- [ ] Add entries to both panes
- [ ] Delete entries
- [ ] Verify budget bars update correctly
- [ ] Check responsive behavior on small screens

---

### Phase 4: Episodes Tab

*(Continue with detailed task breakdown for Episodes, Topics, Loops, and Search tabs following the same pattern)*

#### Task 4.1: Implement Episodes List Component
- [ ] Create two-pane layout structure
- [ ] Add filter buttons (All/Pending/Consolidated)
- [ ] Fetch episodes from `/api/memory/episodes`
- [ ] Render episode list with metadata and badges
- [ ] Implement selection state

#### Task 4.2: Implement Episode Details Component
- [ ] Display episode metadata (status, conversationId, timestamps, skills)
- [ ] Render all six sections with proper formatting
- [ ] Add action buttons (Review, Archive, Delete)
- [ ] Implement Previous/Next navigation

#### Task 4.3: Connect Episode Actions
- [ ] "Review Now" → `POST /api/assistant/reflect`
- [ ] "Archive" → Move to `.Archive/` (backend operation)
- [ ] "Delete" → `DELETE /api/memory/episodes/:filename` with confirmation

#### Task 4.4: Test Episodes Tab
- [ ] Load episodes list
- [ ] Switch between filters
- [ ] View episode details
- [ ] Trigger review action
- [ ] Test archive/delete operations

---

### Phase 5: Topics Tab

#### Task 5.1: Implement Topics List Component
- [ ] Two-pane layout with search box
- [ ] Fetch topics from `/api/memory/topics`
- [ ] Render list with entry counts and budget bars
- [ ] Implement selection state

#### Task 5.2: Implement Topic Details Component
- [ ] Display numbered entries with content
- [ ] Show timestamps and consolidation source
- [ ] Add "Add Entry" and "Delete Topic" buttons

#### Task 5.3: Connect Topic Operations
- [ ] "Add Entry" → Modal with textarea, `POST /api/memory` with `target: "topic"`
- [ ] "Delete Entry" → `DELETE /api/memory?target=topic&topic=<slug>&id=<number>`
- [ ] Budget validation on add

#### Task 5.4: Test Topics Tab
- [ ] Load topics list
- [ ] Search/filter topics
- [ ] Add new entries
- [ ] Delete entries and topics

---

### Phase 6: Memory Loops Tab

#### Task 6.1: Implement Configuration Form
- [ ] Create Fast Loop section with toggle, inputs, dropdowns
- [ ] Create Slow Loop section (same pattern)
- [ ] Create Advanced Settings section
- [ ] Fetch config from `/api/config?namespace=memoryLoops`

#### Task 6.2: Implement Run History Component
- [ ] Fetch history from `/api/logs?category=memory.loops`
- [ ] Display last execution summaries
- [ ] Add manual trigger buttons

#### Task 6.3: Connect Configuration Operations
- [ ] "Save" → `POST /api/config` with namespace and values
- [ ] "Reset" → Restore defaults
- [ ] Manual triggers → `POST /api/memory/consolidate` and `POST /api/assistant/reflect`

#### Task 6.4: Test Loops Tab
- [ ] Load configuration
- [ ] Modify settings and save
- [ ] Trigger loops manually
- [ ] Verify run history updates

---

### Phase 7: Search Tab

#### Task 7.1: Implement Search Interface
- [ ] Search box with Enter key support
- [ ] Source filter buttons (All/Topics/Episodes/Memory)
- [ ] Results count display

#### Task 7.2: Implement Results Display
- [ ] Fetch from `/api/memory/search?q=<query>&maxResults=50`
- [ ] Render results with source, content, score
- [ ] Highlight matched terms
- [ ] Add "Load More" pagination

#### Task 7.3: Test Search Tab
- [ ] Execute searches
- [ ] Apply filters
- [ ] Verify highlighting and scores
- [ ] Test pagination

---

### Phase 8: Polish & Testing

#### Task 8.1: Responsive Design
- [ ] Add media queries for <768px screens
- [ ] Stack two-pane layouts to single column
- [ ] Adjust font sizes and spacing

#### Task 8.2: Loading & Error States
- [ ] Add loading spinners for all async operations
- [ ] Display error messages with retry options
- [ ] Handle empty states gracefully

#### Task 8.3: Accessibility Audit
- [ ] Add ARIA labels to all interactive elements
- [ ] Test keyboard navigation (Tab, Enter, Escape)
- [ ] Verify focus states
- [ ] Run Lighthouse audit (target >90)

#### Task 8.4: TypeScript & Lint
- [ ] Run `npx tsc --noEmit` and fix all errors
- [ ] Run `npm run lint` and fix all warnings
- [ ] Add missing type definitions

#### Task 8.5: Documentation
- [ ] Update `docs/usage/apps/memory.md` with new features
- [ ] Add screenshots if applicable
- [ ] Document API changes in `docs/dev/memory/memory.md`

#### Task 8.6: Final Testing
- [ ] Test all user stories from spec
- [ ] Verify backward compatibility
- [ ] Performance test with large datasets
- [ ] Cross-browser testing (Chrome, Firefox, Safari)

---

## Completion Criteria

All tasks must be completed and verified:
- [ ] All Phase 0-8 tasks checked off
- [ ] `npx tsc --noEmit` passes with no errors
- [ ] `npm run lint` passes with no warnings
- [ ] All user stories from spec work as expected
- [ ] Documentation updated
- [ ] Feature branch ready for merge

---

## Notes for Developer

- Follow BOS Style Guide strictly (opacity colors, text-xs default, lucide-react icons)
- Use mockup at `/mockups/memory-app-redesign.html` as normative UI reference
- Implement loading states and error handling for all async operations
- Keep components small and focused; extract shared logic into utilities
- Write meaningful commit messages describing changes
- Test incrementally after each phase
