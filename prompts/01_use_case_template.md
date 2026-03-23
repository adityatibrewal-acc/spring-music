# Use Case Document – Prompt Template

You are a **Business Analyst** creating a **clear, testable, and enterprise-ready use case document**.

## Instructions
- Use **simple, precise language**
- Avoid assumptions; do not add functionality not explicitly stated
- Number all steps and acceptance criteria
- Ensure acceptance criteria are **testable** and map to the flows
- Use **Markdown headings and bullet points**
- Keep the document professional and structured

---

## Context
- **System / Application Name**: <provide>
- **Business Domain**: <provide>
- **Objective / Problem Statement**: <brief description>
- **Intended Audience**: Business, Tech, QA

---

## 1. Use Case Overview
- **Use Case ID**:
- **Use Case Name**:
- **Description**:
- **Scope**:
- **Level** (User Goal / Sub-function):

---

## 2. Actors
- **Primary Actor**:
- **Secondary Actors**:
- **Stakeholders & Interests** (if applicable):

---

## 3. Preconditions
List all conditions that must be true before the use case starts.

---

## 4. Trigger
Describe the event that initiates this use case.

---

## 5. Main Success Scenario (Basic Flow)
Describe the happy path as numbered steps.
Each step should clearly show **user action** and **system response**.

1.
2.
3.

---

## 6. Alternate Flows
Describe valid variations from the main flow.
Reference the step number from the main flow.

- **AF-1** (From Step X):
- **AF-2** (From Step Y):

---

## 7. Exception / Error Flows
Describe error scenarios and how the system handles them.

- **EF-1**:
- **EF-2**:

---

## 8. Postconditions
- **Success Postconditions**:
- **Failure Postconditions**:

---

## 9. Business Rules
List rules, validations, thresholds, or policies that apply.

---

## 10. Acceptance Criteria ✅
Define when this use case is considered **complete and acceptable**.
Use **Given / When / Then** format.
Ensure coverage of:
- Happy path
- Alternate flows
- Error scenarios

### Acceptance Criteria
- **AC-1 (Happy Path)**  
  - Given  
  - When  
  - Then  

- **AC-2 (Validation / Error)**  
  - Given  
  - When  
  - Then  

- **AC-3 (Authorization / Edge Case)**  
  - Given  
  - When  
  - Then  

---

## 11. Non‑Functional Requirements (if applicable)
- Performance:
- Security:
- Audit / Logging:
- Usability:
- Availability:

---

## 12. Assumptions & Dependencies
- Assumptions:
- External systems / integrations:

---

## 13. Open Issues / Notes
List unresolved questions, risks, or follow-ups.

---

### Output Quality Check
Ensure the document:
- Is unambiguous and testable
- Can be directly used by **Dev and QA**
- Has acceptance criteria traceable to flows