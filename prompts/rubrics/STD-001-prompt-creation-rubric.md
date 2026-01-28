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
| **Consistent Progress Updates** | Update `progress.yaml` in sequence: detail status → pointers → status → work arrays → next_action → timestamp (see Progress.yaml Consistency Rules) |

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
# CONSISTENCY RULES (6-step sequence):
# 1. Update detail-level status FIRST (phases.*.status, tasks.*.status)
# 2. Update top-level pointers SECOND (current_phase, current_step, current_task)
# 3. Update overall status THIRD (status field)
# 4. Update work arrays FOURTH (work_completed, work_in_progress, work_remaining)
# 5. Update next_action FIFTH (must reference current pointers)
# 6. Update last_updated ALWAYS
# 7. Verify all 9 invariants hold (see Progress.yaml Consistency Rules)
#
progress:
  last_updated: "[ISO DateTime]"
  current_phase: "[Phase ID]"
  current_step: "[Step ID]"
  status: "In Progress | Blocked | Complete"
  
  # Task-specific phase tracking
  phases:
    phase_0_discovery:
      status: "Not Started | In Progress | Complete"
      # Phase-specific metrics
      
    phase_1_[name]:
      status: "Not Started | In Progress | Complete"
      # Phase-specific metrics
      
  # What's done
  work_completed:
    - item: "[Completed item]"
      completed_at: "[DateTime]"
      
  # What's in progress
  work_in_progress:
    - item: "[Current item]"
      status: "[What's done, what remains]"
      
  # What's remaining
  work_remaining:
    - "[List of pending items]"
    
  # Any blockers
  blockers:
    - "[Any issues preventing progress]"
    
  # CRITICAL: Exactly what to do next
  next_action: "[EXACTLY what to do next when resuming - be specific]"
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

### Progress.yaml Consistency Rules

#### The Consistency Problem

Progress.yaml has **redundant fields by design** (top-level pointers + detail-level status + work tracking arrays). This redundancy enables fast resumption but creates risk of internal conflicts:

```yaml
# INCONSISTENT - DO NOT DO THIS (real example from production)
progress:
  current_task: "T7"                        # Says T7

  tasks:
    T6_client_interface:
      status: "Complete"                     # T6 is complete
      completed_at: "2026-01-27T20:04:00Z"
    T7_client_success_path:
      status: "Not Started"                  # CONFLICT: Says not started but is current!

  work_in_progress:
    - task: "T6"                             # CONFLICT: T6 is complete, not in progress!
      status: "Starting Client Interface"

  work_remaining:
    - "T6: Client interface"                 # CONFLICT: T6 is complete, not remaining!
    - "T7: Client success path"

  work_completed:
    - task: "T5"                             # CONFLICT: Missing T6 entirely!
      completed_at: "2026-01-27T20:03:00Z"

  next_action: "Create T6: CompaniesHouseClient interface..."  # CONFLICT: Should say T7!
```

**5 distinct conflicts** - When Claude reads this after context compaction, which field is truth?

#### Single Source of Truth Principle

**Detail-level status is always truth. Top-level pointers and arrays are derived.**

| Field Type | Truth Level | Update Order | Purpose |
|------------|-------------|--------------|---------|
| `tasks.T7.status` | **SOURCE OF TRUTH** | **1st** | Authoritative state |
| `phases.phase_X.status` | **SOURCE OF TRUTH** | **1st** | Authoritative state |
| `levels.level_N.status` | **SOURCE OF TRUTH** | **1st** | Authoritative state |
| `current_task` | Derived pointer | 2nd | Fast lookup only |
| `current_phase` | Derived pointer | 2nd | Fast lookup only |
| `current_step` | Derived pointer | 2nd | Fast lookup only |
| `status` | Derived summary | 3rd | Overall status |
| `work_completed` | Derived array | 4th | Completed items list |
| `work_in_progress` | Derived array | 4th | Current items list |
| `work_remaining` | Derived array | 4th | Remaining items list |
| `next_action` | Derived instruction | 5th | Resumption guide |
| `last_updated` | Metadata | **ALWAYS** | Audit trail |

