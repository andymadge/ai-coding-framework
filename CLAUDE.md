# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is an AI Coding Framework—a reusable template and methodology for orchestrating complex, multi-step AI-assisted development workflows using Claude.

**Design Principle:** By default, all rubrics should be self-contained with examples embedded inside. Do not create separate example or template files unless there is a compelling reason and explicit user approval.

## Core Architecture

**State Persistence**: All important information is persisted to `.work/progress.yaml` and artifacts are saved to disk. This enables recovery from context compaction without relying on conversation history.

**Manifest-Driven Execution**: Prompts are orchestrated via `manifest.yaml` which defines execution order, dependencies, and conditional branching.

**Checkpoint Discipline**: Every execution step maintains checkpoints. Interrupted sessions resume by reading `.work/progress.yaml` rather than conversation history.

**Role-Based Prompts**: Each prompt defines a specific expert role (architect, developer, QA strategist, etc.) to improve output quality beyond generic assistance.

## Working with This Framework

### Key Files

- `prompts/rubrics/STD-001-prompt-engineering-rubric.md` — Comprehensive evaluation standards, worked examples, and self-assessment checklist

### When Using This Framework in a Project

1. Review `STD-001-prompt-engineering-rubric.md` to understand the framework
2. Copy only the rubrics you need to the new project's `prompts/rubrics/` directory

Claude Code will handle the rest: creating manifest and progress files, generating prompts, managing execution, and checkpoint recovery.

### Creating New Prompts

Copy the worked example from section 10 in `STD-001-prompt-engineering-rubric.md` and customize it for your task. Evaluate using the self-assessment checklist in section 11 (24 verification items covering structure, role targeting, compaction survival, context management, and quality attributes).
