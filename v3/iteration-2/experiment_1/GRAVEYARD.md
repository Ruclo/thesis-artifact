# GRAVEYARD - Test Generation Lessons Learned

This file documents mistakes made during automated test generation and how to avoid them.
It is read during the exploration phase to prevent repeating past errors.

---

## Entry: Wrong HCO CR field name casing and value type for CommonInstancetypesDeployment

**Category**: ResourceError
**Date**: 2026-02-21
**Test**: tests/install_upgrade_operators/common_instancetypes_deployment/conftest.py::disabled_common_instancetypes_deployment

### What Went Wrong
The generated code used `commonInstancetypesDeployment` (lowercase 'c') as the HCO CR spec field name with string values `"Enabled"` / `"Disabled"`. The actual CRD field is `CommonInstancetypesDeployment` (capital 'C') and its value is an object with a nested boolean `enabled` field, not a string.

### Wrong Code
```python
COMMON_INSTANCETYPES_DEPLOYMENT_KEY = "commonInstancetypesDeployment"
COMMON_INSTANCETYPES_DEPLOYMENT_ENABLED = "Enabled"
COMMON_INSTANCETYPES_DEPLOYMENT_DISABLED = "Disabled"

# Used in fixtures as:
patches={
    hyperconverged_resource: {
        "spec": {COMMON_INSTANCETYPES_DEPLOYMENT_KEY: COMMON_INSTANCETYPES_DEPLOYMENT_DISABLED}
    }
}
```

### Correct Code
```python
COMMON_INSTANCETYPES_DEPLOYMENT_KEY = "CommonInstancetypesDeployment"
COMMON_INSTANCETYPES_DEPLOYMENT_SPEC_ENABLED = {COMMON_INSTANCETYPES_DEPLOYMENT_KEY: {"enabled": True}}
COMMON_INSTANCETYPES_DEPLOYMENT_SPEC_DISABLED = {COMMON_INSTANCETYPES_DEPLOYMENT_KEY: {"enabled": False}}

# Used in fixtures as:
patches={
    hyperconverged_resource: {
        "spec": COMMON_INSTANCETYPES_DEPLOYMENT_SPEC_DISABLED
    }
}
```

### Lesson Learned
HCO CRD field names do not always follow standard camelCase conventions. Some fields like `CommonInstancetypesDeployment` start with a capital letter. Additionally, the field type may be an object with nested properties rather than a simple string enum. Always verify the actual CRD schema before generating code.

### How to Avoid
Before generating code that patches HCO CR spec fields, run `kubectl get crd hyperconvergeds.hco.kubevirt.io -o json` and inspect the `openAPIV3Schema` to confirm: (1) the exact field name casing, (2) the field type (string, boolean, object), and (3) nested property structure. Never assume field names follow standard camelCase or that toggle fields use string enum values.

---

## Entry: Wrong type hints on helper function parameters

**Category**: TypeError
**Date**: 2026-02-21
**Test**: tests/install_upgrade_operators/common_instancetypes_deployment/conftest.py::get_common_cluster_instancetypes

### What Went Wrong
Helper functions that accept a Kubernetes `DynamicClient` parameter were annotated with incorrect types. The `admin_client` parameter was typed as `list[VirtualMachineClusterInstancetype]` (the return type was accidentally used as the parameter type) and `object` instead of `DynamicClient`.

### Wrong Code
```python
def get_common_cluster_instancetypes(admin_client: list[VirtualMachineClusterInstancetype]) -> list[VirtualMachineClusterInstancetype]:
    ...

def wait_for_common_instancetypes_absent(admin_client: object, wait_timeout: int = TIMEOUT_5MIN) -> None:
    ...
```

### Correct Code
```python
from kubernetes.dynamic import DynamicClient

def get_common_cluster_instancetypes(admin_client: DynamicClient) -> list[VirtualMachineClusterInstancetype]:
    ...

def wait_for_common_instancetypes_absent(admin_client: DynamicClient, wait_timeout: int = TIMEOUT_5MIN) -> None:
    ...
```

