# Project Creation Process

**KB Type:** HR (ab_)
**Maintained by:** Karen

This article documents the correct process for initializing a brand-new project from scratch. It is the authoritative reference for the PM and all subagents when starting a new project that has no existing codebase, specs, or KB.

---

## When to Use This Process

Use this process any time the PM is assigned a new project that:
- Has no existing `specs/` directory
- Has no `master_spec.md`
- Has only a project idea document (e.g., `siteIdea.md`, `projectIdea.md`, or a `prompt.md` pointing to one)

---

## Pre-Flight: Required Reading

Before beginning, the PM (or assigned agent) must read:
1. **`agent.md`** — team structure, IOC system, all agent roles and rules
2. **The project idea document** (e.g., `projectIdea.md`) — full understanding of what is being built

---

## Step 0: Setup

### 0a. Read agent.md
Understand the team structure, IOC system, and all behavioral rules before doing anything else.

### 0b. Read the project idea document

Understand what is being built completely before creating any files. After reading, interview the user to ensure full understanding of the project vision. Ask clarifying questions and use non-technical language to confirm your understanding.

#### Step 0b Extension: Reference Source Intake

If `refsrc/` directory exists in the project root, the new project involves forking or otherwise using existing source code as a base. In this case, perform the Reference Source Intake **before** the user interview and SDD creation:

**Trigger:** `refsrc/` directory present at project root

**Process:**
1. For each subfolder in `refsrc/` (each subfolder = one separate program):
   - Spawn a fresh **intake-team**: Pat + Tami + Chad working in parallel
   - Intake-team studies all source code in that program folder
   - Intake-team creates `<program>_ref_spec.md` in `refsrc/` documenting the application's features and functions in detail
2. PM verifies all ref_specs are complete before proceeding
3. Copy all ref_specs from `refsrc/` to `specs/reference/` (project record archive)

**Completion Gate:** SDD creation begins only after all reference specs are verified complete.

> **Note:** The intake-team uses fresh spawns per task to prevent context compaction. "Intake-team" is a functional label, not a standing team.

### 0c. Create supporting folder and file structure

Create the following directories:
```
specs/
specs/reference/         ← archive for reference source specs (if refsrc/ present)
specs/<module>/          ← one per planned module (see Step 2b)
specs/ui/
specs/ui/design/
kb/general/
kb/<project-name>/       ← project-specific KB (name from project)
hr/
refsrc/                  ← reference source code (if forking existing program)
```

Create stub files:
- `session.md` — session log (PM maintains)
- `kb/helpdesk.md` — KB index (Carl maintains)

### 0d. Prime the knowledge base

Before implementation begins, create KB articles in two categories:

> **Note:** If Reference Source Intake was performed in Step 0b, KB priming should happen **after** the ref_spec is complete. The source analysis provides valuable insights about the codebase that should inform KB content.

**1. Carry-over KBs** (`kb/general/`) — these are reusable documents that persist across projects and grow over time:
- **Usage/Patterns KB** — how to use the project's languages and APIs
  - One article per language or tool used in the project
  - Covers: syntax patterns, API usage, project-relevant examples
- **Best Practices KB** — coding quality and maintainability standards
  - One article per language or tool used in the project
  - Covers: naming conventions, organization, performance, common mistakes to avoid
  - **Required for every new project** — establishes the quality standard Chad implements to
- **Security KB** — security requirements for the language/platform
  - A single cross-cutting security article appropriate to the project type
  - Covers: XSS prevention, URL encoding, CSP, sensitive data handling, a per-IOC security checklist
  - **Required for every new project** — Chad must consult this before implementing any IOC that handles dynamic data or user input

> **Note:** These carry-over KBs live in `kb/general/` and are reused across projects. As the knowledge base grows, future projects benefit from accumulated expertise — Chad, Tami and Pat can query Carl for relevant knowledge rather than relearning patterns from scratch.

**2. Project-specific KB** (`kb/<project-name>/`) — unique systems and patterns for this project only:
- One article per key system (e.g., `content_system_kb.md`, `order_system_kb.md`)
- Covers: data formats, architecture decisions, troubleshooting

Update `kb/helpdesk.md` to index all new KB articles.

**Purpose:** KB priming ensures Chad, Tami and Pat can query Carl for relevant knowledge before implementing any IOC. Best practice and security KBs in particular set the bar for code quality from the very first IOC — they are not optional afterthoughts.

---

## Step 1: Karen Watches Over the Process

Karen monitors the project creation process to ensure it follows this KB. If the user directs Karen to update the process (e.g., due to tooling changes, new best practices, or agent workflow evolution), Karen will revise `kb/hr/ab_project_creation_kb.md` accordingly.

---

## Step 2: Project Creation Sequence

### 2a. Software Design Document (SDD)

Before creating the SDD, interview the user to gather detailed requirements. Use careful, probing questions to elicit the information needed. Since the user may not have full knowledge of technical components required, guide them through understanding what approaches and components are needed using clear, non-technical language.

