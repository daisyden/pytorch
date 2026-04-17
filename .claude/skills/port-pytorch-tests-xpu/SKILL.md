---
name: port-pytorch-tests-xpu
description: Port PyTorch unit tests to torch-xpu-ops for XPU backend coverage. Use when copying/migrating tests from pytorch/test to third_party/torch-xpu-ops/test/xpu. Covers two approaches: direct copy and hook override. Follows agent-guidelines for atomic commits and semantic analysis.
---

# Port PyTorch Tests to torch-xpu-ops

This skill guides the porting of PyTorch unit tests to torch-xpu-ops repo for XPU backend coverage.

## Related Skills
- **agent-guidelines** (REQUIRED): Must be loaded first for behavior rules
- **at-dispatch-v2**: For C++ kernel type dispatch work
- **add-uint-support**: For unsigned integer type additions

## When to Use This Skill

Use when:
- Adding XPU test coverage for pytorch operators
- Migrating CUDA-only tests to XPU
- Creating XPU counterparts for existing pytorch tests
- Fixing missing tests identified in CI/test collection

## Key Concepts

### torch-xpu-ops Test Structure

```
third_party/torch-xpu-ops/
├── test/xpu/
│   ├── test_meta_xpu.py           # XPU meta dispatch tests
│   ├── test_transformers_xpu.py   # XPU transformer tests
│   ├── test_modules_xpu.py        # XPU module tests
│   └── xpu_test_utils.py          # Test utilities & XPUPatchForImport
```

### Porting Approaches

There are **two porting approaches** depending on test complexity:

#### Approach 1: Direct Copy (NO XPUPatchForImport)

Used when copying entire test file with minimal modifications.

**Examples:** `test_transformers_xpu.py`, `test_meta_xpu.py`

**Characteristics:**
- Standalone test file copied from pytorch/test
- No XPUPatchForImport usage
- Imports pytorch test code directly
- Modifies instantiations for XPU enablement

**Pattern:**
```python
# Example instantiation modification
instantiate_device_type_tests(
    TestSDPACudaOnly, globals(), only_for=("cuda", "xpu"), allow_xpu=True
)
```

#### Approach 2: Hook Override (WITH XPUPatchForImport)

Used when selectively overriding specific tests/methods without copying entire files.

**Examples:** `test_modules_xpu.py`

**Characteristics:**
- Uses `XPUPatchForImport(False)` context
- Imports only for method override access
- Overrides specific tests/methods after import

**Pattern:**
```python
from xpu_test_utils import XPUPatchForImport

with XPUPatchForImport(False):
    from test_modules import TestModule

# Override specific methods
TestModule._test_gradients_helper = _gradients_helper
TestModule.test_multiple_device_transfer = _test_multiple_device_transfer
```

## Workflow Steps

### Step 1: Find CUDA Test and Map to XPU

1. Locate the CUDA test in pytorch repo:
   ```bash
   grep "test_name" test/*.py  # Find source test
   ```

2. Map CUDA test naming to XPU by changing suffix:
   ```
   test_name_cuda -> test_name_xpu
   ```

3. Check if XPU version already exists:
   ```bash
   grep "test_name_xpu" third_party/torch-xpu-ops/test/xpu/*.py
   ```

### Step 2: Update/Create XPU Test in torch-xpu-ops

If test exists in `third_party/torch-xpu-ops/test/xpu/`:
- Edit the existing test file directly
- Keep changes localized to torch-xpu-ops repo

If test does not exist:
- Copy from pytorch/test to torch-xpu-ops/test/xpu/
- Add `_xpu` suffix to file and class names
- Use appropriate porting approach (see Porting Approaches)

### Step 3: Run Test in Conda Environment

```bash
# Activate pytorch conda environment
source ~/miniforge3/etc/profile.d/conda.sh
conda activate pytorch_opencode_env

# Run from /tmp to avoid local import conflicts
cd /tmp

# Set PYTHONPATH for test discovery
export PYTHONPATH=$HOME/daisy_pytorch/test/functorch:/tmp

# Copy test file to /tmp for isolated execution
cp $HOME/daisy_pytorch/third_party/torch-xpu-ops/test/xpu/functorch/test_vmap_xpu.py /tmp/

# Run specific test
python -m pytest test_vmap_xpu.py -k "test_name_xpu" -v --tb=short
```

### Step 4: Analyze Why Test Does Not Run

Common reasons for tests not discovering or running:

1. **Missing backend in platform list** - Add missing backends
2. **OpInfo dtypesIf condition** - Patch dtypesIf for XPU (see Step 5)
3. **Device restriction decorators** - Extend to include XPU
4. **Skip decorators** - Adjust for XPU-specific conditions
5. **Platform check failures** - Review `PLATFORM_*` variables

### Step 5: Enable Test with Appropriate Solution

#### Solution A: Add Backend to Platform List

```python
# Add missing backend for XPU
if TEST_XPU and SDPBackend.MISSING not in PLATFORM_SPECIFIC_SDPA:
    PLATFORM_SPECIFIC_SDPA.append(SDPBackend.MISSING)
```

#### Solution B: Patch OpInfo dtypesIfCUDA

When `SM53OrLater` or similar conditions exclude XPU:

```python
# Patch at module top
bf16 = torch.bfloat16
_ops = ["op1", "op2"]
for _op in op_db:
    if _op.name in _ops:
        for _dtype_list in [_op.dtypesIfCUDA, _op.dtypesIfXPU, _op.dtypesIf.get("xpu")]:
            if _dtype_list is not None and bf16 not in _dtype_list:
                _dtype_list.add(bf16)
```

**Constraint:** Mutate in place with `.add()`, never use direct assignment for OpInfo dtypes.

#### Solution C: Update Device-Specific Checks

Ensure CUDA-specific checks include device parameter:
```python
# Before (CUDA only)
if backend == SDPBackend.X and randomness == "different":

# After (CUDA + XPU)
if backend == SDPBackend.X and randomness == "different" and device == "cuda":
```

### Step 6: Check Intel torch-xpu-ops Issues for Known Failures

If test fails after enabling:

1. Search intel/torch-xpu-ops GitHub issues:
   - https://github.com/intel/torch-xpu-ops/issues

2. Check for similar documented issues:
   - Keyword: test name or error pattern
   - Label: `xpu`, `backend`, specific operator

3. If known issue found:
   - Document issue URL in commit/PR
   - Enable test anyway (will fail with tracked limitation)
   - Do NOT add skip unless directed

### Step 7: Verify and Commit

```bash
# Verify test discovers
python -m pytest test_vmap_xpu.py --collect-only 2>&1 | grep "test_name"

# Run and verify behavior
python -m pytest test_vmap_xpu.py -k "test_name_xpu" -v --tb=short

# Review diff
git diff third_party/torch-xpu-ops/test/xpu/

# Commit with descriptive message
git add third_party/torch-xpu-ops/test/xpu/test_name_xpu.py
git commit -m "XPU: Enable test_name_xpu tests

Enables XPU coverage for:
- test_name_xpu_variant1
- test_name_xpu_variant2

Solution: [rief description of change]
Reference: https://github.com/intel/torch-xpu-ops/issues/XXXX
"
```

## Key Files Reference

### xpu_test_utils.py Utilities

| Component | Purpose |
|-----------|---------|
| `XPUPatchForImport` | Main class for importing pytorch tests to XPU context |
| `dtypesIfXPUMock` | Mocks dtypesIfCUDA -> dtypesIfXPU translation |
| `align_supported_dtypes` | Aligns op_db dtypes for XPU hardware capabilities |
| `onlyXPU` | XPU device restriction decorator |
| `skipXPU` | XPU skip decorator |

## Dtype Alignment: Core Pattern

**Critical Discovery:** CUDA tests use `@dtypesIfCUDA` decorators for GPU-specific dtypes. XPU tests MUST have equivalent `@dtypesIfXPU` decorators to enable the same dtype coverage.

### The Pattern

Each test with `@dtypesIfCUDA(X, Y, Z)` needs a corresponding `@dtypesIfXPU(X, Y, Z)`:

```python
# CUDA source test_nn.py:
@dtypesIfCUDA(torch.half, torch.float)
def test_softmax_results(self, device, dtype):
    ...

# XPU aligned test_nn_xpu.py:
from torch.testing._internal.common_device_type import (
    dtypes,
    dtypesIfCUDA,
    dtypesIfXPU,  # MUST import dtypesIfXPU
    instantiate_device_type_tests,
    # ... other imports
)

@dtypesIfCUDA(torch.half, torch.float)
@dtypesIfXPU(torch.half, torch.float)  # ALIGN WITH CUDA!
def test_softmax_results(self, device, dtype):
    ...
```

### Step-by-Step Dtype Alignment

1. **Find CUDA dtypesIfCUDA decorator** for the test:

   ```bash
   grep -B1 "def test_name" test/test_nn.py | grep "@dtypesIfCUDA"
   # Example output: @dtypesIfCUDA(torch.half, torch.float, torch.double)
   ```

