# Implementation Plan: Memory App Redesign

**Feature**: `memory-app-redesign`  
**Branch**: `bos/023-memory-app`  
**Status**: Ready for Implementation

---

## Overview

This plan outlines the step-by-step implementation of the Memory App Redesign to expose the new memory loops architecture (episodes, topics, automated loops) through a modern 5-tab interface while maintaining backward compatibility.

---

## Phase Breakdown

### Phase 0: Prerequisites & Verification (1-2 hours)

**Goal**: Ensure all backend infrastructure is in place before UI development.

#### Tasks:
1. **Verify Spec 021 Implementation**
   - Check that `/Documents/Memory/Episodes/` directory structure exists
   - Verify topic files in `/Documents/Memory/Topics/`
   - Confirm watermarks file at `/Documents/Memory/.watermarks.json`
   - Validate scheduler jobs for fast/slow loops in `/Documents/System/scheduler-jobs.json`

2. **Backend API Audit**
   - Test existing endpoints: `GET /api/memory`, `POST /api/memory`, `GET /api/memory/search`
   - Verify manual triggers: `POST /api/memory/consolidate`, `POST /api/assistant/reflect`
   - Check config namespace: `GET /api/config?namespace=memoryLoops`

3. **Identify Gaps**
   - List missing episode endpoints (list, get, delete)
   - List missing topic management endpoints (if any)
   - Document required changes to backend services

**Deliverable**: Gap analysis report with prioritized backend tasks.

---

### Phase 1: Backend API Completion (4-6 hours)

**Goal**: Implement all missing API endpoints for the Memory app.

#### Priority Tasks:

**1.1 Episode Service Endpoints**
```typescript
// File: src/lib/agent/memory/episodes.ts (if not exists)
async function listEpisodes(): Promise<{ pending: EpisodeMeta[]; consolidated: EpisodeMeta[] }>
async function getEpisode(filename: string): Promise<Episode>
async function deleteEpisode(filename: string): Promise<void>
```

**API Routes**: `src/pages/api/memory/episodes.ts`, `src/pages/api/memory/episodes/[filename].ts`

- Implement file scanning under `/Documents/Memory/Episodes/`
- Parse frontmatter for metadata
- Atomic read operations with error handling
- Return properly typed responses

**1.2 Topic Management Endpoints**
```typescript
// Extend existing topic service if needed
async function addTopicEntry(slug: string, content: string): Promise<number>
async function replaceTopicEntry(slug: string, id: number, content: string): Promise<void>
async function deleteTopicEntry(slug: string, id: number): Promise<void>
```

**API Routes**: Extend `src/pages/api/memory.ts` to handle `target: "topic"` operations

- Budget validation (4000 chars per topic)
- Incremental operations only (no full rewrites)
- Entry ID generation and tracking

**1.3 Configuration Integration**
- Ensure `memoryLoops` namespace is registered in `src/lib/config/registry.ts`
- Verify config persistence works for all fields
- Add validation for interval/budget ranges

**1.4 Run History Endpoint** (optional, can use existing logging)
```typescript
GET /api/logs?category=memory.loops&limit=20
```

- Query central logging for loop execution records
- Aggregate start/end times, processed counts, errors

**Acceptance Criteria**:
- All endpoints return correct data types
- Error handling for missing files/invalid operations
- Budget enforcement on topic writes
- TypeScript compilation clean

---

### Phase 2: App Structure & Navigation (2-3 hours)

**Goal**: Create the app skeleton with working tab navigation.

#### Tasks:

**2.1 App Manifest**
```typescript
// File: src/apps/memory/manifest.ts (update existing) or src/apps/memory-app/manifest.ts
import type { AppManifest } from "@/os/types";

const manifest: AppManifest = {
  id: "memory", // or "memory-app" if creating new
  name: "Memory",
  icon: "Brain", // must exist in src/components/desktop/icons.tsx
  defaultWidth: 1200,
  defaultHeight: 800,
  order: 40,
  singleton: true,
  builtin: true,
};

export default manifest;
```

