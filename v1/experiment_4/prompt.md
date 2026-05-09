Act as a Senior SDET. I need you to implement the test plan below.

I am currently in the root of the repository. Before generating any code, you must perform a "Context Exploration" phase to understand the existing testing architecture.

**Step 1: Context Exploration**
1.  Scan the `docs/` folder to understand the project's testing conventions or architecture documentation.
2.  Analyze the `libs/` and `utilities/` folders. Identify existing helper classes, fixtures, and utility functions.
3.  Look for existing test files in `tests/` folder to see how they import these utilities and what standard `pytest` fixtures are available (e.g., clients, namespace helpers). Examine how tests are linked to requirements.

**Step 2: Code Generation**
* Implement the scenarios from the Test Plan below.
* **Strict Constraint:** Do not hallucinate new utilities. You MUST use the existing functions and classes you found in `libs/` and `utilities/`. If a specific helper is missing, implement it locally in the test file using the base clients found.

**Test Plan (STP):**
# Openshift-virtualization-tests Test plan

## **VM Snapshot Restore Stuck with runStrategy RerunOnFailure - Quality Engineering Plan**

| Field                  | Details                                                 |
| :--------------------- | :------------------------------------------------------ |
| **Enhancement(s)**     | TBD                                                     |
| **Feature in Jira**    | [CNV-63819](https://issues.redhat.com/browse/CNV-63819) |
| **Jira Tracking**      | [CNV-63819](https://issues.redhat.com/browse/CNV-63819) |
| **QE Owner(s)**        | TBD                                                     |
| **Owning SIG**         | sig-storage                                             |
| **Participating SIGs** | sig-storage, sig-compute                                |
| **Current Status**     | Draft                                                   |

### I. Motivation and Requirements Review (QE Review Guidelines)

#### **1. Requirement & User Story Review Checklist**

| Check                                  | Status (Y/N) | Details/Notes                                                                                                                                                                           | Comments                                                                                                                                                 |
| :------------------------------------- | :----------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Review Requirements**                |              | Reviewed bug report CNV-63819 regarding VM snapshot restore getting stuck when VM has runStrategy: RerunOnFailure.                                                                      | Snapshot restore never completes. virt-controller immediately tries to start VM, blocking restore.                                                        |
| **Understand Value**                   |              | **Value for RH customers**: Enables snapshot restore for VMs using RerunOnFailure run strategy, critical for backup/recovery workflows.                                                 | Important for backup and disaster recovery scenarios.                                                                                                     |
| **Customer Use Cases**                 |              | Administrators restoring VMs from snapshots for disaster recovery, testing, or rollback scenarios when VMs use RerunOnFailure strategy.                                                 | Common for production VMs using automatic restart policies.                                                                                               |
| **Testability**                        |              | Testable - create VM with RerunOnFailure, take snapshot, stop VM, restore snapshot, verify completion.                                                                                  | 100% reproducible according to bug report.                                                                                                                |
| **Acceptance Criteria**                |              | Snapshot restore must complete successfully for VMs with runStrategy: RerunOnFailure. VM should not auto-start during restore. Restore operation should finish properly.                | Clear acceptance: Snapshot restore completes, VM can be started afterwards.                                                                               |
| **Non-Functional Requirements (NFRs)** |              | **Reliability**: Snapshot restore must work consistently. **Data Integrity**: Restored VM must have correct state. **Consistency**: All run strategies should support snapshot restore.| Snapshot reliability is critical NFR.                                                                                                                     |

**Reference enhancement sources:** Review KubeVirt VM controller reconciliation logic and snapshot restore workflow.

#### **2. Technology and Design Review**

| Check                            | Status | Details/Notes                                                                                                                                           | Comments                                                                                                                                                                      |
| :------------------------------- | :----- | :------------------------------------------------------------------------------------------------------------------------------------------------------ | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Developer Handoff/QE Kickoff** | [ ]    | Pending - Need Dev walkthrough on virt-controller behavior during snapshot restore, RerunOnFailure reconciliation logic.                                | Critical to understand: Why virt-controller tries to start VM during restore, how to defer start until after restore, fix approach.                                           |
| **Technology Challenges**        | [ ]    | VM controller reconciliation loop, runStrategy handling, snapshot restore workflow, race conditions between restore and VM start.                       | Testing requires understanding of controller reconciliation timing.                                                                                                           |
| **Test Environment Needs**       | [ ]    | Cluster with storage class supporting snapshots, VM snapshot functionality enabled.                                                                     | Standard snapshot-capable environment.                                                                                                                                        |
| **API Extensions**               | [ ]    | VM runStrategy field, VirtualMachineSnapshot, VirtualMachineRestore APIs.                                                                               |                                                                                                                                                                               |
| **Topology Considerations**      | [ ]    | Not topology-specific but requires snapshot-capable storage.                                                                                            | Snapshot feature depends on storage class capabilities.                                                                                                                       |

### II. Software Test Plan (STP)

#### 1. Introduction and Scope

- **Purpose:** This document describes the overall approach to testing for [CNV-63819 - VM snapshot restore stuck with runStrategy RerunOnFailure](https://issues.redhat.com/browse/CNV-63819). It outlines the testing objectives, scope, methodology, and resources required to ensure **reliable snapshot restore** across all run strategies.

- **Scope of Testing:**
  - Verify snapshot restore completes for VMs with RerunOnFailure run strategy
  - Test snapshot restore for VMs with other run strategies (regression)
  - Validate VM doesn't auto-start during restore process
  - Test restored VM can be started after restore completes
  - Verify restored VM has correct state and data
  - Test various snapshot restore scenarios with different run strategies

- **Document Conventions:**
  - **runStrategy**: VM field controlling automatic start behavior
  - **RerunOnFailure**: Run strategy that restarts VM automatically on failure
  - **VirtualMachineSnapshot**: Resource capturing VM state
  - **VirtualMachineRestore**: Resource restoring VM from snapshot
  - **virt-controller**: Control plane component managing VM lifecycle

#### 2. Test Objectives

The primary goals of this test plan are to:

- **Verify** snapshot restore works with RerunOnFailure run strategy
- **Ensure** VM doesn't interfere with restore process
- **Validate** restored VM has correct configuration and data
- **Test** snapshot restore across all run strategies
- **Confirm** virt-controller properly handles restore workflow

#### 3. Motivation

This section captures the feature motivation from a QE perspective, focusing on testing goals and priorities.

##### **A. User Stories (Testing Perspective)**

- **As a** VM administrator, **I want** to restore VMs from snapshots regardless of run strategy **so that** I can recover from failures or rollback changes.
  - **QE Focus:** Verify snapshot restore works for all run strategy values.

- **As a** backup operator, **I need** reliable snapshot restore **because** it's critical for disaster recovery procedures.
  - **QE Focus:** Test complete snapshot/restore workflow with RerunOnFailure VMs.

- **As a** platform engineer, **I want** snapshot restore to complete without manual intervention **so that** automated recovery workflows function correctly.
  - **QE Focus:** Verify restore completes automatically without getting stuck.

##### **B. Testing Goals**

- [ ] Verify snapshot restore completes for VM with runStrategy: RerunOnFailure
- [ ] Test that VM doesn't auto-start during restore (blocking issue)
- [ ] Validate VirtualMachineRestore resource reaches Complete state
- [ ] Confirm restored VM can be manually started after restore
- [ ] Test restored VM has correct state and data
- [ ] Verify snapshot restore works for other run strategies (regression)

##### **C. Non-Goals (Testing Scope Exclusions)**

| Non-Goal                                                                           | Rationale                                                                                                                  | PM/ Lead Agreement |
| :--------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------- | :----------------- |
| Testing all snapshot features comprehensively                                      | Focus on RerunOnFailure issue. Comprehensive snapshot testing separate.                                                    | [ ] TBD            |
| Testing on all storage backends                                                    | Use snapshot-capable storage. Full storage matrix tested separately.                                                       | [ ] TBD            |
| Performance testing of snapshot restore                                            | Focus is on functional correctness, not performance.                                                                       | [ ] TBD            |
| Testing online snapshots (while VM running)                                        | Bug report shows offline workflow (stop then restore). Online snapshots tested separately.                                 | [ ] TBD            |
| Testing snapshot cloning                                                           | Focus on restore operation. Cloning tested separately.                                                                     | [ ] TBD            |

#### 4. Test Strategy

##### **A. Types of Testing**

| Item (Testing Type)            | Addressed (Y/N) | Comments                                                                                                              | Details                                                                                                     |
| :----------------------------- | :-------------- | :-------------------------------------------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------- |
| Functional Testing             | Y               | Verify snapshot restore with various run strategies. **(Tier 1)**                                                     | Verifying that the software functions as specified. **(Typically Tier 1)**                                  |
| Automation Testing             | Y               | Automate snapshot/restore workflows, run strategy testing. **(Tier 1 & Tier 2)**                                     | Example: Automate all P1 scenarios, focus on API-level automation. **(Both Tier 1 & Tier 2)**               |
| Performance Testing            | N/A             | Not applicable - focus on functional correctness.                                                                     | Evaluating responsiveness, stability, and scalability under various loads. **(Typically Tier 2)**           |
| Security Testing               | N/A             | No security implications from restore fix.                                                                            | Identifying vulnerabilities and weaknesses. **(Both Tier 1 & Tier 2)**                                      |
| Usability Testing              | Y               | Verify restore process is straightforward for users. **(Tier 2)**                                                     | Assessing ease of use and user-friendliness. **(Typically Tier 2)**                                         |
| Compatibility Testing          | Y               | Test with various storage classes supporting snapshots. **(Tier 1)**                                                  | Checking compatibility with different deployments. **(Both Tier 1 & Tier 2)**                               |
| Regression Testing             | Y               | Ensure restore works for other run strategies (Always, Manual, Halted). **(Tier 1)**                                  | Ensuring new changes do not adversely affect existing functionalities. **(Typically Tier 1)**               |
| Upgrade Testing                | Y               | Verify fix works after upgrade from affected version. **(Tier 2)**                                                    | Verifying successful software upgrade capability. **(Typically Tier 2)**                                    |
| Backward Compatibility Testing | Y               | Ensure existing snapshots can be restored with fix. **(Tier 1)**                                                      | Ensuring the new version works with older versions or data, such as API changes. **(Both Tier 1 & Tier 2)** |

##### **B. Potential Areas to Consider**

| Item                   | Description                                                                                                        | Addressed (Y/N) | Comment                                                                                                                       | Source      |
| :--------------------- | :----------------------------------------------------------------------------------------------------------------- | :-------------- | :---------------------------------------------------------------------------------------------------------------------------- | :---------- |
| **Dependencies**       | Dependent on deliverables from other components/products? Identify what is tested by which team.                   | Y               | Depends on: virt-controller for restore logic, storage backend for snapshot support, CDI for volume operations.               | CNV-63819   |
| **Monitoring**         | Does the feature require metrics and/or alerts?                                                                    | Y               | Should monitor: Snapshot restore completion, restore failures, stuck restore operations.                                      | CNV-63819   |
| **Cross Integrations** | Does the feature affect other features/require testing by other components? Identify what is tested by which team. | Y               | Integration with: VM controller, snapshot controller, storage provisioning.                                                   | CNV-63819   |
| **UI**                 | Does the feature require UI? If so, ensure the UI aligns with the requirements.                                    | Y               | Console should show restore progress and not indicate VM starting during restore.                                             | CNV-63819   |

#### 5. Test Environment

| Environment Component                         | Value                                                                 | Specification Examples                                                                |
| :-------------------------------------------- | :-------------------------------------------------------------------- | :------------------------------------------------------------------------------------ |
| **Cluster Topology**                          | 3-master/3-worker                                                     | [e.g., 3-master/3-worker bare-metal, SNO, Compact Cluster, Hypershift hosted cluster] |
| **OCP & OpenShift Virtualization Version(s)** | OCP 4.18, OpenShift Virtualization 4.18                               | [e.g., OCP 4.20 with OpenShift Virtualization 4.20]                                   |
| **CPU Virtualization**                        | Nodes with VT-x (Intel) or AMD-V (AMD) enabled                        | [e.g., Nodes with VT-x (Intel) or AMD-V (AMD) enabled in BIOS]                        |
| **Compute Resources**                         | Standard worker node resources (8 vCPUs, 32GB RAM)                    | [e.g., Minimum per worker node: 8 vCPUs, 32GB RAM]                                    |
| **Special Hardware**                          | None required                                                         | [e.g., Specific NICs for SR-IOV, GPU etc]                                             |
| **Storage (Disk)**                            | Storage with snapshot support                                         | [e.g., Minimum 500GB per node]                                                        |
| **Storage Class**                             | Storage class with snapshot capability (e.g., ODF Ceph RBD)           | [e.g., ocs-storagecluster-ceph-rbd-virtualization, specific CSI driver like Ceph-RBD] |
| **CNI (Container Network)**                   | OVN-Kubernetes (default)                                              | [e.g., OVN-Kubernetes (default), OpenShift-SDN]                                       |
| **Secondary Networks**                        | Not required for this test                                            | [e.g., Required NetworkAttachmentDefinitions (NADs) for SR-IOV, bridge, macvlan]      |
| **Network Plugins**                           | Standard Multus CNI                                                   | [e.g., Multus CNI for secondary networks]                                             |
| **Network Features**                          | IPv4                                                                  | [e.g., IPv4, IPv6, dual-stack]                                                        |
| **Required Operators**                        | OpenShift Virtualization                                              | [e.g., NMState Operator]                                                              |
| **Platform**                                  | Bare metal or cloud platforms (AWS, Azure, GCP)                       | [e.g., Bare metal, AWS, Azure, GCP etc]                                               |
| **Special Configurations**                    | Storage with VolumeSnapshot CRD installed                             | [e.g., Disconnected/air-gapped cluster, Proxy environment, FIPS mode enabled]         |

#### 6. Entry and Exit Criteria

##### **A. Entry Criteria**

The following conditions must be met before testing can begin:

- Bug fix code is **merged and available** in target build
- Test environment with **snapshot-capable storage** deployed
- **Understanding of virt-controller restore logic** from Dev team
- Test cases **reviewed and approved**
- VMs with different run strategies available for testing

##### **B. Exit Criteria**

The following conditions must be met for testing to be considered complete:

- Snapshot restore **completes successfully** for VMs with RerunOnFailure
- **VirtualMachineRestore reaches Complete** state
- VM **can be started manually** after restore
- All **high-priority defects resolved and verified**
- **Test coverage goals achieved** across run strategies
- **Test automation merged** (required for GA sign-off)
- Regression testing confirms other run strategies still work
- QE sign-off approved

#### 7. Risk Management

| Risk                       | Risk Description                                                                                                            | Mitigation Strategy                                                                                                        | Comments                                                                                     |
| :------------------------- | :-------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------- |
| Timeline/Schedule Risk     | Snapshot restore testing can be time-consuming with large VMs.                                                              | Use smaller VMs for testing, automate snapshot/restore workflows, prioritize P1 scenarios.                                 | Snapshot operations can take time depending on size.                                         |
| Insufficient test coverage | Multiple run strategy values to test.                                                                                       | Test all run strategies (RerunOnFailure, Always, Manual, Halted, Once), document tested matrix.                           | Four main run strategy values to test.                                                       |
| Unstable test environment  | Snapshot operations depend on storage backend stability.                                                                    | Use stable storage backend, validate snapshot capability before testing restore, monitor storage health.                   | Storage backend stability important.                                                         |
| Untestable aspects         | Race conditions in controller reconciliation may be timing-sensitive.                                                       | Multiple test runs, various timing scenarios, monitor controller logs for race conditions.                                 | Controller timing issues can be intermittent.                                                |
| Resource constraints       | Snapshots consume storage resources.                                                                                        | Clean up snapshots after tests, monitor storage usage, use storage quotas.                                                 | Snapshot testing storage-intensive.                                                          |
| Integration concerns       | Fix involves virt-controller, snapshot controller, storage backend coordination.                                            | Test each component's behavior, integration testing with various storage backends, coordinate with storage team.          | Multi-component fix requires careful integration testing.                                    |
| Other                      | Fix may expose other issues in snapshot restore workflow.                                                                   | Comprehensive snapshot restore testing, various restore scenarios, clear bug triage process.                               |                                                                                              |

#### 8. Limitations

- Testing limited to snapshot-capable storage classes
- Cannot test all possible VM configurations and sizes
- Focus on runStrategy as primary variable
- Snapshot restore testing time-consuming - limits test iterations
- May not cover all race conditions in controller reconciliation

### III. Test Case Descriptions & Traceability

#### **1. Test Case Repository**

All test cases for this feature are managed in: **[Link to Polarion - TBD]**

#### **2. Traceability Matrix**

| Requirement / User Story (from Section I) | Mapped Test Scenarios (High-Level)                                                                  | Tier | Priority | Test Type(s)                      | Automation Status         |
| :---------------------------------------- | :------------------------------------------------------------------------------------------------- | :--- | :------- | :-------------------------------- | :------------------------ |
| [CNV-63819] Restore with RerunOnFailure  | Snapshot VM with RerunOnFailure → Stop VM → Restore → Verify restore completes                      | T1   | P1       | Functional, Snapshot               | Planned                   |
| [CNV-63819] VirtualMachineRestore status | Verify VirtualMachineRestore resource reaches Complete (not stuck)                                   | T1   | P1       | Functional, API                    | Planned                   |
| [CNV-63819] VM doesn't auto-start        | During restore, verify VM doesn't start prematurely (blocking restore)                               | T1   | P1       | Functional, VM Lifecycle           | Planned                   |
| [CNV-63819] Manual start after restore   | After restore completes, manually start VM, verify starts successfully                               | T1   | P1       | Functional, VM Lifecycle           | Planned                   |
| [CNV-63819] Restore with Always strategy | Snapshot and restore VM with runStrategy: Always (regression)                                        | T1   | P1       | Regression, Snapshot               | Planned                   |
| [CNV-63819] Restore with other strategies| Test snapshot restore with Manual and Halted run strategies                                          | T1   | P2       | Regression, Snapshot               | Planned                   |
| [CNV-63819] Complete restore workflow    | Create VM (RerunOnFailure) → Add data → Snapshot → Stop → Restore → Start → Verify data intact      | T2   | P1       | E2E, Snapshot, Data Integrity      | Planned                   |
| [CNV-63819] Multiple restore operations  | Take multiple snapshots, restore different snapshots, verify all complete                            | T2   | P2       | E2E, Snapshot                      | Planned                   |

#### **3. Test Scenario Descriptions**

##### **Tier 1 (Functional) Scenarios**

- **Verify restore with RerunOnFailure**
  - **Requirements Covered:** CNV-63819
  - **Priority:** P1
  - **Test Approach:** Create VM with runStrategy: RerunOnFailure, start VM, take snapshot, stop VM, create VirtualMachineRestore, monitor restore status, verify reaches Complete state
  - **Why Tier 1:** Tests single feature (snapshot restore) with specific configuration

- **Verify VirtualMachineRestore completes**
  - **Requirements Covered:** CNV-63819
  - **Priority:** P1
  - **Test Approach:** Monitor VirtualMachineRestore resource during restore operation, verify status progresses to Complete (not stuck indefinitely)
  - **Why Tier 1:** Tests single API resource lifecycle

- **Verify VM doesn't auto-start during restore**
  - **Requirements Covered:** CNV-63819
  - **Priority:** P1
  - **Test Approach:** During restore operation, check VMI (VirtualMachineInstance) resources, verify no VMI created during restore, check virt-controller logs for start attempts
  - **Why Tier 1:** Tests single controller behavior (not starting VM prematurely)

##### **Tier 2 (End-to-End) Scenarios**

- **Complete snapshot restore workflow with data validation**
  - **Requirements Covered:** CNV-63819
  - **Priority:** P1
  - **Test Approach:** End-to-end workflow: Create VM with runStrategy: RerunOnFailure → Start VM → Write test data to disk → Take VirtualMachineSnapshot → Wait for snapshot ready → Stop VM → Create VirtualMachineRestore → Monitor restore progress → Verify restore completes → Start VM manually → Verify test data intact → Validate VM functions correctly
  - **Why Tier 2:** Tests complete user workflow from snapshot through restore and data validation

#### **4. Traceability Maintenance**

- **Owner:** CNV QE Storage Team
- **Update Frequency:** Updated after each test execution and when snapshot features change
- **Review Process:** Weekly sync with Storage and Compute teams during active development
- **Tool:** Jira (requirements) + Polarion (test cases) + Git (automation)

### IV. Sign-off and Approval

#### **1. Final Sign-off Checklist**

| Requirement                        | Status (Y/N) | Notes                                                                         | Source      |
| :--------------------------------- | :----------- | :---------------------------------------------------------------------------- | :---------- |
| **Tier 1 / Tier 2 Tests Defined**  |              | Reviewed with Storage/Compute teams on snapshot restore testing.              | This STP    |
| **Automation Merged**              |              | **Automation must be merged for GA sign-off**.                                | Exit Criteria|
| **Tests in Release Checklist Job** |              | Snapshot restore tests integrated into release checklist jobs.                | TBD         |
| **Documentation Reviewed**         |              | Verify snapshot restore documentation mentions run strategy compatibility.    | TBD         |
| **Feature Sign-off**               |              | QE officially signs off the bug fix.                                          | TBD         |

#### **2. Approval**

This Software Test Plan requires approval from the following stakeholders:

- Reviewers:
  - CNV Storage Team Lead
  - CNV Compute Team (for run strategy behavior)
  - virt-controller maintainers
- Approvers:
  - CNV QE Lead
  - CNV Engineering Manager