### Lesson Learned
When writing type hints for Kubernetes client parameters, always use `DynamicClient` from `kubernetes.dynamic`. Do not confuse return types with parameter types, and do not use `object` as a generic stand-in.

### How to Avoid
For any function that takes a Kubernetes client parameter, always import and use `from kubernetes.dynamic import DynamicClient` as the type hint. Follow the pattern established in existing repository fixtures where `admin_client` is consistently typed as `DynamicClient`.

---

## Entry: Class-scoped fixtures from different VMs in same test class cause scheduling failures

**Category**: ResourceError
**Date**: 2026-02-21
**Test**: tests/virt/screenshot/test_vm_screenshot.py::TestVMScreenshotGuestCompatibility::test_screenshot_with_rhel_guest

### What Went Wrong
Two class-scoped VM fixtures (`screenshot_fedora_vm` and `screenshot_rhel_vm`) were used by different tests within the same test class `TestVMScreenshotGuestCompatibility`. Because both fixtures were class-scoped, the Fedora VM remained alive for the entire class lifetime. When the RHEL test ran (still in the same class), the RHEL VM could not be scheduled because the cluster lacked sufficient memory to run both VMs simultaneously. The RHEL VM timed out with `ErrorUnschedulable: Insufficient memory`.

### Wrong Code
```python
@pytest.mark.tier3
class TestVMScreenshotGuestCompatibility:
    def test_screenshot_with_fedora_guest(self, screenshot_fedora_vm, tmp_path):
        ...

    def test_screenshot_with_rhel_guest(self, screenshot_rhel_vm, tmp_path):
        ...
```

### Correct Code
```python
@pytest.mark.tier3
class TestVMScreenshotFedoraGuest:
    def test_screenshot_with_fedora_guest(self, screenshot_fedora_vm, tmp_path):
        ...


@pytest.mark.tier3
class TestVMScreenshotRhelGuest:
    def test_screenshot_with_rhel_guest(self, screenshot_rhel_vm, tmp_path):
        ...
```

### Lesson Learned
When tests use different class-scoped VM fixtures, all fixtures remain alive for the entire class duration. On resource-constrained clusters, this means multiple VMs run simultaneously and can exceed available node memory. Each VM fixture that provisions a distinct VM should be in its own test class to ensure sequential teardown-then-create behavior.

### How to Avoid
Never put tests that use different class-scoped VM fixtures into the same test class. Each distinct VM fixture should be consumed by its own class so that one VM is torn down before the next is created. This prevents cluster scheduling failures due to insufficient resources.

---

## Entry: run_virtctl_command returns False when virtctl writes info logs to stderr

**Category**: AssertionError
**Date**: 2026-02-21
**Test**: tests/virt/node/hard_reset/test_virtctl_reset.py::TestVirtctlReset::test_virtctl_reset_command_succeeds

### What Went Wrong
The `run_virtctl_command` function (which calls `run_command` from `pyhelper_utils.shell`) has `verify_stderr=True` by default. When `virtctl reset` succeeds, it writes an info-level JSON log message to stderr (e.g., `{"component":"portforward","level":"info","msg":"Reset VMI",...}`). Because stderr is non-empty and `verify_stderr=True`, `run_command` returns `(False, stdout, stderr)` even though the command exit code was 0.

### Wrong Code
```python
command_success, output, error = run_virtctl_command(
    command=["reset", vm.vmi.name],
    namespace=vm.namespace,
)
assert command_success, f"virtctl reset command failed: {error}"
```

### Correct Code
```python
command_success, output, error = run_virtctl_command(
    command=["reset", vm.vmi.name],
    namespace=vm.namespace,
    verify_stderr=False,
)
assert command_success, f"virtctl reset command failed: {error}"
```