**2.2 Main Component Structure**
```tsx
// File: src/apps/memory/index.tsx
"use client";

import { useState, useEffect } from "react";
import { useOSStore } from "@/store/os-provider";
import { Brain, FileText, BookOpen, Settings, Search } from "lucide-react";

export default function MemoryApp({ windowId }: AppProps) {
  const [activeTab, setActiveTab] = useState<"profile" | "episodes" | "topics" | "loops" | "search">("profile");
  const setTitle = useOSStore((s) => s.setTitle);

  useEffect(() => {
    setTitle(windowId, "Memory");
  }, [windowId, setTitle]);

  return (
    <div className="flex h-full flex-col bg-[#15171e]">
      {/* Header */}
      <header className="flex items-center justify-between border-b border-white/10 bg-white/[0.02] px-4 py-3">
        <h1 className="flex items-center gap-2 text-base font-semibold">
          <Brain className="h-6 w-6 text-violet-300" />
          Memory
        </h1>
        {/* Stats badges */}
      </header>

      {/* Tabs */}
      <nav className="flex gap-1 border-b border-white/10 bg-white/[0.02] px-4 py-2">
        {tabs.map((tab) => (
          <button
            key={tab.id}
            onClick={() => setActiveTab(tab.id)}
            className={`rounded px-3 py-1.5 text-xs font-medium transition-colors ${
              activeTab === tab.id ? "bg-white/15 text-white" : "text-white/70 hover:bg-white/10"
            }`}
          >
            <tab.icon className="mr-1 inline h-3.5 w-3.5" />
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
```

**2.3 Tab Component Stubs**
- Create empty components for each tab
- Ensure proper `min-h-0 flex-1` on scroll regions
- Add basic styling to match BOS conventions

**Acceptance Criteria**:
- App launches correctly
- All 5 tabs render and switch
- Active state visible
- Window title set properly

---

### Phase 3: Profile & Notes Tab (2-3 hours)

**Goal**: Implement backward-compatible profile management with enhancements.

#### Tasks:

**3.1 Two-Pane Layout**
```tsx
<div className="grid grid-cols-[320px_1fr] gap-3">
  <div className="flex flex-col rounded-lg border border-white/10 bg-white/[0.02]">
    <div className="border-b border-white/10 bg-white/[0.02] px-3 py-2">
      <h2 className="text-xs font-semibold uppercase tracking-wide text-white/60">User Profile</h2>
    </div>
    <div className="min-h-0 flex-1 overflow-auto p-3">
      {/* User entries list */}
    </div>
  </div>

  <div className="flex flex-col rounded-lg border border-white/10 bg-white/[0.02]">
    {/* Agent Notes pane - same structure */}
  </div>
</div>
```

**3.2 Data Loading**
```tsx
const [userEntries, setUserEntries] = useState<string[]>([]);
const [memoryEntries, setMemoryEntries] = useState<string[]>([]);
const [loading, setLoading] = useState(true);

useEffect(() => {
  const load = async () => {
    const [userRes, memoryRes] = await Promise.all([
      fetch("/api/memory?target=user"),
      fetch("/api/memory?target=memory")
    ]);
    const user = await userRes.json();
    const memory = await memoryRes.json();
    setUserEntries(user.entries);
    setMemoryEntries(memory.entries);
    setLoading(false);
  };
  load();
}, []);
```

**3.3 Budget Display**
```tsx
const userBudget = { used: 847, max: 1200 };
const memoryBudget = { used: 1456, max: 2000 };

function BudgetBar({ used, max }: { used: number; max: number }) {
  const percent = (used / max) * 100;
  const color = percent > 80 ? "bg-red-400" : percent > 50 ? "bg-amber-400" : "bg-emerald-400";
  
  return (
    <div className="mt-2">
      <div className="text-xs text-white/60">{used.toLocaleString()} / {max.toLocaleString()} chars</div>
      <div className="h-1 w-full rounded bg-white/10">
        <div className={`h-full rounded ${color}`} style={{ width: `${percent}%` }} />
      </div>
    </div>
  );
}
```

**3.4 Entry Management**
- Reuse existing add/edit/delete logic from current Memory app
- Add "Add" button with modal or inline input
- Implement delete with confirmation

**3.5 Info Banner**
```tsx
<div className="mt-3 rounded-lg border border-violet-400/20 bg-violet-400/10 p-2 text-xs text-white/90">
  <InfoIcon className="mr-1 inline h-3.5 w-3.5" />
  Changes take effect in your <strong>next conversation</strong>.
</div>
```

**Acceptance Criteria**:
- Both panes load data correctly
- Budget bars display with proper colors
- Add/edit/delete work as before
- Info banner visible

---

### Phase 4: Episodes Tab (3-4 hours)