#### Update Sequence Rules

**ALWAYS update progress.yaml fields in this order:**

```yaml
# STEP 1: Update detail-level status FIRST (source of truth)
tasks:
  T6_client_interface:
    status: "Complete"                      # ← Update detail status FIRST
    completed_at: "2026-01-27T20:04:00Z"   # ← Add timestamp
  T7_client_success_path:
    status: "In Progress"                   # ← Must match current_task pointer
    started_at: "2026-01-27T20:05:00Z"     # ← Add timestamp

# STEP 2: Update top-level pointers SECOND (must match step 1)
current_task: "T7"                          # ← Points to task with "In Progress" status
current_phase: "Phase 1 - Tasks"            # ← Phase containing T7

# STEP 3: Update overall status THIRD (derived from all tasks)
status: "In Progress"                       # ← Still work to do

# STEP 4: Update work tracking arrays FOURTH (must be synchronized)
work_completed:
  - task: "T5"
    completed_at: "2026-01-27T20:03:00Z"
  - task: "T6"                              # ← Add T6 (it's complete)
    completed_at: "2026-01-27T20:04:00Z"
    commit: "abc123f"

work_in_progress:
  - task: "T7"                              # ← T7 now in progress (remove T6)
    status: "Starting client implementation"

work_remaining:
  - "T7: Client success path"               # ← Remove T6 (it's complete)
  - "T8: Client error handling"
  - "T10: Integration tests"

# STEP 5: Update next_action FIFTH (must reference step 2 pointers and current work)
next_action: "Implement T7: CompaniesHouseClientImpl with getRegisteredAddress method. Follow success path from .work/t7-spec.yaml"
                                            # ← References T7, NOT T6

# STEP 6: Update timestamp ALWAYS
last_updated: "2026-01-27T20:05:00Z"       # ← Every update gets new timestamp
```

#### Consistency Invariants (Must Always Hold)

These rules **MUST be true** after every progress.yaml update:

```yaml
# INVARIANT 1: Top-level pointer matches detail-level status
# If current_task = "T7"
# Then tasks.T7.status MUST be "In Progress" (not "Not Started", not "Complete")

# INVARIANT 2: next_action references current pointers
# If current_task = "T7"
# Then next_action MUST mention "T7" or describe T7 work (not T6, not T8)

# INVARIANT 3: Overall status reflects task completion
# If status = "Complete"
# Then ALL tasks.*.status MUST be "Complete" or "Skipped"

# INVARIANT 4: Completed tasks have completed_at timestamps
# If tasks.T6.status = "Complete"
# Then tasks.T6.completed_at MUST exist

# INVARIANT 5: In-progress tasks match current pointer
# If tasks.T7.status = "In Progress"
# Then current_task MUST be "T7" (unless T7 is blocked)

# INVARIANT 6: Only ONE task/phase can be "In Progress"
# If tasks.T7.status = "In Progress"
# Then ALL OTHER tasks.*.status MUST be "Complete", "Not Started", or "Skipped"

# INVARIANT 7: work_completed contains ALL completed tasks
# If tasks.T6.status = "Complete"
# Then work_completed array MUST contain T6 entry

# INVARIANT 8: work_in_progress contains ONLY current task
# If current_task = "T7"
# Then work_in_progress array MUST contain T7 entry
# And work_in_progress MUST NOT contain T6 (or any other completed task)

# INVARIANT 9: work_remaining does NOT contain completed tasks
# If tasks.T6.status = "Complete"
# Then work_remaining array MUST NOT contain "T6: ..." entry
```

#### Worked Example: Incorrect vs Correct Update

**INCORRECT UPDATE (Creates 5 Conflicts):**

Starting from T6 complete, moving to T7 - but updated inconsistently:

