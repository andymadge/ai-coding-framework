# Prompt Engineering Rubric for Claude Code

This rubric defines standards for generating well-structured prompts that survive context compaction, enable resumable execution, and maintain project state across sessions.

---

## Quick Reference

```
prompts/
├── manifest.yaml           # Execution order, parallelisation, dependencies
├── 001-requirements.md     # First prompt in sequence
├── 002-technical-design.md
├── 003-implementation.md
└── ...

.work/
└── progress.yaml           # Progress tracking, decisions, outputs
```

---

## 1. Prompt File Structure

Each prompt file must follow this XML-structured format:

```markdown
# [NNN] [Descriptive Title]

<prompt_metadata>
  <id>NNN</id>
  <name>descriptive-kebab-case-name</name>
  <category>requirements|planning|design|bdd|tdd|implementation|review|documentation</category>
  <estimated_effort>small|medium|large</estimated_effort>
</prompt_metadata>

<prerequisites>
  <prompt id="NNN">Description of what must be complete</prompt>
  <artifact path="relative/path">Description of required artifact</artifact>
  <state_check path=".work/progress.yaml" key="some.key" value="expected_value"/>
</prerequisites>

<context>
  <persistent>
    <!-- References to project-level context files -->
    <file path="CLAUDE.md" sections="conventions,architecture"/>
    <file path="docs/adr/001-tech-stack.md"/>
  </persistent>
  <session>
    <!-- Current task state -->
    <file path=".work/progress.yaml"/>
  </session>
  <inline>
    <!-- Minimal context needed for this specific prompt -->
    Any brief context that doesn't warrant a separate file.
  </inline>
</context>

<role>
  <identity>[Specific expert role, e.g., "Senior API Architect", "Security Engineer"]</identity>
  <expertise>[Key skills and knowledge areas relevant to this task]</expertise>
  <mindset>[How this role approaches problems, what they prioritise]</mindset>
</role>

<objective>
  Clear, single-sentence statement of what this prompt accomplishes.
</objective>

<instructions>
  <step id="1">
    <action>Specific action to take</action>
    <checkpoint>What to write to progress.yaml after this step</checkpoint>
  </step>
  <step id="2">
    <action>Next action</action>
    <output artifact="NNN-output-name.ext">Description of artifact to create</output>
    <checkpoint>State update</checkpoint>
  </step>
</instructions>

<outputs>
  <artifact path="NNN-output-name.ext" type="specification|code|test|documentation">
    Description of this output
  </artifact>
  <state_update>
    key.to.update: new_value
    another.key: value
  </state_update>
</outputs>

<verification>
  <criterion>How to verify this prompt completed successfully</criterion>
  <criterion>Another verification check</criterion>
</verification>

<on_failure>
  <condition trigger="description of failure scenario">
    <action>What to do</action>
    <escalation>When to stop and ask for human input</escalation>
  </condition>
</on_failure>

<next_prompts>
  <prompt id="NNN+1" condition="default">Next prompt in sequence</prompt>
  <prompt id="NNN+2" condition="if some condition">Alternative path</prompt>
</next_prompts>
```

---

## 2. Manifest File Structure

The `prompts/manifest.yaml` defines execution order and dependencies:

```yaml
version: "1.0"
project: project-name
description: Brief description of this prompt sequence

execution:
  - group: 1
    prompts:
      - id: "001"
        name: requirements-gathering
        file: 001-requirements.md
        
  - group: 2
    prompts:
      - id: "002"
        name: technical-design
        file: 002-technical-design.md
        depends_on: ["001"]
        
  - group: 3
    parallel: true  # These can run concurrently
    prompts:
      - id: "003"
        name: api-specification
        file: 003-api-specification.md
        depends_on: ["002"]
      - id: "004"
        name: database-schema
        file: 004-database-schema.md
        depends_on: ["002"]
        
  - group: 4
    prompts:
      - id: "005"
        name: implementation
        file: 005-implementation.md
        depends_on: ["003", "004"]

categories:
  requirements:
    prompts: ["001"]
  planning:
    prompts: ["002"]
  design:
    prompts: ["003", "004"]
  implementation:
    prompts: ["005"]

bdd_tdd_enabled: false  # Set true to enforce test-first workflow
```

---

## 3. State File Structure

The `.work/progress.yaml` tracks progress and decisions:

```yaml
meta:
  project: project-name
  manifest: prompts/manifest.yaml
  created_at: 2025-01-24T10:00:00Z
  last_updated: 2025-01-24T14:30:00Z
  last_prompt_completed: "002"

prompts:
  "001":
    status: completed  # pending|in_progress|completed|failed|skipped
    started_at: 2025-01-24T10:00:00Z
    completed_at: 2025-01-24T10:45:00Z
    artifacts_created:
      - path: 001-output-requirements.md
        verified: true
    decisions:
      - id: d001
        description: "Chose PostgreSQL over MySQL for JSON support"
        rationale: "Better JSONB performance for our use case"
        
  "002":
    status: completed
    started_at: 2025-01-24T11:00:00Z
    completed_at: 2025-01-24T12:30:00Z
    artifacts_created:
      - path: 002-output-technical-design.md
        verified: true
    decisions:
      - id: d002
        description: "REST over GraphQL"
        rationale: "Team familiarity, simpler caching"

  "003":
    status: in_progress
    started_at: 2025-01-24T14:00:00Z
    current_step: 2
    checkpoints:
      - step: 1
        completed_at: 2025-01-24T14:15:00Z
        note: "Identified 12 API endpoints"

errors:
  - prompt_id: "003"
    step: 2
    timestamp: 2025-01-24T14:30:00Z
    error: "Ambiguous requirement for authentication flow"
    resolution: pending  # pending|resolved|escalated

context_summary:
  tech_stack:
    language: Java 21
    framework: Spring Boot 3.x
    database: PostgreSQL 15
  key_decisions:
    - "d001: PostgreSQL for JSON support"
    - "d002: REST API design"
  current_focus: "API specification for user management endpoints"
```

---

## 4. Rubric Evaluation Criteria

Use this checklist to evaluate prompt quality:

### 4.1 Structure (Required)

| Criterion | Weight | Description |
|-----------|--------|-------------|
| Has valid metadata | ★★★ | Includes id, name, category, effort estimate |
| Role is specific | ★★★ | Targeted expert identity, not generic "assistant" |
| Prerequisites defined | ★★★ | Lists required prompts, artifacts, state checks |
| Clear objective | ★★★ | Single sentence describing outcome |
| Numbered steps | ★★★ | Each step has action and checkpoint |
| Outputs specified | ★★★ | Artifacts named with NNN prefix |
| Verification criteria | ★★☆ | How to confirm success |
| Failure handling | ★★☆ | What to do when things go wrong |

### 4.2 Role Targeting (Required)

| Criterion | Weight | Description |
|-----------|--------|-------------|
| Specific identity | ★★★ | Named expertise, not "helpful assistant" or "AI" |
| Relevant expertise | ★★★ | Skills directly applicable to the task |
| Clear mindset | ★★☆ | How this role thinks and prioritises |
| Appropriate seniority | ★★☆ | Junior/mid/senior matches task complexity |
| Domain alignment | ★★☆ | Role matches the prompt category |

### 4.3 Compaction Survival (Critical)

| Criterion | Weight | Description |
|-----------|--------|-------------|
| Checkpoints after decisions | ★★★ | Every significant decision writes to progress.yaml |
| Checkpoints after subtasks | ★★★ | Progress saved incrementally |
| No implicit state | ★★★ | All important context in files, not memory |
| Resumable from any step | ★★★ | Can continue after interruption |
| State file read first | ★★★ | Prompt begins by loading .work/progress.yaml |

### 4.4 Context Management (Required)

| Criterion | Weight | Description |
|-----------|--------|-------------|
| References persistent context | ★★★ | Points to CLAUDE.md, ADRs, etc. |
| Loads session state | ★★★ | Reads .work/progress.yaml |
| Minimal inline context | ★★☆ | Only essential info embedded |
| No duplicated context | ★★☆ | References files rather than copying |

### 4.5 Quality Attributes (Required)

| Criterion | Weight | Description |
|-----------|--------|-------------|
| Idempotent | ★★★ | Running twice doesn't break things |
| Traceable | ★★☆ | Outputs link to inputs and decisions |
| Self-contained | ★★☆ | Prompt + referenced files = complete |
| Appropriately scoped | ★★☆ | Not too large, not too granular |

### 4.6 BDD/TDD Integration (When Enabled)

| Criterion | Weight | Description |
|-----------|--------|-------------|
| Test prompt precedes implementation | ★★★ | Code prompts require test prompts complete |
| Acceptance criteria in BDD format | ★★☆ | Given/When/Then for features |
| Test artifacts verified | ★★☆ | Tests exist before implementation runs |

---

## 5. Prompt Categories

### 5.1 Requirements Gathering

