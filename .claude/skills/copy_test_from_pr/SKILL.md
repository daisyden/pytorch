# Copy Test from PR Skill

## Overview
This skill guides the process of copying/migrating test classes from an existing PyTorch PR branch to a target XPU test file. It provides a systematic approach for porting tests to the Intel XPU backend.

## Preconditions

### Environment Requirements
- Working directory: `/home/daisydeng/daisy_pytorch`
- conda environment: `pytorch_opencode_env` must be activated for testing
- Git repository must be clean or have tracked changes
- Network access to download files from GitHub

### conda Environment Setup
```bash
export PATH="/home/daisydeng/miniforge3/bin:$PATH"
eval "$(/home/daisydeng/miniforge3/bin/conda shell.bash hook)"
conda activate pytorch_opencode_env
```

### Typical Test File Locations
- Source PR tests: `test/test_<module>.py` (PyTorch main repo)
- Target XPU tests: `third_party/torch-xpu-ops/test/xpu/test_<module>_xpu.py`

## Required Tools

### Subagent - Explore (Recommended for Analysis)
```bash
Task(tool="explore", description="...", prompt="...")
```
**Purpose**: Comprehensive codebase exploration with deep analysis
- Fast agent specialized for exploring codebases
- Use "very thorough" level for complex test files
- Can identify class boundaries, dependencies, and patterns

### File Operations
| Tool | Purpose | Usage |
|------|---------|-------|
| `read` | Read file contents with line numbers | `read file.py --offset 1 --limit 100` |
| `grep` | Search file contents with regex | `grep pattern --include "*.py"` |
| `glob` | Find files by pattern | `glob pattern "**/test_*.py"` |
| `edit` | Edit files with exact string matching | `edit file.py --oldString "X" --newString "Y"` |
| `write` | Write files | `write content file.py` |

### Code Analysis
| Tool | Purpose | Command |
|------|---------|---------|
| `bash + ast` | Parse Python AST for syntax validation | `python3 -c "import ast; ast.parse(open('file.py').read())"` |
| `bash + curl` | Download and analyze remote files | `curl -sL URL -o /tmp/file.py` |
| `bash + wc` | Count lines and analyze structure | `wc -l file.py` |
| `bash + sed` | Extract sections by line numbers | `sed -n 'start,end p' file.py` |

## Scenario Determination

Before starting, determine which scenario applies:

### Scenario A: Existing _xpu Test File Exists
- Target file `test_<module>_xpu.py` already exists in `torch-xpu-ops/test/xpu/`
- Need to COPY specific class(es) from PR into existing target
- Follow **Scenario A Workflow** below

### Scenario B: No _xpu Test File Exists
- Source file `test/<module>.py` has NO counterpart `test_<module>_xpu.py` in `torch-xpu-ops/test/xpu/`
- Need to CREATE new file by copying entire PR file and adapting
- Follow **Scenario B Workflow** below

### How to Determine Scenario
```bash
# Check if _xpu version exists
SOURCE_MODULE="nn"  # or other module name
ls -la /home/daisydeng/daisy_pytorch/third_party/torch-xpu-ops/test/xpu/test_${SOURCE_MODULE}_xpu.py 2>/dev/null && echo "Scenario A" || echo "Scenario B"
```

---

## Scenario A: Copy Class into Existing _xpu File

### Phase 1: Deep Source Analysis

#### Step 1.1: Download Source File
```bash
# Download from PR branch
BRANCH_NAME="<pr_branch_name>"
SOURCE_FILE="test_<module>.py"  # e.g., test_nn.py
curl -sL "https://raw.githubusercontent.com/daisyden/pytorch/$BRANCH_NAME/test/$SOURCE_FILE" -o /tmp/source_test.py
wc -l /tmp/source_test.py
```

#### Step 1.2: Analyze Class Structure with Explore Agent
```bash
Task(tool="explore", description="Analyze source class structure", 
prompt="""Analyze the file /tmp/source_test.py thoroughly.

Find and report:
1. All class definitions (class Xxx:)
2. For each class:
   - Exact line boundaries (start/end)
   - Class docstring (first multi-line string)
   - Key methods and decorators
   - Helper functions defined within/near the class
3. Import statements and their line numbers
4. Any @onlyAccelerator or @deviceDecorator usage
5. Find instantiate_device_type_tests() calls and their parameters

Return a structured report with all findings.""",
subagent_type="explore")
```