```yaml
# After completing T6, Claude only updates SOME fields:
progress:
  last_updated: "2026-01-27T20:04:00Z"
  current_task: "T7"                        # ✓ Updated to T7
  status: "In Progress"                     # ✓ Still in progress

  tasks:
    T6_client_interface:
      status: "Complete"                     # ✓ Marked complete
      completed_at: "2026-01-27T20:04:00Z"
    T7_client_success_path:
      status: "Not Started"                  # ⚠️ CONFLICT 1: Says not started but current_task=T7!

  work_completed:
    - task: "T5"                             # ⚠️ CONFLICT 2: Missing T6!
      completed_at: "2026-01-27T20:03:00Z"

  work_in_progress:
    - task: "T6"                             # ⚠️ CONFLICT 3: T6 is complete, not in progress!
      status: "Starting Client Interface"

  work_remaining:
    - "T6: Client interface"                 # ⚠️ CONFLICT 4: T6 is complete, not remaining!
    - "T7: Client success path"

  next_action: "Create T6: CompaniesHouseClient interface..."  # ⚠️ CONFLICT 5: Should say T7!
```

**Result:** Inconsistent state. After compaction, Claude doesn't know:
- Is T7 started or not started?
- Is T6 still in progress or complete?
- What should be done next - T6 or T7?

**CORRECT UPDATE (Follows 6-Step Sequence):**

```yaml
# After completing T6, Claude updates in CORRECT SEQUENCE:
progress:
  # STEP 6: Update timestamp ALWAYS
  last_updated: "2026-01-27T20:05:00Z"     # ← New timestamp for this update

  # STEP 2: Update top-level pointer SECOND
  current_task: "T7"                        # ← Points to T7 (matches T7 status)
  current_phase: "Phase 1 - Tasks"

  # STEP 3: Update overall status THIRD
  status: "In Progress"                     # ← Still work to do (more tasks remain)

  # STEP 1: Update detail-level status FIRST
  tasks:
    T6_client_interface:
      status: "Complete"                     # ✓ SOURCE OF TRUTH updated first
      completed_at: "2026-01-27T20:04:00Z"  # ✓ Completion timestamp
    T7_client_success_path:
      status: "In Progress"                  # ✓ Now in progress (matches current_task)
      started_at: "2026-01-27T20:05:00Z"    # ✓ Start timestamp

  # STEP 4: Update work arrays FOURTH (synchronized with detail status)
  work_completed:
    - task: "T5"
      completed_at: "2026-01-27T20:03:00Z"
      commit: "147524a"
    - task: "T6"                             # ✓ T6 added (source of truth says Complete)
      completed_at: "2026-01-27T20:04:00Z"
      commit: "8f4e2c1"

  work_in_progress:
    - task: "T7"                             # ✓ T7 now in progress (T6 removed)
      status: "Implementing CompaniesHouseClientImpl"

  work_remaining:
    - "T7: Client success path"              # ✓ T6 removed (it's complete)
    - "T8: Client error handling"
    - "T10: Integration tests"
    - "T11: Documentation"

  blockers: []

  # STEP 5: Update next_action FIFTH (references current pointers)
  next_action: "Implement T7: Create CompaniesHouseClientImpl with getRegisteredAddress method. Follow API from T6 interface and success path specs."
                                             # ✓ References T7, NOT T6

# INVARIANTS CHECK:
# ✓ 1. current_task="T7" matches tasks.T7.status="In Progress"
# ✓ 2. next_action mentions "T7"
# ✓ 3. status="In Progress" with incomplete tasks remaining
# ✓ 4. T6 has completed_at timestamp
# ✓ 5. T7 status="In Progress" matches current_task="T7"
# ✓ 6. Only T7 has status="In Progress"
# ✓ 7. work_completed contains T6 (and all completed tasks)
# ✓ 8. work_in_progress contains only T7 (not T6)
# ✓ 9. work_remaining does not contain T6
```

