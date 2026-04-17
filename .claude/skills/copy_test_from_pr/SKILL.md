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

## Working Logic - Deep Analysis Approach

### Phase 1: Deep Source Analysis

#### Step 1.1: Download Source File
```bash
# Download from PR branch
BRANCH_NAME="daisyden/test_nn_stage1"
curl -sL "https://raw.githubusercontent.com/daisyden/pytorch/$BRANCH_NAME/test/test_nn.py" -o /tmp/source_test.py
wc -l /tmp/source_test.py
```

#### Step 1.2: Analyze Class Structure with Explore Agent
```bash
Task(tool="explore", description="Analyze test_nn class structure", 
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
5. Look for energy_aware_test, skipCUDAIf, or other test decorators
6. Find instantiate_device_type_tests() calls and their parameters
7. Check for any conditional test definitions (if TEST_XPU:, etc.)

Return a structured report with all findings.""",
subagent_type="explore")
```

#### Step 1.3: Identify Test Class Dependencies
```bash
# Use explore agent to identify all imports and decorators
Task(tool="explore", description="Analyze imports and dependencies",
prompt="""Deep analysis of /tmp/source_test.py for all dependencies.

1. Find ALL import statements and categorize them:
   - Standard library imports (os, sys, unittest, etc.)
   - torch imports
   - testing._internal imports
   - Third-party imports (hypothesis, scipy, numpy)

2. For each import category, identify:
   - Specific names/objects imported
   - Whether they're used in the target test class
   - Potential compatibility issues with installed PyTorch

3. Check for decorator usage:
   - @given, @parametrize, @onlyCPU, @onlyCUDA, @onlyXPU
   - @xfail, @skipIf for platform-specific skipping
   - Custom decorators

4. Find function definitions outside classes:
   - Helper functions used by tests
   - Module-level setup/teardown functions
   
Return organized findings with specific line numbers.""",
subagent_type="explore")
```

### Phase 2: Analyze Target File Existing Structure

#### Step 2.1: Check Target File Class Structure
```bash
# Use explore agent to understand existing target
Task(tool="explore", description="Analyze target XPU test structure",
prompt="""Analyze /home/daisydeng/daisy_pytorch/third_party/torch-xpu-ops/test/xpu/test_nn_xpu.py

Find and report:
1. All current class definitions
2. Existing import structure (lines 1-100)
3. Test instantiation at end of file
4. Any existing onlyAccelerator or device-specific handling
5. Skip/test configuration at end of file
6. Check for any TestNN or similar class already present

Return detailed findings for modification planning.""",
subagent_type="explore")
```

#### Step 2.2: Identify Insertion Points
```bash
# Deep scan for patterns
Task(tool="explore", description="Find insertion points and patterns",
prompt="""Analyze /home/daisydeng/daisy_pytorch/third_party/torch-xpu-ops/test/xpu/test_nn_xpu.py

Find:
1. Line number where "class TestFusionEval" or similar class begins
2. All instantiate_device_type_tests() calls and their parameters
3. Pattern of device type suffixes (_cpu, _xpu, _cuda)
4. Any existing skip decorators or xfail patterns
5. Structure of if __name__ == "__main__": block

Return precise line numbers and context for each finding.""",
subagent_type="explore")
```

### Phase 3: Extract Source Class (Precise Boundaries)

#### Step 3.1: Determine Exact Class Boundaries
```bash
# Use explore findings to set exact boundaries
# Example for test_nn.py TestNN class:
# - Start: grep -n "class TestNN(NNTestCase):" | cut -d: -f1
# - End: grep -n "^class TestNNDeviceType" | head -1 | cut -d: -f1
```

#### Step 3.2: Extract with Verification
```bash
# Extract class definition
LINE_START=83
LINE_END=8147
sed -n "${LINE_START},${LINE_END}p" /tmp/source_test.py > /tmp/extracted_class.py

# Verify structure
python3 -c "
content = open('/tmp/extracted_class.py').read()
print(f'Lines: {len(content.splitlines())}')
print(f'Starts with class: {content.startswith(\"class \")}')
print(f'Has class end: {\"class TestNNDeviceType\" in content}')
"
```