### Phase 2: Analyze Target File Existing Structure

#### Step 2.1: Check Target File Structure
```bash
Task(tool="explore", description="Analyze target XPU test structure",
prompt="""Analyze /home/daisydeng/daisy_pytorch/third_party/torch-xpu-ops/test/xpu/test_<module>_xpu.py

Find and report:
1. All current class definitions
2. Existing import structure (lines 1-100)
3. Test instantiation at end of file
4. Any existing onlyAccelerator or device-specific handling

Return detailed findings for modification planning.""",
subagent_type="explore")
```

### Phase 3: Extract Source Class (Precise Boundaries)

#### Step 3.1: Determine Exact Class Boundaries
```bash
# Use explore findings or grep to identify exact boundaries
CLASS_NAME="TestNN"
grep -n "^class ${CLASS_NAME}(" /tmp/source_test.py | head -1
grep -n "^class " /tmp/source_test.py | head -5  # Find next class
```

#### Step 3.2: Extract with Verification
```bash
# Extract class definition
LINE_START=<start_line>
LINE_END=<end_line>
sed -n "${LINE_START},${LINE_END}p" /tmp/source_test.py > /tmp/extracted_class.py
```

### Phase 4: Copy and Adapt Source to Target

#### Step 4.1: Insert Class into Target
```python
# Python script for safe insertion
with open('third_party/torch-xpu-ops/test/xpu/test_<module>_xpu.py', 'r') as f:
    target_content = f.read()

with open('/tmp/extracted_class.py', 'r') as f:
    source_class = f.read()

# Find insertion point (before first existing class definition)
INSERT_MARKER = '<existing_class_marker>'
insert_pos = target_content.find(INSERT_MARKER)
new_content = target_content[:insert_pos] + source_class + '\n\n' + target_content[insert_pos:]

with open('third_party/torch-xpu-ops/test/xpu/test_<module>_xpu.py', 'w') as f:
    f.write(new_content)
```

#### Step 4.2: Apply Import Fixes (See Common Issues Section)

### Phase 5: Verify and Add Instantiations

#### Step 5.1: Verify Syntax with AST
```bash
python3 -c "import ast; ast.parse(open('third_party/torch-xpu-ops/test/xpu/test_<module>_xpu.py').read()); print('Syntax OK')"
```

#### Step 5.2: Add XPU Test Instantiation
```python
# Add allow_xpu=True to instantiation
content = content.replace(
    'instantiate_device_type_tests(TestNN, globals()',
    'instantiate_device_type_tests(TestNN, globals(), allow_xpu=True'
)
```

### Phase 6-9: Verify Tests, Run, Check Issues, Document
Follow remaining phases from Verification onward.

---

## Scenario B: Create New _xpu File from Scratch

When no `_xpu` counterpart exists, create an entire new test file.

### Phase B1: Setup and Download

#### Step B1.1: Determine Target Path
```bash
SOURCE_MODULE="nn"  # e.g., "nn" from test/test_nn.py
TARGET_DIR="/home/daisydeng/daisy_pytorch/third_party/torch-xpu-ops/test/xpu"
TARGET_FILE="${TARGET_DIR}/test_${SOURCE_MODULE}_xpu.py"

# Verify directory exists
ls -la "${TARGET_DIR}/" 2>/dev/null | head -10
```

#### Step B1.2: Download Source File
```bash
BRANCH_NAME="<pr_branch_name>"
SOURCE_FILE="test_${SOURCE_MODULE}.py"
curl -sL "https://raw.githubusercontent.com/daisyden/pytorch/$BRANCH_NAME/test/$SOURCE_FILE" -o /tmp/full_source_test.py
wc -l /tmp/full_source_test.py
```

