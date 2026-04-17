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

### Step 1: Identify Missing Test

Determine if a test is missing for XPU:

1. Check pytorch test source:
   ```bash
   grep "test_name" pytorch/test/*.py
   ```

2. Check if XPU counterpart exists:
   ```bash
   grep "test_name" third_party/torch-xpu-ops/test/xpu/*.py
   ```

3. Run test collection to discover:
   ```bash
   python -m pytest test/xpu/test_meta_xpu.py --collect-only 2>&1 | grep -i "test_name"
   ```

### Step 2: Analyze Source Test

Read and understand the pytorch source test:

1. **File location**: Find pytorch test in `test/test_*.py`
2. **Test class**: Identify class (e.g., `TestMetaCUDA`, `TestSDPACudaOnly`)
3. **Test method**: Identify specific test method
4. **Dependencies**: Check imports, fixtures, utilities used
5. **Device dependency**: Note if uses `@onlyCUDA`, `device='cuda'`, CUDA-specific checks

### Step 3: Determine Porting Approach

Choose based on device-specific logic:

**Direct Copy if:**
- Test is device-agnostic
- Uses standard `device` parameter
- No CUDA-specific code patterns
- Test pattern applies universally to XPU

**Hook Override if:**
- Test has CUDA-specific logic that needs XPU override
- Limited number of tests require modification
- Want to maintain single source of truth
- Can generalize CUDA-specific to XPU

### Step 4: Locate/Determine Implementation

**For Approach 1 (Direct Copy):**

1. Copy pytorch test to xpu folder with `_xpu` suffix
2. Add XPU-specific imports:
   ```python
   import torch.xpu  # For XPU device checks
   ```
3. Modify instantiations:
   ```python
   only_for=("cuda", "xpu")  # Extend CUDA-only tests
   allow_xpu=True            # Enable XPU instantiation
   ```
4. Check for XPU-specific adaptations needed

**For Approach 2 (Hook Override):**

1. Use existing or create xpu test file
2. Import source test via XPUPatchForImport:
   ```python
   from xpu_test_utils import XPUPatchForImport

   with XPUPatchForImport(False):
       from test_modules import TestModule
   ```
3. Define XPU-specific override functions
4. Assign overrides to TestClass:
   ```python
   TestModule.method_name = _xpu_method_name
   ```

### Step 5: Handle Complex Cases

#### Handling dtypesIfCUDA for OpInfo

When OpInfo has CUDA-only dtype conditions:

**Problem:** OpInfo often has `SM53OrLater` checks:
```python
# common_methods_invocations.py
dtypesIfCUDA=floating_and_complex_types_and(torch.float16,
    *[torch.bfloat16] if SM53OrLater else [])
```
On XPU, `SM53OrLater` evaluates to False, missing bfloat16.

**Solution:** Patch OpInfo dtypes directly:
```python
# At module top in test file
for _op in op_db:
    if _op.name == "addbmm":
        # Update all dtype lists
        for _dtype_list in [_op.dtypesIfCUDA, _op.dtypesIfXPU, _op.dtypesIf.get("xpu")]:
            if _dtype_list is not None and torch.bfloat16 not in _dtype_list:
                _dtype_list.add(torch.bfloat16)
        break
```

**Critical Constraint:** Do NOT use direct assignment for dtypesIf:
```python
# WRONG - bypasses setter validation
_op.dtypesIfCUDA = frozenset(...)

# CORRECT - mutates in place
_dtype_list = set(_op.dtypesIfCUDA)
_dtype_list.add(torch.bfloat16)
_op.dtypesIfCUDA = _dtype_list  # Still fails - setter expects original type
```

**Best Practice:** Mutate existing set in place using `.add()`:
```python
_dtype_list.add(bf16)  # In-place modification
```

#### Handling onlyCUDA Decorator

Convert CUDA-only to XPU:
```python
# XPUPatchForImport monkey-patches onlyCUDA -> onlyXPU
# So tests with @onlyCUDA automatically run on XPU
```

If explicitly needed:
```python
# Import the onlyXPU variant
from xpu_test_utils import onlyOn
```

### Step 6: Verify Test Discovery

Run test collection:
```bash
python -m pytest test/xpu/test_meta_xpu.py --collect-only 2>&1 | grep -i "test_name.*bfloat16"
```