**Goal**: Full episode browsing, details, and management.

#### Tasks:

**4.1 Two-Pane Layout with Filters**
```tsx
<div className="grid grid-cols-[320px_1fr] gap-3">
  {/* Left: Episode List */}
  <div className="flex flex-col rounded-lg border border-white/10 bg-white/[0.02]">
    <div className="border-b border-white/10 px-3 py-2">
      <div className="flex items-center justify-between">
        <h2 className="text-xs font-semibold uppercase tracking-wide text-white/60">Episodes</h2>
        <button className="rounded p-1 hover:bg-white/10"><RefreshCw className="h-3.5 w-3.5" /></button>
      </div>
      {/* Filter buttons */}
      <div className="mt-2 flex gap-1">
        {["All", "Pending", "Consolidated"].map((filter) => (
          <button
            key={filter}
            onClick={() => setFilter(filter)}
            className={`rounded px-2 py-0.5 text-[10px] transition-colors ${
              filter === activeFilter ? "bg-white/10 text-white" : "text-white/60 hover:bg-white/5"
            }`}
          >
            {filter}
          </button>
        ))}
      </div>
    </div>
    <div className="min-h-0 flex-1 overflow-auto p-2">
      {episodes.map((episode) => (
        <div
          key={episode.filename}
          onClick={() => setSelectedEpisode(episode)}
          className={`cursor-pointer rounded-lg p-2 transition-colors ${
            selectedEpisode?.filename === episode.filename ? "bg-violet-400/10 border-violet-400/50" : "hover:bg-white/5"
          } border border-transparent`}
        >
          <div className="flex items-start justify-between">
            <span className="text-xs font-medium">{formatFilename(episode.filename)}</span>
            <span className={`rounded px-1.5 py-0.5 text-[9px] font-semibold uppercase ${
              episode.status === "pending" ? "bg-amber-400/15 text-amber-300" : "bg-emerald-400/15 text-emerald-300"
            }`}>
              {episode.status}
            </span>
          </div>
          <div className="mt-1 flex gap-2 text-[10px] text-white/50">
            <span><ClockIcon className="mr-0.5 inline h-2.5 w-2.5" />{formatTime(episode.createdAt)}</span>
            <span><MessageCircleIcon className="mr-0.5 inline h-2.5 w-2.5" />{episode.turnCount} turns</span>
            <span><WrenchIcon className="mr-0.5 inline h-2.5 w-2.5" />{episode.skillsUsed.length} skills</span>
          </div>
        </div>
      ))}
    </div>
  </div>

  {/* Right: Episode Details */}
  <div className="flex flex-col rounded-lg border border-white/10 bg-white/[0.02]">
    <div className="border-b border-white/10 px-3 py-2">
      <div className="flex items-center justify-between">
        <h2 className="text-xs font-semibold uppercase tracking-wide text-white/60">Episode Details</h2>
        <div className="flex gap-1">
          <button className="rounded border border-white/10 px-2 py-0.5 text-[10px] hover:bg-white/5">Review</button>
          <button className="rounded border border-red-400/20 px-2 py-0.5 text-[10px] text-red-300 hover:bg-red-400/10">Delete</button>
        </div>
      </div>
    </div>
    <div className="min-h-0 flex-1 overflow-auto p-4">
      {selectedEpisode ? <EpisodeDetail episode={selectedEpisode} /> : <EmptyState />}
    </div>
  </div>
</div>
```

**4.2 Episode Detail Component**
```tsx
function EpisodeDetail({ episode }: { episode: Episode }) {
  return (
    <div>
      <h3 className="mb-2 text-sm font-semibold">{episode.title}</h3>
      
      {/* Metadata */}
      <div className="mb-3 flex flex-wrap gap-3 text-[10px] text-white/60">
        <span><strong>Status:</strong> {episode.status}</span>
        <span><strong>Conversation:</strong> <code className="font-mono">{episode.conversationId}</code></span>
        <span><strong>Created:</strong> {formatDateTime(episode.createdAt)}</span>
        <span><strong>Skills Used:</strong> {episode.skillsUsed.join(", ")}</span>
      </div>

      {/* Sections */}
      {["Task & Outcome", "What Worked", "What Failed", "Corrections Received", "Durable Lesson Candidates", "Profile Suggestions"].map((section) => (
        <div key={section} className="mb-3">
          <h4 className="mb-1 text-[10px] font-semibold uppercase tracking-wide text-violet-300">{section}</h4>
          <div className="rounded-lg bg-white/[0.03] p-2.5 text-xs leading-relaxed text-white/85">
            {episode.sections[section]} || "No content"
          </div>
        </div>
      ))}

      {/* Navigation */}
      <div className="mt-4 flex gap-2">
        <button className="rounded border border-white/10 px-2 py-1 text-xs hover:bg-white/5"><ArrowLeftIcon className="mr-1 inline h-3.5 w-3.5" />Previous</button>
        <button className="rounded border border-white/10 px-2 py-1 text-xs hover:bg-white/5">Next<ArrowRightIcon className="ml-1 inline h-3.5 w-3.5" /></button>
      </div>
    </div>
  );
}
```