```xml
<category_guidance name="requirements">
  <purpose>Capture and clarify what needs to be built</purpose>
  <suggested_roles>
    <role>Business Analyst with [domain] expertise</role>
    <role>Product Manager with technical background</role>
    <role>Requirements Engineer specialising in [domain]</role>
  </suggested_roles>
  <typical_outputs>
    <output>Requirements specification</output>
    <output>User stories or use cases</output>
    <output>Acceptance criteria</output>
    <output>Questions for stakeholders</output>
  </typical_outputs>
  <key_checkpoints>
    <checkpoint>Core requirements identified</checkpoint>
    <checkpoint>Ambiguities documented</checkpoint>
    <checkpoint>Scope boundaries defined</checkpoint>
  </key_checkpoints>
</category_guidance>
```

### 5.2 Technical Planning

```xml
<category_guidance name="planning">
  <purpose>Define technical approach and architecture</purpose>
  <suggested_roles>
    <role>Solution Architect with [tech stack] expertise</role>
    <role>Technical Lead experienced in [domain]</role>
    <role>Enterprise Architect for cross-cutting concerns</role>
  </suggested_roles>
  <typical_outputs>
    <output>Technical design document</output>
    <output>Architecture Decision Records (ADRs)</output>
    <output>Component diagrams</output>
    <output>Technology selections</output>
  </typical_outputs>
  <key_checkpoints>
    <checkpoint>Architecture pattern selected</checkpoint>
    <checkpoint>Key technical decisions recorded</checkpoint>
    <checkpoint>Integration points identified</checkpoint>
  </key_checkpoints>
</category_guidance>
```

### 5.3 BDD (When Applicable)

```xml
<category_guidance name="bdd">
  <purpose>Define behaviour specifications before implementation</purpose>
  <suggested_roles>
    <role>QA Engineer with BDD expertise</role>
    <role>Test Architect specialising in behaviour-driven design</role>
    <role>Business Analyst bridging requirements to scenarios</role>
  </suggested_roles>
  <typical_outputs>
    <output>Feature files (Gherkin)</output>
    <output>Scenario outlines</output>
    <output>Step definitions skeleton</output>
  </typical_outputs>
  <key_checkpoints>
    <checkpoint>Happy path scenarios defined</checkpoint>
    <checkpoint>Edge cases identified</checkpoint>
    <checkpoint>Acceptance criteria mapped to scenarios</checkpoint>
  </key_checkpoints>
  <enforcement>
    When bdd_tdd_enabled: true in manifest, implementation prompts 
    must have corresponding BDD prompts completed first.
  </enforcement>
</category_guidance>
```

### 5.4 TDD (When Applicable)

```xml
<category_guidance name="tdd">
  <purpose>Write tests before implementation code</purpose>
  <suggested_roles>
    <role>Test Engineer with [language/framework] expertise</role>
    <role>Senior Developer practising test-first development</role>
    <role>Quality Engineer focusing on testability</role>
  </suggested_roles>
  <typical_outputs>
    <output>Unit test files</output>
    <output>Integration test files</output>
    <output>Test fixtures and mocks</output>
  </typical_outputs>
  <key_checkpoints>
    <checkpoint>Test cases for requirements identified</checkpoint>
    <checkpoint>Tests written and failing</checkpoint>
    <checkpoint>Test infrastructure in place</checkpoint>
  </key_checkpoints>
  <enforcement>
    When bdd_tdd_enabled: true in manifest, implementation prompts 
    must have corresponding TDD prompts completed first.
  </enforcement>
</category_guidance>
```

### 5.5 Implementation

```xml
<category_guidance name="implementation">
  <purpose>Write production code</purpose>
  <suggested_roles>
    <role>Senior [Language] Developer with [framework] experience</role>
    <role>Backend Engineer specialising in [domain]</role>
    <role>Full-Stack Developer for end-to-end features</role>
  </suggested_roles>
  <typical_outputs>
    <output>Source code files</output>
    <output>Configuration files</output>
    <output>Database migrations</output>
  </typical_outputs>
  <key_checkpoints>
    <checkpoint>Core logic implemented</checkpoint>
    <checkpoint>Error handling added</checkpoint>
    <checkpoint>Code compiles/runs</checkpoint>
    <checkpoint>Tests pass (if TDD enabled)</checkpoint>
  </key_checkpoints>
</category_guidance>
```

### 5.6 Code Review

