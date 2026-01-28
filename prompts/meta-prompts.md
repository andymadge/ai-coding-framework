These are prompts for generating or updating other prompts or rubrics from scratch. They can be useful when switching or assessing different AI tools/agents/LLMs.

## Prompt to Generate 'Prompt Creation Rubric'
```
Create a comprehensive prompt engineering rubric for AI coding assistants working on long-running tasks that survive context loss/compaction.

**IMPORTANT:** Follow Anthropic's XML prompting guide at https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/use-xml-tags
All prompt templates must use XML tags for structure (e.g., <context>, <methodology>, <critical_reminders>, <begin>).

**Core Principles:**
- Disk over memory: persist all state to `.work/progress.yaml` (must be in git)
- Checkpoint triggers: after completions, every 5-10 min, before risky ops, after findings, before transitions
- Large file handling: chunk files >100KB into 300-500 line segments, summarize to disk
- Single source of truth: store task status once in tasks map, derive work arrays on read
- No arbitrary limits: never use `head -N` on file discovery

**Progress.yaml Format (YAML template):**

    progress:
    last_updated: "[ISO DateTime]"
    current:
        phase: "[Phase ID]"
        task: "[Task ID]"
    next_action: "[Exactly what to do next]"
    tasks:
        [task_id]:
        status: not_started|in_progress|complete|skipped
        started_at: "[ISO DateTime]"
        completed_at: "[ISO DateTime]"
        commit: "[Git SHA]"
    blockers: []
    # NEVER store work_completed, work_in_progress, work_remaining arrays - derive on read


**Update Pattern:** 2 atomic operations: (1) Update task status in tasks map, (2) Update pointer + next_action + timestamp

**Include these sections:**
1. When to use these patterns (multi-phase tasks, large files, >10-15 min work)
2. Prompt structure template using XML tags (<context>, <foundational_principles>, <context_compaction_survival>, <large_file_handling>, <methodology>, <output_specifications>, <critical_reminders>, <begin>)
3. Context compaction survival pattern with XML template
4. Large file handling pattern with chunked reading strategy
5. Avoiding arbitrary limits pattern (the `head -N` problem)
6. Progress tracking with validation rules
7. Checkpoint strategies
8. Begin section template (check for existing progress first)
9. Critical reminders template
10. Customization guide
11. Quick reference card

**Directory structure:**

    .work/              # MUST be in git - enables team collaboration
    ├── progress.yaml
    ├── source-discovery.yaml
    ├── source-summaries/
    └── large-file-summaries/


Make it practical with copy-paste templates, real examples, and a self-assessment checklist.

```

To use the generated prompt creation rubric, use prompt similar to this:

```
<Summarise the project we are building>. I want to build a set of prompts which will take me through requirements, architecture, planning, implementation and testing using TDD. I want you to follow the XML prompting guide from Anthropic at https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/use-xml-tags and ensure our prompt creation rubric is followed @prompts/rubrics/STD-001-prompt-creation-rubric.md
<provide some more requirements if known, or just allow the AI to ask questions to clarify>.
```
