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

### Step 4: Determine Files to Lint

#### Option A: Files from a PR

If a PR URL is provided (e.g., `https://github.com/intel/torch-xpu-ops/pull/3383`):

```bash
# Fetch and checkout PR
PR_NUM=3383
git fetch intel pull/${PR_NUM}/head:pr-${PR_NUM}
git checkout pr-${PR_NUM}

# Find modified/added files
git fetch origin main
git diff origin/main...HEAD --name-only --diff-filter=AM > new_files.txt
```

#### Option B: All test files

```bash
FILES=$(git diff origin/main...HEAD --name-only --diff-filter=AM | grep -E '\.(py|h|cpp|hpp)$')
```

### Step 5: Run Deep Analysis on Lint Output

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

### Step 6: Apply Fixes or Auto-Formatting

#### For individual fixes:

```bash
# Add noqa to individual lambda
edit file.py old_string new_string  # noqa comment on same line as lambda
git add file.py
git commit -m "brief descriptive message"
```

#### For auto-formatting:

```bash
# Run full lintrunner on specific files
lintrunner -a <files>

# Check for remaining issues
lintrunner -a <files> 2>&1 | grep -v "Advice.*B950" | grep -E "^\s+Error"
```

### Step 7: Merge Commits for PR

If contributing to a PR, squash lint commits:

```bash
git log --oneline lint-commits-start~1..HEAD  # check commits to merge
git reset --soft <first-commit-sha>~1
git commit -m "Lint: apply fixes to PR #NNN

Summary of fixes:
- E731 noqa for triton grid lambdas
- Auto-formatting via lintrunner

8 files changed, 189 insertions(+), 77 deletions(-)"
```

### Step 8: Push to Appropriate Branch

#### For intel PR:

```bash
PR_BRANCH=pr-NNN
git push intel HEAD:${PR_BRANCH}
# or force push if needed
git push intel HEAD:${PR_BRANCH} --force
```

#### For daisyden fork:

```bash
DEST_BRANCH=daisyden/dynamo_xpu  # or provided branch name
git push daisyden HEAD:${DEST_BRANCH} --force
```

## Constraints

1. **DO NOT skip steps**: Always run lintrunner init, analyze, then fix
2. **DO NOT use pattern matching** for code analysis - use semantic understanding
3. **Merge related commits**: Multiple lint commits for same PR should be squashed
4. **Test environment**: Always use `~/miniforge3/envs/pytorch_opencode_env`
5. **No force push to main**: Only force push to feature branches
6. **Preserve file content**: Auto-formatting should not change test logic

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

- [ ] All E731 noqa comments added on SAME line as lambda
- [ ] B950 noqa added on closing triple quote for expected outputs
- [ ] No B950 noqa on content that should be reformatted
- [ ] Auto-formatting applied consistently
- [ ] Commits merged into single atomic commit for PR
- [ ] Tests still pass after fixes (if verification possible)
- [ ] Pushed to correct branch (intel PR or daisyden fork)
- [ ] Force push only to feature branches, never main

## Example Workflows

### Workflow 1: Fix new PR lint issues

```
1. Load skill for fix-lintrunner-issues
2. Checkout intel PR: #3383
3. Run lintrunner init
4. Get changed files: git diff origin/main...HEAD --name-only
5. Run lintrunner -a on all changed files
6. For each error:
   - Read context with 15-20 line window
   - Classify: fixable/expected/auto-format
   - Apply appropriate fix
   - Always commit atomic changes
7. Merge lint commits
8. Push to intel PR branch
```

### Workflow 2: Fix fork branch lint issues

```
1. Load skill for fix-lintrunner-issues
2. Ensure on daisyden/dynamo_xpu branch
3. Run lintrunner init (if not done)
4. Identify files needing fixes
5. Run lintrunner -a on files
6. Deep analysis of each error type
7. Apply fixes with proper noqa comments
8. Merge all lint commits into one
9. Force push to daisyden/dynamo_xpu
```