# Java Unit Testing Skill

A house style for Java unit tests (JUnit 5 + AssertJ + Mockito), packaged as a
tool-agnostic Agent Skill.

## What it is

`SKILL.md` plus the `references/` directory are the entire skill. `SKILL.md`
holds the rules and stays high-level; the reference files hold detailed patterns
and load on demand. Any agent that supports the Agent Skills format reads
`SKILL.md` directly.

Two thin adapters route agents that use a different discovery convention to the
same `SKILL.md`. They contain no rules of their own:
- `AGENTS.md`: for agents that read an `AGENTS.md` file.
- `.kiro/steering/java-unit-testing.md`: a Kiro steering rule scoped to Java
  test files.

## Layout

```
java-unit-test-skill/
├── SKILL.md                     # rules and workflow
├── references/
│   ├── PATTERNS.md                # code for every test pattern
│   ├── ASSERTJ.md                 # AssertJ usage, traps, migration
│   ├── UNIT-TESTING-PRACTICES.md  # state-first testing, test doubles, determinism, coverage
│   └── CHECKLIST.md               # pre-flight checklist + pitfalls
├── versions.json                # assumed tool versions (update first when bumping)
├── LICENSE                      # Apache-2.0
├── AGENTS.md                    # adapter → SKILL.md
├── .kiro/steering/
│   └── java-unit-testing.md     # adapter → SKILL.md (scoped to *Test.java)
└── README.md
```

## Install

Place the `java-unit-test-skill/` folder where the agent looks for skills, then
point the agent at it. Common locations:
- A skills directory the agent scans (the whole folder), so `SKILL.md` is
  discovered automatically.
- The repository root, alongside `AGENTS.md`, for agents that read `AGENTS.md`.
- A project's `.kiro/steering/` directory (the steering adapter), for Kiro.

Keep the folder intact so the adapters can reach `SKILL.md` and `references/`.