#### Step B1.3: Deep Analyze Full Source File
```bash
Task(tool="explore", description="Full source file analysis",
prompt="""Complete deep analysis of /tmp/full_source_test.py

1. Structure Analysis:
   - Total line count
   - All import statements with line numbers
   - All class definitions with boundaries
   - All decorator usage (@onlyCPU, @onlyCUDA, @given, etc.)
   - Instantiate_device_type_tests() calls

2. XPU Compatibility Analysis:
   - Identify all CUDA-specific decorators/imports
   - Find all TEST_CUDA conditional code
   - Check for CUDA-only kernel usages
   - Identify bfloat16/float16 precision tests

3. Required Modifications List:
   - Imports that need local definitions
   - Decorators needing XPU equivalents
   - Conditional code needing XPU handling

Return comprehensive findings for complete porting.""",
subagent_type="explore")
```

### Phase B2: Create Adapted Copy

#### Step B2.1: Create Target File from Source
```python
# Read full source
with open('/tmp/full_source_test.py', 'r') as f:
    content = f.read()

# Remove ACCELERATOR_TYPE from import if present
content = content.replace(', ACCELERATOR_TYPE', '')

# Remove onlyAccelerator from import if present
content = content.replace(', onlyAccelerator', '')
content = content.replace('onlyAccelerator', '')

# Fix double commas
content = content.replace(',,', ',')

# Add local definitions after load_tests line
INSERT_MARKER = 'load_tests = load_tests  # noqa: PLW0127\n'
LOCAL_DEFS = '''
# Local ACCELERATOR_TYPE for XPU compatibility
def _get_accelerator_type():
    if torch.xpu.is_available():
        return "xpu"
    elif torch.cuda.is_available():
        return "cuda"
    elif torch.backends.mps.is_available():
        return "mps"
    else:
        return "cpu"

ACCELERATOR_TYPE = _get_accelerator_type()

# onlyAccelerator decorator for XPU - simplified pass-through
onlyAccelerator = lambda func: func

'''
content = content.replace(INSERT_MARKER, INSERT_MARKER + LOCAL_DEFS)
```

#### Step B2.2: Fix Root-Level Instantiation
```python
# Find and update instantiation section
OLD_INSTANTIATE = '''instantiate_device_type_tests(TestNNDeviceType, globals(), allow_mps=True)

device_list = None

# # https://github.com/pytorch/pytorch/issues/177119
# if os.environ.get('PYTORCH_TEST_WITH_DYNAMO', '0') == '1':
#     device_list = ('cpu', )

instantiate_device_type_tests(TestNN, globals(), except_for=device_list)'''

NEW_INSTANTIATE = '''instantiate_device_type_tests(
    TestNNDeviceType, globals(), allow_mps=True, allow_xpu=True
)
instantiate_device_type_tests(TestNN, globals(), allow_xpu=True)'''

content = content.replace(OLD_INSTANTIATE, NEW_INSTANTIATE)
```

#### Step B2.3: Write Adapted Source to Target
```python
# Write adapted content to target
TARGET_PATH = '/home/daisydeng/daisy_pytorch/third_party/torch-xpu-ops/test/xpu/test_<module>_xpu.py'
with open(TARGET_PATH, 'w') as f:
    f.write(content)
print(f'Created {TARGET_PATH}')
```

### Phase B3: Verify File Creation

#### Step B3.1: Syntax Verification
```bash
python3 -c "import ast; ast.parse(open('/home/daisydeng/daisy_pytorch/third_party/torch-xpu-ops/test/xpu/test_<module>_xpu.py').read()); print('Syntax OK')"
```

#### Step B3.2: Test Collection
```bash
cd /tmp  # Run from /tmp to avoid local torch
export PATH="/home/daisydeng/miniforge3/bin:$PATH"
eval "$(/home/daisydeng/miniforge3/bin/conda shell.bash hook)"
conda activate pytorch_opencode_env

python3 -m pytest /home/daisydeng/daisy_pytorch/third_party/torch-xpu-ops/test/xpu/test_<module>_xpu.py --collect-only 2>&1 | head -30
```

