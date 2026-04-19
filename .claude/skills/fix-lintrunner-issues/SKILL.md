# Fix Lintrunner Issues

## Description

Fix lint violations and apply formatting fixes to PyTorch XPU test files using lintrunner. Use when porting test cases, updating a PR, or fixing lint issues. The skill identifies, analyzes, and resolves lint errors ensuring code quality for Intel XPU backend tests.

## Skill Integration

**This skill follows agent-guidelines AND extends it with specific constraints.**

Always apply agent-guidelines rules including:
- Mandatory post-write commit protocol (ask user before committing)
- Deep semantic analysis instead of pattern matching
- Atomic commits for each fix
- All constraints defined in agent-guidelines

## Preconditions

Before starting, ensure:
1. Working directory is at PyTorch repo root `/home/daisydeng/daisy_pytorch`
2. Git remotes are properly configured (both `intel` and `daisyden` for torch-xpu-ops)
3. Password-less authentication tokens are configured for GitHub push operations
4. Miniforge environment is available at `~/miniforge3/bin/activate`

## Instructions

### Step 1: Setup Test Environment

```bash
source ~/miniforge3/bin/activate ~/miniforge3/envs/pytorch_opencode_env
```

Always activate the correct environment before running lint or git commands.

### Step 2: Navigate to torch-xpu-ops

```bash
cd third_party/torch-xpu-ops
```

All lint operations should be performed within the torch-xpu-ops directory.

### Step 3: Run lintrunner init

Initialize all linters before checking:

```bash
lintrunner init
```

This installs required dependencies:
- FLAKE8 and extensions (flake8-bugbear, flake8-pyi, etc.)
- CLANGFORMAT, CLANGTIDY
- MYPY and type stubs
- RUFF, BLACK, ISORT, SHELLCHECK
- BAZEL_LINTER

### Step 4: Get PR Branch Before Running Lintrunner

**CRITICAL**: Before running lintrunner on a PR, you MUST identify and checkout the correct PR branch. All commits MUST be made to the PR branch, not to your local fork branch.

#### Get PR Branch Reference

Use the GitHub API to find the correct branch for a PR:

```bash
# Fetch PR info and extract branch reference
PR_NUM=3385
curl -s "https://api.github.com/repos/intel/torch-xpu-ops/pulls/${PR_NUM}" | grep -E '"head":|"label"|"ref"' | head -10

# Extract the branch reference from head info
# Example output: "label": "daisyden:daisyden/dtype_align_series"
# This tells us the PR is from daisyden:daisyden/dtype_align_series branch
```

#### Checkout Correct PR Branch

```bash
# Get the branch name from the API response
# Format: owner:branch-name (e.g., "daisyden:daisyden/dtype_align_series")
REMOTE=owner  # extracted from "label" field
BRANCH=branch-name  # extracted from "ref" field (e.g., daisyden/dtype_align_series)

# Fetch the PR branch
git fetch ${REMOTE} ${BRANCH}:${REMOTE}/${BRANCH}

# Checkout the PR branch
git checkout ${REMOTE}/${BRANCH}

# Verify we're on the correct branch
git status
git log --oneline -3
```

#### Verify PR Commits

After checkout, verify the PR contains expected commits:

```bash
git log --oneline -5  # Should show PR commits
git diff daisyden/dtype_align_series..HEAD --stat  # Compare with base
```

### Step 5: Run Lintrunner on PR Branch

Now run lintrunner on the PR branch:

```bash
# Always activate correct environment
source ~/miniforge3/bin/activate ~/miniforge3/envs/pytorch_opencode_env

# Run lintrunner init (first time only)
lintrunner init

# Find modified/added files in this PR
git diff origin/main...HEAD --name-only --diff-filter=AM > new_files.txt
```

### Step 6: Determine Files to Lint

#### Option A: Files from a PR

```bash
# Read files from new_files.txt or use git directly
FILES=$(git diff origin/main...HEAD --name-only --diff-filter=AM | grep -E '\.(py|h|cpp|hpp)$')
```

#### Option B: All test files

```bash
FILES=$(git diff origin/main...HEAD --name-only --diff-filter=AM | grep -E '\.(py|h|cpp|hpp)$')
```

### Step 7: Run Deep Analysis on Lint Output