Create `specs/<ProjectName>_sdd.md`. The SDD is a complete, detailed design document covering:
- Project overview and goals
- Scope (in scope / out of scope)
- Technical architecture and stack
- Site/system structure (pages, modules, file organization)
- Feature descriptions (detailed — enough to derive specs from)
- Data models
- UI/UX principles
- Workflow descriptions (e.g., order flow, content management workflow)
- Constraints and known limitations

The SDD is the source of truth for all design decisions. Specs are derived from the SDD, not the other way around.

Wait for user input before proceeding to the next step.

### 2b. Master Specification (`specs/master_spec.md`)

Based on the SDD, create `specs/master_spec.md` covering:
- Module list with IOC counts per module and status table
- Brief description of each module
- Module dependency/implementation order
- IOC status definitions
- Link to each module's spec file

Determine modules by grouping related features. A module should be:
- A cohesive area of functionality (e.g., Gallery, Order, Foundation)
- Large enough to warrant its own spec (usually ≥ 2 IOCs)
- Small enough to be understood in one sitting

### 2c. Feature Spec Files

For each module listed in `master_spec.md`, create `specs/<module>/<module>_spec.md` containing:
- Module overview
- IOC table (ID, name, file, status)
- Brief description of each IOC
- Pass criteria for the module

Store each spec in its own subdirectory: `specs/<module>/`.

### 2d. IOC Files

For each IOC in every spec, create `specs/<module>/<ioc_name>_ioc.md`. IOC files are **descriptive documents — no code**. Each IOC file must describe:
- Purpose (what this unit of work accomplishes)
- Functions/methods required (name, parameters, return value, behavior)
- Data structures involved
- Event handling (if applicable)
- CSS requirements (if applicable)
- Edge cases and error handling
- Dependencies on other IOCs (if applicable)
- Notes for the implementer

**Good IOC files prevent repeated back-and-forth.** A well-written IOC gives Chad everything needed to implement without asking questions.

---

### 2e. Dependency Chart (`specs/depc.md`)

After all IOC files are complete but before creating the Implementation Plan, create `specs/depc.md`. This file is a visual flowchart (using Mermaid or similar) showing the dependency relationships between all modules and IOCs in the project.

**Purpose:** Helps the PM and team understand:
- Which modules depend on other modules
- Which IOCs must be completed before others can begin
- Critical path through the implementation
- Where parallel work is possible

This ensures the Implementation Plan orders tasks correctly based on actual dependencies.

---

### 2f. Implementation Plan (`imp_plan.md`)

After all IOC files are written, create `imp_plan.md` at the project root. This file is a bullet-pointed task list ordered from most critical (blocks the most other work) to least critical (polish/content). It is derived from the SDD and specs and covers the full build from first file to production launch.

**What a good `imp_plan.md` includes:**
- Every implementation task needed to ship the project, derived from all IOC files
- Ordering by criticality/dependency: infrastructure and prerequisites first, polish and content last
- IOC references in brackets (`[F-01]`) so tasks trace back to specs
- Noted dependencies: which tasks require another task to be complete first
- Design-dependent tasks marked (⚑ Tami first or equivalent) to flag pre-requisites
- Configuration-sourced values marked (⚙ config) to flag no-hardcode rules
- A production launch checklist at the end covering config changes, deployment, and live testing

**Format:** flat or nested bullet points — not a table. Numbered sections are acceptable for grouping.

---

## Step 3: Wait for Instruction

After `imp_plan.md` is written, **do not begin implementation.** Report completion to the user and wait for explicit instruction to begin. Implementation order may require discussion, and UI IOCs require Tami's design specs before Chad can proceed.

---

## File Naming Summary

| Artifact | Location | Naming Pattern |
|----------|----------|----------------|
| SDD | `specs/` | `<ProjectName>_sdd.md` |
| Master spec | `specs/` | `master_spec.md` |
| Dependency chart | `specs/` | `depc.md` |
| Feature spec | `specs/<module>/` | `<module>_spec.md` |
| IOC file | `specs/<module>/` | `<ioc-name>_ioc.md` |
| Implementation plan | project root | `imp_plan.md` |
| Reference spec | `specs/reference/` | `<program>_ref_spec.md` |
| General KB | `kb/general/` | `<subject>_kb.md` |
| Project KB | `kb/<project-name>/` | `<subject>_kb.md` |

---

## Quality Checks Before Completing Initialization

- [ ] All modules from the SDD are represented in `master_spec.md`
- [ ] Every module has a spec file with a complete IOC table
- [ ] Every IOC in every spec has a corresponding `_ioc.md` file
- [ ] `depc.md` exists at `specs/depc.md` showing module and IOC dependencies
- [ ] `imp_plan.md` exists at project root, ordered most critical → least critical, referencing all IOCs
- [ ] If `refsrc/` was present, reference specs exist in both `refsrc/` and `specs/reference/`
- [ ] Usage/Patterns KB exists for each language used
- [ ] Best Practices KB exists for each language used
- [ ] Security KB exists (at minimum one cross-cutting `web_security_kb.md`)
- [ ] `kb/helpdesk.md` indexes all KB articles created during priming
- [ ] `session.md` has a Session 1 entry documenting what was done
- [ ] No implementation (code) has been written — only documentation

---

*Last updated: 2026-03-25 by Karen*