Verify specific test:
```bash
python -m pytest test/xpu/test_meta_xpu.py -k "test_dispatch_meta_inplace_addbmm_xpu_bfloat16" -v
```

### Step 7: Run Individual Test

Execute the ported test:
```bash
python -m pytest test/xpu/test_meta_xpu.py::TestMetaXPU::test_name -v --timeout=120
```

Verify PASS/FAIL status.

### Step 8: Commit Changes

Follow agent-guidelines for atomic commits:

```bash
# Show diff
git diff test/xpu/test_meta_xpu.py

# Stage and commit
git add test/xpu/test_meta_xpu.py
git commit -m "XPU: Add bfloat16 dtype support for addbmm operator tests

Enables previously missing XPU tests for addbmm with bfloat16:
- test_dispatch_meta_inplace_addbmm_xpu_bfloat16
- test_dispatch_meta_outplace_addbmm_xpu_bfloat16

Root cause: OpInfo dtypesIfCUDA has SM53OrLater check excluding bfloat16 on XPU
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
1. OpInfo skip decorators
2. Device restriction (`only_for="cuda"` missing `xpu`)
3. dtypesIf condition not including XPU

**Fix:** Apply appropriate porting approach

### Issue 2: bfloat16 Tests Missing

**Symptom:** Tests for dtype `bfloat16` not generating

**Root cause:** `SM53OrLater` condition in OpInfo dtypesIfCUDA

**Fix:** Patch OpInfo dtypes directly at module load time:
```python
# See Step 5: Handling dtypesIfCUDA for OpInfo
```

### Issue 3: dtypesIf Setter Error

**Symptom:** `AssertionError: Expected _dispatch_dtypes or None`

**Cause:** Direct assignment bypasses OpInfo property setter

**Fix:** Mutate in place:
```python
_dtype_list.add(torch.bfloat16)
```

### Issue 4: XPUPatchForImport Scope

**Symptom:** Changes not taking effect

**Cause:** Import ordering or scope issues

**Fix:** Ensure patch applied early, before test decorators evaluated

### Issue 5: CUDNN_ATTENTION Backend Not Supported on XPU

**Symptom:** SDPA backend3 tests fail with:
```
RuntimeError: No viable backend for scaled_dot_product_attention was found
```

**Root cause:** XPU lacks cuDNN hardware/software support

**Decision:**
- If backend is known unsupported on XPU → enable test anyway (will fail with documented issue)
- Track failures for future kernel implementation milestones

**Implementation:**
```python
# Add CUDNN_ATTENTION to platform-specific backends
if TEST_XPU and SDPBackend.CUDNN_ATTENTION not in PLATFORM_SPECIFIC_SDPA:
    PLATFORM_SPECIFIC_SDPA.append(SDPBackend.CUDNN_ATTENTION)

# DO NOT add skip for CUDNN_ATTENTION on XPU - let it fail
# Only skip for OTHER device-specific reasons:
if device == "cuda" and backend == SDPBackend.CUDNN_ATTENTION and condition:
    raise unittest.SkipTest("...")  # CUDA-specific skip only
```

## Handling CUDNN_ATTENTION on XPU (Known Limitation)

For SDPA/CUDNN_ATTENTION tests, XPU lacks cuDNN hardware/software support.
When enabling such tests:

### Test Execution Environment

```bash
# Activate pytorch environment (conda)
source ~/miniforge3/etc/profile.d/conda.sh
conda activate pytorch_opencode_env

# Run from /tmp to avoid local functorch import conflicts
cd /tmp

# Set PYTHONPATH for test discovery
export PYTHONPATH=/home/daisydeng/daisy_pytorch/test/functorch:/tmp

# Run backend3 tests
python -m pytest test_vmap_xpu.py -k "backend3 and (test_randomness or test_sdpa)" -v --tb=short
```

### Test Naming Convention

XPU tests use `_xpu` suffix instead of `_cuda`:
```bash
# Wrong (CUDA naming)
test_randomness_backend3_randomness_error_cuda

