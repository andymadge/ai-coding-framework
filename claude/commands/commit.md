Review all uncommitted changes and create logical, atomic commits following these rules:

   1. **Atomic Commits**: Group related changes into separate commits, where each commit represents one logical unit of work that could be reverted independently

   2. **Partial File Commits**: When a single file contains multiple unrelated changes, use `git add -p` to stage and commit related hunks separately

   3. **Conventional Commits**: Format all commit messages using the Conventional Commits specification:
      - feat: new feature
      - fix: bug fix
      - docs: documentation changes
      - style: formatting, missing semicolons, etc.
      - refactor: code restructuring without changing behavior
      - test: adding or updating tests
      - chore: maintenance tasks, dependencies, tooling
      - perf: performance improvements
      - ci: CI/CD changes
      - build: build system or external dependency changes

   4. **Commit Message Format**:
```
      <type>[optional scope]: <description>
      
      [optional body]
      
      [optional footer(s)]
```
      
      - Description: imperative mood ("add" not "added"), lowercase, no period, max 50 chars
      - Body: explain what and why (not how), wrap at 72 chars, separated by blank line
      - Footer: breaking changes, issue references

   5. **Quality Guidelines**:
      - Each commit should pass tests if run in isolation
      - Commits should tell a story of the work progression
      - Separate refactoring from feature/bug fix commits
      - Keep formatting/whitespace changes in separate commits
      - File renames and modifications must be in separate commits - first commit the rename with no changes, then commit modifications
      - Reference issue/ticket numbers in footer when applicable

   Before committing, show me the proposed commit structure for approval.