### Lesson Learned
Many virtctl commands write info-level log output to stderr even on success. The `run_virtctl_command` wrapper inherits `verify_stderr=True` from `run_command`, which treats any stderr output as a failure. For virtctl commands that are known to write logs to stderr, pass `verify_stderr=False` explicitly.

### How to Avoid
When using `run_virtctl_command`, always pass `verify_stderr=False` unless you specifically need to validate that stderr is empty. Virtctl subcommands commonly write structured JSON logs to stderr even on successful execution.

---

## Entry: ocp_resources retry_cluster_exceptions re-raises last_exp not TimeoutExpiredError

**Category**: TypeError
**Date**: 2026-02-21
**Test**: tests/virt/node/hard_reset/test_vmi_reset_negative.py::TestVMIResetOnStoppedVM::test_reset_stopped_vmi

### What Went Wrong
The generated code expected `TimeoutExpiredError` to be raised when calling `vmi.reset()` on a non-existent or stopped VMI. However, `ocp_resources.resource.Resource.retry_cluster_exceptions` catches `TimeoutExpiredError` and re-raises `exp.last_exp` (the original `ApiException`) via `raise exp.last_exp from exp`. So the actual exception that propagates is `kubernetes.client.exceptions.ApiException(404)`, not `TimeoutExpiredError`.

### Wrong Code
```python
from timeout_sampler import TimeoutExpiredError

with pytest.raises(TimeoutExpiredError):
    vmi.reset()
```

### Correct Code
```python
from kubernetes.client.exceptions import ApiException

with pytest.raises(ApiException, match="404"):
    vmi.reset()
```

### Lesson Learned
When `ocp_resources` calls a Kubernetes API that returns a 404 (or other HTTP error), the retry mechanism in `retry_cluster_exceptions` catches the `TimeoutExpiredError` and re-raises the underlying `ApiException` with `raise exp.last_exp from exp`. The exception that reaches the test is always the raw `ApiException`, not `TimeoutExpiredError`.

### How to Avoid
When writing negative tests that expect a Kubernetes API error (404, 409, etc.) from `ocp_resources` methods like `reset()`, `pause()`, etc., always catch `kubernetes.client.exceptions.ApiException` with a `match` on the HTTP status code, NOT `TimeoutExpiredError` or `NotFoundError`. The `ocp_resources` retry layer unwraps the timeout and re-raises the raw API exception.

---

## Entry: Resetting a paused VMI succeeds without error

**Category**: LogicError
**Date**: 2026-02-21
**Test**: tests/virt/node/hard_reset/test_vmi_reset_negative.py::TestVMIResetOnPausedVM::test_reset_paused_vmi

### What Went Wrong
The generated code assumed that calling `vmi.reset()` on a paused VMI would raise an error. In reality, the KubeVirt reset API accepts the reset call on a paused VMI and processes it successfully. The test was written as a negative test expecting `Exception` with match `"paused"`, but no exception was raised.

### Wrong Code
```python
with pytest.raises(Exception, match="(paused|Paused)"):
    paused_vm.vmi.reset()
```

### Correct Code
```python
# Reset on paused VMI succeeds - test as a positive case
paused_vm.vmi.reset()
```

### Lesson Learned
The KubeVirt hard reset API does not reject reset calls on paused VMIs. A hard reset is a hardware-level operation that works regardless of the guest pause state. Do not assume that pause state prevents reset operations.

### How to Avoid
Before writing negative tests that assume an API operation will fail on a paused VMI, verify the actual KubeVirt API behavior. Hard reset is a hardware signal that bypasses guest-level state. Only write negative reset tests for states where the VMI truly does not exist (stopped VM with no running VMI, non-existent VMI name).

---

## Entry: Unprivileged user (namespace admin) already has VMI reset permission

**Category**: LogicError
**Date**: 2026-02-21
**Test**: tests/virt/node/hard_reset/test_vmi_reset_negative.py::TestVMIResetRBAC::test_unprivileged_user_cannot_reset_without_permission