### Phase B4: Fix Collection Errors
```bash
# Diagnose any errors
python3 -m pytest /home/daisydeng/daisy_pytorch/third_party/torch-xpu-ops/test/xpu/test_<module>_xpu.py --collect-only 2>&1 | grep -E "ERROR|NameError|ImportError"

# Fix using explore agent
Task(tool="explore", description="Fix collection errors",
prompt="""Fix the pytest collection errors in:
/home/daisydeng/daisy_pytorch/third_party/torch-xpu-ops/test/xpu/test_<module>_xpu.py

1. Identify all missing imports
2. Find broken multi-line import statements
3. Check for syntax errors

Apply necessary fixes and verify.""",
subagent_type="explore")
```

### Phase B5: Run Tests

#### Step B5.1: Run All Tests in New File
```bash
cd /tmp
export PYTEST_ADDOPTS="-n 1 --timeout=60"

timeout 600 python3 -m pytest \
    /home/daisydeng/daisy_pytorch/third_party/torch-xpu-ops/test/xpu/test_<module>_xpu.py \
    -v --tb=short 2>&1 > /tmp/test_<module>_xpu_results.txt

# Extract summary
tail -30 /tmp/test_<module>_xpu_results.txt

# Extract failures
grep "FAILED" /tmp/test_<module>_xpu_results.txt | head -30
```

#### Step B5.2: Record Results
```bash
cp /tmp/test_<module>_xpu_results.txt /home/daisydeng/daisy_pytorch/test_<module>_xpu_failures.txt
```

### Phase B6: Check Known Issues

#### Step B6.1: Search GitHub for Each Failure
```bash
grep "FAILED" /home/daisydeng/daisy_pytorch/test_<module>_xpu_failures.txt | cut -d: -f4 | cut -d_ -f1-5 | sort -u | while read test; do
    echo "=== Checking: $test ==="
    curl -s "https://api.github.com/search/issues?q=repo:intel/torch-xpu-ops+$test+is:issue+state:open&per_page=3" | \
    python3 -c "import sys,json; d=json.load(sys.stdin); print('  Found:', d['total_count'], 'open issues') if d.get('total_count',0)>0 else print('  No open issue')" 2>/dev/null
done
```

**IMPORTANT: PR Submission Rules**
- Clone fresh from `intel/torch-xpu-ops` origin/main before creating PR
- Ask user to provide specific commit range (e.g., `start_hash...end_hash`) before submission
- Create separate PRs for distinct purposes; do not mix unrelated commits
- Maintain folder structure mapping: `test/xpu/<source_subdir>/` mirrors `test/<source_subdir>/`
  - `test/dynamo/` → `test/xpu/dynamo/`
  - `test/functorch/` → `test/xpu/functorch/`
  - `test/nn/` → `test/xpu/nn/`

### Phase B7: Commit Changes

#### Step B7.1: Stage and Commit
```bash
cd /home/daisydeng/daisy_pytorch

# Add file to git (if tracking separate test repo)
git add third_party/torch-xpu-ops/test/xpu/test_<module>_xpu.py

# Or if using separate torch-xpu-ops git:
cd /home/daisydeng/daisy_pytorch/third_party/torch-xpu-ops
git add test/xpu/test_<module>_xpu.py
git commit -m "Add test_<module>_xpu.py: Port test_<module>.py to XPU backend

- Ported test classes from pytorch PR branch
- Added local ACCELERATOR_TYPE and onlyAccelerator definitions
- Enabled XPU device tests with allow_xpu=True
- Updated test instantiation structure

Authored with Claude."

git log --oneline -2
```

---

## Common Issues and Fixes Reference

### Import Failure Fixes (Deep Analysis Approach)

When encountering ImportError or NameError during test collection, apply deep analysis:

**Step 1: Identify the missing import**
```bash
# Check test collection error to identify missing import
python -m pytest test/<module>.py --collect-only 2>&1 | grep "ImportError\|NameError"
```

**Step 2: Check original pytorch test folder for pattern**
```bash
# Look at the original test file in pytorch/test/<subdir>/
ls -la pytorch/test/<subdir>/

# Check how other pytorch tests in same folder define missing utilities
grep -n "def requires_gpu\|requires_gpu =" pytorch/test/<subdir>/*.py | head -10
```