### Phase 4: Copy and Adapt Source to Target

#### Step 4.1: Insert Class into Target
```python
# Python script for safe insertion
with open('third_party/torch-xpu-ops/test/xpu/test_nn_xpu.py', 'r') as f:
    target_content = f.read()

with open('/tmp/extracted_class.py', 'r') as f:
    source_class = f.read()

# Find insertion point (before class TestFusionEval)
INSERT_MARKER = 'class TestFusionEval(TestCase):'
insert_pos = target_content.find(INSERT_MARKER)

if insert_pos != -1:
    new_content = target_content[:insert_pos] + source_class + '\n\n' + target_content[insert_pos:]
else:
    # Alternative: insert before instantiate_device_type_tests
    INSTANTIATE_MARKER = 'instantiate_device_type_tests(TestNNDeviceType'
    insert_pos = target_content.find(INSTANTIATE_MARKER)
    new_content = target_content[:insert_pos] + source_class + '\n\n' + target_content[insert_pos:]

with open('third_party/torch-xpu-ops/test/xpu/test_nn_xpu.py', 'w') as f:
    f.write(new_content)
print('Class inserted successfully')
```

#### Step 4.2: Deep Import Analysis and Fixes

##### Analysis: Check Available Imports in Installed PyTorch
```bash
# Analyze AND fix missing imports
python3 << 'EOF'
import subprocess
import sys

# Check what constants are available
checks = {
    'ACCELERATOR_TYPE': "from torch.testing._internal.common_utils import ACCELERATOR_TYPE",
    'onlyAccelerator': "from torch.testing._internal.common_device_type import onlyAccelerator"
}

for name, import_stmt in checks.items():
    try:
        exec(compile(import_stmt, '<string>', 'exec'), {'__builtins__': __builtins__})
        print(f"✓ {name}: AVAILABLE")
    except (ImportError, NameError, AttributeError) as e:
        print(f"✗ {name}: NOT AVAILABLE - {e}")
        print(f"  Workaround needed: Define locally")
EOF
```

##### Fix Strategy by Import Type

**Type 1: Constants/Variables from common_utils**
```python
# Problem: ACCELERATOR_TYPE, TEST_ACCELERATOR might not exist
# Solution: Add local definition after AMPERE_OR_ROCM
'''
# Local definitions for XPU compatibility
_MARKER_ = "AMPERE_OR_ROCM = TEST_WITH_ROCM or torch.cuda.is_tf32_supported()"

_REPLACEMENT_ = """AMPERE_OR_ROCM = TEST_WITH_ROCM or torch.cuda.is_tf32_supported()

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
"""

# Apply fix
content = content.replace(_MARKER_, _REPLACEMENT_)
```

**Type 2: Device Decorators from common_device_type**
```python
# Problem: onlyAccelerator may not be available
# Solution: Add identity decorator after imports
'''
_MARKER_ = ")  # end of from torch.testing imports"

_REPLACEMENT_ = """)
# onlyAccelerator for XPU - identity pass-through since it's not exported
onlyAccelerator = lambda func: func
"""

# Apply fix
content = content.replace(_MARKER_, _REPLACEMENT_)
```

**Type 3: Broken Import Statements**
```bash
# Find and fix broken multi-line imports
# Common pattern: missing items with onlyAccelerator excised

# Find broken patterns
grep -n "onlyNativeDeviceTypes,,\|""," /tmp/target.py | head -10

# Fix with precise edit
edit file.py --oldString "get_all_device_types,\n\nfrom hypothesis" --newString "get_all_device_types\n\nfrom hypothesis"
```

**Type 4: Import Adaptation for XPU-Specific Tests**
```python
# Problem: # devices may be CUDA-specific
# Solution: Add XPU equivalents or use identity pass-through
'''
# Original
@onlyCUDA

# Adapted for XPU
@onlyXPU  # or use @deviceDecorator("xpu") if available
'''
```

### Phase 5: Verify and Add Instantiations