```xml
<category_guidance name="review">
  <purpose>Evaluate code quality and correctness</purpose>
  <suggested_roles>
    <role>Senior Developer with code review expertise</role>
    <role>Tech Lead enforcing team standards</role>
    <role>Security Engineer for security-focused review</role>
    <role>Performance Engineer for optimisation review</role>
  </suggested_roles>
  <typical_outputs>
    <output>Review comments/findings</output>
    <output>Refactoring suggestions</output>
    <output>Approval or rejection decision</output>
  </typical_outputs>
  <key_checkpoints>
    <checkpoint>Code reviewed against standards</checkpoint>
    <checkpoint>Issues categorised by severity</checkpoint>
    <checkpoint>Recommendations documented</checkpoint>
  </key_checkpoints>
</category_guidance>
```

### 5.7 Documentation

```xml
<category_guidance name="documentation">
  <purpose>Create project documentation</purpose>
  <suggested_roles>
    <role>Technical Writer with [domain] knowledge</role>
    <role>Developer Advocate for API documentation</role>
    <role>Senior Developer documenting architecture</role>
  </suggested_roles>
  <typical_outputs>
    <output>README files</output>
    <output>API documentation</output>
    <output>Architecture documentation</output>
    <output>Runbooks</output>
  </typical_outputs>
  <key_checkpoints>
    <checkpoint>Target audience identified</checkpoint>
    <checkpoint>Key sections drafted</checkpoint>
    <checkpoint>Examples included</checkpoint>
  </key_checkpoints>
</category_guidance>
```

---

## 6. Execution Workflow

### 6.1 Starting a New Sequence

```xml
<workflow name="initialisation">
  <step>Read prompts/manifest.yaml</step>
  <step>Check if .work/progress.yaml exists</step>
  <step>If not, create initial progress.yaml with meta section</step>
  <step>Identify first prompt in execution order</step>
  <step>Begin prompt execution</step>
</workflow>
```

### 6.2 Resuming After Interruption

```xml
<workflow name="resume">
  <step>Read .work/progress.yaml</step>
  <step>Find last completed prompt from meta.last_prompt_completed</step>
  <step>Check for any in_progress prompts</step>
  <step>If in_progress exists, resume from current_step</step>
  <step>Otherwise, proceed to next prompt in manifest</step>
</workflow>
```

### 6.3 After Context Compaction

```xml
<workflow name="compaction_recovery">
  <step>IMMEDIATELY read .work/progress.yaml</step>
  <step>Read context_summary section for quick orientation</step>
  <step>Load referenced persistent context files</step>
  <step>Resume from last checkpoint</step>
  <step>Do NOT rely on any information not in files</step>
</workflow>
```

---

## 7. Checkpoint Guidelines

### When to Write Checkpoints

Write to `.work/progress.yaml` after:

1. **Significant decisions** - Technology choices, architecture patterns, approach selections
2. **Completed substeps** - Each numbered step in a prompt
3. **Artifact creation** - When an output file is created
4. **Error encounters** - Log failures for debugging
5. **Context changes** - New information that affects future prompts

### Checkpoint Format

```yaml
# In prompts.NNN.checkpoints
- step: 2
  completed_at: 2025-01-24T14:15:00Z
  note: "Brief description of what was accomplished"
  artifacts:
    - path/to/created/file.ext
  decisions:
    - id: dNNN
      description: "What was decided"
      rationale: "Why"
```

---

## 8. Failure Handling Patterns

### 8.1 Ambiguous Requirements

```xml
<on_failure>
  <condition trigger="requirement is unclear or contradictory">
    <action>Document the ambiguity in progress.yaml errors section</action>
    <action>List possible interpretations</action>
    <action>Mark prompt as blocked, not failed</action>
    <escalation>Request human clarification before proceeding</escalation>
  </condition>
</on_failure>
```

### 8.2 Missing Prerequisites

```xml
<on_failure>
  <condition trigger="required artifact or state not found">
    <action>Check if prerequisite prompt was skipped</action>
    <action>Log missing dependency in progress.yaml</action>
    <escalation>Cannot proceed - prerequisite must complete first</escalation>
  </condition>
</on_failure>
```

### 8.3 Technical Errors

```xml
<on_failure>
  <condition trigger="code doesn't compile, tests fail, etc">
    <action>Log error details in progress.yaml</action>
    <action>Attempt automated fix if pattern is recognised</action>
    <action>After 2 failed attempts, pause and document</action>
    <escalation>Request human review of error</escalation>
  </condition>
</on_failure>
```

---

## 9. .work Folder Placement Guidance

Place `.work/` folders at the appropriate scope:

| Scenario | Location | Example |
|----------|----------|---------|
| Single feature | Feature directory | `features/user-auth/.work/` |
| Module-level work | Module root | `backend/.work/` |
| Cross-cutting work | Project root | `.work/` |
| Microservice | Service directory | `services/payment/.work/` |

