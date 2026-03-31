# Basic Prompt Patterns for AI Coding Assistants

> **Note:** This document contains educational examples of prompt patterns commonly used with AI coding assistants. All examples are illustrative and generic. No proprietary or confidential material is included.

---

## Overview

These annotated examples demonstrate prompt patterns that appear frequently in AI coding tool workflows. Understanding these patterns helps developers write more effective prompts and build better integrations.

---

## Pattern 1: The Task-Context-Format Pattern

**Structure:** Provide the task, then the relevant context, then specify the output format.

```
Task: Add input validation to the `createUser` function.

Context:
```typescript
// src/api/users.ts
export async function createUser(data: unknown) {
  const user = await db.users.create(data);
  return user;
}
```

The project uses Zod for validation. See `src/lib/schemas.ts` for existing schemas.

Format: Return only the updated function. Do not include the import statements.
```

**Why it works:**
- The model knows exactly what to do (task)
- The model has the relevant code (context)
- The model knows what to produce (format)

---

## Pattern 2: Step-by-Step Decomposition

**Structure:** Break the task into explicit numbered steps before asking the model to act.

```
Please complete the following steps in order:

1. Read `src/services/EmailService.ts`
2. Identify all methods that send emails
3. For each method, check whether it validates the recipient email address
4. List any methods that do NOT validate the recipient
5. Add validation to those methods using the `isValidEmail` utility from `src/utils/validators.ts`
6. Run the tests in `tests/EmailService.test.ts`
```

**Why it works:**
- Prevents the model from skipping exploration and jumping straight to code
- Creates a verifiable checklist of actions
- Reduces errors caused by incomplete understanding

---

## Pattern 3: Role + Constraint Prompt

**Structure:** Assign a role and state explicit constraints before the task.

```
You are a security engineer reviewing a Node.js web application.

Constraints:
- Focus ONLY on security issues, not general code quality
- Flag OWASP Top 10 vulnerabilities with their OWASP category
- Do not suggest architectural changes; focus on minimal fixes
- Rank findings by severity: Critical, High, Medium, Low

Please review the following authentication middleware:

```typescript
app.use((req, res, next) => {
  const token = req.query.token;
  if (token === process.env.SECRET_KEY) {
    next();
  } else {
    res.status(401).send('Unauthorized');
  }
});
```
```

**Why it works:**
- The role focuses the model on the relevant domain
- The constraints prevent irrelevant output
- The structured ranking makes findings actionable

---

## Pattern 4: The Rubber Duck Debug Pattern

**Structure:** Ask the model to explain the code before attempting to fix it.

```
I have a bug but I'm not sure what's causing it. 

Before suggesting any fixes, please:
1. Explain what this function is doing, line by line
2. Identify any assumptions the function makes about its inputs
3. Describe what would happen if those assumptions were violated

Here is the function:

```python
def process_items(items):
    result = []
    for i in range(len(items)):
        if items[i] > items[i+1]:
            result.append(items[i])
    return result
```

The error I'm seeing is: `IndexError: list index out of range`
```

**Why it works:**
- Forces the model to understand before acting
- The model's explanation often reveals the bug
- Reduces the chance of the model producing a plausible-but-wrong fix

---

## Pattern 5: Minimal Change Instruction

**Structure:** Explicitly tell the model to make the smallest possible change.

```
The following test is failing:

```
FAIL: test_user_login
AssertionError: Expected status 200, got 401
```

Here is the relevant production code:

```python
def login(username, password):
    user = User.query.filter_by(username=username).first()
    if user and user.password == password:
        return {"status": 200, "token": generate_token(user)}
    return {"status": 401}
```

Fix the failing test with the MINIMAL change possible. Do not refactor, rename, or change any behavior beyond what is needed to make this specific test pass.
```

**Why it works:**
- Prevents the model from making unnecessary changes
- Reduces the risk of introducing regressions
- Produces a focused, reviewable diff

---

## Pattern 6: Test-First Prompt

**Structure:** Provide the test first and ask the model to write code that passes it.

```
Here is a test I have written for a function that doesn't exist yet:

```typescript
describe('slugify', () => {
  it('converts spaces to hyphens', () => {
    expect(slugify('Hello World')).toBe('hello-world');
  });
  it('removes special characters', () => {
    expect(slugify('Hello, World!')).toBe('hello-world');
  });
  it('collapses multiple hyphens', () => {
    expect(slugify('Hello   World')).toBe('hello-world');
  });
  it('trims leading and trailing hyphens', () => {
    expect(slugify('  Hello World  ')).toBe('hello-world');
  });
});
```

Write the `slugify` function that makes all of these tests pass. The function should be in `src/utils/slugify.ts`.
```

**Why it works:**
- The test is the specification; the model has an unambiguous target
- Edge cases are already documented in the tests
- The model can verify its own solution against the tests

---

## Pattern 7: Incremental Refactoring

**Structure:** Ask for one specific refactoring at a time instead of a general "improve this code" prompt.

Instead of:
```
Improve this code.
```

Use:
```
Apply a SINGLE refactoring to this function: extract the email validation logic into a separate function named `validateEmailAddress`. Do not make any other changes.
```

**Why it works:**
- Each change is small, reviewable, and reversible
- The model focuses on one concern at a time
- Easier to catch mistakes in a small diff than a large rewrite

---

## Pattern 8: The "Before You Start" Checklist

**Structure:** Before tackling a complex task, ask the model to confirm its understanding.

```
Before you start making any changes, please answer these questions:

1. Which files will you need to modify?
2. Are there any tests that cover this functionality?
3. Are there any other parts of the codebase that might be affected by your changes?
4. Are there any edge cases or gotchas you've noticed?

After I confirm your understanding, you can proceed with the changes.
```

**Why it works:**
- Surfaces misunderstandings before they become bugs
- Creates a shared mental model between developer and assistant
- Allows the developer to correct the plan before execution

---

## Summary Table

| Pattern | Best For |
|---|---|
| Task-Context-Format | Clear, well-scoped tasks |
| Step-by-Step Decomposition | Complex multi-step workflows |
| Role + Constraint | Domain-specific analysis (security, performance, etc.) |
| Rubber Duck Debug | Debugging unfamiliar code |
| Minimal Change | Bug fixes with low regression risk |
| Test-First | Implementing new features with clear specs |
| Incremental Refactoring | Improving code quality safely |
| Before You Start Checklist | Large or risky changes |