# Correct (XPU naming)
test_randomness_backend3_randomness_error_xpu
```

### Expected Behavior

After enabling CUDNN_ATTENTION on XPU:
1. **Test discovers** - Test found in collection
2. **Test runs** - Executes but fails with known issue
3. **Error message:** `RuntimeError: No viable backend for scaled_dot_product_attention was found`
4. **This is EXPECTED** - Document in PR/commit message

### Commit Pattern for Known Failing Tests

```bash
git commit -m "XPU: Enable CUDNN_ATTENTION backend3 SDPA tests (will fail with known issue)

Enables previously skipped SDPA backend3 tests for XPU tracking:
- test_randomness_backend3_randomness_different_xpu
- test_randomness_backend3_randomness_error_xpu
- test_randomness_backend3_randomness_same_xpu
- test_sdpa_backend3_xpu

Tests fail with: No viable backend for scaled_dot_product_attention
See: https://github.com/intel/torch-xpu-ops/issues/3229
"
```

## Preconditions for SDPA Test Porting

1. **Environment Setup:**
   - Working conda env: `pytorch_opencode_env`
   - Intel oneAPI: `/opt/intel/oneapi/compiler/2025.0/`
   - PyTorch with XPU support

2. **torch-xpu-ops Structure:**
   ```
   third_party/torch-xpu-ops/test/xpu/functorch/
   ├── test_vmap_xpu.py           # Contains SDPA tests
   └── PLATFORM_SPECIFIC_SDPA     # Backend list
   ```

3. **Key Variables:**
   - `PLATFORM_SPECIFIC_SDPA`: List of enabled backends
   - `TEST_XPU`: Boolean, XPU is available
   - `SDPBackend.CUDNN_ATTENTION`: Backend enum value 3

## Template Outline

### For Direct Copy Test

```python
# third_party/torch-xpu-ops/test/xpu/test_{{name}}_xpu.py
"""
XPU specific tests for {{name}}
Migrated from pytorch/test/test_{{name}}.py
"""

# Required imports
import torch
import itertools

# Import source test utilities
from torch.testing._internal.common_utils import (
    TestCase, run_tests, parametrize, instantiate_device_type_tests
)

# XPU-specific imports
from xpu_test_utils import XPUPatchForImport

# Patch op_db if needed for dtype coverage
for _op in op_db:
    if _op.name == "{{op_name}}":
        for _dtype_list in [_op.dtypesIfCUDA, _op.dtypesIfXPU, _op.dtypesIf.get("xpu")]:
            if _dtype_list is not None and torch.bfloat16 not in _dtype_list:
                _dtype_list.add(torch.bfloat16)
        break

# Import source test
with XPUPatchForImport(False):
    from test_{{name}} import Test{{Name}}Class

# Only if method overrides needed:
# Test{{Name}}Class.test_method = _xpu_test_method

# XPU instantiation
instantiate_device_type_tests(
    Test{{Name}}Class, globals(), only_for="xpu", allow_xpu=True
)

if __name__ == "__main__":
    run_tests()
```

### For Hook Override Test

```python
# third_party/torch-xpu-ops/test/xpu/test_{{name}}_xpu.py
"""
XPU specific test overrides for {{name}}
Uses XPUPatchForImport for selective overrides
"""

import torch
from xpu_test_utils import XPUPatchForImport

# Import source test
with XPUPatchForImport(False):
    from test_{{name}} import Test{{Name}}Class

# Define XPU-specific test override
def _xpu_test_method(self, device):
    # XPU implementation
    ...

# Apply overrides
Test{{Name}}Class.test_method = _xpu_test_method

# Register for collection
# Source test's instantiation already handles XPU discovery
```

## Checklist Before Commit

- [ ] Test identified as missing
- [ ] Source test analyzed
- [ ] Appropriate approach chosen (Direct Copy or Hook Override)
- [ ] Implementation complete
- [ ] Test discovery verified (--collect-only)
- [ ] Test execution passes
- [ ] Atomic commit created
- [ ] PR description prepared (if upstream contribution)

## Boundaries

**This skill covers:**
- Unit test porting from pytorch/test to torch-xpu-ops/test/xpu/
- OpInfo dtypes patching for XPU coverage
- XPUPatchForImport usage patterns
- Test discovery and execution verification
- SDPA backend3 (CUDNN_ATTENTION) XPU test enabling with known limitations

**This skill does NOT cover:**
- C++ kernel implementation
- Operator registration
- Build system changes
- Documentation writing