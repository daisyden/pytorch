---
name: fix-issues-identified-by-comments
description: Fix code issues identified by PR review comments (e.g., Copilot AI reviews). Use when fixing XPU compatibility, CUDA-to-XPU migration, or addressing specific review feedback. Performs deep semantic analysis of comment issues before applying fixes.
---

# Fix Issues Identified by Comments

## Description

Fix issues identified by PR review comments such as Copilot AI code reviews. This skill performs deep semantic analysis of comment feedback, identifies the root cause of issues, and applies targeted fixes while following PyTorch conventions.

Use when:
- Addressing Copilot AI review comments on PRs
- Fixing XPU compatibility issues found in code review
- Migrating CUDA-specific code to support Intel XPU
- Applying fixes based on specific PR feedback

## Skill Integration

**This skill follows agent-guidelines AND extends it with specific constraints.**

Always apply agent-guidelines rules including:
- Mandatory post-write commit protocol (ask user before committing)
- Deep semantic analysis instead of pattern matching
- Atomic commits for each fix
- All constraints defined in agent-guidelines

## Preconditions

Before starting, ensure:

1. **Working directory**: PyTorch repo root `/home/daisydeng/daisy_pytorch`
2. **Git remotes configured**: Both `intel` and `daisyden` remotes for torch-xpu-ops
3. **GitHub authentication**: Password-less tokens configured for push operations
4. **Environment**: Miniforge environment available at `~/miniforge3/bin/activate`
5. **PR context**: URL or PR number provided to understand context of comments

## Used Tools

The following tools are available and MUST be used appropriately:

### Read Tool
- **Purpose**: Load files at specific line numbers for context
- **Usage**: Always read 15-30 lines around each issue location
- **Constraint**: Use targeted reads, not full file scans

### Edit Tool
- **Purpose**: Apply targeted code fixes
- **Usage**: Make surgical changes, preserve surrounding code
- **Constraint**: Include same-line `# noqa:` comments when appropriate

### Glob/Grep Tools
- **Purpose**: Find similar patterns in codebase for consistency
- **Usage**: Use AFTER analysis to FIND similar code, NOT for analysis itself
- **Constraint**: DO NOT use for analysis - only for pattern discovery

### Bash Tool
- **Purpose**: Run git commands, checkouts, commits, and pushes
- **Functions**:
  - `git status -s`: Check modified files
  - `git diff`: Show changes before commit
  - `git add/commit`: Apply atomic commits
  - `git push`: Push to correct branch

### Task Tool (explore subagent)
- **Purpose**: Deep semantic analysis of complex code patterns
- **Usage**: When issue requires understanding cross-file dependencies
- **Constraint**: Used for ANALYSIS only, not for code generation

### todowrite Tool
- **Purpose**: Track multi-step fix progress
- **Usage**: List all issues from comment, track completion

### question Tool
- **Purpose**: Ask user approval before commits
- **Usage**: MANDATORY before each commit per agent-guidelines
- **Constraint**: Must ask, wait for response, then act

## Working Workflow

### Step 1: Setup Environment

```bash
# Activate correct environment
source ~/miniforge3/bin/activate ~/miniforge3/envs/pytorch_opencode_env

# Navigate to torch-xpu-ops
cd third_party/torch-xpu-ops
```

### Step 2: Identify Target Branch

```bash
# For PR-based work, get the correct branch
PR_NUM=3383
# Fetch and checkout PR branch
git fetch owner branch:owner/branch
git checkout owner/branch
```

### Step 3: Analyze Comment Issues

For each issue from the comment:

1. **Extract issue details**:
   - File path and line numbers from comment
   - Description of what needs to be fixed
   - Suggested fix (if provided)

2. **Read context** (MANDATORY):
   ```bash
   # Read 15-30 lines around each issue location
   read file.py offset=N-10 limit=40
   ```

3. **Understand semantic purpose**:
   - What is the code trying to do?
   - Why was it written this way?
   - What would break if changed incorrectly?

4. **Check similar patterns**:
   ```bash
   # Find similar code for consistency reference
   grep -n "pattern_type" related/files/*.py
   ```

### Step 4: Apply Deep Analysis

DO NOT use pattern matching or regex substitution for analysis. Instead:

#### Analysis Protocol

```
FOR EACH ISSUE:

1. INTENT RECOGNITION
   - Read surrounding code (15-30 lines)
   - Understand what function/module does
   - Identify requirements (triton, CUDA, XPU, etc.)

2. ERROR/ISSUE CLASSIFICATION
   - Is this a XPU vs CUDA compatibility issue?
   - Is this a hard-coded backend assumption?
   - Is this a missing import/decorator?
   - Is this an expected test that needs modification?

3. SOLUTION VALIDATION
   - Design minimal fix that maintains behavior
   - Check if similar fixes exist in codebase
   - Verify fix works for both CUDA and XPU if applicable

4. APPLY TARGETED FIX
   - Use Edit tool with exact string match
   - Include noqa comments on same line when appropriate
```

### Step 5: Validate Fix

After applying each fix:

```bash
# Check consistency with similar patterns
grep -n "new_pattern" *.py | head -10

# Verify no regressions in related tests
git diff path/to/fixed.file

# Check for new lint issues
lintrunner -a path/to/fixed.file
```

### Step 6: Track Progress

```bash
# Track each fix
todowrite --add "Fix issue #1: XPU device context" --status pending
todowrite --add "Fix issue #2: CUDA Event/Stream" --status completed
```

### Step 7: User Approval Before Each Commit (MANDATORY)

Following agent-guidelines, you MUST ask for approval:

```
**Issue fixed**: [brief description]
**File modified**: [file path]
**Line changed**: [line numbers]
**Change summary**: [what was modified]

Should I commit this fix?
```

Wait for user response before committing.

### Step 8: Push Changes

After user approves (if PR work):

```bash
# Push to correct branch
git push origin HEAD:branch-name
```

## Common XPU Compatibility Patterns

When fixing CUDA-to-XPU issues, use these patterns:

### Pattern 1: Import Changes
```python
# BEFORE (CUDA-only)
from torch.testing._internal.triton_utils import requires_cuda_and_triton
from torch.testing._internal.common_cuda import requires_cuda

# AFTER (XPU-compatible)
from torch.testing._internal.inductor_utils import GPU_TYPE
from torch.testing._internal.triton_utils import requires_gpu_and_triton
```

### Pattern 2: Hard-coded Device Strings
```python
# BEFORE
torch.Stream(device="cuda")
x = torch.ones(2, 2, device="cuda")

# AFTER
torch.Stream(device=GPU_TYPE)
x = torch.ones(2, 2, device=GPU_TYPE)
```

### Pattern 3: Decorator Changes
```python
# BEFORE
@requires_cuda
@requires_cuda_and_triton

# AFTER
@requires_gpu_and_triton
```

### Pattern 4: Device Module Selection
```python
# BEFORE
with torch.cuda.stream(s):
    x = torch.sin(x)

# AFTER
device_module = torch.get_device_module(GPU_TYPE)
with device_module.stream(s):
    x = torch.sin(x)
```

### Pattern 5: Backend Capability Checks
```python
# BEFORE
if not torch.cuda.is_bf16_supported():
    raise unittest.SkipTest("requires bf16")

# AFTER
if GPU_TYPE == "cuda":
    bf16_supported = torch.cuda.is_bf16_supported()
elif GPU_TYPE == "xpu":
    bf16_supported = getattr(torch.xpu, "is_bf16_supported", lambda: False)()
else:
    bf16_supported = False
if not bf16_supported:
    raise unittest.SkipTest("requires bf16")
```

### Pattern 6: Event/Class Instantiation
```python
# BEFORE
event = torch.cuda.Event() if torch.cuda.is_available() else torch.xpu.Event()

# AFTER
Event = torch.cuda.Event if torch.cuda.is_available() else torch.xpu.Event
event = Event()
```

## Constraints

1. **PR branch first**: Always get correct PR branch before making changes
2. **Context required**: ALWAYS read 15-30 lines before analysis
3. **No pattern matching for analysis**: Use Read tool for semantic understanding
4. **Deep analysis over shortcuts**: Understand code intent before fixing
5. **Minimal fixes**: Only change what is necessary
6. **Preserve test logic**: Fixes should not change test behavior
7. **Ask before commit**: MANDATORY per agent-guidelines
8. **Atomic commits**: One issue per commit
9. **No force push to main**: Only to feature/PR branches
10. **Verify consistency**: Check similar patterns before applying fix

## Validation Checklist

Before considering a fix complete:

- [ ] Issue analyzed with context read (15-30 lines)
- [ ] Root cause identified
- [ ] Solution validated against similar patterns
- [ ] Fix applied with targeted Edit
- [ ] Changes verified with git diff
- [ ] User approval obtained (MANDATORY)
- [ ] Committed with descriptive message
- [ ] Pushed to correct branch
- [ ] Progress tracked in todowrite

## Example Complete Workflow

### Input
PR #3383 Copilot AI comment:
> `test_cuda_device` runs when either CUDA or XPU is available, but the implementation unconditionally uses `torch.cuda.device(...)`. This will raise on XPU-only test runs.

### Analysis
1. Read context at line 673 of test_ctx_manager_xpu.py
2. Understand: Function sets device context using CUDA API
3. Classify: XPU compatibility issue
4. Solution: Use torch.get_device_module(GPU_TYPE).device(...)

### Fix Applied
```python
# BEFORE
with torch.cuda.device(x.device.index - 1):
    x = torch.sin(x + 1)

# AFTER  
with torch.get_device_module(GPU_TYPE).device(x.device.index - 1):
    x = torch.sin(x + 1)
```

### Validation
```bash
git diff test/xpu/dynamo/test_ctx_manager_xpu.py | grep -A2 "test_cuda_device"
```

### Approval Request
```
**Issue fixed**: test_cuda_device XPU compatibility
**File modified**: test/xpu/dynamo/test_ctx_manager_xpu.py  
**Line changed**: 673
**Change summary**: Changed torch.cuda.device() to torch.get_device_module(GPU_TYPE).device()

Should I commit this fix?
```

### Push
```bash
git push daisyden dynamo_xpu
```