**Result:** All fields are synchronized. After compaction, Claude knows exactly:
- T7 is in progress (source of truth + pointer + arrays all agree)
- T6 is complete (in work_completed, not in work_in_progress or work_remaining)
- Next action is to work on T7 (next_action references T7)

#### Transition Example: Clean Phase Boundary

**Completing Phase 2 and Moving to Phase 3:**

```yaml
# Before transition (working on phase 2, step 2.3)
progress:
  last_updated: "2025-01-06T14:00:00Z"
  current_phase: "2"
  current_step: "2.3"
  status: "In Progress"

  phases:
    phase_1:
      status: "Complete"
      completed_at: "2025-01-06T13:00:00Z"
    phase_2:
      status: "In Progress"                   # ← Currently working here
      current_step: "2.3"
      steps_completed: ["2.1", "2.2"]
    phase_3:
      status: "Not Started"                   # ← Next phase

  work_completed:
    - item: "Phase 1 - Discovery"
      completed_at: "2025-01-06T13:00:00Z"
    - item: "Phase 2, Step 2.1 - Design"
      completed_at: "2025-01-06T13:30:00Z"
    - item: "Phase 2, Step 2.2 - Implementation"
      completed_at: "2025-01-06T13:50:00Z"

  work_in_progress:
    - item: "Phase 2, Step 2.3 - Testing"
      status: "Writing integration tests"

  work_remaining:
    - "Phase 2, Step 2.3 - Testing"
    - "Phase 3 - Deployment"

  next_action: "Complete step 2.3 - write integration tests for Payment service"

# After completing step 2.3 (LAST step of phase 2) - 6-STEP SEQUENCE:

# STEP 1: Update detail-level status FIRST
  phases:
    phase_2:
      status: "Complete"                      # ← Mark phase 2 complete
      completed_at: "2025-01-06T14:30:00Z"   # ← Add timestamp
      steps_completed: ["2.1", "2.2", "2.3"] # ← Add final step
    phase_3:
      status: "In Progress"                   # ← Start phase 3
      started_at: "2025-01-06T14:30:00Z"     # ← Add timestamp
      current_step: "3.1"                     # ← Detail-level pointer

# STEP 2: Update top-level pointers SECOND
  current_phase: "3"                          # ← Now points to phase 3
  current_step: "3.1"                         # ← First step of phase 3

# STEP 3: Update overall status THIRD
  status: "In Progress"                       # ← Still work to do

# STEP 4: Update work arrays FOURTH
  work_completed:
    - item: "Phase 1 - Discovery"
      completed_at: "2025-01-06T13:00:00Z"
    - item: "Phase 2, Step 2.1 - Design"
      completed_at: "2025-01-06T13:30:00Z"
    - item: "Phase 2, Step 2.2 - Implementation"
      completed_at: "2025-01-06T13:50:00Z"
    - item: "Phase 2, Step 2.3 - Testing"    # ← Add completed step
      completed_at: "2025-01-06T14:30:00Z"

  work_in_progress:
    - item: "Phase 3, Step 3.1 - Design"     # ← Phase 3 work (phase 2 removed)
      status: "Starting notification service architecture"

  work_remaining:
    - "Phase 3, Step 3.1 - Design"           # ← Phase 2 removed
    - "Phase 3, Step 3.2 - Implementation"
    - "Phase 3, Step 3.3 - Testing"

# STEP 5: Update next_action FIFTH
  next_action: "Begin phase 3, step 3.1 - design Notification service architecture. Read requirements from .work/phase3-requirements.yaml"
                                             # ← References phase 3, step 3.1

# STEP 6: Update timestamp ALWAYS
  last_updated: "2025-01-06T14:30:00Z"       # ← Matches transition time
```

**Result:** Clean transition with no conflicts. All fields are synchronized at phase boundary.

#### Self-Validation Checklist

Before writing progress.yaml, verify:

