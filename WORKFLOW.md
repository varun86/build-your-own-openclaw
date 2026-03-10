# Implementation Workflow - Build Your Own OpenClaw

## Overview

This document defines the step-by-step process for implementing each tutorial step. Following this workflow ensures that tutorial code stays aligned with picklebot patterns and avoids unnecessary code drift.

### Core Principles

1. **Copy from picklebot first** - Use battle-tested code as the foundation
2. **Trim future code** - Remove pieces needed only in later steps
3. **Keep it working** - Every step must be runnable
4. **Shared workspace** - Config and workspace files in `default_workspace/` are shared across all steps

### Why "Copy from Picklebot" Over "Merge from Previous Step"?

When a file needs changes, you might be tempted to:
- ❌ Take previous step's code → add features from picklebot → complicated merge

This is hard because:
- Previous step may have drifted from picklebot patterns
- Hard to identify exactly what to add
- Risk of missing picklebot improvements

Instead, always:
- ✅ Take picklebot's code → trim to this step's needs → simpler and cleaner

This works because:
- Picklebot is the source of truth for patterns
- Easier to remove code than to add/merge
- Guarantees alignment with production patterns

---

## Step Implementation Workflow

### Phase 1: Planning (Before Writing Code)

1. **Identify Required Files**
   - Read PLAN.md step description
   - List all files to be created/modified
   - Map each file to picklebot source: `../pickle-bot/src/picklebot/...`

2. **Categorize Each File**
   - **Type A**: New file (doesn't exist in previous step)
   - **Type B**: Existing file, no new features needed
   - **Type C**: Existing file, new features needed

### Phase 2: Implementation (File by File)

**For Type A (New file):**
1. Copy from picklebot → trim to essentials
2. Remove production complexity, keep core logic

**For Type B (Existing, no changes):**
1. Copy from previous step (no changes needed)
2. Verify it still works with new dependencies

**For Type C (Existing, needs changes):**
1. Copy from picklebot (not from previous step)
2. Trim to include ONLY features up to this step
3. Done - no merge needed, picklebot is the source of truth

### Phase 3: Validation

1. Code runs: `uv run my-bot chat`
2. Test the new feature
3. Verify patterns match picklebot
4. Write/update README.md (follow concise format - see "README Format" section below)

---

## How to Trim Picklebot Code

### What to KEEP (Needed for THIS Step)
- ✅ Core logic required for current step's feature
- ✅ Essential dependencies and imports
- ✅ Error handling that affects current functionality
- ✅ Logging/retries/metrics if they're actually used

### What to REMOVE (Future Step Code)
- ❌ Code only needed in later tutorial steps
- ❌ Imports for features not yet implemented
- ❌ Configuration options for future features
- ❌ Helper functions for future capabilities
- ❌ "Forward-looking" abstractions

### Trimming Strategy

1. **Check PLAN.md** - What does THIS step need?
2. **Read picklebot file** - Identify which parts serve this step
3. **Remove future pieces** - Anything not needed until Step X
4. **Keep it working** - Don't break current functionality

### Example

**Step 01 (Tools) trimming:**
- ✅ Keep: ToolRegistry, basic tools (read/write/bash)
- ❌ Remove: SkillLoader (needed in Step 02)
- ❌ Remove: DispatchEvent handling (needed in Step 13)
- ❌ Remove: Concurrency control (needed in Step 16)

---

## Common Scenarios

### Scenario 1: "Picklebot file is very different from previous step"

**When:** Major refactoring happened in picklebot (e.g., Step 08 event-driven refactor)

**Solution:**
1. Copy from picklebot (it's the target architecture)
2. Trim to this step's features
3. Document the refactor in README.md

**Note:** Don't try to "evolve" from previous step - just use picklebot and trim.

### Scenario 2: "Not sure which picklebot file to use"

**When:** Multiple files in picklebot seem relevant

**Solution:**
1. Check PLAN.md - it usually specifies the exact file
2. Look for similar naming in picklebot structure
3. When in doubt, ask or pick the simpler one

### Scenario 3: "Picklebot doesn't have this file"

**When:** Tutorial needs a file that doesn't exist in picklebot

**Solution:**
1. Double-check PLAN.md - are we sure this file is needed?
2. Check if similar functionality exists under different name
3. Only then: write new code (document why in README.md)

### Scenario 4: "Previous step code looks wrong"

**When:** Previous implementation drifted from picklebot patterns

**Solution:**
1. Trust picklebot over previous step
2. Replace with trimmed picklebot version
3. Note the correction in commit message

---

## Quick Reference: Implementing a Step

### Before You Start
- [ ] Read PLAN.md step description
- [ ] List all files needed
- [ ] Map files to picklebot sources
- [ ] Note: `default_workspace/` config files are shared across all steps

### For Each File
- [ ] Determine type: A (new) / B (no changes) / C (needs changes)
- [ ] **Type A**: Copy from picklebot → trim future code
- [ ] **Type B**: Copy from previous step
- [ ] **Type C**: Copy from picklebot → trim to this step (no merge)

### Trimming Checklist
- [ ] Keep only what's needed for THIS step
- [ ] Remove code for future steps
- [ ] Remove unused imports
- [ ] Keep core logic intact

### Validation Checklist
- [ ] `uv run my-bot chat` works
- [ ] New feature works as described
- [ ] Matches picklebot patterns
- [ ] README.md updated (follow concise format from Step 00)
- [ ] Commit with clear message

### README Format

Follow the concise format from Step 00's README:

1. **Title + one-line description**
2. **Prerequisites** (only if needed)
3. **What We will Build?**
   - Simple architecture diagram
   - Key components (bullet list)
4. **Key Changes**
   - Code snippets with file links
   - Show only the most important parts
5. **Notes** (optional, only if needed)
6. **How to Run**
   - Command + example interaction
7. **What's Next** (link to next step)

**Keep it extremely concise.** No lengthy explanations, no deep dives, no alternatives discussion. The code speaks for itself.

### File Priority Order
1. ⭐ **picklebot code** (always preferred - trim to essentials)
2. ⭐ **previous step code** (only if no picklebot reference AND no changes needed)
3. ⭐ **new code** (only if no reference exists anywhere)

**Key insight:** For files that need changes, always start from picklebot and trim. Don't try to merge previous step with picklebot - picklebot is the source of truth.

---

## How This Connects to PLAN.md

**PLAN.md** tells you **WHAT** to build:
- Which features each step needs
- Which picklebot files to reference
- What the end result should look like

**WORKFLOW.md** tells you **HOW** to build it:
- The process for implementing each file
- How to trim picklebot code
- How to validate your work

### Usage Pattern

1. Read PLAN.md → Understand the step's requirements
2. Follow WORKFLOW.md → Implement file by file
3. Return to PLAN.md → Verify you met the requirements

### Updates to PLAN.md

PLAN.md will reference this workflow in its "Implementation Principles" section:

```markdown
## Implementation Principles

### **CRITICAL: Follow the Workflow**

See [WORKFLOW.md](./WORKFLOW.md) for the step-by-step implementation process.
This ensures consistency with picklebot patterns across all steps.
```
