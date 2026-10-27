---
name: requirements-analyzer
description: Analyze a requirements document (.md or .txt), optionally with a second file of the user's own notes on it. Goes through the document paragraph by paragraph, quotes each original paragraph, and turns it into numbered functional requirements with clarifications, formulas, JSON request/response structures and ASCII illustrations where useful. Asks clarifying questions before writing if the notes don't fill the gaps. Use when the user gives a requirements, problem or spec file to analyze, or runs /requirements-analyzer.
argument-hint: <requirements-file> [notes-file]
---

# Requirements Analyzer

Turn a requirements document into clear, numbered functional requirements, one original paragraph at a time.

Arguments: `$ARGUMENTS`

## Inputs

1. **Main requirements file** (required): the problem or spec, `.md` or `.txt`.
2. **Notes file** (optional): a separate `.md` or `.txt` file holding the user's own input on the main file (answers, decisions, extra details).

The user says which file is which, either in the arguments or in their message. If it isn't clear which file is the main one and which is the notes, or a path doesn't exist, ask. Do not guess.

## Process

### 1. Read everything carefully

- Read the **whole** main file. Do not skim and do not skip any part of it.
- If a notes file was given, read the whole notes file too.
- Do not use outside assumptions. Every requirement must come from the main file or the notes file.

### 2. Split the main file into paragraphs

- A paragraph is a block of text separated by a blank line. A heading belongs to the paragraph under it. A list, table, code block or JSON block belongs to the paragraph that introduces it.
- Number the paragraphs in order: P1, P2, P3, ...
- Keep every paragraph, including intro and context ones. If a paragraph contains no requirement, still list it and say so.

### 3. Find the gaps, then ask before writing

For each paragraph, look for anything that is unclear, missing or could be read more than one way. For example:

- Undefined terms, or the same thing called by different names
- Missing field names, data types, units, ranges or default values
- Formulas with unclear inputs, rounding, order of operations or edge cases (division by zero, negatives, empty lists)
- Request or response shapes that aren't fully specified
- Missing validation rules, error cases or status codes
- Contradictions between paragraphs, or between the main file and the notes

Check whether the notes file answers each gap.

**If any gaps are still open after reading the notes, stop. Do not write the output file yet.** Ask the user all the open questions in one message:

- Group the questions by paragraph (`P3: ...`).
- Quote the part of the original text each question is about.
- Where it helps, offer the likely options so the user can choose.

Wait for the answers, then write the output. If there are no open gaps, go straight to writing.

### 4. Write the output file

Write the analysis to a new file **next to the main file**, named `<main-file-name>.requirements.md`. For example, `docs/orders.txt` becomes `docs/orders.requirements.md`. If that file already exists, ask before overwriting it.

Use the template below.

### 5. Report in chat

Reply with a short summary, not the full analysis:

- The output file path
- The number of paragraphs and functional requirements
- Any conflicts found between the main file and the notes, and how they were resolved

## Output template

````markdown
# Requirements Analysis: <document title or file name>

**Source:** `<main file path>`
**Notes:** `<notes file path>` (or "none")

## Summary

<2 to 4 sentences: what the system or feature does, as described in the document.>

## Glossary

| Term | Meaning |
|------|---------|
| <term from the document> | <definition, from the document or the notes> |

---

## P1: <short title>

### Original

> <the paragraph, quoted word for word from the main file>

### Functional requirements

- **FR-1.1** The system shall <one testable behavior>.
- **FR-1.2** The system shall <...>.

### Clarifications

- <what the paragraph means in precise terms: exact fields, types, limits, defaults, edge cases>
- <answers taken from the notes file or from the user, marked "(from notes)" or "(from user)">

### Formulas

<Only if the paragraph has a calculation. Write the formula, define every variable, and add a worked example.>

```
total = sum(item.price * item.quantity) - discount

where:
  item.price     decimal, >= 0
  item.quantity  integer, >= 1
  discount       decimal, >= 0, cannot be more than the subtotal

example: (10.00 * 2) + (5.00 * 1) - 3.00 = 22.00
```

### Request / response

<Only if the paragraph describes input, output or an API. Show the full structure with realistic values, and list the fields.>

```json
{
  "customer_id": 42,
  "items": [
    { "product_id": 7, "quantity": 2 }
  ]
}
```

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| customer_id | integer | yes | must exist |

### Illustration

<Only if a flow, state change, relationship or layout is easier to understand as a picture. Use ASCII.>

```
[Cart] --checkout--> [Pending] --pay--> [Paid] --ship--> [Shipped]
                         |
                         +--cancel--> [Cancelled]
```

---

## P2: <short title>

...

---

## Requirements index

| ID | Requirement | Paragraph |
|----|-------------|-----------|
| FR-1.1 | <short version> | P1 |
````

## Rules

- **Quote the original word for word.** Do not reword, shorten or fix the source text in the "Original" block.
- **One behavior per requirement.** Each FR describes one thing and can be tested on its own. Start it with "The system shall".
- **Number requirements by paragraph:** FR-<paragraph>.<n>, so every requirement traces back to its source.
- **Include only the sections that apply.** Leave out Formulas, Request / response or Illustration for a paragraph that has none of them. Do not add placeholder text.
- **Formulas:** always define every variable with its type and unit, state rounding and edge cases, and give a worked example with real numbers.
- **JSON:** use realistic example values and `snake_case` keys, unless the document shows a different style. Then follow the document.
- **ASCII illustrations:** keep them under 80 characters wide, and only draw one when it makes something clearer.
- **Paragraph with no requirements:** keep the "Original" block and write "No functional requirements. Context only." under it.
- **Never invent requirements.** If something is not in the main file, the notes or the user's answers, it is a gap. Ask about it; do not fill it in.