- [ ] **Step 1 Complete:** Detail-level status updated FIRST (tasks.*.status or phases.*.status)
- [ ] **Step 2 Complete:** Top-level pointers updated SECOND (current_task, current_phase, current_step)
- [ ] **Step 3 Complete:** Overall status updated THIRD (status field reflects all work)
- [ ] **Step 4 Complete:** Work arrays synchronized FOURTH (work_completed, work_in_progress, work_remaining)
- [ ] **Step 5 Complete:** next_action updated FIFTH (references current pointers and describes next work)
- [ ] **Step 6 Complete:** Timestamp updated ALWAYS (last_updated is current)
- [ ] **Invariants Check:** All 9 invariants hold (detail matches pointers, arrays synchronized, next_action accurate)
- [ ] **No Orphans:** No task appears in multiple conflicting states (both complete and in_progress)

#### Common Mistakes to Avoid

| Mistake | Why It's Wrong | Correct Approach |
|---------|----------------|------------------|
| Update current_task first, detail status later | Creates inconsistent window where pointer doesn't match source of truth | Update detail status FIRST, then pointers |
| Leave old task in work_in_progress when moving on | Multiple tasks appear active | Remove completed task from work_in_progress, add to work_completed |
| Forget to add completed task to work_completed | work_completed doesn't reflect actual completed work | Always add completed tasks to work_completed array with timestamp |
| next_action references previous task | Causes Claude to repeat completed work | Update next_action to reference current pointers (current_task, current_phase) |
| Forget to update timestamp | Can't tell when checkpoint was made | ALWAYS update last_updated |
| Copy-paste checkpoint without updating all fields | Creates conflicts like T6 marked complete but still in work_remaining | Follow 6-step sequence EVERY time - no shortcuts |
| Update only current_task, ignore detail status | Source of truth (tasks.T7.status) is stale | Update detail-level status FIRST, then derive pointers from it |
| Remove task from work_remaining but not add to work_completed | Task disappears from tracking entirely | Completed tasks move from work_remaining → work_completed (don't delete) |

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

2a. **UPDATE PROGRESS.YAML CONSISTENTLY (6-step sequence)**
   - Update detail-level status FIRST (tasks.T7.status, phases.phase_2.status)
   - Update top-level pointers SECOND (current_task, current_phase)
   - Update overall status THIRD (status field)
   - Update work arrays FOURTH (work_completed, work_in_progress, work_remaining)
   - Update next_action FIFTH (must reference current pointers)
   - Update last_updated ALWAYS
   - Verify all 9 invariants hold (see Progress.yaml Consistency Rules)

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

### Progress.yaml Update Sequence (6 Steps)
1. Detail-level status (tasks.T7.status, phases.phase_2.status) ← **SOURCE OF TRUTH**
2. Top-level pointers (current_task, current_phase, current_step)
3. Overall status (status field)
4. Work arrays (work_completed, work_in_progress, work_remaining) ← **MUST SYNC**
5. next_action (must reference step 2 pointers)
6. Timestamp (last_updated) ← **ALWAYS**

**Rule:** Detail → Pointers → Status → Arrays → next_action → Timestamp

**Arrays must be synchronized:**
- work_completed: Add completed task, remove from in_progress
- work_in_progress: Add current task, remove completed
- work_remaining: Remove completed task

**See:** Progress.yaml Consistency Rules section for full details and examples

### Memory Patterns
1. Summarise then discard
2. Reference not re-read
3. Incremental output building
4. Targeted chunk access

---

## Version History

| Version | Date       | Changes                                            |
| ------- | ---------- | -------------------------------------------------- |
| 1.2     | 2026-01-28 | Added Progress.yaml Consistency Rules; defined 6-step update sequence, source of truth, 9 invariants including work array synchronization |
| 1.1     | 2026-01-28 | Consolidated checkpoint trigger guidance into canonical list; resolved inconsistencies in progress.yaml update timing |
| 1.0     | 2026-01-06 | Initial rubric created from 01c, 01e, 01f patterns |

---

## Related Documents

- 