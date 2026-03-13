# /fix

Fix a bug using a **strict Test-Driven Debugging workflow executed by specialized agents**.

The workflow includes:

* optional **bugfix branch creation**
* **failing test before implementation**
* **automated verification**
* **explicit user approval before creating a PR**
* **use of the repository Pull Request template**

---

# Input

The command accepts:

• a **Jira ticket ID or URL**
• a **bug description**

Examples:

```
/fix PROJ-123
```

```
/fix https://jira.company.com/browse/PROJ-123
```

```
/fix Search crashes when query contains emoji
```

---

# Workflow Overview

Execution order:

1. Branch selection or creation
2. Natalia reproduces the bug with a failing test
3. Pedro implements the fix
4. Pedro checks all tests are passing and lint command passes as well
5. Verify tests
6. Yago prepares the PR request
7. Wait for user confirmation before creating the PR

If **Natalia cannot reproduce the bug with a failing test, pause the workflow and tell the user.** 

Pedro **must not write code until a failing test exists.** unless the user confirms there is no way to reproduce the bug using automated tests

---

# Step 0 — Branch Selection (Optional)

Before starting development, determine which branch should be used.

Ask the user:

```
Should we work on the current branch or create a new bugfix branch?

1) Use current branch
2) Create a new bugfix branch
```

---

## If the user chooses **current branch**

Record the current branch name:

```
git branch --show-current
```

Continue the workflow using that branch.

---

## If the user chooses **create a new branch**

Create a bugfix branch.

Branch naming convention:

```
cpf/fix-<ticket-or-short-description>
```

Examples:

```
cpf/fix-PROJ-123
cpf/fix-search-emoji-crash
```

Commands:

```
git checkout main
git pull
git checkout -b cpf/fix-<name>
```

Record the branch name for later PR creation.

---

# Step 1 — Natalia (Automation Engineer)

Agent: **Natalia**

Role: debugging and automated testing specialist.

---

## Responsibilities

### Understand the Bug

If the input is a **Jira ticket**, extract:

* description
* reproduction steps
* expected behavior
* actual behavior

If the input is a **bug description**, analyze it directly.

---

### Investigate the Repository

Identify:

* likely modules involved
* relevant files
* existing tests

---

### Reproduce the Bug

Create a **deterministic automated test** that reproduces the issue.

Requirements:

• minimal
• isolated
• deterministic

---

### Verify Test Failure

Run tests and confirm:

```
NEW TEST: FAILS
```

The failure must represent the reported bug.

---

## Natalia Output

Bug summary

Root cause hypothesis

Affected files

Failing test path

Test code

Execution output confirming failure

---

# Gate 1 — TDD Enforcement

Pedro **cannot begin** until:

• failing test exists
• failing test fails consistently

If these conditions are not met:

```
PAUSE WORKFLOW AND TELL THE USER
```

---

# Step 2 — Pedro (Software Engineer)

Agent: **Pedro**

Role: implement the minimal fix.

Pedro begins **only after Gate 1 passes**.

---

## Responsibilities

1. Analyze Natalia’s failing test
2. Identify the root cause
3. Implement the **minimal code change required**
4. Run tests
5. Iterate until:

```
ALL TESTS PASS
```

---

## Rules

• do not modify Natalia's test logic
• avoid unrelated refactors
• keep the change minimal

---

## Pedro Output

Files modified

Code diff

Explanation of the fix

Test results confirming success

---

# Gate 2 — Verification

Run the full test suite.

Requirements:

```
Natalia test: PASS
All existing tests: PASS
```

If any test fails:

Pedro must resolve before continuing.

---

# Step 3 — Yago (Release Engineer)

Agent: **Yago**

Role: prepare the Pull Request proposal.

---

## Responsibilities

### Prepare Commit

```
git add .
git commit -m "Fix: <ticket or short bug description>"
```

---

### Push Branch

```
git push origin <branch-name>
```

---

### Prepare Pull Request Draft

Yago **must not create the PR yet**.

Instead:

1. Locate the repository **Pull Request template**
2. Populate it with the relevant information
3. Present the PR content for review

The repository template must be used exactly as defined.

---

## PR Proposal Output

Branch name

Commit message

Proposed PR title

Proposed PR body (filled using the repo template)

---

# Final Step — User Confirmation

Ask the user:

```
Do you want me to create the Pull Request using this information? (yes/no)
```

If the user answers **yes**:

Create the PR using the repository template.

If the user answers **no**:

Allow edits before creating the PR.

---

# Final Output

Branch name

Failing test path

Files modified

Commit message

Proposed PR content

Confirmation request