#### Step 5.1: Verify Syntax with AST
```bash
python3 -c "
import ast
try:
    with open('third_party/torch-xpu-ops/test/xpu/test_nn_xpu.py') as f:
        ast.parse(f.read())
    print('Syntax: VALID')
except SyntaxError as e:
    print(f'Syntax Error at line {e.lineno}: {e.msg}')
    with open('third_party/torch-xpu-ops/test/xpu/test_nn_xpu.py') as f:
        lines = f.readlines()
    if e.lineno:
        print(f'Problem line: {lines[e.lineno-1].strip()[:80]}')
"
```

#### Step 5.2: Add XPU Test Instantiation
```python
# Find and update instantiation block
# Look for: instantiate_device_type_tests(TestNNDeviceType, globals(), allow_mps=True)

_OLD_INSTANTIATE_ = '''instantiate_device_type_tests(
    TestNNDeviceType, globals(), allow_mps=True
)
instantiate_device_type_tests(TestNN, globals(), except_for=device_list)'''

_NEW_INSTANTIATE_ = '''instantiate_device_type_tests(
    TestNNDeviceType, globals(), allow_mps=True, allow_xpu=True
)
instantiate_device_type_tests(TestNN, globals(), allow_xpu=True)'''

content = content.replace(_OLD_INSTANTIATE_, _NEW_INSTANTIATE_)
```

### Phase 6: Test Collection and Validation

#### Step 6.1: Collect Tests (Run from /tmp to avoid local torch)
```bash
export PATH="/home/daisydeng/miniforge3/bin:$PATH"
eval "$(/home/daisydeng/miniforge3/bin/conda shell.bash hook)"
conda activate pytorch_opencode_env
cd /tmp

# Collect tests
python3 -m pytest /home/daisydeng/daisy_pytorch/third_party/torch-xpu-ops/test/xpu/test_nn_xpu.py --collect-only 2>&1 | head -50

# Check for errors
python3 -m pytest /home/daisydeng/daisy_pytorch/third_party/torch-xpu-ops/test/xpu/test_nn_xpu.py --collect-only 2>&1 | grep -E "ERROR|error|FAILED"
```

#### Step 6.2: Diagnose Collection Errors
```bash
# Parse errors with explore agent
Task(tool="explore", description="Diagnose test collection errors",
prompt="""Diagnose the pytest collection error from:
PATH: /home/daisydeng/daisy_pytorch/third_party/torch-xpu-ops/test/xpu/test_nn_xpu.py

Look for:
1. Missing import names
2. Syntax errors in import statements
3. Broken function/class definitions
4. Duplicate definitions
5. Indentation issues
6. String/multi-line string problems

Review the file structure and provide specific fixes needed.""",
subagent_type="explore")
```

### Phase 7: Run Tests and Record Results

#### Step 7.1: Run Specific Test Class
```bash
export PATH="/home/daisydeng/miniforge3/bin:$PATH"
eval "$(/home/daisydeng/miniforge3/bin/conda shell.bash hook)"
conda activate pytorch_opencode_env
cd /tmp

export PYTEST_ADDOPTS="-n 1 --timeout=60"

# Run TestNN class tests only
timeout 600 python3 -m pytest \
    /home/daisydeng/daisy_pytorch/third_party/torch-xpu-ops/test/xpu/test_nn_xpu.py::TestNN \
    -v --tb=short 2>&1 > /tmp/test_results.txt

# Extract summary
tail -30 /tmp/test_results.txt

# Extract failures
grep "FAILED" /tmp/test_results.txt | head -20

# Show pass/fail counts
grep -E "^=.*=.*$" /tmp/test_results.txt | tail -3
```

#### Step 7.2: Record Results
```bash
# Copy results to workspace
cp /tmp/test_results.txt /home/daisydeng/daisy_pytorch/test_nn_xpu_failures.txt
```

### Phase 8: Check Known Issues