### What Went Wrong
The generated code assumed that the unprivileged user (namespace admin) would NOT have permission to reset VMIs without an explicit `kubevirt.io:edit` RoleBinding. In reality, the namespace-admin role already includes the `virtualmachineinstances/reset` subresource permission. The test expected `ForbiddenError` but no exception was raised.

### Wrong Code
```python
with pytest.raises(ForbiddenError):
    rbac_vm.vmi.reset()
```

### Correct Code
```python
# Namespace admin already has reset permission - no negative RBAC test needed
# Remove or convert to positive test
rbac_vm.vmi.reset()
```

### Lesson Learned
The KubeVirt RBAC model grants `virtualmachineinstances/reset` permission to namespace admin users. The `unprivileged_client` in this test framework is actually a namespace admin (it can create VMs, manage resources in its namespace). A truly unprivileged RBAC test would require creating a user with a more restricted role that explicitly lacks the reset subresource permission.

### How to Avoid
Before writing RBAC negative tests, verify what permissions the `unprivileged_client` fixture actually has. In this test framework, `unprivileged_client` is a namespace admin with broad permissions. For testing RBAC denial of new subresources, you need a custom ServiceAccount or User with a restricted ClusterRole that explicitly excludes the subresource being tested.

---

## Entry: SSH timeout too short after VMI hard reset

**Category**: TimeoutError
**Date**: 2026-02-21
**Test**: tests/virt/node/hard_reset/conftest.py::hard_reset_vm_after_reset

### What Went Wrong
After performing a VMI hard reset, `wait_for_running_vm(vm=hard_reset_vm)` was called with the default `ssh_timeout=TIMEOUT_2MIN` (120 seconds). A hard reset is more disruptive than a normal reboot - the guest OS must cold-boot from scratch, which can take longer for SSH to become available. The SSH connection timed out with `SSHException: Error reading SSH protocol banner` after 120 seconds.

### Wrong Code
```python
hard_reset_vm.vmi.reset()
wait_for_running_vm(vm=hard_reset_vm)
```

### Correct Code
```python
from utilities.constants import TIMEOUT_10MIN

hard_reset_vm.vmi.reset()
wait_for_running_vm(vm=hard_reset_vm, ssh_timeout=TIMEOUT_10MIN)
```

### Lesson Learned
A VMI hard reset simulates a hardware reset button press, causing a full cold boot of the guest OS. SSH readiness after a hard reset takes significantly longer than after a soft reboot because the guest must complete the entire boot sequence from BIOS/UEFI. The default `ssh_timeout=TIMEOUT_2MIN` is insufficient. Use `TIMEOUT_10MIN` to allow adequate time for the cold boot + SSH daemon startup.

### How to Avoid
When waiting for a VM to become SSH-accessible after a hard reset operation, always pass an increased `ssh_timeout` (at least `TIMEOUT_10MIN`) to `wait_for_running_vm`. The default 2-minute timeout is only appropriate for warm reboots where the guest OS restarts quickly.

---

## Entry: VM with maxSockets and memory_guest too large for resource-constrained cluster

**Category**: ResourceError
**Date**: 2026-02-22
**Test**: tests/virt/node/hotplug/test_cpu_hotplug_max_limit.py::TestCPUHotplugMaxSocketsLimit::test_max_sockets_limited_by_max_vcpus

### What Went Wrong
The generated code created VMs with `cpu_max_sockets=EIGHT_CPU_SOCKETS` (8) and `memory_guest=FOUR_GI_MEMORY` ("4Gi"). When CPU hotplug is enabled via `maxSockets`, KubeVirt's virt-launcher pod requests memory proportional to the maximum number of vCPUs (overhead for QEMU vCPU threads, NUMA topology, etc.). With `maxSockets=8` and `memory_guest=4Gi`, the total pod memory request exceeded available memory on all worker nodes (~2-3GiB free per node). The VM failed to schedule with `ErrorUnschedulable: Insufficient memory`.

