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

## **Unable to Disable Common-Instancetypes Deployment from HCO - Quality Engineering Plan**

---

### Metadata & Tracking

| Field                  | Details                                                                |
| :--------------------- | :--------------------------------------------------------------------- |
| **Enhancement(s)**     | N/A - Bug Fix / Feature Enhancement                                    |
| **Feature in Jira**    | [CNV-61256](https://issues.redhat.com/browse/CNV-61256)                |
| **Jira Tracking**      | [CNV-59564](https://issues.redhat.com/browse/CNV-59564) (Bug Fix)      |
| **QE Owner(s)**        | Stuart Gott                                                            |
| **Owning SIG**         | sig-hco                                                                |
| **Participating SIGs** | sig-hco, sig-compute                                                   |
| **Current Status**     | Draft                                                                  |

### Related GitHub Pull Requests

| PR Link | Repository | Source Jira Issue | Description |
| :------ | :--------- | :---------------- | :---------- |
| [kubevirt/hyperconverged-cluster-operator#3471](https://github.com/kubevirt/hyperconverged-cluster-operator/pull/3471) | kubevirt/hyperconverged-cluster-operator | CNV-59564 | fix(api): Introduce CommonInstancetypesDeployment configurable |

**Note:** All PRs listed above were reviewed for implementation details, code changes, and review comments to inform this test plan.

---

### **I. Motivation and Requirements Review (QE Review Guidelines)**

#### **1. Requirement & User Story Review Checklist**

| Check                                  | Done | Details/Notes                                                                                                                                                                           | Comments |
| :------------------------------------- | :--- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :------- |
| **Review Requirements**                | [x]  | Reviewed bug: Users cannot disable common-instancetypes deployment via HCO. Need configurable option. |          |
| **Understand Value**                   | [x]  | Allows users to manage their own instancetypes instead of using bundled ones. |          |
| **Customer Use Cases**                 | [x]  | Enterprise customers with custom instancetype requirements. |          |
| **Testability**                        | [x]  | Testable by setting CommonInstancetypesDeployment in HCO CR. |          |
| **Acceptance Criteria**                | [x]  | Users should be able to disable common-instancetypes deployment via HCO configuration. |          |
| **Non-Functional Requirements (NFRs)** | [x]  | Configurability; Flexibility. |          |

#### **2. Technology and Design Review**

| Check                            | Done | Details/Notes                                                                                                                           | Comments |
| :------------------------------- | :--- | :------------------------------------------------------------------------------------------------------------------------------------------------------ | :------- |
| **Developer Handoff/QE Kickoff** | [ ]  | Pending - need walkthrough on CommonInstancetypesDeployment field. |          |
| **Technology Challenges**        | [x]  | Fix introduces CommonInstancetypesDeployment field in HCO spec to control deployment. |          |
| **Test Environment Needs**       | [x]  | Standard cluster. |          |
| **API Extensions**               | [x]  | New field: spec.commonInstancetypesDeployment with values: Enabled, Disabled. |          |
| **Topology Considerations**      | [x]  | Standard topology. |          |

### **II. Software Test Plan (STP)**

#### **1. Scope of Testing**

**In Scope:**

- Verify CommonInstancetypesDeployment field accepts valid values
- Verify setting to Disabled removes common-instancetypes
- Verify setting to Enabled deploys common-instancetypes
- Verify default behavior (enabled)
- Verify HCO reconciliation respects setting

**Document Conventions:** 
- HCO: HyperConverged Cluster Operator
- common-instancetypes: Bundled VM instance types (predefined VM sizes)
- CommonInstancetypesDeployment: New HCO spec field

#### **2. Testing Goals**

- [ ] Verify common-instancetypes can be disabled
- [ ] Confirm deployment state matches configuration
- [ ] Validate field validation and defaults

#### **3. Non-Goals (Testing Scope Exclusions)**

| Non-Goal                               | Rationale              | PM/ Lead Agreement |
| :------------------------------------- | :--------------------- | :----------------- |
| Testing custom instancetypes           | Separate feature | [ ] Name/Date      |
| Performance testing                    | Not relevant | [ ] Name/Date      |
| Testing instancetype functionality     | Covered elsewhere | [ ] Name/Date      |

#### **4. Test Strategy**

##### **A. Types of Testing**

| Item (Testing Type)            | Applicable (Y/N or N/A) | Comments |
| :----------------------------- | :---------------------- | :------- |
| Functional Testing             | Y                       | Core configuration |
| Automation Testing             | Y                       | Must be automated |
| Performance Testing            | N/A                     | Not relevant |
| Security Testing               | N/A                     | No security changes |
| Usability Testing              | N/A                     | No UI changes |
| Compatibility Testing          | N/A                     | Standard feature |
| Regression Testing             | Y                       | HCO configuration |
| Upgrade Testing                | Y                       | Test setting preserved |
| Backward Compatibility Testing | Y                       | Default behavior |

##### **B. Potential Areas to Consider**

| Item                   | Description                                                                                                        | Applicable (Y/N or N/A) | Comment |
| :--------------------- | :----------------------------------------------------------------------------------------------------------------- | :---------------------- | :------ |
| **Dependencies**       | SSP (Scheduling, Scale and Performance) operator                                                                   | Y                       | Manages instancetypes |
| **Monitoring**         | HCO status conditions                                                                                              | Y                       | Track deployment state |
| **Cross Integrations** | HCO, SSP, instancetypes                                                                                            | Y                       | All involved |
| **UI**                 | May show/hide instancetypes                                                                                        | Y                       | Check UI impact |

#### **5. Test Environment**

| Environment Component                         | Configuration | Specification Examples                                                                        |
| :-------------------------------------------- | :------------ | :-------------------------------------------------------------------------------------------- |
| **Cluster Topology**                          | Required      | Standard cluster                                                                              |
| **OCP & OpenShift Virtualization Version(s)** | Required      | OCP 4.17+ with OpenShift Virtualization 4.17+                                                 |
| **CPU Virtualization**                        | Required      | VT-x or AMD-V enabled                                                                         |
| **Compute Resources**                         | Required      | Standard requirements                                                                         |
| **Special Hardware**                          | N/A           | None required                                                                                 |
| **Storage**                                   | Required      | Standard storage                                                                              |
| **Network**                                   | Required      | OVN-Kubernetes                                                                                |
| **Required Operators**                        | Required      | OpenShift Virtualization Operator                                                             |
| **Platform**                                  | Required      | Any supported platform                                                                        |
| **Special Configurations**                    | N/A           | None                                                                                          |

#### **5.5. Testing Tools & Frameworks**

| Category           | Tools/Frameworks                                                  |
| :----------------- | :---------------------------------------------------------------- |
| **Test Framework** | Standard OpenShift Virtualization test framework                  |
| **CI/CD**          | Standard CI pipeline                                              |
| **Other Tools**    | kubectl, oc for CR manipulation                                   |

#### **6. Entry and Exit Criteria**

##### **A. Entry Criteria**

- [x] Requirements and design documents are **approved and merged**
- [ ] Test environment is **set up and configured**
- [ ] Test cases are **reviewed and approved**
- [ ] PR #3471 is merged

##### **B. Exit Criteria**

- [ ] Common-instancetypes can be disabled
- [ ] Default behavior preserved
- [ ] Setting persists across reconciliation
- [ ] Test automation merged

#### **7. Risks and Limitations**

| Risk Category        | Specific Risk for This Feature                                                                                 | Mitigation Strategy                                                                            | Status |
| :------------------- | :------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------- | :----- |
| Timeline/Schedule    | Standard timeline                                                                                              | N/A                                                                                            | [ ]    |
| Test Coverage        | Need to test all states                                                                                        | Test enabled, disabled, default                                                                | [ ]    |
| Test Environment     | Standard environment                                                                                           | N/A                                                                                            | [ ]    |
| Untestable Aspects   | N/A                                                                                                            | N/A                                                                                            | [ ]    |
| Resource Constraints | N/A                                                                                                            | N/A                                                                                            | [ ]    |
| Dependencies         | SSP operator behavior                                                                                          | Test with actual SSP                                                                           | [ ]    |
| Other                | N/A                                                                                                            | N/A                                                                                            | [ ]    |

#### **8. Known Limitations**

- Feature requires HCO version with CommonInstancetypesDeployment support
- Disabling removes all common instancetypes (users must provide their own if needed)

---

### **III. Test Case Descriptions & Traceability**

#### **1. Requirements-to-Tests Mapping**

| Requirement ID | Requirement Summary   | Test Scenario(s)                                           | Test Type(s)                | Priority |
| :------------- | :-------------------- | :--------------------------------------------------------- | :-------------------------- | :------- |
| CNV-61256      | Disable instancetypes | Set CommonInstancetypesDeployment: Disabled | Functional, Tier 1 | P0 |
| CNV-61256      | Enable instancetypes | Set CommonInstancetypesDeployment: Enabled | Functional, Tier 1 | P0 |
| CNV-61256      | Default behavior | Verify default is Enabled | Functional, Tier 1 | P0 |
| CNV-61256      | Persistence | Setting persists after HCO reconcile | Functional, Tier 1 | P1 |

#### **Test Scenarios - Tier 1 (Functional)**

**Scenario 1: Disable Common-Instancetypes**
- **Preconditions:** Fresh CNV installation with common-instancetypes
- **Steps:**
  1. Verify common-instancetypes deployed (default)
  2. Edit HCO CR: set commonInstancetypesDeployment: Disabled
  3. Wait for HCO reconciliation
  4. Check for common-instancetype resources
- **Expected Result:** All common-instancetype resources removed
- **Priority:** P0

**Scenario 2: Enable Common-Instancetypes**
- **Preconditions:** Common-instancetypes disabled
- **Steps:**
  1. Verify common-instancetypes not deployed
  2. Edit HCO CR: set commonInstancetypesDeployment: Enabled
  3. Wait for HCO reconciliation
  4. Check for common-instancetype resources
- **Expected Result:** Common-instancetype resources deployed
- **Priority:** P0

**Scenario 3: Default Behavior**
- **Preconditions:** Fresh CNV installation
- **Steps:**
  1. Install CNV without setting commonInstancetypesDeployment
  2. Check HCO status
  3. Verify common-instancetypes deployed
- **Expected Result:** Default is enabled, instancetypes deployed
- **Priority:** P0

**Scenario 4: Setting Persistence**
- **Preconditions:** Setting configured
- **Steps:**
  1. Set commonInstancetypesDeployment: Disabled
  2. Trigger HCO reconciliation (edit another field)
  3. Verify common-instancetypes remain disabled
- **Expected Result:** Setting persists across reconciliations
- **Priority:** P1

#### **Test Scenarios - Tier 2 (E2E)**

**Scenario 5: Upgrade with Disabled Setting**
- **Preconditions:** CNV with common-instancetypes disabled
- **Steps:**
  1. Disable common-instancetypes
  2. Upgrade CNV
  3. Verify setting preserved
  4. Verify instancetypes still disabled
- **Expected Result:** Setting preserved through upgrade
- **Priority:** P2

---

### **IV. Sign-off and Approval**

This Software Test Plan requires approval from the following stakeholders:

- **Reviewers:**
  - [QE Team Lead]
  - [HCO Team Representative]
- **Approvers:**
  - [QE Manager]
  - [Product Owner]