**4.3 API Integration**
```tsx
const [episodes, setEpisodes] = useState<{ pending: EpisodeMeta[]; consolidated: EpisodeMeta[] }>({ pending: [], consolidated: [] });

useEffect(() => {
  const load = async () => {
    const res = await fetch("/api/memory/episodes");
    const data = await res.json();
    setEpisodes(data);
  };
  load();
}, []);
```

**4.4 Actions Implementation**
- Review Now: `POST /api/assistant/reflect` with conversationId
- Archive: Move file to `.Archive/` (backend operation)
- Delete: Confirm then `DELETE /api/memory/episodes/:filename`

**Acceptance Criteria**:
- Episodes list loads and filters correctly
- Details display all sections
- Actions trigger backend operations
- Navigation works between episodes

---

### Phase 5: Topics Tab (2-3 hours)

**Goal**: Topic browsing and entry management.

#### Tasks:

**5.1 Two-Pane Layout**
```tsx
<div className="grid grid-cols-[320px_1fr] gap-3">
  {/* Left: Topic List */}
  <div className="flex flex-col rounded-lg border border-white/10 bg-white/[0.02]">
    <div className="border-b border-white/10 px-3 py-2">
      <div className="flex items-center justify-between">
        <h2 className="text-xs font-semibold uppercase tracking-wide text-white/60">Topics</h2>
        <button className="rounded bg-white/10 px-2 py-0.5 text-[10px] hover:bg-white/20"><PlusIcon className="mr-1 inline h-3.5 w-3.5" />New</button>
      </div>
      <input type="text" placeholder="Search topics..." className="mt-2 w-full rounded border border-white/10 bg-black/30 px-2 py-1 text-xs outline-none focus:border-white/20" />
    </div>
    <div className="min-h-0 flex-1 overflow-auto p-2">
      {topics.map((topic) => (
        <div key={topic.slug} onClick={() => setSelectedTopic(topic)} className={`cursor-pointer rounded-lg p-2 transition-colors ${selectedTopic?.slug === topic.slug ? "bg-violet-400/10" : "hover:bg-white/5"}`}>
          <div className="text-xs font-medium">{topic.slug}</div>
          <div className="mt-1 flex gap-2 text-[10px] text-white/50">
            <span><FileIcon className="mr-0.5 inline h-2.5 w-2.5" />{topic.entryCount} entries</span>
            <span><RulerIcon className="mr-0.5 inline h-2.5 w-2.5" />{formatBudget(topic.charUsage, topic.budget)}</span>
          </div>
          <div className="mt-1 h-1 w-full rounded bg-white/10">
            <div className={`h-full rounded ${getBudgetColor(topic.charUsage, topic.budget)}`} style={{ width: `${(topic.charUsage / topic.budget) * 100}%` }} />
          </div>
        </div>
      ))}
    </div>
  </div>

  {/* Right: Topic Details */}
  <div className="flex flex-col rounded-lg border border-white/10 bg-white/[0.02]">
    <div className="border-b border-white/10 px-3 py-2">
      <div className="flex items-center justify-between">
        <h2 className="text-xs font-semibold uppercase tracking-wide text-white/60">{selectedTopic?.slug}</h2>
        <div className="flex gap-1">
          <button className="rounded bg-white/10 px-2 py-0.5 text-[10px] hover:bg-white/20"><PlusIcon className="mr-1 inline h-3.5 w-3.5" />Entry</button>
          <button className="rounded border border-red-400/20 px-2 py-0.5 text-[10px] text-red-300 hover:bg-red-400/10">Delete</button>
        </div>
      </div>
    </div>
    <div className="min-h-0 flex-1 overflow-auto p-3">
      {selectedTopic?.entries.map((entry, idx) => (
        <div key={entry.id} className="mb-2 rounded-lg border-l-2 border-violet-400 bg-white/[0.03] p-2.5">
          <div className="text-xs font-semibold text-violet-300">#{idx + 1}</div>
          <div className="text-xs leading-relaxed text-white/85">{entry.text}</div>
          <div className="mt-1 text-[9px] text-white/50">{formatDateTime(entry.timestamp)} | {entry.source}</div>
        </div>
      ))}
    </div>
  </div>
</div>
```