#### Step 8.1: Search for Specific Test Failures in GitHub
```bash
# For each failed test, search exactly:
FAILED_TESTS="test_batchnorm_2D_train_NCHW_vs_cpu_mixed_bfloat16"

for test in $FAILED_TESTS; do
    echo "=== Searching: $test ==="
    curl -s "https://api.github.com/search/issues?q=repo:intel/torch-xpu-ops+$(echo $test | sed 's/ /+/g')+is:issue+state:open&per_page=5" | \
    python3 -c "
import sys, json
try:
    data = json.load(sys.stdin)
    if data.get('total_count', 0) > 0:
        for item in data['items']:
            print(f'  FOUND #{item[\"number\"]}: {item[\"title\"]}')
    else:
        print('  No open issue found')
except: pass
"
done
```

#### Step 8.2: Check Skip Lists
```bash
# Check if known issues are in skip lists
grep -r "test_batchnorm\|test_cosine\|test_upsampling\|test_interpolate" \
    /home/daisydeng/daisy_pytorch/third_party/torch-xpu-ops/test/xpu/skip_list*.py 2>/dev/null
```

### Phase 9: Document Results

#### Step 9.1: Save Failure Analysis
```bash
# Create summary document
cat > /home/daisydeng/daisy_pytorch/test_porting_summary.md << 'EOF'
# Test Porting Summary

## Source Information
- PR Branch: daisyden/test_nn_stage1
- Source File: test/test_nn.py
- Target File: third_party/torch-xpu-ops/test/xpu/test_nn_xpu.py
- Date: $(date +%Y-%m-%d)

## Tests Ported
- TestNN (NNTestCase base class with all parametrized tests)

## Issues Encountered
1. ACCELERATOR_TYPE - Not in installed PyTorch (added local definition)
2. onlyAccelerator - Not exported (added identity function)
3. Import continuation issues (fixed manually)

## Test Results
- 627 passed
- 7 failed (listed below)
- 422 skipped
- 3 xfailed

## Failed Tests (Not in Known Issues)
| Test Name | Issue Search |
|-----------|--------------|
| test_batchnorm_2D_train_NCHW_vs_cpu_mixed_bfloat16_xpu_bfloat16 | Not found |
| test_cosine_similarity_mixed_precision_xpu | Not found |
| test_interpolate_buffer_overflow_xpu | Not found |
| test_upsampling_bfloat16_xpu | Not found |

## Recommendations
1. Create GitHub issues for untracked failures
2. Consider skipping known precision-sensitive tests
3. Monitor XPU kernel improvements for bfloat16 support

EOF
```

## Common Issues and Fixes Reference

| Issue | Symptom | Fix |
|-------|---------|-----|
| ACCELERATOR_TYPE missing | `ImportError: cannot import name 'ACCELERATOR_TYPE'` | Add local `_get_accelerator_type()` function |
| onlyAccelerator missing | `NameError: name 'onlyAccelerator' is not defined` | Add `onlyAccelerator = lambda func: func` |
| onlyXPU available | `NameError: name 'onlyXPU' is not defined` | Import `onlyXPU` from common_device_type |
| Broken import continuation | `SyntaxError: trailing comma` | Remove orphan commas or add missing items |
| Device type mismatch | `NameError: name 'device' is not defined` | Use proper device parameter or globals() |
| Tensor type not available | `AssertionError: Tensor-likes are not close` | Adjust precision tolerances |

## Validation Checklist

- [ ] Source class boundaries correctly identified
- [ ] All imports analyzed for compatibility
- [ ] Missing imports have local workarounds
- [ ] Python syntax validated with AST
- [ ] Test collection succeeds (0 errors)
- [ ] XPU instantiation added
- [ ] Tests run to completion
- [ ] Failures recorded and analyzed
- [ ] Known issues searched in GitHub
- [ ] Results documented

## Generic Application

This procedure applies to ANY test class porting:

1. **Identify Source**: Locate test class in PyTorch PR branch
2. **Analyze with Explore Agent**: Get deep understanding of structure
3. **Find Target Insertion Point**: Understand existing target structure
4. **Extract with Verification**: Use sed/awk for precise extraction
5. **Adapt Imports**: For each missing import, apply appropriate workaround
6. **Add XPU Support**: Include `allow_xpu=True` in instantiation
7. **Verify**: Test collection before running
8. **Run and Document**: Execute tests and record failures
9. **Check Known Issues**: Search GitHub with exact test names