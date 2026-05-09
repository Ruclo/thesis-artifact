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

## **StorageProfile snapshotClass Not Honored for VM Snapshot - Quality Engineering Plan**

---

### Metadata & Tracking

| Field                  | Details                                                           |
| :--------------------- | :---------------------------------------------------------------- |
| **Enhancement(s)**     | N/A - Bug Fix                                                     |
| **Feature in Jira**    | [CNV-61266](https://issues.redhat.com/browse/CNV-61266)           |
| **Jira Tracking**      | [CNV-54866](https://issues.redhat.com/browse/CNV-54866) (Bug Fix) |
| **QE Owner(s)**        | Stuart Gott                                                       |
| **Owning SIG**         | sig-storage                                                       |
| **Participating SIGs** | sig-storage, sig-compute                                          |
| **Current Status**     | Draft                                                             |

### Related GitHub Pull Requests

| PR Link                                                                    | Repository        | Source Jira Issue | Description                                                                      |
| :------------------------------------------------------------------------- | :---------------- | :---------------- | :------------------------------------------------------------------------------- |
| [kubevirt/kubevirt#13711](https://github.com/kubevirt/kubevirt/pull/13711) | kubevirt/kubevirt | CNV-54866         | VMSnapshot: honor StorageProfile snapshotClass when choosing volumesnapshotclass |
| [kubevirt/kubevirt#13723](https://github.com/kubevirt/kubevirt/pull/13723) | kubevirt/kubevirt | CNV-54866         | [release-1.4] VMSnapshot: honor StorageProfile snapshotClass (backport)          |

**Note:** All PRs listed above were reviewed for implementation details, code changes, and review comments to inform this test plan.

---

### **I. Motivation and Requirements Review (QE Review Guidelines)**

#### **1. Requirement & User Story Review Checklist**

| Check                                  | Done | Details/Notes                                                                                                                        | Comments |
| :------------------------------------- | :--- | :----------------------------------------------------------------------------------------------------------------------------------- | :------- |
| **Review Requirements**                | [x]  | Reviewed bug: StorageProfile's snapshotClass setting is ignored when creating VM snapshots. Snapshots use wrong VolumeSnapshotClass. |          |
| **Understand Value**                   | [x]  | Critical for customers using specific snapshot classes for their storage backends.                                                   |          |
| **Customer Use Cases**                 | [x]  | Customers with multiple storage backends needing specific snapshot configurations.                                                   |          |
| **Testability**                        | [x]  | Testable by configuring StorageProfile with snapshotClass and creating VM snapshot.                                                  |          |
| **Acceptance Criteria**                | [x]  | VM snapshots should use the snapshotClass defined in StorageProfile.                                                                 |          |
| **Non-Functional Requirements (NFRs)** | [x]  | Storage compatibility; Data integrity.                                                                                               |          |

#### **2. Technology and Design Review**

| Check                            | Done | Details/Notes                                                                                 | Comments |
| :------------------------------- | :--- | :-------------------------------------------------------------------------------------------- | :------- |
| **Developer Handoff/QE Kickoff** | [ ]  | Pending - need walkthrough on snapshotClass selection logic.                                  |          |
| **Technology Challenges**        | [x]  | Fix adds check for StorageProfile.snapshotClass before falling back to label-based selection. |          |
| **Test Environment Needs**       | [x]  | Cluster with storage backend supporting snapshots.                                            |          |
| **API Extensions**               | [x]  | No API changes - behavior fix in snapshot controller.                                         |          |
| **Topology Considerations**      | [x]  | Standard topology with snapshot-capable storage.                                              |          |

### **II. Software Test Plan (STP)**

#### **1. Scope of Testing**

**In Scope:**

- Verify VMSnapshot uses snapshotClass from StorageProfile
- Verify fallback to label-based selection when StorageProfile has no snapshotClass
- Verify snapshot creation succeeds with correct VolumeSnapshotClass
- Verify snapshot restore works with ProfileProfile-specified snapshotClass

**Document Conventions:**

- StorageProfile: CDI resource defining storage characteristics per StorageClass
- snapshotClass: Field in StorageProfile specifying VolumeSnapshotClass to use
- VolumeSnapshotClass: Kubernetes resource defining snapshot provisioner settings

#### **2. Testing Goals**

- [ ] Verify 100% adherence to StorageProfile snapshotClass
- [ ] Confirm fallback behavior works correctly
- [ ] Validate snapshot/restore cycle with correct classes

#### **3. Non-Goals (Testing Scope Exclusions)**

| Non-Goal                             | Rationale               | PM/ Lead Agreement |
| :----------------------------------- | :---------------------- | :----------------- |
| Storage backend testing              | Focus on KubeVirt logic | [ ] Name/Date      |
| Snapshot performance testing         | Functionality focus     | [ ] Name/Date      |
| Testing unsupported storage backends | Out of scope            | [ ] Name/Date      |

#### **4. Test Strategy**

##### **A. Types of Testing**

| Item (Testing Type)            | Applicable (Y/N or N/A) | Comments                    |
| :----------------------------- | :---------------------- | :-------------------------- |
| Functional Testing             | Y                       | Core snapshot functionality |
| Automation Testing             | Y                       | Must be automated           |
| Performance Testing            | N/A                     | Not in scope                |
| Security Testing               | N/A                     | No security changes         |
| Usability Testing              | N/A                     | No UI changes               |
| Compatibility Testing          | Y                       | Different storage backends  |
| Regression Testing             | Y                       | Existing snapshot tests     |
| Upgrade Testing                | N/A                     | Not upgrade specific        |
| Backward Compatibility Testing | Y                       | Existing StorageProfiles    |

##### **B. Potential Areas to Consider**

| Item                   | Description                                   | Applicable (Y/N or N/A) | Comment            |
| :--------------------- | :-------------------------------------------- | :---------------------- | :----------------- |
| **Dependencies**       | CDI, StorageProfile, VolumeSnapshotClass      | Y                       | Core dependencies  |
| **Monitoring**         | Snapshot success/failure metrics              | Y                       | Track success rate |
| **Cross Integrations** | CDI, snapshot-controller, storage provisioner | Y                       | All involved       |
| **UI**                 | Snapshot creation in console                  | Y                       | Verify UI works    |

#### **5. Test Environment**

| Environment Component                         | Configuration | Specification Examples                                              |
| :-------------------------------------------- | :------------ | :------------------------------------------------------------------ |
| **Cluster Topology**                          | Required      | Standard cluster with snapshot support                              |
| **OCP & OpenShift Virtualization Version(s)** | Required      | OCP 4.17+ with OpenShift Virtualization 4.17+                       |
| **CPU Virtualization**                        | Required      | VT-x or AMD-V enabled                                               |
| **Compute Resources**                         | Required      | Standard requirements                                               |
| **Special Hardware**                          | N/A           | None required                                                       |
| **Storage**                                   | Required      | Snapshot-capable storage (ODF, Ceph, etc.)                          |
| **Network**                                   | Required      | OVN-Kubernetes                                                      |
| **Required Operators**                        | Required      | OpenShift Virtualization, OpenShift Data Foundation (or equivalent) |
| **Platform**                                  | Required      | Any platform with snapshot support                                  |
| **Special Configurations**                    | Required      | StorageProfile with snapshotClass configured                        |

#### **5.5. Testing Tools & Frameworks**

| Category           | Tools/Frameworks                                 |
| :----------------- | :----------------------------------------------- |
| **Test Framework** | Standard OpenShift Virtualization test framework |
| **CI/CD**          | Standard CI pipeline                             |
| **Other Tools**    | kubectl, storage verification tools              |

#### **6. Entry and Exit Criteria**

##### **A. Entry Criteria**

- [x] Requirements and design documents are **approved and merged**
- [ ] Test environment is **set up and configured** with snapshot-capable storage
- [ ] Test cases are **reviewed and approved**
- [ ] PR #13711 is merged

##### **B. Exit Criteria**

- [ ] StorageProfile snapshotClass honored
- [ ] Fallback behavior works
- [ ] Snapshot/restore cycle succeeds
- [ ] Test automation merged

#### **7. Risks and Limitations**

| Risk Category        | Specific Risk for This Feature                  | Mitigation Strategy                 | Status |
| :------------------- | :---------------------------------------------- | :---------------------------------- | :----- |
| Timeline/Schedule    | Need snapshot-capable storage                   | Use ODF or equivalent               | [ ]    |
| Test Coverage        | Various storage backends may behave differently | Test on ODF as primary              | [ ]    |
| Test Environment     | Need multiple VolumeSnapshotClasses             | Configure test environment properly | [ ]    |
| Untestable Aspects   | All storage backends                            | Focus on supported configurations   | [ ]    |
| Resource Constraints | Snapshot storage consumption                    | Clean up snapshots after tests      | [ ]    |
| Dependencies         | External snapshot controller                    | Verify controller is healthy        | [ ]    |
| Other                | N/A                                             | N/A                                 | [ ]    |

#### **8. Known Limitations**

- Requires snapshot-capable storage backend
- StorageProfile must be properly configured with snapshotClass
- Fix only affects new snapshots, not existing ones

---

### **III. Test Case Descriptions & Traceability**

#### **1. Requirements-to-Tests Mapping**

| Requirement ID | Requirement Summary        | Test Scenario(s)                                         | Test Type(s)       | Priority |
| :------------- | :------------------------- | :------------------------------------------------------- | :----------------- | :------- |
| CNV-61266      | Honor snapshotClass        | Create VM snapshot with StorageProfile snapshotClass set | Functional, Tier 1 | P0       |
| CNV-61266      | Fallback behavior          | Create snapshot without snapshotClass in StorageProfile  | Functional, Tier 1 | P0       |
| CNV-61266      | Restore with correct class | Restore VM from snapshot using correct class             | Functional, Tier 1 | P1       |

#### **Test Scenarios - Tier 1 (Functional)**

**Scenario 1: StorageProfile snapshotClass Honored**

- **Preconditions:** StorageProfile with snapshotClass set
- **Steps:**
  1. Configure StorageProfile with specific snapshotClass
  2. Create VM using that StorageClass
  3. Create VMSnapshot
  4. Inspect VolumeSnapshot's volumeSnapshotClassName
- **Expected Result:** VolumeSnapshot uses StorageProfile's snapshotClass
- **Priority:** P0

**Scenario 2: Fallback to Label-Based Selection**

- **Preconditions:** StorageProfile without snapshotClass
- **Steps:**
  1. Ensure StorageProfile has no snapshotClass
  2. Create VM and VMSnapshot
  3. Verify VolumeSnapshotClass selected via labels
- **Expected Result:** Fallback selection works correctly
- **Priority:** P0

**Scenario 3: Restore with Correct VolumeSnapshotClass**

- **Preconditions:** Snapshot created with StorageProfile snapshotClass
- **Steps:**
  1. Create snapshot using StorageProfile snapshotClass
  2. Restore VM from snapshot
  3. Verify restoration succeeds
- **Expected Result:** Restore works with StorageProfile-specified class
- **Priority:** P1

#### **Test Scenarios - Tier 2 (E2E)**

**Scenario 4: Multiple Storage Classes**

- **Preconditions:** Multiple StorageClasses with different snapshotClasses
- **Steps:**
  1. Create VMs on different StorageClasses
  2. Create snapshots for each
  3. Verify each uses correct snapshotClass
- **Expected Result:** All snapshots use correct class
- **Priority:** P2

---

### **IV. Sign-off and Approval**

This Software Test Plan requires approval from the following stakeholders:

- **Reviewers:**
  - [QE Team Lead]
  - [Storage SIG Representative]
- **Approvers:**
  - [QE Manager]
  - [Product Owner]