**5.2 Entry Management**
- Add Entry: Modal with textarea, submit to `POST /api/memory` with `target: "topic"`
- Delete Entry: Confirm then `DELETE /api/memory?target=topic&topic=<slug>&id=<number>`
- Budget validation on add (reject if over limit)

**Acceptance Criteria**:
- Topics list loads with budgets
- Entries display with provenance
- Add/delete operations work
- Budget enforcement active

---

### Phase 6: Memory Loops Tab (3-4 hours)

**Goal**: Configuration interface and run history.

#### Tasks:

**6.1 Configuration Sections**
```tsx
<div className="max-w-4xl">
  {/* Fast Loop */}
  <div className="mb-4 rounded-lg border border-white/10 bg-white/[0.02] p-3.5">
    <h3 className="mb-3 flex items-center gap-2 text-xs font-semibold uppercase tracking-wide text-white/80">
      <ZapIcon className="h-3.5 w-3.5" />
      Fast Loop Configuration
    </h3>
    <div className="grid grid-cols-[140px_1fr] gap-x-3 gap-y-3">
      <label className="flex items-center gap-2 text-xs text-white/60">
        <input type="checkbox" checked={config.fastLoop.enabled} onChange={(e) => updateConfig({ fastLoop: { ...config.fastLoop, enabled: e.target.checked } })} className="h-3.5 w-3.5" />
        Enable Fast Loop
      </label>
      <div className="col-start-2 grid grid-cols-[140px_1fr] gap-x-3">
        <label className="text-xs text-white/60">Tick Interval (sec)</label>
        <input type="number" value={config.fastLoop.tickIntervalSec} onChange={(e) => updateConfig({ fastLoop: { ...config.fastLoop, tickIntervalSec: Number(e.target.value) } })} className="rounded border border-white/10 bg-black/30 px-2 py-1 text-xs outline-none focus:border-white/20" />
        
        <label className="text-xs text-white/60">Idle Threshold (sec)</label>
        <input type="number" value={config.fastLoop.idleThresholdSec} onChange={(e) => updateConfig({ fastLoop: { ...config.fastLoop, idleThresholdSec: Number(e.target.value) } })} className="rounded border border-white/10 bg-black/30 px-2 py-1 text-xs outline-none focus:border-white/20" />
        
        <label className="text-xs text-white/60">Turn Cap</label>
        <input type="number" value={config.fastLoop.turnCap} onChange={(e) => updateConfig({ fastLoop: { ...config.fastLoop, turnCap: Number(e.target.value) } })} className="rounded border border-white/10 bg-black/30 px-2 py-1 text-xs outline-none focus:border-white/20" />
      </div>
    </div>
  </div>

  {/* Slow Loop - similar structure */}
  {/* Advanced Settings - similar structure */}

  {/* Action Buttons */}
  <div className="mt-4 flex gap-2">
    <button onClick={saveConfig} className="rounded bg-white/10 px-3 py-1.5 text-xs hover:bg-white/20"><SaveIcon className="mr-1 inline h-3.5 w-3.5" />Save Configuration</button>
    <button onClick={resetDefaults} className="rounded border border-white/10 px-3 py-1.5 text-xs hover:bg-white/5"><RotateCcwIcon className="mr-1 inline h-3.5 w-3.5" />Reset Defaults</button>
  </div>
</div>
```

