# Prompt Engineering Techniques for AI Coding Assistants

> **Note:** This document is educational and analytical. All techniques and examples are derived from publicly available research and documentation. No proprietary or confidential material is included.

---

## Overview

**Prompt engineering** is the practice of structuring inputs to a large language model (LLM) in ways that produce better, more reliable outputs. For AI coding assistants, effective prompt engineering can mean the difference between a model that reliably produces correct, idiomatic code and one that makes frequent errors.

---

## 1. System Prompts

A **system prompt** is an instruction block supplied to the model before the user's message. It establishes:

- The model's **role and persona** (e.g., "You are an expert software engineer")
- **Rules and constraints** (e.g., "Never delete files without confirmation")
- **Available tools and their descriptions**
- **Output format requirements** (e.g., "Always respond with valid JSON")
- **Contextual background** (e.g., the project's coding conventions)

### Example System Prompt Structure

```
You are a senior software engineer assisting with a TypeScript codebase.

Rules:
- Always write type-safe code
- Prefer functional patterns
- Run tests after every change
- Ask for clarification if the task is ambiguous

Available tools:
- read_file(path): read a file from the codebase
- write_file(path, content): write content to a file
- run_command(cmd): execute a shell command
```

---

## 2. Few-Shot Examples

**Few-shot prompting** provides the model with examples of the desired input/output behavior before presenting the actual task. This is particularly effective for tasks with a specific format or style.

### Example

```
Here are examples of how to document functions:

Input: function add(a, b) { return a + b; }
Output:
/**
 * Adds two numbers together.
 * @param a - The first number
 * @param b - The second number
 * @returns The sum of a and b
 */
function add(a: number, b: number): number { return a + b; }

---

Now document this function:
Input: function fetchUser(id) { return db.users.findById(id); }
```

---

## 3. Chain-of-Thought (CoT) Prompting

**Chain-of-thought prompting** encourages the model to reason step-by-step before producing a final answer. This is especially useful for complex debugging or refactoring tasks.

### Technique: Explicit CoT Instruction

```
Before writing any code, think through the following:
1. What is the root cause of this bug?
2. What are all the places in the codebase that might be affected?
3. What is the minimal change that fixes the issue without introducing regressions?

Then, implement your solution.
```

### Technique: Scratchpad / Thinking Block

Some models support a dedicated "thinking" or scratchpad section where they reason privately before producing a response. This reduces the chance of errors caused by the model committing prematurely to an approach.

---

## 4. Role Specification

Assigning the model a specific **role** can significantly improve output quality. The role anchors the model's behavior and vocabulary.

| Role | Effect |
|---|---|
| "You are a TypeScript expert" | Produces idiomatic TypeScript, avoids JavaScript-isms |
| "You are a security engineer" | Focuses on security implications, flags vulnerabilities |
| "You are a senior code reviewer" | Provides constructive criticism, flags anti-patterns |
| "You are a test engineer" | Focuses on coverage, edge cases, and testability |

---

## 5. Structured Output Formats

Requiring structured output (JSON, XML, Markdown with specific headings) makes model responses easier to parse and integrate into tools.

### Example: Require JSON for tool calls

```
When you need to use a tool, always respond with a JSON object in this format:
{
  "tool": "<tool_name>",
  "arguments": { ... }
}

Do not include any text outside of the JSON object when making a tool call.
```

---

## 6. Constraint and Guardrail Prompts

**Guardrails** prevent the model from taking harmful or undesirable actions. For coding assistants, common guardrails include:

- **Destructive operation confirmation**: "Always ask the user before deleting files or running `DROP TABLE` statements."
- **Scope limitation**: "Only modify files within the `src/` directory."
- **Dependency restrictions**: "Do not add new npm dependencies without approval."
- **Secrets handling**: "Never log or output API keys, passwords, or tokens."

### Example Guardrail Instructions

```
Important constraints:
- Never modify files outside the project root
- Never run commands that modify system state (e.g., apt install) without confirmation
- If you encounter a secret or credential in a file, do not include it in your response
```

---

## 7. Context Injection

AI coding assistants often inject additional context into the prompt dynamically:

- **File contents** – The relevant source files for the current task
- **Error messages** – Stack traces and compiler errors
- **Test results** – Which tests passed or failed
- **Git diff** – Recent changes to the codebase
- **Documentation** – Relevant API docs or README sections

Effective context injection involves selecting the *most relevant* information without exceeding the model's context window.

---

## 8. Iterative Refinement Prompts

Rather than asking for a complete solution in one shot, iterative refinement breaks the task into smaller steps:

```
Step 1: Read the failing test and identify what behavior it expects.
Step 2: Identify the production code that should implement this behavior.
Step 3: Write the minimal implementation that makes the test pass.
Step 4: Run the test suite to confirm.
Step 5: Refactor if needed.
```

---

## 9. Negative Instructions

Explicitly telling the model what **not** to do can be as important as telling it what to do:

```
Do NOT:
- Add unnecessary comments
- Change unrelated code
- Use `any` type in TypeScript
- Introduce new dependencies for simple utility functions
```

---

## 10. Self-Consistency and Verification Prompts

After producing a solution, prompting the model to verify its own work reduces errors:

```
After writing the code:
1. Re-read the original requirements
2. Check that every requirement is addressed
3. Look for any syntax errors or typos
4. Confirm that the code will not break any existing functionality
```

---

## References

- Wei et al. (2022), "Chain-of-Thought Prompting Elicits Reasoning in Large Language Models": https://arxiv.org/abs/2201.11903
- Brown et al. (2020), "Language Models are Few-Shot Learners" (GPT-3 paper): https://arxiv.org/abs/2005.14165
- Anthropic prompt engineering documentation (public): https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview
- OpenAI prompt engineering guide (public): https://platform.openai.com/docs/guides/prompt-engineering
