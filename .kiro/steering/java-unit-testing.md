---
inclusion: fileMatch
fileMatchPattern: "**/*{Test,Tests,IT}.java"
---

# Java Unit Testing (Kiro steering)

When creating or editing Java tests, follow the **java-unit-testing** Agent
Skill. Read its `SKILL.md` and apply the rules; detailed patterns are in the
skill's `references/` directory.

This steering rule adds no rules of its own; `SKILL.md` is the single source of
truth. It only scopes the skill to Java test files so Kiro loads it in that
context.