**6.2 Run History**
```tsx
<div className="rounded-lg border border-white/10 bg-white/[0.02] p-3.5">
  <h3 className="mb-3 flex items-center gap-2 text-xs font-semibold uppercase tracking-wide text-white/80">
    <ActivityIcon className="h-3.5 w-3.5" />
    Run History
  </h3>
  
  {history.map((run) => (
    <div key={run.id} className="mb-2 rounded-lg bg-white/[0.03] p-2.5">
      <div className="flex items-center justify-between">
        <span className="text-xs font-semibold">{run.loopType === "fast" ? <ZapIcon className="mr-1 inline h-3.5 w-3.5" /> : <ClockIcon className="mr-1 inline h-3.5 w-3.5" />}{run.loopType === "fast" ? "Fast Loop" : "Slow Loop"}</span>
        <span className="text-[9px] text-white/50">{formatRelativeTime(run.timestamp)}</span>
      </div>
      <div className="mt-1 flex flex-wrap gap-2 text-[10px] text-white/60">
        {run.stats.map((stat) => (
          <span key={stat.label}>{stat.icon} {stat.value}</span>
        ))}
      </div>
    </div>
  ))}

  <div className="mt-3 flex gap-2">
    <button onClick={() => triggerLoop("fast")} disabled={triggering} className="rounded border border-white/10 px-2 py-1 text-xs hover:bg-white/5"><ZapIcon className="mr-1 inline h-3.5 w-3.5" />Run Fast Loop Now</button>
    <button onClick={() => triggerLoop("slow")} disabled={triggering} className="rounded border border-white/10 px-2 py-1 text-xs hover:bg-white/5"><ClockIcon className="mr-1 inline h-3.5 w-3.5" />Run Slow Loop Now</button>
  </div>
</div>
```

**6.3 Configuration State**
```tsx
const [config, setConfig] = useState<MemoryLoopsConfig | null>(null);
const [saving, setSaving] = useState(false);

useEffect(() => {
  const load = async () => {
    const res = await fetch("/api/config?namespace=memoryLoops");
    const data = await res.json();
    setConfig(data);
  };
  load();
}, []);

const saveConfig = async () => {
  setSaving(true);
  try {
    await fetch("/api/config", {
      method: "POST",
      body: JSON.stringify({ namespace: "memoryLoops", values: config })
    });
    // Show success toast
  } finally {
    setSaving(false);
  }
};
```

**Acceptance Criteria**:
- All config fields load and display correctly
- Toggle switches, inputs, dropdowns work
- Save persists to backend
- Manual triggers execute loops
- Run history displays recent executions

---

### Phase 7: Search Tab (2-3 hours)

**Goal**: Cross-surface search with relevance ranking.

#### Tasks:

**7.1 Search Interface**
```tsx
<div className="max-w-4xl">
  <div className="mb-3 flex gap-2">
    <input 
      type="text" 
      placeholder="Search across all memory surfaces..." 
      value={query}
      onChange={(e) => setQuery(e.target.value)}
      onKeyPress={(e) => e.key === "Enter" && performSearch()}
      className="flex-1 rounded border border-white/10 bg-black/30 px-3 py-1.5 text-xs outline-none focus:border-white/20"
    />
    <button onClick={performSearch} disabled={searching} className="rounded bg-white/10 px-3 py-1.5 text-xs hover:bg-white/20 disabled:opacity-40"><SearchIcon className="mr-1 inline h-3.5 w-3.5" />Search</button>
  </div>

  <div className="mb-3 flex gap-1">
    {["All Sources", "Topics", "Episodes", "Memory"].map((filter) => (
      <button
        key={filter}
        onClick={() => setSourceFilter(filter)}
        className={`rounded px-2 py-0.5 text-[10px] transition-colors ${sourceFilter === filter ? "bg-white/10 text-white" : "text-white/60 hover:bg-white/5"}`}
      >
        {filter}
      </button>
    ))}
  </div>

  <div className="mb-2 text-[10px] text-white/60">
    Found <strong>{results.length}</strong> matches{query ? ` for "${query}"` : ""}
  </div>

  {results.map((result) => (
    <div key={result.source} onClick={() => navigateToSource(result.source)} className="cursor-pointer rounded-lg bg-white/[0.03] p-2.5 transition-colors hover:border-violet-400/30 hover:bg-violet-400/10">
      <div className="mb-1 text-[9px] font-mono text-violet-300">{result.source}</div>
      <div className="text-xs leading-relaxed text-white/85" dangerouslySetInnerHTML={{ __html: highlightMatches(result.content, query) }} />
      <div className="mt-1 text-[9px] text-white/50">Relevance: {Math.round(result.score * 100)}% | {result.timestamp || result.status}</div>
    </div>
  ))}

  {results.length > 20 && (
    <button onClick={loadMore} className="mt-2 rounded border border-white/10 px-3 py-1.5 text-xs hover:bg-white/5">Load More Results</button>
  )}
</div>
```