Use the explore agent for deep semantic analysis of lint reports. DO NOT rely on simple pattern matching or regex search.

#### For each lint error/warning:

1. **Read context**: Load the file at the error line with surrounding lines
2. **Understand semantics**: Analyze what the code does, not just the pattern
3. **Identify root cause**: Determine if this is:
   - A genuine code issue (E731 lambda assignment)
   - An expected test registration (META_NO_CREATE_UNBACKED)
   - Auto-formatting needed (line-too-long, trailing whitespace)
4. **Categorize by type**:
   - **Fixable with noqa**: E731 lambdas, intentional complex expressions
   - **Fixable with auto-format**: FLAKE8 formatting, RUFF fixes
   - **Expected test errors**: mypy PyTEST registration checks
   - **Requires code change**: Real bugs or type issues

#### Example analysis workflow:

```
For E731 lambda warnings:
- Check if lambda is for triton grid (requires inline context)
- If triton grid or capture_triton, add # noqa: E731 to SAME line as lambda
- Do NOT try to convert to def - these lambdas are REQUIRED

For META_NO_CREATE_UNBACKED errors:
- These are EXPECTED in test files using ShapeEnv.create_unbacked_symint()
- Document as known false positives, no action needed
- If test uses create_unbacked APIs, this error is intentional

For line-too-long (B950):
- Check if inside string block or expected output
- For expected outputs: add # noqa: B950 on SAME line as closing triple quote
- For actual code: reformat to fit within 120 chars
```

### Step 8: Apply Fixes or Auto-Formatting

Apply all fixes to the PR branch, then commit:

```bash
# Add noqa to individual lambda
edit file.py old_string new_string  # noqa comment on same line as lambda
git add file.py

# Run auto-formatting
lintrunner -a <files>

# Commit on PR branch (MANDATORY ask-user before commit)
git commit -m "Lint: apply fixes to PR #${PR_NUM}

Summary of fixes:
- E731 noqa for triton grid lambdas  
- Auto-formatting via lintrunner

8 files changed, 189 insertions(+), 77 deletions(-)"
```

### Step 9: Push to PR Branch

**CRITICAL**: Push to the correct PR branch:

```bash
# Push to PR branch (owner/branch from Step 4)
git push origin HEAD:${BRANCH}

# If changes needed, force push to PR branch only
git push origin HEAD:${BRANCH} --force

# For intel PRs - push to intel remote
git push intel HEAD:refs/heads/${BRANCH}
```

## Constraints

1. **PR branch required before lint**: ALWAYS get PR branch via curl (Step 4) before running lintrunner
2. **DO NOT skip PR branch step**: Committing to wrong branch wastes time and causes confusion
3. **Merge related commits**: Multiple lint commits for same PR should be squashed
4. **Test environment**: Always use `~/miniforge3/envs/pytorch_opencode_env`
5. **No force push to main**: Only force push to PR feature branches, never main
6. **Preserve file content**: Auto-formatting should not change test logic
7. **Ask user before commit**: Follow agent-guidelines commit protocol

## Used Tools

- **Read**: Load files at specific lines for context
- **Edit**: Apply targeted fixes with # noqa comments
- **Glob/Grep**: Find patterns for counting only, NOT for analysis
- **Bash**: Run lint commands and git operations
- **Task (explore subagent)**: For deep semantic analysis of complex lint reports
- **todowrite**: Track progress for multi-step lint fixes

## Deep Analysis Protocol

When encountering lint errors, follow this analysis chain:

1. **Intent recognition**: What is the code trying to do?
2. **Error classification**: Is this a style issue or semantic error?
3. **Solution validation**: Will the fix maintain intended behavior?
4. **Risk assessment**: Could fix break test logic?

### Example: Analyzing B950 line-too-long

WRONG approach:
```
grep for lines > 120 chars and auto-break them
```

RIGHT approach (semantic):
```
1. Read context: is this in expected inline output or actual code?
2. Check B950 location: line 752 in test_higher_order_ops_xpu.py
3. Analyze content: torch.ops.aten._assert_scalar.default(...)
4. Determine intent: This is torch.compile output for tracking
5. Decision: For expected outputs in assertExpectedRaisesInline,
   add # noqa: B950 on closing triple quote; for actual code,
   break long variable assignments
```

