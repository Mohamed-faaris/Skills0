---
name: repo-debugger
description: Use when an existing repository has failing tests, incorrect outputs, runtime errors, or unexpected database behavior and the task is to locate and minimally correct verified implementation bugs.
---

# Repository Debugger

## Core objective

Debug the existing code only. Find the actual cause of the reported failure and correct the smallest responsible implementation detail while preserving the existing algorithm, architecture, and intended behavior.

**Evidence first. No guessing. No invented code. Debug; do not redesign.**

These rules apply to every language, framework, repository, and database. They are mandatory throughout the task.

## Non-negotiable rules

1. **Never hallucinate or predict code.** Do not invent source code, function names, variables, schemas, collection names, fields, query operators, return values, or runtime behavior.
2. **Ask when information is missing.** Request the exact file, function, screenshot, complete problem statement, test input/output, database schema/sample document, query, or log needed to continue. Ask focused questions; do not fill gaps with assumptions.
3. **Debug only.** Do not add features, redesign behavior, refactor, optimize unrelated code, or create new abstractions.
4. **Do not create new functions or methods to solve a bug.** Correct the existing call, argument, condition, reference, query, or other faulty line in place. Do not introduce helper functions, classes, modules, wrappers, or new execution paths. If the evidence indicates that a new function or substantial redesign is genuinely required, stop and report that the issue falls outside this minimal-debugging approach; ask before proceeding.
5. **Preserve existing logic.** Keep the original algorithm and control flow. Change only the incorrect implementation detail demonstrated by evidence.
6. **Expect multiple independent bugs.** Several bugs may exist across files or inside the same function, including multiple mistakes in one call or expression. Continue inspecting relevant paths after finding the first bug.
7. **No total line-change cap.** Many small fixes can add up to more than 30 lines. Judge each bug separately. A typical bug is an incorrect call, argument, condition, variable/property reference, or query detail requiring roughly 1–3 changed lines. A larger individual fix requires renewed diagnosis and an explicit explanation.
8. **Do not force the minor-bug theory.** If evidence points to an algorithmic, architectural, data-integrity, or otherwise substantial issue, report it clearly. Do not disguise it as a small patch.
9. **Do not claim verification without evidence.** Run tests only when the environment and required commands are available. Otherwise report exactly what remains unverified.

## MongoDB object-query investigation priority

When the repository uses MongoDB, Mongoose, or a MongoDB-compatible query layer, inspect object-based queries early and carefully because query construction and selection are a high-priority investigation area. This is a priority for inspection, **not an assumption that MongoDB is necessarily the cause**.

Use only the actual code and database evidence supplied or inspected. Check relevant details such as:

- Which query method is actually called (\`find\`, \`findOne\`, \`findById\`, \`updateOne\`, \`findOneAndUpdate\`, \`deleteOne\`, \`aggregate\`, or another method present in the code).
- Whether the chosen method matches the existing caller's intent and input type.
- Filter object keys, field names, nesting, and values.
- MongoDB operators and their placement, such as \`$in\`, \`$or\`, \`$and\`, \`$set\`, \`$unset\`, and \`$exists\`, when they appear in the real code.
- ObjectId versus string handling, only where the actual schema/input establishes the expected type.
- Projection, sort, pagination, and update options when relevant to the failing behavior.
- Whether the code handles the actual result shape and asynchronous behavior correctly.
- Whether a query is built correctly before it reaches the database.

Do not invent a collection, schema, field, operator, query, or replacement method. If the schema, sample document, expected query behavior, or relevant model definition is missing, ask the user for it. Do not assume that a query is wrong merely because it looks unfamiliar.

## Workflow

### 1. Collect the initial evidence

Accept evidence in stages. The user may provide repository files, code screenshots, test cases, expected and actual output, stack traces, database details, or execution logs across multiple messages.

First identify:

- The failing behavior and exact test case.
- Expected versus actual result.
- The relevant repository, files, and function(s), if known.
- The runtime/database context that is explicitly established.

If any required information is absent, ask for the specific missing item before diagnosing. For screenshots, request a clear image showing the complete relevant function and enough surrounding context to identify its inputs and callers.

### 2. Inspect existing code

Read the relevant existing implementation and trace the actual call path. Follow existing references into related functions or model definitions only as needed. Do not assume the bug is in the first file shown or in the database layer.

### 3. Build a bug inventory

Track each distinct issue separately. Mark each as:

- **Confirmed** — directly supported by source and test/runtime evidence.
- **Needs evidence** — a plausible location, but not yet proven.
- **Not a minor implementation bug** — evidence indicates a larger underlying problem.

Do not present a hypothesis as a confirmed bug. Do not stop after the first bug if other relevant paths remain unexplored.

### 4. Request targeted diagnostics when necessary

If static inspection cannot establish the cause, ask for the missing evidence or provide temporary logs based on actual identifiers and expressions in the supplied source.

For JavaScript, a simple object inspection can be:

\`\`\`javascript
console.log("DEBUG objectName:", objectName);
\`\`\`

For nested objects:

\`\`\`javascript
console.dir(objectName, { depth: null });
\`\`\`

Use the real variable names from the user's code. Never invent \`objectName\`, \`input\`, \`result\`, or other identifiers as if they exist; placeholders must be clearly labelled, or avoid giving a snippet until the real names are known. State the exact file and insertion point, what to run, and what output to return. Keep instrumentation narrow and temporary. Do not add helper functions for logging. Remove temporary logs after diagnosis when appropriate, and identify their removal explicitly.

Use the corresponding existing logging mechanism for other languages. Do not add permanent logging infrastructure.

### 5. Apply only verified in-place corrections

For each confirmed bug, correct the smallest existing expression or statement that causes it. Prefer changing a wrong function call, argument, condition, property reference, query method, filter key, operator, or result handling in place.

Do not create new functions, methods, classes, files, modules, helpers, wrappers, or dependencies. Do not rewrite an entire function when a one-line correction is sufficient. Do not change unrelated code. If no safe in-place correction is supported by evidence, stop and ask the user rather than generating a new implementation.

### 6. Validate and inspect the diff

After the user applies or authorizes the patch:

- Re-run the exact failing case when possible.
- Check relevant existing passing cases and focused boundary cases.
- Confirm the observed result matches the expected behavior.
- Inspect the diff for unrelated edits, accidental rewrites, and leftover temporary logs.
- Report tests that could not be run.

Do not broaden the task into unrelated cleanup.

## Required response format

### Findings

- Confirmed bugs: count
- Needs-evidence items: count
- Files/functions involved

### Bug N — short description

- **Status:** Confirmed / Needs evidence / Not a minor implementation bug
- **Location:** exact file, function, and line or nearby expression
- **Evidence:** the supplied source/test/log that establishes the issue
- **Current code:** exact existing code, when available
- **Minimal correction:** exact replacement, only when confirmed
- **Reason:** why this specific change fixes the demonstrated issue

For issues needing more information, state the exact file, screenshot, schema, sample document, query output, or log required. Do not provide speculative replacement code.

### Validation

- Tests actually run and their results
- Tests not run and why
- Diff/change summary
- Temporary logs to remove, if any

## Final operating rule

**Inspect what exists. Ask for what is missing. Prove each bug. Correct it in place. Preserve everything else.**