**7.2 Search Implementation**
```tsx
const [results, setResults] = useState<SearchResult[]>([]);
const [searching, setSearching] = useState(false);

const performSearch = async () => {
  if (!query.trim()) return;
  setSearching(true);
  try {
    const res = await fetch(`/api/memory/search?q=${encodeURIComponent(query)}&maxResults=50`);
    const data = await res.json();
    setResults(data.results);
  } finally {
    setSearching(false);
  }
};

const highlightMatches = (text: string, query: string) => {
  const regex = new RegExp(`(${query})`, "gi");
  return text.replace(regex, '<strong>$1</strong>');
};
```

**Acceptance Criteria**:
- Search box accepts input and executes on Enter/Click
- Results display with provenance and scores
- Highlighting works for matched terms
- Filters restrict results by source
- "Load More" pagination works

---

### Phase 8: Polish & Testing (2-3 hours)

**Goal**: Final refinements, bug fixes, documentation.

#### Tasks:

**8.1 Responsive Design**
```tsx
// Add media query for small screens
<div className="grid grid-cols-[320px_1fr] gap-3 md:grid-cols-[320px_1fr]">
  {/* On screens <768px, this stacks to single column */}
</div>
```

**8.2 Loading States**
```tsx
{loading ? (
  <div className="flex items-center justify-center py-12">
    <LoaderIcon className="h-6 w-6 animate-spin text-white/40" />
  </div>
) : error ? (
  <div className="rounded-lg border border-red-400/20 bg-red-400/10 p-3 text-xs text-red-300">{error}</div>
) : (
  {/* content */}
)}
```

**8.3 Error Handling**
- Wrap all API calls in try/catch
- Display user-friendly error messages
- Add retry buttons for transient failures

**8.4 Accessibility Audit**
- Add ARIA labels to icons: `aria-label="Add entry"`
- Ensure keyboard navigation works (Tab, Enter, Escape)
- Check focus states on all interactive elements
- Run Lighthouse accessibility audit

**8.5 TypeScript & Lint**
```bash
npx tsc --noEmit
npm run lint
```
- Fix all errors and warnings
- Add missing type definitions

**8.6 Documentation Updates**
Update `docs/usage/apps/memory.md`:
- Describe new 5-tab structure
- Explain episodes, topics, loop configuration
- Document search functionality
- Update screenshots (if applicable)

**8.7 User Testing**
- Test all user stories from spec
- Verify backward compatibility
- Check edge cases (empty states, large datasets)
- Performance testing with 100+ episodes/topics

**Acceptance Criteria**:
- Responsive on all screen sizes
- Loading/error states displayed appropriately
- Accessibility score >90
- No TypeScript errors or lint warnings
- Documentation complete and accurate

---

## Risk Mitigation

### Known Risks

1. **Backend API Gaps**: If episode/topic endpoints are missing, Phase 1 becomes critical path.
   - **Mitigation**: Prioritize backend completion; use mock data for UI development if needed.

2. **Performance with Large Datasets**: 100+ episodes may slow list rendering.
   - **Mitigation**: Implement virtual scrolling or pagination; test with realistic data sizes.

3. **Browser Compatibility**: New CSS features may not work in all browsers.
   - **Mitigation**: Test in Chrome, Firefox, Safari; use autoprefixer for vendor prefixes.

4. **State Management Complexity**: Multiple tabs with independent state could become unwieldy.
   - **Mitigation**: Use React Context or Zustand for shared state; keep tab state local when possible.

### Rollback Plan

If issues arise:
1. Revert to existing Memory app (backup current version)
2. Deploy in feature-flagged mode if supported
3. Gather user feedback before full rollout

---

## Success Metrics

- **Development**: All phases completed within estimated time (+/- 20%)
- **Quality**: Zero critical bugs; accessibility score >90
- **Performance**: Page load <1s; API responses <200ms
- **User Satisfaction**: Positive feedback from initial testers

---

## Next Steps

1. **Immediate**: Begin Phase 0 (Prerequisites & Verification)
2. **After Phase 0**: Confirm backend requirements with team
3. **Parallel Development**: UI can proceed with mock data while backend is built
4. **Integration Testing**: Once both phases complete, integrate and test end-to-end