### Wrong Code
```python
cpu_max_sockets=EIGHT_CPU_SOCKETS,
cpu_sockets=FOUR_CPU_SOCKETS,
memory_guest=FOUR_GI_MEMORY,
```

### Correct Code
```python
cpu_max_sockets=FOUR_CPU_SOCKETS,
cpu_sockets=TWO_CPU_SOCKETS,
memory_guest=VM_MEMORY_GUEST,  # "2Gi"
```

### Lesson Learned
The virt-launcher pod's memory request scales with `maxSockets` because KubeVirt pre-allocates resources for the maximum number of vCPUs. On resource-constrained test clusters, using high `maxSockets` values (e.g., 8) combined with significant guest memory (e.g., 4Gi) causes `ErrorUnschedulable`. Reduce both `maxSockets` and `memory_guest` to the minimum needed to test the feature. For CPU hotplug limit tests, `maxSockets=4` and `memory_guest=VM_MEMORY_GUEST` ("2Gi") are sufficient.

### How to Avoid
Before choosing `cpu_max_sockets` and `memory_guest` values, check cluster worker node available memory with `kubectl top nodes` and `kubectl describe nodes`. Use the smallest values that still exercise the feature under test. For hotplug limit tests, `FOUR_CPU_SOCKETS` as maxSockets and `VM_MEMORY_GUEST` ("2Gi") are safer defaults than `EIGHT_CPU_SOCKETS` and `FOUR_GI_MEMORY`.

---

## Entry: CPU hotplug migration fails on resource-constrained clusters

**Category**: ResourceError
**Date**: 2026-02-22
**Test**: tests/virt/node/hotplug/test_cpu_hotplug_max_limit.py::TestCPUHotplugMaxSocketsLimit::test_hotplug_cpu_up_to_max_sockets

### What Went Wrong
CPU hotplug via `hotplug_spec_vm_and_verify_hotplug` triggers a live migration to apply the CPU change. The migration creates a second virt-launcher pod on another node. On resource-constrained clusters, this target pod cannot be scheduled because no worker node has sufficient free memory for a second VM pod. The migration gets stuck in `Scheduling` state and eventually times out with `TimeoutExpiredError: VMIM stuck in Scheduling state`.

### Wrong Code
```python
hotplug_spec_vm_and_verify_hotplug(
    vm=max_limit_vm,
    client=unprivileged_client,
    sockets=FOUR_CPU_SOCKETS,
)
```

### Correct Code
```python
# On resource-constrained clusters, hotplug with migration verification
# requires enough free memory on at least 2 nodes to run both source
# and target virt-launcher pods simultaneously.
# If cluster resources are insufficient, the positive hotplug tests
# will fail at migration scheduling, but negative tests (webhook
# rejection) will still pass since they don't trigger migration.
hotplug_spec_vm_and_verify_hotplug(
    vm=max_limit_vm,
    client=unprivileged_client,
    sockets=FOUR_CPU_SOCKETS,
)
```

### Lesson Learned
CPU hotplug with `hotplug_spec_vm_and_verify_hotplug` requires live migration, which needs enough free memory on at least two worker nodes to run source and target virt-launcher pods simultaneously. On clusters with limited memory, the migration target pod cannot be scheduled, causing the test to fail. Negative hotplug tests (testing webhook rejection for values exceeding maxSockets) do NOT require migration because the API rejects the request before any migration is triggered.

### How to Avoid
When writing CPU hotplug tests, ensure negative tests (testing rejection above maxSockets) do NOT depend on positive hotplug tests via `@pytest.mark.dependency`. Negative tests use `hotplug_spec_vm` which is rejected at the webhook level without triggering migration, so they work on any cluster. Positive hotplug tests require clusters with sufficient resources for live migration - do not make other tests depend on them.

---