2. **Check XPU file** for existing `@dtypesIfXPU`:

   ```bash
   grep -A1 "@dtypesIfXPU" third_party/torch-xpu-ops/test/xpu/test_nn_xpu.py
   ```

3. **Add/update @dtypesIfXPU** to match @dtypesIfCUDA exactly:

   ```python
   # Before (WRONG - missing dtypes):
   @dtypesIfCUDA(torch.half, torch.float, torch.double)
   @dtypesIfXPU(torch.half)  # WRONG!

   # After (CORRECT - aligned with CUDA):
   @dtypesIfCUDA(torch.half, torch.float, torch.double)
   @dtypesIfXPU(torch.half, torch.float, torch.double)  # CORRECT!
   ```

4. **Verify import** - ensure `dtypesIfXPU` is imported:

   ```python
   from torch.testing._internal.common_device_type import (
       dtypes,
       dtypesIfCUDA,
       dtypesIfXPU,  # Add this import
       # ... other imports
   )
   ```

5. **Collect tests** to verify dtype variants appear:

   ```bash
   python -m pytest test_nn_xpu.py -k "test_name_xpu" --collect-only 2>&1 | grep "Function"
   # Should see: test_name_xpu_float16, test_name_xpu_float32, etc.
   ```

### Common Dtype Gaps Found

| CUDA dtypesIfCUDA | XPU Required dtypesIfXPU |
|-------------------|-------------------------|
| `torch.half` | `torch.half` (float16) |
| `torch.float` | `torch.float` (float32) |
| `torch.double` | `torch.double` (float64) |
| `torch.bfloat16` | `torch.bfloat16` (bfloat16) |
| `torch.complex128` | `torch.complex128` |
| Combinations | Match exactly |

### Validation Checklist

After adding @dtypesIfXPU decorators:

- [ ] `dtypesIfXPU` imported in test file header
- [ ] Each `@dtypesIfCUDA(A, B, C)` has corresponding `@dtypesIfXPU(A, B, C)`
- [ ] Test collection shows expected dtype variants: `test_name_xpu_float16`, `test_name_xpu_bfloat16`, etc.
- [ ] Tests pass (or fail with documented known limitations)

### XPUPatchForImport Initialization

```python
class XPUPatchForImport:
    def __init__(self, patch_test_case=True) -> None:
        test_dir = os.path.join(
            os.path.dirname(os.path.abspath(__file__)), "../../../../test"
        )
        # Patches: onlyCUDA, dtypesIfCUDA, onlyNativeDeviceTypes, etc.
```

### Method `_enter__` Key Patches

```python
# Key patches in __enter__
common_device_type.onlyCUDA = common_device_type.onlyXPU
common_device_type.skipXPU = _skipXPU
common_device_type.dtypesIfCUDA = get_dtypesIf_mock("cuda")
common_device_type.dtypesIfXPU = get_dtypesIf_mock("xpu")
```

## Common Issues & Solutions

### Issue 1: Test Not Discovered

**Symptom:** Test exists in pytorch but not in torch-xpu-ops collection

**Check:**
1. Check if test exists in torch-xpu-ops test directory
2. OpInfo skip decorators or device restrictions
3. dtypesIf condition not including XPU

**Fix:** Apply Steps 2-5 above

### Issue 2: dtypesIf Setter Error

**Symptom:** `AssertionError: Expected _dispatch_dtypes or None`

**Cause:** Direct assignment bypasses OpInfo property setter

**Fix:** Mutate in place using `.add()`:
```python
_dtype_list.add(torch.bfloat16)
```

### Issue 3: Backend Not Supported on XPU (Known Limitation)

**Symptom:** Test fails with unknown backend or no viable backend error

**Example:** `RuntimeError: No viable backend for scaled_dot_product_attention was found`

**Root cause:** XPU lacks cuDNN hardware/software for certain backends

**Decision:**
- If backend is known unsupported on XPU → enable test anyway (tracks limitation)
- Check intel/torch-xpu-ops issues for documented cases

**Implementation:**
```python
# Force add unsupported backend to platform list
if TEST_XPU and SDPBackend.UNSUPPORTED not in PLATFORM_LIST:
    PLATFORM_LIST.append(SDPBackend.UNSUPPORTED)

# Remove skip that blocks XPU testing:
# if device == "xpu" and backend == SDPBackend.UNSUPPORTED:
#     raise unittest.SkipTest("...")  # REMOVE this
```

## Template Outline

### For Direct Copy Test

