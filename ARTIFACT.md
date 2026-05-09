# Artifact Guide

For each STP run, inspect:

1. `claude.log` for the agent action trace.
2. `changes.patch` for generated code.
3. `test_run.log` for final runtime validation.
4. GRAVEYARD snapshots for memory experiments.

## Reproducibility Limits

The logs can be inspected from this repository. Full dynamic re-execution requires
access to the private OpenShift Virtualization test repository, credentials, and QE cluster.