**Step 3: Check existing xpu tests for reference**
```bash
# See how existing xpu tests handle similar imports
cat third_party/torch-xpu-ops/test/xpu/<subdir>/test_<other>_xpu.py | head -30
```

**Common patterns found through analysis:**

| Missing Import | How to Fix |
|----------------|------------|
| `requires_cuda`, `requires_gpu` | Define locally using `torch.cuda.is_available()` or `torch.xpu.is_available()` |
| `test_functions` (cross-file dependency) | Add `PYTORCH_TEST_PATH` pointing to pytorch test source folder if file uses relative imports |
| `requires_gpu_and_triton` | Import from `torch.testing._internal.triton_utils` |

**Example: Defining local requires_gpu**
```python
# From pytorch/test/dynamo/test_logging.py pattern
import torch
import unittest

requires_gpu = unittest.skipUnless(
    torch.cuda.is_available() or (hasattr(torch, 'xpu') and torch.xpu.is_available()),
    "requires cuda or xpu"
)
```

**Example: Cross-file dependency (when original uses `from . import test_functions`)**
```python
# Add path to pytorch test source
from pathlib import Path
import sys

PYTORCH_TEST_PATH = str(Path(__file__).resolve().parents[5] / "test" / "dynamo")
if PYTORCH_TEST_PATH not in sys.path:
    sys.path.insert(0, PYTORCH_TEST_PATH)

# Then import the module
from test_functions import *
```

**Rule**: Always prefer defining utilities locally over depending on missing pytorch utilities. Only use path manipulation when original test truly depends on sibling modules.

| Issue | Symptom | Fix |
|-------|---------|---|
| ACCELERATOR_TYPE missing | `ImportError: cannot import name 'ACCELERATOR_TYPE'` | Add local `_get_accelerator_type()` function |
| onlyAccelerator missing | `NameError: name 'onlyAccelerator' is not defined` | Add `onlyAccelerator = lambda func: func` |
| onlyXPU available | `NameError: name 'onlyXPU' is not defined` | Import `onlyXPU` from common_device_type |
| requires_cuda/requires_gpu missing | `NameError: name 'requires_gpu' is not defined` | Define locally using `torch.cuda.is_available()` or `torch.xpu.is_available()` |
| Broken import continuation | `SyntaxError: trailing comma` | Remove orphan commas |
| Device type mismatch | `NameError: name 'device' is not defined` | Use proper device parameter |
| File ends with execute code | No tests collected | Remove `if __name__` block or restructure instantiation |

## Validation Checklist

- [ ] Source file correctly identified
- [ ] Scenario determined (A or B)
- [ ] All imports analyzed for compatibility
- [ ] Missing imports have local workarounds
- [ ] Python syntax validated with AST
- [ ] Test collection succeeds (0 errors)
- [ ] XPU instantiation added (allow_xpu=True)
- [ ] Tests run to completion
- [ ] Failures recorded and analyzed
- [ ] Known issues searched in GitHub
- [ ] Results documented
- [ ] File committed (if applicable git tracking)

## Working Logic Summary

### Scenario A (Existing _xpu File):
1. Download source → Analyze class boundaries
2. Extract specific class → Insert into existing target
3. Apply import fixes → Add XPU instantiation
4. Verify syntax → Run tests → Check issues → Document

### Scenario B (New _xpu File):
1. Download full source → Deep analyze entire file
2. Create adapted copy (remove/add local definitions)
3. Fix instantiation → Write to target path
4. Verify syntax → Fix any collection errors
5. Run all tests → Check known issues → Commit

## Generic Application

This procedure applies to ANY test class/file porting:

1. **Scenario Detection**: Check if `_xpu` version exists
2. **Analyze**: Use explore agent for deep understanding
3. **Adapt**: For each missing import, apply appropriate workaround
4. **Support**: Include `allow_xpu=True` in all instantiations
5. **Verify**: Test collection before running
6. **Run and Document**: Execute tests and record failures
7. **Check Known Issues**: Search GitHub with exact test names
8. **Commit**: Stage and commit to appropriate repository