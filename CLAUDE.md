# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is an AI Coding Framework—a reusable template and methodology for orchestrating complex, multi-step AI-assisted development workflows using Claude.

**Design Principle:** By default, all rubrics should be self-contained with examples embedded inside. Do not create separate example or template files unless there is a compelling reason and explicit user approval.

## Core Architecture

**State Persistence**: All important information is persisted to `.work/progress.yaml` using single source of truth format. Task status is stored once in the `tasks` map; work arrays are derived on read, not stored. This eliminates synchronization conflicts and enables reliable recovery from context compaction without relying on conversation history. **The `.work/` directory must be committed to git** to enable team collaboration and machine switching.

**Manifest-Driven Execution**: Prompts are orchestrated via `manifest.yaml` which defines execution order, dependencies, and conditional branching.

**Checkpoint Discipline**: Every execution step maintains checkpoints. Interrupted sessions resume by reading `.work/progress.yaml` rather than conversation history.

**Role-Based Prompts**: Each prompt defines a specific expert role (architect, developer, QA strategist, etc.) to improve output quality beyond generic assistance.

## Working with This Framework

### Key Files

- `prompts/rubrics/STD-001-prompt-creation-rubric.md` — Comprehensive evaluation standards for Claude prompts, worked examples, and self-assessment checklist
- `prompts/rubrics/STD-002-python-development-rubric.md` — Python development standards covering type hints, Pythonic patterns, testing, minimal documentation, security, and performance

### When Using This Framework in a Project

1. **For prompt design**: Review `STD-001-prompt-creation-rubric.md` to understand context compaction survival, large file handling, and prompt structure
2. **For Python projects**: Review `STD-002-python-development-rubric.md` for coding standards (type hints, testing, minimal documentation, security)
3. Copy only the rubrics you need to the new project's `prompts/rubrics/` directory

Claude Code will handle the rest: creating manifest and progress files, generating prompts, managing execution, and checkpoint recovery.

### Rubric Selection Guide

- **All projects**: STD-001 for prompt engineering across any domain
- **Python projects**: STD-002 for development standards covering type hints, error handling, testing, documentation, security, and performance
- **Future**: Additional rubrics (STD-003, etc.) for other languages/domains as needed

### Creating New Prompts

Copy the worked example from section 11 in `STD-001-prompt-creation-rubric.md` and customize it for your task. Evaluate using the self-assessment checklist in section 12 covering structure, role targeting, compaction survival, context management, and quality attributes.

### Python Development Guidelines

For Python code contributions or reviews, consult `STD-002-python-development-rubric.md`:
- **Type hints**: Required on all function signatures
- **Documentation**: Minimal—only document non-obvious intent
- **Testing**: Standard pytest patterns with Arrange-Act-Assert structure
- **Code style**: Pythonic patterns, clear naming, single responsibility
- **Security**: Input validation at boundaries, parameterized queries, no hardcoded secrets
- **Self-assessment**: Use the 43-item checklist to evaluate code quality