**Guidance:**
- Keep `.work/` close to the code it relates to
- Avoid deeply nested `.work/` folders
- One active `.work/` per logical unit of work
- Add `.work/` to `.gitignore` if state shouldn't be committed

---

## 10. Example Prompt

```markdown
# 003 API Specification

<prompt_metadata>
  <id>003</id>
  <name>api-specification</name>
  <category>design</category>
  <estimated_effort>medium</estimated_effort>
</prompt_metadata>

<prerequisites>
  <prompt id="002">Technical design must be complete</prompt>
  <artifact path="002-output-technical-design.md">Technical design document</artifact>
  <state_check path=".work/progress.yaml" key="prompts.002.status" value="completed"/>
</prerequisites>

<role>
  <identity>Senior API Architect</identity>
  <expertise>REST API design, OpenAPI specification, HTTP semantics, 
    authentication patterns, and API versioning strategies</expertise>
  <mindset>Designs APIs from the consumer's perspective. Prioritises 
    consistency, discoverability, and backwards compatibility. 
    Questions unclear requirements before assuming intent.</mindset>
</role>

<context>
  <persistent>
    <file path="CLAUDE.md" sections="api-conventions"/>
    <file path="docs/adr/002-rest-api-design.md"/>
  </persistent>
  <session>
    <file path=".work/progress.yaml"/>
  </session>
  <inline>
    This API serves a mobile application with offline-first requirements.
  </inline>
</context>

<objective>
  Define the REST API specification for all user management endpoints.
</objective>

<instructions>
  <step id="1">
    <action>Read .work/progress.yaml and 002-output-technical-design.md</action>
    <checkpoint>Confirmed prerequisites met</checkpoint>
  </step>
  <step id="2">
    <action>Identify all user management operations from requirements</action>
    <checkpoint>operations_identified: [list of operations]</checkpoint>
  </step>
  <step id="3">
    <action>Define endpoint paths, methods, request/response schemas</action>
    <output artifact="003-output-api-spec.yaml">OpenAPI specification</output>
    <checkpoint>endpoints_defined: true</checkpoint>
  </step>
  <step id="4">
    <action>Document error responses and edge cases</action>
    <checkpoint>error_handling_defined: true</checkpoint>
  </step>
  <step id="5">
    <action>Verify specification against requirements</action>
    <checkpoint>verified: true</checkpoint>
  </step>
</instructions>

<outputs>
  <artifact path="003-output-api-spec.yaml" type="specification">
    OpenAPI 3.0 specification for user management API
  </artifact>
  <state_update>
    prompts.003.status: completed
    prompts.003.artifacts_created: ["003-output-api-spec.yaml"]
    meta.last_prompt_completed: "003"
  </state_update>
</outputs>

<verification>
  <criterion>All user operations from requirements have corresponding endpoints</criterion>
  <criterion>Request/response schemas are fully defined</criterion>
  <criterion>Error responses cover authentication, validation, not-found cases</criterion>
</verification>

<on_failure>
  <condition trigger="requirement ambiguity about user data model">
    <action>Document ambiguity in progress.yaml</action>
    <action>Propose reasonable default interpretation</action>
    <escalation>If multiple valid interpretations, ask human to choose</escalation>
  </condition>
</on_failure>

<next_prompts>
  <prompt id="005" condition="default">Implementation</prompt>
</next_prompts>
```

---

## 11. Self-Assessment Checklist

Before finalising a generated prompt, verify:

- [ ] Metadata complete (id, name, category, effort)
- [ ] Role is specific and targeted (not generic "assistant")
- [ ] Role expertise matches the task requirements
- [ ] Role mindset describes how they approach problems
- [ ] Prerequisites explicitly listed
- [ ] Context references files, not embedded content
- [ ] Objective is one clear sentence
- [ ] Each step has an action and checkpoint
- [ ] Outputs use NNN-prefix naming
- [ ] Verification criteria are testable
- [ ] Failure handling covers likely issues
- [ ] State updates specified
- [ ] Next prompts identified

---

## 12. Integration with CLAUDE.md

Add this section to your project's CLAUDE.md:

```markdown
## Prompt Execution

This project uses structured prompts in `prompts/`.

### Starting Work
1. Read `prompts/manifest.yaml` for execution order
2. Check `.work/progress.yaml` for current progress
3. Execute prompts in order, respecting dependencies

### After Compaction
1. FIRST read `.work/progress.yaml`
2. Load `context_summary` section for orientation
3. Resume from last checkpoint

### Checkpoint Discipline
- Write to progress.yaml after every significant decision
- Write to progress.yaml after completing each step
- Never rely on context memory for important information
```
