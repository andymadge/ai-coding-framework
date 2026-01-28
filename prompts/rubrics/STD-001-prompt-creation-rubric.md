# Prompt Engineering Rubric for Claude Code Prompts

## Overview

This rubric provides reusable patterns and templates for building robust Claude Code prompts that:
- Survive context compaction across long-running tasks
- Handle large files that exceed token limits
- Maintain state and enable reliable resumption
- Follow consistent structural patterns

Use this document as a reference when creating new prompts or enhancing existing ones.

---

## Table of Contents

1. [When to Use These Patterns](#when-to-use-these-patterns)
2. [Core Principles](#core-principles)
   - [Checkpoint Triggers (Canonical List)](#checkpoint-triggers-canonical-list)
3. [Prompt Structure Template](#prompt-structure-template)
4. [Context Compaction Survival Pattern](#context-compaction-survival-pattern)
5. [Large File Handling Pattern](#large-file-handling-pattern)
6. [Avoiding Arbitrary Limits Pattern](#avoiding-arbitrary-limits-pattern)
7. [Progress Tracking Patterns](#progress-tracking-patterns)
   - [Progress.yaml Consistency Rules](#progressyaml-consistency-rules)
8. [Checkpoint Strategies](#checkpoint-strategies)
9. [Begin Section Template](#begin-section-template)
10. [Critical Reminders Template](#critical-reminders-template)
11. [Customisation Guide](#customisation-guide)

---

## When to Use These Patterns

### Always Include Context Compaction Survival When:
- Task involves multiple phases or steps
- Work will take more than 10-15 minutes of Claude time
- Task involves reading/processing multiple source files
- Output involves creating multiple files or documents
- Task has natural checkpoint boundaries (phases, levels, personas, etc.)

### Always Include Large File Handling When:
- Source files may exceed 50KB
- Working with QUINT specifications (often large)
- Processing documentation sets
- Reading architecture documents
- Any task where you say "read all the files in..."

### Skip These Patterns When:
- Simple single-file transformations
- Quick Q&A or analysis tasks
- Tasks completable in a single response
- No file I/O required

---

## Core Principles

| Principle                           | Description                                                                                      |
| ----------------------------------- | ------------------------------------------------------------------------------------------------ |
| **Disk Over Memory**                | Write everything to `.work/` directory; context will be lost but disk persists                   |
| **Progress After Every Unit**       | Update `progress.yaml` after completing work AND before risky operations (see Checkpoint Triggers below) |
| **Summarise Then Discard**          | For large files: read chunk → extract key info → write summary → forget chunk                    |
| **Reference Not Re-read**           | Once summarised, reference the summary file; only re-read original for quotes                    |
| **Clear Next Action**               | Always document exactly what to do next for cold resumption                                      |
| **Check Before Starting**           | First action is always checking for existing progress; never restart completed work              |
| **Complete Then Move**              | Finish one unit of work completely before starting another                                       |
| **No Arbitrary Limits**             | NEVER use `head -N` or `tail -N` to limit file discovery; process ALL files or show count + warn |
| **Separate Work from Deliverables** | `.work/` is for internal tracking; `docs/` is for final artifacts humans review                  |
| **Single Source of Truth** | Store task status once in `tasks` map; derive work arrays on read, never store (see Progress.yaml Format) |

### Checkpoint Triggers (Canonical List)

Update `progress.yaml` in these situations:

| Trigger Type | When | Examples |
|--------------|------|----------|
| **Completion checkpoints** | After completing any significant work unit | Phase done, level done, file processed, document created, spec written |
| **Time-based checkpoints** | Every 5-10 minutes of active work | Regardless of completion state, save progress periodically |
| **Pre-operation checkpoints** | Before starting any complex/risky operation | Before multi-step refactor, before bulk file operations, before data transformations |
| **Discovery checkpoints** | After discovering significant findings | Critical issue found, unexpected pattern detected, blocker identified |
| **Transition checkpoints** | Before moving to next phase/level/step | Enables clean resumption at phase boundaries |

**Rule of thumb:** If you would be frustrated to lose this progress after context compaction, checkpoint NOW.

---

## Output Directory Separation

**Critical:** The `.work/` directory is for INTERNAL tracking only. Final deliverables must go to visible directories.

### Directory Purposes

| Directory              | Purpose                                | Visibility             | Contents                                        |
| ---------------------- | -------------------------------------- | ---------------------- | ----------------------------------------------- |
| **`.work/`**           | Internal tracking, compaction survival | Hidden (dot directory) | `progress.yaml`, inventories, intermediate data |
| **`docs/qa-reports/`** | QA analysis deliverables               | Visible, reviewable    | `*-REPORT.md` files                             |
| **`docs/analysis/`**   | Code analysis deliverables             | Visible, reviewable    | Analysis reports, findings                      |
| **`docs/sysdocs/`**    | System documentation                   | Visible, reviewable    | Architecture, API, onboarding docs              |

### Standard QA Output Structure

```
repository/
├── .work/                              # INTERNAL - Never deliverables
│   └── qa-analysis/
│       └── {category}/
│           ├── progress.yaml           # Compaction survival
│           ├── *-inventory.yaml        # Intermediate data
│           └── *-findings.yaml         # Raw findings
│
└── docs/                               # DELIVERABLES - Human review
    └── qa-reports/
        └── {category}/
            └── {CATEGORY}-REPORT.md    # Final report
```

### Why This Matters

1. **Dot directories are hidden** - Users won't find reports in file explorers
2. **Source control** - Reports in `docs/` can be committed and tracked
3. **CI/CD integration** - Pipelines can publish `docs/` artifacts easily
4. **Clear separation** - Working state vs deliverables are obviously different

---

## Prompt Structure Template

A well-structured Claude Code prompt follows this pattern:

```xml
# [PROMPT TITLE]

<context>
<project>[Project name and description]</project>
<role>[What role Claude is playing]</role>
<objective>[What this prompt achieves]</objective>
</context>

<foundational_principles>
[Key principles that guide all work - numbered list]
</foundational_principles>

<context_compaction_survival>
[See pattern below]
</context_compaction_survival>

<large_file_handling>
[See pattern below]
</large_file_handling>

<methodology>
[Phases and steps - the actual work to be done]
</methodology>

<output_specifications>
[What files/artifacts to produce and their format]
</output_specifications>

<critical_reminders>
[Key points that must not be forgotten - numbered list]
</critical_reminders>

<begin>
[Instructions for starting/resuming work]
</begin>
```

---

## Context Compaction Survival Pattern

### Template (Copy and Customise)

```xml
<context_compaction_survival>
  <critical_warning>
  THIS WORK WILL SPAN MULTIPLE CONTEXT COMPACTIONS.
  [Description of why this task is extensive].
  You WILL lose context multiple times during this work.
  You MUST implement strategies to survive compaction and resume work correctly.
  </critical_warning>
  
  <work_tracking_directory>
    <path>[OUTPUT_DIR]/.work/</path>
    <purpose>Persistent work state that survives context compaction</purpose>
    <critical>Create this directory FIRST before any other work</critical>
    
    <required_files>
      <file name="progress.yaml">
        <purpose>Track current [phase/level/step] and exactly what to do next</purpose>
        <updated>After EVERY [significant work unit], EVERY [milestone]</updated>
        <critical>MUST be updated frequently - this is your resumption lifeline</critical>
      </file>
      
      <file name="source-discovery.yaml">
        <purpose>Complete catalogue of all source files with sizes</purpose>
        <created>Phase 0 during discovery</created>
        <used_by>All subsequent phases for source lookup</used_by>
      </file>
      
      <!-- Add task-specific tracking files here -->
      <file name="[task-specific].yaml">
        <purpose>[What this tracks]</purpose>
        <created>[When created]</created>
        <format>[Format description]</format>
      </file>
      
      <directory name="source-summaries/">
        <purpose>Summary of each source file</purpose>
        <created>During discovery phase</created>
        <format>One .yaml per source document with key content extracted</format>
      </directory>
      
      <directory name="large-file-summaries/">
        <purpose>Chunked summaries of files too large to read at once</purpose>
        <created>When large files encountered during discovery</created>
        <format>One .yaml per large file with chunk-by-chunk summaries</format>
      </directory>
    </required_files>
  </work_tracking_directory>
  
  <progress_tracking_schema>
```yaml
# progress.yaml - UPDATE AFTER EVERY SIGNIFICANT WORK UNIT
#
# UPDATE PATTERN (2 atomic operations):
# 1. Update task status in tasks map (tasks.T7.status = in_progress)
# 2. Update pointer + next_action + timestamp (current.task = T7)
#
# DO NOT STORE: work_completed, work_in_progress, work_remaining arrays
# (derive these on read by filtering tasks by status)
#
progress:
  last_updated: "[ISO DateTime]"

  # Navigation pointers (fast resumption)
  current:
    phase: "[Phase ID]"
    task: "[Task ID or Step ID]"

  # Resumption instruction
  next_action: "[EXACTLY what to do next when resuming - be specific]"

  # Source of truth: task/phase status
  tasks:
    [task_id]:
      status: not_started | in_progress | complete | skipped
      started_at: "[ISO DateTime if started]"
      completed_at: "[ISO DateTime if complete]"
      commit: "[Git SHA if complete]"
      notes: "[Optional notes]"

  # OR for phase-based tracking:
  phases:
    phase_0_discovery:
      status: not_started | in_progress | complete
      # Phase-specific metrics

    phase_1_[name]:
      status: not_started | in_progress | complete
      # Phase-specific metrics

  # Optional: blocker tracking
  blockers: []

# DERIVED VIEWS (compute on read, never store):
#   work_completed = [k for k, v in tasks.items() if v.status == "complete"]
#   work_in_progress = [k for k, v in tasks.items() if v.status == "in_progress"]
#   work_remaining = [k for k, v in tasks.items() if v.status == "not_started"]
```
  </progress_tracking_schema>
  
  <resumption_protocol>
  WHEN CONTEXT IS COMPACTED OR SESSION RESUMES:
  
  1. IMMEDIATELY check for existing progress:
     ```bash
     cat [OUTPUT_DIR]/.work/progress.yaml 2>/dev/null || echo "NO_PROGRESS_FILE"
     ```
     
  2. IF progress file exists:
     - Read current_phase, current_step
     - Read next_action (this tells you EXACTLY what to do)
     - Check which [phases/levels/items] are complete
     - Load relevant .work/ files (source-discovery.yaml, summaries as needed)
     - Resume from next_action - do NOT restart from beginning
     - Do NOT re-read source files - use .work/ summaries
     
  3. IF no progress file (fresh start):
     - Initialize .work/ directory structure
     - Begin with Phase 0 (Discovery/Prerequisites)
     
  4. After each significant unit of work:
     - Update progress.yaml immediately
     - Write next_action clearly for potential resumption
     
  5. CHECKPOINT REQUIREMENTS (see Checkpoint Triggers in Core Principles):
     - After completing any [significant work unit]
     - Every 5-10 minutes of active work
     - Before starting any complex/risky operation
     - After discovering significant findings
     - Before transitions to next [phase/level/step]
  </resumption_protocol>
  
  <compaction_safe_practices>
    <practice>Write progress.yaml using canonical Checkpoint Triggers (see Core Principles)</practice>
    <practice>Always checkpoint: after completions, every 5-10 min, before risky operations</practice>
    <practice>Write summaries to disk, don't keep in context memory</practice>
    <practice>Reference .work/ files instead of re-reading large sources</practice>
    <practice>Complete one [unit] fully before starting another</practice>
    <practice>Document "next_action" with enough detail to resume cold</practice>
    <practice>Use .work/*.yaml as source of truth, not context memory</practice>
    <practice>Never rely on context to remember what [phases/levels] are done</practice>
  </compaction_safe_practices>
</context_compaction_survival>
```

### Customisation Points

Replace these placeholders when using the template:

| Placeholder               | Replace With                  | Examples                                |
| ------------------------- | ----------------------------- | --------------------------------------- |
| `[OUTPUT_DIR]`            | Actual output directory path  | `/home/ubuntu/src/project/docs/output`  |
| `[significant work unit]` | What constitutes a checkpoint | "spec file", "ADR", "page", "component" |
| `[phase/level/step]`      | Your task's hierarchy         | "phase", "level", "persona", "stage"    |
| `[major deliverable]`     | Key outputs                   | "spec file", "document", "verification" |
| `[task-specific].yaml`    | Additional tracking files     | "level-status.yaml", "adr-index.yaml"   |

---

## Large File Handling Pattern

### Template (Copy and Customise)

```xml
<large_file_handling>
  <critical_warning>
  Some [file type] may exceed token limits and cannot be read in one operation.
  This is especially likely for:
  - [List of file types that tend to be large]
  - [Another type]
  - [Another type]
  You MUST detect and handle large files appropriately.
  </critical_warning>
  
  <detection_strategy>
  During [discovery phase]:
  
  1. Get file sizes for ALL source files:
     ```bash
     find [SOURCE_DIR] -type f \( -name "*.md" -o -name "*.qnt" -o -name "*.yaml" \) -exec ls -la {} \;
     ```
     
  2. Categorise by size:
     - Small: < 50KB (safe to read entirely)
     - Medium: 50-100KB (usually OK, monitor for truncation)
     - Large: > 100KB (requires chunked reading)
     
  3. For large files, calculate estimated chunks:
     - Assume ~300-500 lines per chunk as safe default
     - Use: wc -l [file] to get line count
     - Denser content (specs, code) → smaller chunks (~300 lines)
     - Prose content (docs) → larger chunks (~500 lines)
     
  4. Record in source-discovery.yaml:
```yaml
source_files:
  - file: "[filename]"
    path: "[full path]"
    size_bytes: [size]
    size_category: "small | medium | large"
    line_count: [lines]
    requires_chunked_reading: true | false
    estimated_chunks: [N]  # if large
    content_type: "[description of content]"
    
  large_files_summary:
    count: [N]
    total_size_mb: [size]
    files:
      - "[filename1]"
      - "[filename2]"
```
  </detection_strategy>
  
  <chunked_reading_strategy>
  For files marked as "large":
  
  1. Read file in sections using line ranges:
     ```
     view /path/to/file.md [1, 300]
     view /path/to/file.md [301, 600]
     view /path/to/file.md [601, 900]
     # etc.
     ```
     
  2. After reading EACH chunk, immediately extract:
     - [Key item type 1 relevant to your task]
     - [Key item type 2]
     - [Key item type 3]
     - Cross-references to other files
     
  3. Write chunk summary to large-file-summaries/:
```yaml
# large-file-summaries/[FILE_ID].yaml
file: "[filename]"
path: "[full path]"
total_lines: [N]
total_chunks: [N]
chunks_processed: [N]
fully_summarised: true | false

chunk_summaries:
  - chunk: 1
    lines: "1-300"
    content_type: "[What this chunk contains]"
    key_items:
      - "[Item 1]"
      - "[Item 2]"
    # Task-specific extracted data
    [custom_field]:
      - [extracted data]
      
  - chunk: 2
    lines: "301-600"
    content_type: "[What this chunk contains]"
    key_items:
      - "[Item 3]"
    # ... continue pattern

aggregate_summary:
  total_[items]: [N]
  by_category:
    [category1]: [N]
    [category2]: [N]
  key_topics:
    - "[Topic 1]"
    - "[Topic 2]"
```
  </chunked_reading_strategy>
  
  <using_summaries_for_work>
  When doing work that references large files:
  
  1. FIRST: Read the summary from large-file-summaries/ (small file, fits in context)
  
  2. Use aggregate_summary for high-level information
  
  3. If specific detail needed:
     - Check chunk_summaries to find which chunk has the content
     - Read ONLY that chunk: view [file] [start, end]
     - Extract the specific [item] needed
     - Do NOT keep entire file in context
     
  4. Cite using file + chunk reference:
     "[Source: [filename], Chunk N, Lines X-Y]"
     
  5. For comprehensive outputs:
     - Use aggregate_summary from the summary file
     - Pull specific details chunk by chunk as needed
     - Write output incrementally, saving after each section
     - If compacted, resume from saved progress
  </using_summaries_for_work>
  
  <memory_efficient_patterns>
    <pattern name="Summarise then discard">
      Read chunk → Extract key info → Write to summary file → Move to next chunk
      Don't try to keep entire large file in context.
    </pattern>
    
    <pattern name="Reference not re-read">
      Once summarised, reference the summary file.
      Only re-read original when exact wording/syntax needed.
    </pattern>
    
    <pattern name="Incremental output building">
      For outputs requiring large file content:
      - Write output section by section
      - Save after each section
      - Update progress.yaml with what's done
      - If compacted, resume from saved progress
    </pattern>
    
    <pattern name="Targeted chunk access">
      Need specific item? Don't re-read whole file.
      1. Read summary to find which chunk has it
      2. Read only that chunk
      3. Extract what you need
      4. Discard chunk from context
    </pattern>
  </memory_efficient_patterns>
</large_file_handling>
```

### Size Thresholds Reference

| Category | Size     | Line Count (est.) | Handling                                             |
| -------- | -------- | ----------------- | ---------------------------------------------------- |
| Small    | < 50KB   | < 800 lines       | Read entirely, still summarise to disk               |
| Medium   | 50-100KB | 800-1500 lines    | Usually OK, summarise anyway, monitor for truncation |
| Large    | > 100KB  | > 1500 lines      | **Chunked reading mandatory**                        |

### Chunk Size Recommendations

| Content Type      | Lines per Chunk | Rationale                 |
| ----------------- | --------------- | ------------------------- |
| QUINT specs       | 300-400         | Dense, many definitions   |
| Code files        | 300-400         | Dense, need context       |
| Markdown docs     | 400-500         | Prose is less dense       |
| YAML/JSON         | 200-300         | Structured, easy to break |
| Architecture docs | 300-400         | Mixed content             |

---

## Avoiding Arbitrary Limits Pattern

### The Problem

Arbitrary limits like `head -20` or `tail -50` in file discovery cause **silent data loss**:
- Files beyond the limit are never processed
- Verification passes on partial data
- Issues in truncated files go undetected
- As the project grows, more data is silently dropped

### The Rule: NEVER Truncate File Discovery

```bash
# ❌ DANGEROUS - Silent data loss
for file in $(find . -name "*.cs" | head -50); do
  process "$file"
done

# ❌ DANGEROUS - Verifies only 20 of potentially 500+ files
for file in $(find "$DIR" -name "*.g.cs" | head -20); do
  verify "$file"
done
```

### Safe Patterns

#### Pattern 1: Process ALL Files (Preferred)
```bash
# ✅ SAFE - Processes everything
for file in $(find . -name "*.cs"); do
  process "$file"
done
```

#### Pattern 2: Show Count + Process All
```bash
# ✅ SAFE - Transparent about volume
TOTAL=$(find . -name "*.cs" | wc -l)
echo "Processing $TOTAL files..."
PROCESSED=0
for file in $(find . -name "*.cs"); do
  ((PROCESSED++))
  echo "[$PROCESSED/$TOTAL] $(basename "$file")"
  process "$file"
done
```

#### Pattern 3: Warn if Multiple When Expecting Single
```bash
# ✅ SAFE - Use when you expect exactly one file
COUNT=$(find "$DIR" -maxdepth 1 -name "*.csproj" | wc -l)
if [ "$COUNT" -eq 0 ]; then
  echo "ERROR: No .csproj found in $DIR"
  exit 1
elif [ "$COUNT" -gt 1 ]; then
  echo "WARNING: Multiple .csproj files in $DIR:"
  find "$DIR" -maxdepth 1 -name "*.csproj"
  echo "Using first one - verify this is correct"
fi
FILE=$(find "$DIR" -maxdepth 1 -name "*.csproj" | head -1)
```

#### Pattern 4: Explicit Sampling (When Justified)
```bash
# ✅ ACCEPTABLE - Only when full processing is impossible
# Must be explicitly justified and transparent
TOTAL=$(find . -name "*.cs" | wc -l)
SAMPLE_SIZE=100

if [ "$TOTAL" -gt "$SAMPLE_SIZE" ]; then
  echo "⚠️ SAMPLING: Checking $SAMPLE_SIZE of $TOTAL files (random sample)"
  echo "   Full verification would take too long"
  FILES=$(find . -name "*.cs" | shuf | head -$SAMPLE_SIZE)
else
  FILES=$(find . -name "*.cs")
fi

for file in $FILES; do
  verify "$file"
done
```

### When head -1 IS Safe

`head -1` is acceptable when extracting a **single value**, not when limiting results:

```bash
# ✅ SAFE - Extracting single value from command output
JAVA_VERSION=$(java -version 2>&1 | head -1)

# ✅ SAFE - Parsing YAML for specific field
STATUS=$(grep "status:" progress.yaml | head -1 | awk '{print $2}')

# ✅ SAFE - Getting first match from grep (intentional)
FIRST_ERROR=$(grep "ERROR" build.log | head -1)
```

### When tail -N IS Safe

`tail -N` is acceptable for **display purposes** (showing recent output):

```bash
# ✅ SAFE - Display only, not processing
echo "Last 20 lines of build output:"
cat build.log | tail -20

# ✅ SAFE - Showing test summary
quint test spec.qnt 2>&1 | tail -20
```

### Checklist for Prompt Authors

Before finalising any prompt, verify:

- [ ] No `head -N` (N>1) in any `find` pipeline
- [ ] No `tail -N` limiting file discovery
- [ ] All file loops process complete results
- [ ] Any single-file selection (`head -1`) warns if multiple found
- [ ] Sampling is explicit, justified, and transparent
- [ ] File counts are displayed before processing

### Real-World Impact

| Pattern             | Files Found | Files Processed | Data Loss |
| ------------------- | ----------- | --------------- | --------- |
| `find \| head -20`  | 509         | 20              | **96%**   |
| `find \| head -50`  | 509         | 50              | **90%**   |
| `find \| head -100` | 509         | 100             | **80%**   |
| `find` (no limit)   | 509         | 509             | **0%**    |

---

## Progress Tracking Patterns

### Basic Progress.yaml Structure

```yaml
progress:
  last_updated: "2025-01-06T10:30:00Z"
  current_phase: "2"
  current_step: "2.3"
  status: "In Progress"
  
  phases:
    phase_0:
      status: "Complete"
      completed_at: "2025-01-06T09:00:00Z"
    phase_1:
      status: "Complete"
      completed_at: "2025-01-06T10:00:00Z"
    phase_2:
      status: "In Progress"
      steps_completed: ["2.1", "2.2"]
      current_step: "2.3"
      
  work_completed:
    - item: "Source discovery"
      completed_at: "2025-01-06T09:00:00Z"
    - item: "File summaries"
      completed_at: "2025-01-06T09:30:00Z"
      
  work_in_progress:
    - item: "Component design - Payment service"
      status: "API defined, implementation pending"
      
  work_remaining:
    - "Component design - Notification service"
    - "Integration testing specs"
    - "Documentation"
    
  blockers: []

  next_action: "Complete Payment service implementation in phase 2, step 2.3. Read existing API from .work/payment-api.yaml and generate implementation."

# CONSISTENCY CHECK (9 invariants):
# ✓ current_phase="2" matches phases.phase_2.status="In Progress"
# ✓ current_step="2.3" matches phases.phase_2.current_step="2.3"
# ✓ next_action references "phase 2, step 2.3" (current pointers)
# ✓ Only phase_2 has status="In Progress"
# ✓ Completed phases (phase_1) have completed_at timestamps
# ✓ work_completed contains completed phases/items
# ✓ work_in_progress contains only current phase 2 work
# ✓ work_remaining does not contain completed items
# ✓ last_updated is present
```

### Progress.yaml Format (Single Source of Truth)

#### Design Philosophy

Progress.yaml stores task status once in the `tasks` map. Everything else (work arrays, overall status) is derived on read. This eliminates conflict risk while maintaining fast resumption.

**Key principle:** Store state once, derive views on demand.

#### Format Structure

**Stored Fields:**

| Field | Purpose | Example |
|-------|---------|---------|
| `tasks.*.status` | **SOURCE OF TRUTH** - Actual task state | `in_progress`, `complete`, `not_started`, `skipped` |
| `phases.*.status` | **SOURCE OF TRUTH** - Actual phase state | `in_progress`, `complete`, `not_started` |
| `current.task` | Navigation pointer for fast resumption | `"T7"` |
| `current.phase` | Navigation pointer for fast resumption | `"Phase 1 - Tasks"` |
| `next_action` | Resumption instruction (what to do next) | `"Implement T7: Create CompaniesHouseClientImpl..."` |
| `last_updated` | Audit timestamp | `"2026-01-27T20:05:00Z"` |
| `blockers` | Optional blocker tracking | `[]` or list of blocker descriptions |

**Derived Fields (NEVER store these):**

| Field | How to Derive | Example |
|-------|---------------|---------|
| `work_completed` | Filter tasks where `status == "complete"` | `["T1", "T2", "T3", "T4", "T5", "T6"]` |
| `work_in_progress` | Filter tasks where `status == "in_progress"` | `["T7"]` |
| `work_remaining` | Filter tasks where `status == "not_started"` | `["T8", "T10", "T11"]` |
| `status` (overall) | `"in_progress"` if any in_progress, `"complete"` if all done, else `"not_started"` | `"in_progress"` |

**Critical:** Arrays like `work_completed`, `work_in_progress`, `work_remaining` are **computed on read** from the tasks map. Never store them in the file - this eliminates synchronization conflicts.

#### Format Template

```yaml
# progress.yaml - NEW FORMAT (Single Source of Truth)
# Work arrays are DERIVED on read, never stored
progress:
  last_updated: "[ISO DateTime]"

  # Navigation pointers (fast resumption)
  current:
    phase: "[Phase ID]"
    task: "[Task ID]"

  # Resumption instruction
  next_action: "[EXACTLY what to do next when resuming]"

  # Source of truth: task status (the ONLY place storing actual state)
  tasks:
    [task_id]:
      status: not_started | in_progress | complete | skipped
      started_at: "[ISO DateTime if in_progress or complete]"
      completed_at: "[ISO DateTime if complete]"
      commit: "[Git SHA if complete]"
      notes: "[Optional notes]"
      files_created: [...]  # Optional
      files_to_create: [...] # Optional
      tests_passing: true/false  # Optional

  blockers: []

# HOW TO DERIVE WORK ARRAYS (do this on read, never store):
#   work_completed = [k for k, v in tasks.items() if v.status == "complete"]
#   work_in_progress = [k for k, v in tasks.items() if v.status == "in_progress"]
#   work_remaining = [k for k, v in tasks.items() if v.status == "not_started"]
```

#### Update Pattern (2 Atomic Operations)

**Scenario:** Complete T6 → Start T7

```yaml
# BEFORE
current:
  task: "T6"
tasks:
  T6_client_interface:
    status: in_progress
    started_at: "2026-01-27T19:55:00Z"
  T7_client_success_path:
    status: not_started

# AFTER (2 operations)

# OPERATION 1: Update task status in tasks map
tasks:
  T6_client_interface:
    status: complete  # ← Changed
    started_at: "2026-01-27T19:55:00Z"
    completed_at: "2026-01-27T20:04:00Z"  # ← Added
    commit: "8f4e2c1"  # ← Added
  T7_client_success_path:
    status: in_progress  # ← Changed
    started_at: "2026-01-27T20:05:00Z"  # ← Added

# OPERATION 2: Update pointer + next_action + timestamp
current:
  task: "T7"  # ← Changed
next_action: "Implement T7: Create CompaniesHouseClientImpl"  # ← Changed
last_updated: "2026-01-27T20:05:00Z"  # ← Changed
```

**That's it.** No work arrays to synchronize. Just 2 atomic updates.


#### Validation Rules (Simple)

After updating progress.yaml, verify:

- [ ] Exactly **ONE** task has `status: in_progress` (or zero if between tasks)
- [ ] `current.task` matches the task with `status: in_progress`
- [ ] `next_action` describes work for `current.task`
- [ ] Completed tasks have `completed_at` timestamp
- [ ] `last_updated` is current
- [ ] **NO** `work_completed`, `work_in_progress`, or `work_remaining` arrays in file

**That's it.** 6 checks instead of 9 invariants + 6-step sequence.

#### Common Mistakes to Avoid

| Mistake | Why It's Wrong | Correct Approach |
|---------|----------------|------------------|
| Storing work arrays in file | Creates redundancy and conflict risk | Derive arrays on read from tasks |
| Multiple tasks with `status: in_progress` | Unclear what's current | Only ONE task should be `in_progress` at a time |
| `current.task` doesn't match `in_progress` task | Pointer inconsistency | Always update task status BEFORE updating pointer |
| Forgetting to update `next_action` | Stale resumption instruction | Update `next_action` whenever `current.task` changes |
| Using uppercase status values | Inconsistent with convention | Use lowercase: `in_progress`, `complete`, `not_started`, `skipped` |

#### Derivation Logic for Claude

When resuming work after context compaction, Claude computes derived views:

```python
# Read progress.yaml
with open('.work/progress.yaml') as f:
    data = yaml.safe_load(f)

tasks = data['progress']['tasks']

# Derive work arrays
work_completed = [
    {"task": k, "completed_at": v.get("completed_at"), "commit": v.get("commit")}
    for k, v in tasks.items()
    if v["status"] == "complete"
]

work_in_progress = [
    {"task": k, "status": v.get("notes", "In progress")}
    for k, v in tasks.items()
    if v["status"] == "in_progress"
]

work_remaining = [
    k for k, v in tasks.items()
    if v["status"] == "not_started"
]

# Derive overall status
if all(v["status"] in ["complete", "skipped"] for v in tasks.values()):
    overall_status = "complete"
elif any(v["status"] == "in_progress" for v in tasks.values()):
    overall_status = "in_progress"
else:
    overall_status = "not_started"

# Resume from current pointer
current_task = data['progress']['current']['task']
next_action = data['progress']['next_action']
```

**Claude doesn't need to write this code** - the resumption protocol just reads `current.task` and `next_action`, then checks task status as needed. Derived arrays are for human review only.

---
---

### Task-Specific Progress Extensions

#### For Multi-Level Tasks (like 01c cross-context testing)
```yaml
levels:
  level_1:
    status: "Complete"
    specs_created: 5
    counterexamples_found: 2
  level_2:
    status: "In Progress"
    current_context: "Pipeline"
```

#### For Multi-Persona Tasks (like 01f verification)
```yaml
personas:
  security_architect:
    status: "Complete"
    findings_critical: 1
    findings_high: 3
  cost_analyst:
    status: "In Progress"
    sections_reviewed: 3
```

#### For Multi-Phase Architecture (like 01e)
```yaml
phases:
  discovery:
    status: "Complete"
  high_level:
    status: "Complete"
    adrs_created: 5
  detailed:
    status: "In Progress"
    components_designed: 3
    components_remaining: 4
```

---

## Checkpoint Strategies

### When to Checkpoint

**See canonical Checkpoint Triggers in Core Principles section above.**

Summary of checkpoint actions:

| Trigger Type | Action |
| ------------ | ------ |
| Completion checkpoints | Update progress.yaml with completed status, write summary |
| Time-based checkpoints (5-10 min) | Quick progress.yaml update with current state |
| Pre-operation checkpoints | Save current state before starting risky work |
| Discovery checkpoints | Write findings to appropriate file immediately |
| Transition checkpoints | Update phase status, document next_action clearly, **verify consistency** (see Progress.yaml Consistency Rules) |

### Checkpoint File Naming

```
.work/
├── progress.yaml                    # Always present
├── source-discovery.yaml            # After discovery
├── [phase]-checkpoint.yaml          # After each phase
├── [level]-checkpoint.yaml          # After each level
├── source-summaries/
│   └── [FILE_ID].yaml              # Per source file
├── large-file-summaries/
│   └── [FILE_ID].yaml              # Per large file
└── [task-specific]/
    └── [task-specific-files].yaml  # As needed
```

---

## Begin Section Template

Use this template for the `<begin>` section of any prompt:

```xml
<begin>
=====================================
CRITICAL: CHECK FOR EXISTING PROGRESS FIRST
=====================================
This work may have been started before context compaction.

FIRST ACTION - Check for existing progress:
```bash
cat [OUTPUT_DIR]/.work/progress.yaml 2>/dev/null || echo "NO_PROGRESS_FILE"
```

IF progress file exists:
- Read current_phase, current_step, next_action
- Resume from where you left off
- Do NOT restart from beginning
- Use .work/ summaries, not re-reading sources

IF no progress file (fresh start):
- Proceed with Phase 0 (Discovery/Prerequisites)
- Create .work/ directory structure first

=====================================
CRITICAL: COMPACTION SURVIVAL
=====================================
This work WILL span multiple context compactions.

CHECKPOINT PROGRESS.YAML:
- After completing any significant work unit
- Every 5-10 minutes of active work
- Before starting complex/risky operations
- After discovering significant findings
- Before transitions to next phase/level/step

ALWAYS:
- Write summaries to .work/ directories, not to context memory
- Complete one unit of work fully before starting another
- Document next_action clearly for resumption

=====================================
CRITICAL: LARGE FILE HANDLING
=====================================
Some source files exceed token limits.

For files >100KB:
- Read in chunks of ~300-500 lines
- Summarise each chunk immediately
- Write to .work/large-file-summaries/
- Use summaries for subsequent work, not re-reading original

=====================================
BEGIN NOW
=====================================
FIRST: Check for existing progress (see command above)

IF resuming: Follow next_action from progress.yaml

IF fresh start: 
1. Create .work/ directory structure
2. Run discovery on source files
3. Proceed with Phase 0

[Add any task-specific starting instructions here]
</begin>
```

---

## Critical Reminders Template

Use this template for the `<critical_reminders>` section:

```xml
<critical_reminders>
================================================================================
                    CRITICAL REMINDERS
================================================================================

1. **STATE IN FILES, NOT CONTEXT**
   - progress.yaml is truth
   - Context may compact any time
   - Checkpoint: after completions, every 5-10 min, before risky ops, after findings, before transitions

2. **CHECK BEFORE STARTING**
   - Always read progress.yaml first
   - Resume from next_action if exists
   - Never restart completed work

2a. **UPDATE PROGRESS.YAML (2-step pattern)**
   - Update task status FIRST (tasks.T7.status = in_progress)
   - Update pointer + next_action SECOND (current.task = T7, next_action = "...")
   - DO NOT store work_completed, work_in_progress, work_remaining arrays
   - Derive work arrays on read by filtering tasks by status
   - Only ONE task should have status=in_progress at a time

3. **LARGE FILES NEED CHUNKING**
   - Files >100KB require chunked reading
   - Summarise to .work/ as you go
   - Reference summaries, not originals

4. **COMPLETE BEFORE MOVING ON**
   - Finish one [unit] before starting another
   - Write checkpoint before transitions
   - Document what comes next

5. **[TASK-SPECIFIC REMINDER 1]**
   - [Details]

6. **[TASK-SPECIFIC REMINDER 2]**
   - [Details]

[Add more task-specific reminders as needed]

</critical_reminders>
```

---

## Customisation Guide

### Step 1: Determine Task Characteristics

Answer these questions:
1. How many phases/levels/steps? → Determines progress structure
2. What are the major deliverables? → Determines checkpoint triggers
3. What source files are involved? → Determines large file handling
4. What task-specific state needs tracking? → Determines additional .work/ files

### Step 2: Customise Progress Tracking

Based on your task structure:
- **Linear phases**: Use `phase_0`, `phase_1`, etc.
- **Levels**: Use `level_1`, `level_2`, etc.
- **Personas/Actors**: Use named personas
- **Parallel work streams**: Use named work streams

### Step 3: Customise Large File Handling

Based on your source files:
- **QUINT specs**: Extract invariants, state machines, types
- **Architecture docs**: Extract decisions, components, constraints
- **Requirements**: Extract requirements, acceptance criteria
- **Code files**: Extract interfaces, key functions, dependencies

### Step 4: Add Task-Specific Tracking

Common additions:
- `counterexamples.yaml` - For verification tasks
- `adr-index.yaml` - For architecture tasks
- `findings.yaml` - For review/audit tasks
- `[entity]-status.yaml` - For multi-entity tasks

### Step 5: Test Resumption

Before finalising a prompt:
1. Run it until partway through
2. Simulate compaction (start fresh context)
3. Verify it resumes correctly from progress.yaml
4. Verify it uses summaries instead of re-reading files

---

## Quick Reference Card

### First Action (Always)
```bash
cat [OUTPUT_DIR]/.work/progress.yaml 2>/dev/null || echo "NO_PROGRESS_FILE"
```

### Directory Structure
```
.work/
├── progress.yaml           # ALWAYS - update frequently
├── source-discovery.yaml   # ALWAYS - file catalogue
├── source-summaries/       # Per-file summaries
├── large-file-summaries/   # Chunked summaries
└── [task-specific]/        # As needed
```

### Size Thresholds
- Small: < 50KB → Read entirely
- Medium: 50-100KB → Monitor for truncation
- Large: > 100KB → **Chunk required**

### Checkpoint Triggers (Update progress.yaml)
- ✅ After completing work units (phases, levels, files, documents)
- ✅ Every 5-10 minutes of active work (time-based safety net)
- ✅ Before complex/risky operations (pre-operation safety)
- ✅ After discovering significant findings (capture insights immediately)
- ✅ Before phase/level/step transitions (clean resumption points)

### Progress.yaml Update Pattern (2 Steps)
1. Update task status (tasks.T7.status = "in_progress") ← **SOURCE OF TRUTH**
2. Update pointer + next_action + timestamp (current.task = "T7", next_action = "...", last_updated = "...")

**Rule:** Task status → Pointer + next_action

**DO NOT STORE work arrays** - derive on read:
- work_completed = filter tasks where status == "complete"
- work_in_progress = filter tasks where status == "in_progress"
- work_remaining = filter tasks where status == "not_started"

**Key invariant:** Only ONE task has status="in_progress" at a time

**See:** Progress.yaml Format section for full template and examples

### Memory Patterns
1. Summarise then discard
2. Reference not re-read
3. Incremental output building
4. Targeted chunk access

---

## Version History

| Version | Date       | Changes                                            |
| ------- | ---------- | -------------------------------------------------- |
| 1.3     | 2026-01-28 | Redesigned progress.yaml format (single source of truth); work arrays derived on read; 2-step update pattern |
| 1.1     | 2026-01-28 | Consolidated checkpoint trigger guidance into canonical list; resolved inconsistencies in progress.yaml update timing |
| 1.0     | 2026-01-06 | Initial rubric created from 01c, 01e, 01f patterns |

---

## Related Documents

- 