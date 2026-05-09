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