```python
# third_party/torch-xpu-ops/test/xpu/test_name_xpu.py
"""
XPU specific tests for test_name.
Modified from pytorch/test/test_name.py
"""

import torch
from torch.testing._internal.common_utils import (
    TestCase, run_tests, instantiate_device_type_tests
)
from xpu_test_utils import XPUPatchForImport

# Patch op_db if needed for dtype coverage
for _op in op_db:
    if _op.name in ["op1", "op2"]:
        for _dtype_list in [_op.dtypesIfCUDA, _op.dtypesIfXPU, _op.dtypesIf.get("xpu")]:
            if _dtype_list is not None and torch.bfloat16 not in _dtype_list:
                _dtype_list.add(torch.bfloat16)

# Import source test
with XPUPatchForImport(False):
    from test_name import TestNameClass

# XPU instantiation
instantiate_device_type_tests(
    TestNameClass, globals(), only_for=("cuda", "xpu"), allow_xpu=True
)

if __name__ == "__main__":
    run_tests()
```

### For Hook Override Test

```python
# third_party/torch-xpu-ops/test/xpu/test_name_xpu.py
"""
XPU specific test overrides for test_name.
Uses XPUPatchForImport for selective overrides.
"""

import torch
from xpu_test_utils import XPUPatchForImport

with XPUPatchForImport(False):
    from test_name import TestNameClass

def _xpu_test_method(self, device):
    # XPU implementation
    ...

TestNameClass.test_method = _xpu_test_method
```

## Case Study: NestedTensor SDPA Tests

### Background

Tests like `test_sdpa_with_packed_in_proj` use nested tensors with SDPA. These tests may have CUDA-specific conditions:

```python
# NestedTensor SDPA test has PLATFORM check
@unittest.skipIf(
    not PLATFORM_SUPPORTS_FUSED_ATTENTION,
    "Platform doesn't support flash or mem-efficient attention",
)
```

### Challenge

XPU may lack support for certain features even when dtype alignment is correct. This results in tests that:
1. **Discover successfully** (dtype variants found)
2. **Fail at runtime** with "No viable backend" error

### Approach: Enable-But-Track-Fail

For XPU limitations (not bugs):

1. **Enable the test** - don't add skip just because XPU fails
2. **Document the known limitation** - reference intel/torch-xpu-ops issue
3. **Track the gap** - let CI show where kernel support is missing

```python
# Example: SDPA with packed in_proj on XPU has known limitation
@skipXPUIf(
    True,  # Always skip due to known limitation
    "XPU nestedtensor SDPA requires fused attention kernel support"
)
# OR use @expectedFailureXPU decorator
```

### Decision Matrix

| Situation | Action |
|-----------|--------|
| Type bug (wrong logic) | Fix the test implementation |
| Missing dtype coverage | Add @dtypesIfXPU to align with CUDA |
| Missing kernel on XPU | Enable anyway, document limitation, reference issue |
| XPU根本没实现功能 | Add skipXPUIf with clear message |

## Checklist Before Commit

- [ ] CUDA test located and mapped to XPU naming (_cuda -> _xpu)
- [ ] XPU test updated/created in third_party/torch-xpu-ops/test/xpu/
- [ ] `dtypesIfXPU` import verified
- [ ] @dtypesIfXPU aligned with CUDA @dtypesIfCUDA
- [ ] Test runs in pytorch_opencode_env conda environment
- [ ] Test discovery verified (--collect-only)
- [ ] Failure analyzed: Is it missing dtypes, XPU limitation, or a bug?
- [ ] Intel torch-xpu-ops issues checked for similar cases
- [ ] Solution implemented and test re-run
- [ ] Known limitations documented with issue references
- [ ] Atomic commit created
- [ ] PR description prepared (if upstream contribution)

## Boundaries

**This skill covers:**
- Porting CUDA tests to XPU by mapping _cuda suffix to _xpu
- Editing existing tests in third_party/torch-xpu-ops/test/xpu/
- Running tests in pytorch_opencode_env conda environment
- Analyzing and resolving missing test discovery
- Aligning XPU @dtypesIfXPU decorators with CUDA @dtypesIfCUDA pattern
- Enabling tests with known limitations tracked via intel/torch-xpu-ops issues
- Restarting unsupported backends (like CUDNN_ATTENTION) for tracking
- Handling nestedtensor/SDPA tests with XPU-specific constraints

**This skill does NOT cover:**
- C++ kernel implementation
- Operator registration
- Build system changes
- Documentation writing
- Pattern-matching specific test files (each case is unique)