### Example: Analyzing E731 lambda warnings

WRONG approach:
```
Replace all lambda with def using regex substitution
```

RIGHT approach (semantic):
```
1. Read context at line 1039 in test_aot_autograd_cache_xpu.py
2. Check surrounding code: `capture_triton(kernel)[grid](...)`
3. Recognize pattern: Triton grid functions are called inline
4. Understand constraint: Triton requires callable with meta param
5. Decision: Add # noqa: E731 to SAME LINE as lambda definition
6. Verification: Test must still run and pass
```

## Handling Known False Positives

### META_NO_CREATE_UNBACKED errors

These errors appear when tests use:
- `ShapeEnv().create_unbacked_symint()`
- `ShapeEnv().create_unbacked_symfloat()`
- `ShapeEnv().create_unbacked_symbool()`

These are EXPECTED in PyTorch test files for shape environment testing.
Document these but do NOT attempt to "fix" the code.

### Lambda warnings in triton context

Triton grid functions LIKE `grid = lambda meta: (...)` are REQUIRED
because they are called in inline context and cannot be converted
to regular def functions in the same scope.

## Validation Checklist

Before pushing:

- [ ] PR branch identified via curl before starting lint work
- [ ] Checked out correct PR branch (verified with `git status` and `git log`)
- [ ] All E731 noqa comments added on SAME line as lambda
- [ ] B950 noqa added on closing triple quote for expected outputs
- [ ] No B950 noqa on content that should be reformatted
- [ ] Auto-formatting applied consistently
- [ ] Commits merged into single atomic commit for PR
- [ ] Tests still pass after fixes (if verification possible)
- [ ] Pushed to correct PR branch (owner/branch from Step 4)
- [ ] Force push only to PR feature branches, never main

## Example Workflows

### Workflow 1: Fix PR lint issues (CORRECT - PR BRANCH FIRST)

```
1. Load skill for fix-lintrunner-issues
2. Get PR branch via curl (Step 4):
   curl -s "https://api.github.com/repos/intel/torch-xpu-ops/pulls/3385" | grep '"head":'
   # Extract owner and branch from: "label": "daisyden:daisyden/dtype_align_series"
3. Fetch and checkout PR branch:
   git fetch daisyden daisyden/dtype_align_series:daisyden/dtype_align_series
   git checkout daisyden/dtype_align_series
4. Verify we're on PR branch (not local fork branch)
5. Run lintrunner init
6. Get changed files: git diff origin/main...HEAD --name-only
7. Run lintrunner -a on all changed files
8. For each error:
   - Read context with 15-20 line window
   - Classify: fixable/expected/auto-format
   - Apply appropriate fix
   - Always commit to PR branch (ask-user before commit)
9. Push to PR branch: git push origin HEAD:daisyden/dtype_align_series
```

### Workflow 2: Fix fork branch lint issues

```
1. Load skill for fix-lintrunner-issues
2. Ensure on correct daisyden branch (e.g., daisyden/dynamo_xpu)
3. Run lintrunner init (if not done)
4. Identify files needing fixes
5. Run lintrunner -a on files
6. Deep analysis of each error type
7. Apply fixes with proper noqa comments
8. Merge all lint commits into one
9. Force push to daisyden branch
```

### Workflow 3: Quick PR lint check

```
PR_NUM=3385
# Step 1: Get PR branch
BRANCH=$(curl -s "https://api.github.com/repos/intel/torch-xpu-ops/pulls/${PR_NUM}" | python3 -c "import sys,json; print(json.load(sys.stdin)['head']['ref'])")
OWNER=$(curl -s "https://api.github.com/repos/intel/torch-xpu-ops/pulls/${PR_NUM}" | python3 -c "import sys,json; print(json.load(sys.stdin)['head']['label'].split(':')[0])")

# Step 2: Checkout PR branch
git fetch ${OWNER} ${BRANCH}:PR-${PR_NUM}
git checkout PR-${PR_NUM}

# Step 3: Run lint and push
source ~/miniforge3/bin/activate ~/miniforge3/envs/pytorch_opencode_env
lintrunner init
lintrunner -a $(git diff origin/main...HEAD --name-only --diff-filter=AM)
git add -A && git commit -m "Lint fixes" && git push ${OWNER} HEAD:${BRANCH}
```