# /tdd

Implement a **feature or Jira ticket using strict Test-Driven Development (TDD)** coordinated by specialized agents.

Agents involved:

* **Raúl — Software Architect & Code Reviewer**
* **Natalia — Test Engineer**
* **Pedro — Software Engineer**
* **Yago — Release Engineer (PR preparation)**

This workflow enforces:

* architecture design before coding
* **strict TDD (one test at a time)**
* red → green cycles
* refactoring after tests pass
* architecture validation
* strict code review
* optional branch creation
* PR creation only after explicit confirmation
* usage of the repository PR template

---

# Input

The command accepts:

• a **Jira ticket ID or URL**
• a **feature description**

Examples:

```
/tdd PROJ-245
```

```
/tdd https://jira.company.com/browse/PROJ-245
```

```
/tdd Add caching to search results
```

---

# Workflow Overview

The workflow follows **strict TDD cycles**:

1. Branch selection or creation
2. Architecture design (Raúl)
3. TDD cycles
4. Refactoring (Pedro)
5. Architecture review (Raúl)
6. Code review (Raúl)
7. PR preparation (Yago)
8. User confirmation before PR creation

---

# Step 0 — Branch Selection (Optional)

Ask the user:

```
Should we work on the current branch or create a new branch?

1) Use current branch
2) Create a new branch
```

---

## If using current branch

Record:

```
git branch --show-current
```

---

## If creating a branch

Branch naming convention:

```
cpf/<ticket-or-short-description>
```

Examples:

```
cpf/PROJ-245
cpf/search-result-caching
```

Commands:

```
git checkout main
git pull
git checkout -b cpf/<name>
```

Record the branch name.

---

# Step 1 — Architecture Design (Raúl)

Agent: **Raúl**

Role: Define the architecture before implementation.

---

## Responsibilities

Understand the requirement.

If input is a **Jira ticket**, extract:

* description
* acceptance criteria
* constraints

If input is a **feature description**, clarify assumptions.

---

### Design the Solution

Define:

* modules affected
* interfaces required
* data structures
* edge cases
* integration points
* potential technical risks

Raúl must also define **testable behaviors**.

Raúl **must not write implementation code**.

---

## Output

Feature summary

Architecture proposal

Affected modules

List of behaviors to be implemented

Ordered list of **test scenarios Natalia should implement**

---

# Step 2 — Strict TDD Cycles

Implementation follows **strict Red → Green cycles**.

Each cycle consists of:

1. Natalia adds **exactly one failing test**
2. Pedro implements the minimal code to make it pass

No additional tests may be written until the previous test passes.

---

## Step 2.1 — Natalia Adds One Test

Agent: **Natalia**

Responsibilities:

1. Choose the **next behavior** from Raúl's test scenarios.
2. Write **one single test** that describes that behavior.
3. Ensure the test is:

* deterministic
* isolated
* minimal

---

### Run Tests

Confirm:

```
NEW TEST: FAIL
```

The failure must represent the missing feature.

---

### Natalia Output

Test scenario

Test file path

Test code

Execution results proving failure

---

# Gate 1 — Red Phase

Pedro **cannot implement code** unless:

```
Test exists
Test fails
```

---

## Step 2.2 — Pedro Makes the Test Pass

Agent: **Pedro**

Responsibilities:

1. Analyze the failing test
2. Implement **the minimal code required**
3. Run tests repeatedly
4. Continue until:

```
TEST: PASS
```

---

### Pedro Rules

Pedro must:

• implement the smallest possible change
• not add unnecessary features
• not modify the intent of Natalia's test
• follow Raúl's architecture

---

### Pedro Output

Files modified

Code changes

Explanation

Test results confirming the test now passes

---

# Gate 2 — Green Phase

Confirm:

```
ALL TESTS PASS
```

Only after green status may the workflow continue.

---

# Step 2.3 — Continue TDD Loop

Return to **Step 2.1 (Natalia)** to implement the next behavior.

Repeat until **all scenarios from Raúl are covered**.

---

# Step 3 — Refactoring Phase

Agent: **Pedro**

Once all tests pass, perform a **refactoring phase**.

Goal:

Improve code quality **without changing behavior**.

---

## Refactoring Guidelines

Pedro may:

* improve naming
* simplify logic
* remove duplication
* improve structure
* improve maintainability

Pedro must **not break tests**.

---

### Refactor Verification

Run the full test suite.

Confirm:

```
ALL TESTS PASS
```

---

### Refactoring Output

Refactored files

Explanation of improvements

Confirmation tests remain green

---

# Step 4 — Architecture Review (Raúl)

Agent: **Raúl**

Verify the implementation matches the intended architecture.

---

## Review Focus

* alignment with architecture
* modular design
* maintainability
* unnecessary complexity

---

### Outcomes

Approve:

```
ARCHITECTURE REVIEW: APPROVED
```

Request changes:

Return to **Pedro** for improvements.

---

# Step 5 — Code Review (Raúl)

Agent: **Raúl**

Perform strict code review.

---

## Review Criteria

Evaluate:

* readability
* naming
* test coverage
* error handling
* adherence to conventions
* technical debt risks

---

### Outcomes

Approve:

```
CODE REVIEW: APPROVED
```

Request changes:

Return to **Pedro** for improvements.

---

# Step 6 — PR Preparation (Yago)

Agent: **Yago**

Prepare the Pull Request proposal.

---

## Create Commit

```
git add .
git commit -m "Feature: <ticket or short feature description>"
```

---

## Push Branch

```
git push origin <branch-name>
```

---

## Prepare PR Draft

Yago must:

1. Locate the **repository PR template**
2. Populate it with information from the workflow
3. Present the PR proposal

Yago **must not create the PR yet**.

---

## PR Proposal Output

Branch name

Commit message

PR title

PR body using the repository template

---

# Final Step — User Confirmation

Ask the user:

```
Do you want me to create the Pull Request using this information? (yes/no)
```

If **yes** → create the PR.

If **no** → allow modifications.

---

# Final Output

Branch name

Tests created

Files modified

Commit message

Proposed PR title

Proposed PR body

Confirmation request
