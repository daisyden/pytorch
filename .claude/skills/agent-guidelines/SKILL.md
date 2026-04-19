---
name: agent-guidelines
description: Meta skill that defines agent behavior rules for PyTorch development. This skill MUST be loaded at the start of any coding task. Use when beginning new tasks, making code changes, creating commits, or performing any analysis.
---

# Agent Behavior Guidelines

This skill defines mandatory behaviors for all agent interactions. Every agent must follow these rules consistently.

**IMPORTANT: This skill applies to ALL other skills in the skills directory.** When any skill is loaded or executed:
- All rules in this skill MUST be followed
- Skills can add constraints but cannot override these core rules
- Before completing any task, verify compliance with all rules here

**Skills created after this rule MUST include reference to this skill:**
```markdown
This skill follows agent-guidelines AND extends it with [specific constraints].
Always apply agent-guidelines rules including the mandatory post-write commit protocol.
```

## Session Configuration

```yaml
force_multi_turn: true
allow_reflection: true
replay_like_chat: true
keep_thought_process: true
```

### allow_tools
The agent has access to all available tools:
- Read, Write, Edit file operations
- Bash command execution
- Grep, Glob for search
- WebFetch for URL content
- Question for user input
- Task for launching subagents
- Skill for loading skills
- todowrite for task tracking

## Code Change Rules

### Minimal, Targeted Changes
- Only make minimal, targeted changes
- Never introduce unnecessary refactoring
- Never add features not requested
- Avoid "drive-by" fixes or style changes
- Changes should be surgical and precise

### Git Commit Rules

After EVERY code change, make a small, atomic git commit immediately:

1. **One change = one commit**. Do not bundle multiple edits.
2. **Use clear, short English commit messages**.
3. **DO NOT** amend, rebase, reset, or force push.
4. **DO NOT** touch `.git`, `.gitignore`, or git config.
5. **Show a brief diff before commit**.
6. **MANDATORY: After any Write/Edit tool call that creates/modifies tracked files**
   - IMMEDIATELY check git status
   - Run `git add <file>` on modified files
   - Run `git commit -m "description"`
   - DO NOT proceed to next task until commit is done

### Mandatory Post-Write Protocol

**CRITICAL**: After using Write or Edit tools, you MUST execute this sequence:

```bash
# Step 1: Check what was modified
git status -s

# Step 2: If tracked files exist, ASK USER before committing
FILES=$(git status -s | grep "^[ M]" | awk '{print $2}')
if [ -n "$FILES" ]; then
    echo "Files modified that need commit: $FILES"
    # PROMPT USER: Use Question tool to ask if they want to commit
fi
```

**MANDATORY USER PROMPT**: When files are modified, you MUST ask:

> "Should I commit these changes: [list modified files]?"

**Wait for user confirmation** before running git add and git commit.

Exception: If user explicitly requests "commit all changes" or similar groupings, you MAY commit without prompt but still show diff and ask about description.

### I Will Provide Examples After Each Unique Edit

When committing complex changes, always provide context:

```bash
# Make small change to file
git diff file.py

# If change looks correct, commit immediately
git add file.py
git commit -m "Fix: short descriptive message of what changed"

# For multi-file changes in same work session
git add file1.py file2.py
git commit -m "Implement: feature X with Y

Changes:
- Modified file1.py to add new function
- Updated file2.py for compatibility

Fixes: #1234"
```

**What SHOULD trigger IMMEDIATE commit:**
- Write tool creates new file
- Edit tool modifies existing file
- Multiple edits forming single logical change

**What CAN be batched (only if same session):**
- Multiple closely related edits (all part of one feature/bug fix)
- Group commits ONLY for PR submission when requested otherwise

**Enforcement Checklist:**
- [ ] Any Write call → commit follows immediately
- [ ] Any Edit call → commit follows immediately  
- [ ] Verify with `git log -1 --oneline` after commit
- [ ] Check `git status` is clean before next task

### Commit Message Style
- Use imperative mood ("Add feature" not "Added feature")
- Keep first line under 72 characters
- Add detailed description in body if needed
- Reference issues/PRs when applicable

## Extraction & Summary Rules

### Prohibited Patterns
- DO NOT use pattern match, regex, keyword match, or string match
- DO NOT rely on fixed formats or positions
- DO NOT assume output structures

### LLM Semantic Understanding
- All analysis MUST use LLM semantic understanding only
- Understand context, not just text
- Infer intent from code structure
- Extract meaning, not just data

### Structured Output
- All results MUST have evidence
- Cite line numbers and file paths
- Quote relevant code snippets
- Provide reasoning chain

Example (correct):
```
Analysis: Function `foo` at file.py:42 appears to handle error cases
by checking null pointers before dereferencing.

Evidence:
- file.py:43: `if (ptr == nullptr)` - guards against null dereference
- file.py:45: `return ptr->value()` - actual usage after check

This follows the pattern of defensive programming where guards precede usage.
```

## Step-by-Step Rule

### No Skipping Steps
- DO NOT skip steps
- DO NOT take shortcuts
- Maintain complete thinking process
- Verify each step before proceeding

### Think Aloud
- Document thought process
- Show reasoning chain
- Explain why decisions made
- Keep audit trail of analysis

### Continuous Chat Behavior
- Act as if in a continuous chat, not cold start
- Reference earlier context
- Maintain conversation flow
- Remember previous decisions

## Analysis Guidelines

### Semantic Analysis Only
When analyzing code:
1. Read and understand the code semantically
2. Identify what it actually does, not just what it matches
3. Use context to determine intent
4. Check actual behavior, not potential text patterns

### Evidence-Based Conclusions
All conclusions MUST cite:
- Exact file path and line numbers
- Specific code snippets that support the conclusion
- Logical reasoning connecting evidence to conclusion

### Avoid Assumptions
- Don't assume patterns based on naming
- Don't assume behavior from documentation alone
- Always verify with actual code inspection
- Check both positive and negative cases

## Tool Usage Guidelines

### Before Using Tools
- Have a clear purpose for each tool call
- Know what you're looking for
- Have expected outcome in mind

### After Tool Results
- Always interpret results semantically
- Connect new information to existing context
- Update understanding based on findings
- Note incomplete or unexpected results

### Tool Patterns to Avoid
- DON'T collect unnecessary information
- DON'T run multiple searches without purpose
- DON'T read large files without targeted reason
- DO use smallest effective tool scope

## Error Handling

### When Unexpected Results
- Pause and analyze what happened
- Don't assume and continue
- Check your understanding is correct
- Ask clarifying questions if needed

### When Uncertain
- Admit uncertainty
- Don't guess or hallucinate
- Search for more evidence
- Ask user for clarification

## Interaction Guidelines

### Ask Questions Appropriately
- Ask when requirements unclear
- Ask for confirmation on major decisions
- Offer options with recommendations
- Never assume user intent

### Provide Clear Status
- State what's been done
- State what will be done
- State any blockers
- Ask for guidance when needed

### Keep User Informed
- Brief updates on significant actions
- Not constant status updates
- Just enough to maintain context
- Alert to issues immediately